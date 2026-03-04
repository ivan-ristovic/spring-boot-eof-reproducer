# Issue description

`sun.nio.ch.EPoll#wait` behaves differently on JVM and SVM.
On JVM, it returnes non-zero once per request

# Reproduce steps

Find: `org.apache.tomcat.util.net.NioEndpoint.class#run`
The method performs `Selector.select` in a loop and processes ready keys
Place a breakpoint at line `802`:
```java
Iterator<SelectionKey> iterator = this.keyCount > 0 ? this.selector.selectedKeys().iterator() : null;
```
The breakpoint will be hit approximately once per second (as that is the select timeout)
You can use a conditional breakpoint `this.keyCount > 0`
Send a request

## JVM

Observe `keyCount == 1`, the key is processed regularly and the response is returned.
In the next iteration, `keyCount == 0`

## SVM

Debug with gdb, notable breakpoint:
```bash
b org.apache.tomcat.util.net.NioEndpoint$Poller::run`
```

Observe same behavior as JVM, `keyCount == 1`, the key is processed regularly.
However, in the next iteration, `Selector.select()` returns `1` again so `keyCount == 1`
The loop after line `802` attempts to process the key again, and dispatches a worker
The worker throws an exception after attempting to read the socket, and the exception gets discarded silently.

Exception thrown at:
```bash
#0  org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper::fillReadBuffer(NioEndpoint.java:1339)
    (this=org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper = {...}, block=false, buffer=java.nio.HeapByteBuffer = {...}) at org/apache/tomcat/util/net/NioEndpoint.java:1339
#1  0x0000555556764326 in org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper::read(NioEndpoint.java:1223)
    (this=org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper = {...}, block=false, to=java.nio.HeapByteBuffer = {...}) at org/apache/tomcat/util/net/NioEndpoint.java:1223
#2  0x00005555566203de in org.apache.coyote.http11.Http11InputBuffer::fill(Http11InputBuffer.java:776)
    (this=org.apache.coyote.http11.Http11InputBuffer = {...}, block=false)
    at org/apache/coyote/http11/Http11InputBuffer.java:776
#3  0x00005555566215c6 in org.apache.coyote.http11.Http11InputBuffer::parseRequestLine(Http11InputBuffer.java:334)
    (this=org.apache.coyote.http11.Http11InputBuffer = {...}, keptAlive=false, connectionTimeout=60000, keepAliveTimeout=60000) at org/apache/coyote/http11/Http11InputBuffer.java:334
#4  0x000055555662c651 in org.apache.coyote.http11.Http11Processor::service(Http11Processor.java:270)
    (this=org.apache.coyote.http11.Http11Processor = {...}, socketWrapper=org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper = {...}) at org/apache/coyote/http11/Http11Processor.java:270
```

