---
layout: post
title: "Roll-call - A microservices pattern to find service instances and their metadata"
date: 2016-12-04 16:33:00 +0530
tags: [microservices, architecture]
---
I am working on a product whose back-end is implemented as collection of micro services. We had a requirement to implement an API that should respond information about running instances of services and their metadata. As the services are independent of each other and already communicating through message queues, we decided to achieve this through message passing. The following section explains how we implemented it and the challenges faced.

One of the back-end service(Service-A) is exposed to internet through HTTP APIs. The front-end communicates with back-end through this service. Initially we thought of making this service aware of all the running instances of the services through service registry like mechanism. But we felt we are complicating Service-A implementation and adding unnecessary coupling with other services. In this approach, every time we add new service we need to make changes in Service-A to make it aware of the new service instances. So we dropped this plan.

![without Roll-call](/assets/roll-call-microservices-architecture-without-roll-call.png)

One of the back-end service(Service-A) is exposed to internet through HTTP APIs. The front-end communicates with back-end through this service. Initially we thought of making this service aware of all the running instances of the services through service registry like mechanism. But we felt we are complicating Service-A implementation and adding unnecessary coupling with other services. In this approach, every time we add new service we need to make changes in Service-A to make it aware of the new service instances. So we dropped this plan.

The brainstorming continued, one idea lead to another idea and suddenly we remembered how roll-calls were happened in our schools/colleges. The teacher will announce he/she is going to do roll-call then each student will announce their presence by saying their name/roll-no. While the students were announcing their presence, the teacher notes down the present and absent students name/roll-no.

We realized a similar approach can be used to implement the requirement. We made the Service-A to broadcast a "roll-call" message to all running service instances. Upon receiving this message, the service instances will respond an "alive" message with metadata like name, ip, instance-id, etc. After broadcasting the "roll-call" message, Service-A will wait for fixed amount time and collect all "alive" messages during this period. This process is triggered by an API call and the metadata about service instances are returned as response.

![with Roll-call](/assets/roll-call-microservices-architecture-with-roll-call.png)

The advantage of this approach is we don't need to make Service-A aware of all running service instances. We can add new services or their instances at anytime and the same API will be able to fetch information about newly added service/instance.

There was one caveat we encountered during the implementation. As the API taking few seconds(i.e. fixed wait time) to respond, if the same API is called again during this period, we might miss or get duplicate metadata in any of the API calls. We solved this problem by creating request specific temporary queues and send the queue name along with the "roll-call" message. The services are made to send their "alive" to the queue specified in the "roll-call" message. As we are using RabbitMQ for our message based communication, creating disposable, temporary queue is quite easy. The problem can also be solved by including "request-id" in the "roll-call" message and collecting the "alive" messages corresponding to specific "request-id".

Please let me know if you find this pattern useful. I love to get your feedback and improve it if you find any shortfalls.