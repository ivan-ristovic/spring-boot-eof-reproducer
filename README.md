# spring-boot-eof-reproduceer

## Issue summary

Tomcat throws `EOFException` once per every request. I believe multiple executors get dispatched to process requests, and one of them always fails silently. I am not sure if this is the issue with Tomcat or Spring.

Might relate to #38940 and #35126

## Steps to reproduce

Please use the following [repository](https://github.com/ivan-ristovic/spring-boot-eof-reproducer).
This is a slightly modified quickstart demo from [start.spring.io](https://start.spring.io/), with Spring Boot 4.1.0, using Maven and Java 25. Dependencies include Spring Web and GraalVM Native Support.

The application defines a simple _Hello world_ controller:
```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;

@RestController
@SpringBootApplication
public class DemoApplication {

	@RequestMapping("/")
	String home() {
		return "Hello World!";
	}

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}
```

1. Build the demo (a convenience `build_jvm` script is provided):
```bash
$ ./build_jvm
```
or manually:
```bash
$ ./mvnw package
```
1. Run the demo with logging enabled (a convenience `run_jvm` script is provided):
```bash
$ ./run_jvm
```
or manually:
```bash
$ ./mvnw spring-boot:run -Dspring-boot.run.jvmArguments="-Dlogging.level.root=TRACE"
```
1. Send a request (a convenience `send_request` script is provided):
```bash
$ ./send_request
```
or manually:
```bash
$ curl $@ localhost:8080
```
1. Observe the following exception trace:
```java
java.io.EOFException
	at org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper.fillReadBuffer(NioEndpoint.java:1339) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper.read(NioEndpoint.java:1223) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.coyote.http11.Http11InputBuffer.fill(Http11InputBuffer.java:776) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.coyote.http11.Http11InputBuffer.parseRequestLine(Http11InputBuffer.java:334) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:270) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:63) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:903) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1779) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:52) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:946) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:480) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:57) ~[tomcat-embed-core-11.0.18.jar:11.0.18]
	at java.base/java.lang.Thread.run(Thread.java:1474) ~[na:na]
```

I have also included convenience scripts that exercise the same behavior on GraalVM (`build_svm` and `run_svm` scripts). You will need to provide a GraalVM installation using the `GRAALVM_HOME` environment variable.

## Hardwaare details

I have been able to reproduce on both AMD and x64 architectures, on Linux systems (Debian 13 and Arch .

