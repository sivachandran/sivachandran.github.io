---
layout: post
title: "Logging overhead in hot code path"
featured: true
author: sivachandran
comments: true
tags: [ go, optimization ]
---

This post will describe how logging in a hot code path can induce significant overhead to the implementation. Though the overhead is described in the context Go program, the logging overhead should be similar in other languages as well.

I was trying to improve a particular API throughput of a web service. I was using Gatling script to do the load testing of the API. When I started, the API was hardly touching 200 Requests-Per-Second(RPS). 

I couldn't see anything glaringly wrong with the implementation. So, I started with an empty implementation(returning just 200 OK) of the API and added the actual implementation part by part. I observed the throughput reduced significantly when a particular part of the implementation was added. This particular part of the implementation logs a portion of the request to the console/stdout.

Though I am aware of the overhead induced by logging, I didn't expect the overhead to be this severe. All my previous observations of logging overhead were with real-time systems like audio-video playback engine. So, it was a surprise for me to know the overhead has a significant impact, even on non-real-time systems like web service.

To understand the overhead of logging in a hot code path, I came up with the following Go program that measures various logging mechanisms overhead.

```go
package main

import (
	"bufio"
	"bytes"
	"fmt"
	"os"
	"time"
)

const (
	N = 1_00_000
)

func sumOfN(n int, logFunc func(sum int)) int {
	var sum int
	for i := 1; i <= n; i++ {
		sum += i

		if logFunc != nil {
			logFunc(sum)
		}
	}

	return sum
}

func main() {
	summaryBuf := &bytes.Buffer{}
	summary := bufio.NewWriter(summaryBuf)

	fmt.Fprintf(summary, "CPU time taken to compute sum of 1..%d:-\n", N)

	startTime := time.Now()
	_ = sumOfN(N, nil)
	fmt.Fprintf(summary, "With nil log func:\t\t %f secs\n", time.Since(startTime).Seconds())

	startTime = time.Now()
	_ = sumOfN(N, func(sum int) {})
	fmt.Fprintf(summary, "With nop log func:\t\t %f secs\n", time.Since(startTime).Seconds())

	startTime = time.Now()
	_ = sumOfN(N, func(sum int) { fmt.Println(sum) })
	fmt.Fprintf(summary, "With stdout log func:\t\t %f secs\n", time.Since(startTime).Seconds())

	startTime = time.Now()
	tmpFile, _ := os.CreateTemp("", "*")
	defer tmpFile.Close()
	_ = sumOfN(N, func(sum int) { fmt.Fprintln(tmpFile, sum) })
	fmt.Fprintf(summary, "With file log func:\t\t %f secs\n", time.Since(startTime).Seconds())

	startTime = time.Now()
	tmpFile, _ = os.CreateTemp("", "*")
	bufTmpFile := bufio.NewWriter(tmpFile)
	defer tmpFile.Close()
	_ = sumOfN(N, func(sum int) { fmt.Fprintln(bufTmpFile, sum) })
	bufTmpFile.Flush()
	fmt.Fprintf(summary, "With buffered file log func:\t %f secs\n", time.Since(startTime).Seconds())

	summary.Flush()
	fmt.Println(summaryBuf.String())
}
```

**Output**
```
CPU time taken to compute sum of 1..100000:-
With nil log func:               0.000041 secs
With nop log func:               0.000194 secs
With stdout log func:            0.937555 secs
With file log func:              0.296046 secs
With buffered file log func:     0.011329 secs
```

As we see from the program output, stdout based logging makes the hot code path(sum of 1..N) ~5000 times slower. The overhead reduces if we use file(~1500x slower) or buffered file(~57x slower) based logging.

Logs are added to gain insight into execution flow. So, we can't eliminate logs from the hot code path. But we could use a more efficient logging mechanism like memory buffer, IPC(Win32 OutputDebugString) for hot code paths. At the minimum, we should mark hot code path logs as debug logs and disable them in load testing and production deployment.

I have made all hot code path logs into debug logs in the web service API that I was investigating. With the debug logs disabled, the particular API could reach 1000 RPS(5x improvement).
