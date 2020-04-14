---
layout: post
title: "Handling pre-shutdown notification in C# Windows service"
date: 2012-03-04 22:27:00 +0530
tags: [dotnet, win32]
---
In this post I am going to explain how pre-shutdown notification can be enabled and handled in C# Windows service.

## The need for pre-shutdown notification
Starting from Windows Vista on wards Windows provides an mechanism to notify applications/services about the system shutdown/reboot. Though shutdown notification was already supported through `ServiceBase` class's `CanShutdown` property and `OnShutdown` method their functionality is very limited.

The limitation of `OnShutdown` method is, limited time given for the cleanup work and there is no guarantee that other essential services(e.g. DB, EventLog) are running. The last point is the fundamental reason why Microsoft brought this pre-shutdown notification.

## Enabling pre-shutdown notification
If we are developing a Win32 application it is really easy to enable pre-shutdown notification. All we need to do is include `SERVICE_ACCEPT_PRESHUTDOWN` flag in `dwControlsAccepted` field of service's `SERVICE_STATUS` structure.

But in .NET's ServiceBase class there is no direct way to enable pre-shutdown notification. We can achieve the goal by accessing the ServiceBase class's internals through .NET reflection.

```c#
const int SERVICE_ACCEPT_PRESHUTDOWN = 0x100;
const int SERVICE_CONTROL_PRESHUTDOWN = 0xf;

MyService
{
  ...

  FieldInfo acceptedCommandsFieldInfo =
      typeof(ServiceBase).GetField("acceptedCommands", BindingFlags.Instance | BindingFlags.NonPublic);
  if (acceptedCommandsFieldInfo == null)
    throw ApplicationException("acceptedCommands field not found");
    
  int value = (int)acceptedCommandsFieldInfo.GetValue(this);
  acceptedCommandsFieldInfo.SetValue(this, value | SERVICE_ACCEPT_PRESHUTDOWN);
}
```
In the constructor of the service `MyService` we add the `SERVICE_ACCEPT_PRESHUTDOWN` flag to `acceptedCommands` private field of ServiceBase using reflection. This enables the pre-shutdown notification.

## Handling pre-shutdown notification

The pre-shutdown notification is delivered through the ServiceBase class's `OnCustomCommand` method. This method is called whenever a command is sent to the service which is not supported by ServiceBase. The method provides a parameter `command` which holds the command code. So to handle the pre-shutdown notification we need to check whether the `command` matches with pre-shutdown notification code.
```c#
protected override void OnCustomCommand(int command)
{
    if (command == SERVICE_CONTROL_PRESHUTDOWN)
    {
      // do the clean-up here
    }
    else
        base.OnCustomCommand(command);
}
```
So that is all we need to enable and handle pre-shutdown notification.