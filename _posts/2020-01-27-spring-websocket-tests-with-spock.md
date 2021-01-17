---
layout: post
title: Painless WebSocket tests with Spock
---

Going beyond the usual REST endpoints to also accept WebSocket connections is a fairly common expectation for your Spring web application in 2020. So is giving up on plain JUnit to leverage the beauty of [Spock](http://spockframework.org/spock/docs/1.3/index.html). If not done right, this mix will backfire at you with tests that are flaky and hard to debug - the usual price for going asynchronous.

Read on to see how easy and gratifying your WebSocket coverage can be once you learn how to target the usual suspects like race conditions and false positives. Make sure to help yourself to the source code [available on GitHub](https://github.com/jjarzynski/spring-ws-spock-tests).

-----

## The Echo

Setting up Spring to handle incoming connections requires a handler bean and a configuration to associate it with an address.

The easiest way to go about a handler is extending one of the helper classes like `TextWebSocketHandler`. For the sake of argument, we will implement [RFC 862](https://tools.ietf.org/html/rfc862): _the Echo Protocol_. One small caveat is we will ignore empty messages to make the service a little more interesting to test.

This is what such handler can look like:

```java
public class EchoSocket extends TextWebSocketHandler {

    @Override
    public void handleTextMessage(WebSocketSession session,
                                  TextMessage message) {

        if (message.getPayload().isEmpty()) return;

        session.sendMessage(message);
    }
}
```

Then a `WebSocketConfigurer` implementation is necessary to register it under specified address. The above handler gets declared as a bean and attached to `/echo`:

```java
@Configuration
@EnableWebSocket
public class SocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(
            WebSocketHandlerRegistry registry) {
        registry.addHandler(echo(), "/echo");
    }

    @Bean
    public EchoSocket echo() {
        return new EchoSocket();
    }
}
```

With these two classes in place the app should accept WebSocket sessions at `localhost:8080/echo` and always reply with received text. Next thing to do is start adding tests to see if it works.

## First contact

Spock does not include an easy way to open WebSocket connections so we need a client library for that:

```xml
<dependency>
    <groupId>com.neovisionaries</groupId>
    <artifactId>nv-websocket-client</artifactId>
    <version>2.8</version>
</dependency>
```

The first thing to validate is that a connection gets accepted. If we let Spring pick a random port to start on, we need to know the value so we inject it with `@LocalServerPort`.

Another thing to bear in mind is forcing immediate disconnect of the socket we used in each of the tests. Otherwise as the number and complexity of tests grow, so will the number of unclosed sockets interfering in unexpected ways:

```java
@SpringBootTest(
        classes = SpringWsApplication.class, 
        webEnvironment = RANDOM_PORT)
class EchoSocketTest extends Specification {

    @LocalServerPort int port

    def "socket opens"() {
        when:
        def socket = new WebSocketFactory()
                .createSocket("http://localhost:${port}/echo")
                .connect()

        then:
        socket.isOpen()

        cleanup:
        socket.disconnect(WebSocketCloseCode.NORMAL, null, 0)
    }
}
```

So far so good, the test goes through so we know we can open and close the socket.

Things become less obvious when we start sending and receiving messages. WebSocket communication, unlike plain HTTP calls, is not conducted in a request-response fashion. When a text message gets sent to a socket, synchronously waiting for a response is not the best idea.

## Here come the Mocks

Before we get to this, let's pull out two methods so they can be reused and taken for granted:

```java
def openSocket() {
    new WebSocketFactory()
            .createSocket("http://localhost:${port}/echo")
            .connect()
}

static def closeSocket(socket) {
    socket.disconnect(WebSocketCloseCode.NORMAL, null, 0)
}
```

Now looking at the `WebSocket` class from the client library submitting a text message seems easy (`sendText(String message)`) but we need to figure out a way to receive responses. There is no `receive()` method so what we need to do instead is register a `WebSocketListener`. And how do we validate that the listener was called? Spock documentation has a whole section on [Interaction Based Testing](http://spockframework.org/spock/docs/1.3/interaction_based_testing.html) and what they mean by that using mocks:

```java
def "responds with original message"() {
    given:
    def socket = openSocket()

    and:
    def listenerMock = Mock(WebSocketListener)
    socket.addListener(listenerMock)

    when:
    socket.sendText("Hello")

    then:
    1 * listenerMock.onTextMessage(_, "Hello")

    cleanup:
    closeSocket(socket)
}
```

So we create a mock, add it as a socket listener, submit a text messages and expect the mock to be called exactly once because this is how the Echo Protocol works. We then try to run the test and surprisingly it fails:

```
Too few invocations for:

1 * listenerMock.onTextMessage(_, "Hello")   (0 invocations)

Unmatched invocations (ordered by similarity):

None
```

What could have possibly gone wrong? We are sending a text message and waiting for a response - are we really though? Immediately after `socket.sendText()` the test goes to the next line to execute the assertion. It expects the mock to have already been called exactly once rather than wait for `EchoSocket` to receive the message and respond. Great: we got ourselves a race condition. If you add a short `sleep()` right before the assertion, it is likely to pass. But then you are doing two things wrong:

1. You are making the test run longer than necessary. E.g. 100 milliseconds is most likely more than it needs.
2. You cannot be sure that 100 milliseconds will always be enough. Every 10th or 1000th execution can still fail and you and your team need to acknowledge that this is just the way this test is. Or all your WebSocket tests for that matter - if you follow down this path.

## Don't wait!

Which proves a need for a synchronisation mechanism to coordinate between the thread that executes the test and the one that calls our `WebSocketListener` after the response arrives. Spock provides a construct called `BlockingVariable`, which is not really explained in their [documentation](http://spockframework.org/spock/docs/1.3/all_in_one.html), maybe because it is pretty straight-forward.

The simplest way to describe it is a wrapper around a value of a given type. The value is set like with any other setter but when  `get()` is called, it is either going to return the value immediately if it has already been set or wait for a given time for it to be set and then fail.

The way you want to use it is:
1. Instantiate it with the expected type and maximum time to wait for `get()` to return.
2. Implement a `WebSocketListener` that forwards any received text to the `BlockingVariable`.
3. Try to get the value in the assertions block. Once it arrives, you can also assert what the text is and if it never arrives the call to `get()` has all the right to fail the test.

```java
def "responds with original message"() {
    given:
    def latch = new BlockingVariable<String>(0.5)

    and:
    def socket = openSocket()
    socket.addListener(new WebSocketAdapter() {
        @Override
        void onTextMessage(WebSocket websocket, 
                           String text) {
            latch.set(text)
        }
    })

    when:
    socket.sendText("Hello")

    then:
    with(latch.get()) {
        it == "Hello"
    }

    cleanup:
    closeSocket(socket)
}
```

This should do it - no race conditions anymore. The test is forced to wait for the thread that receives the message to respond. It also takes exactly as much time as it needs to and not any more than that. The moment the value is set, method `get()` returns and all remaining assertions get executed.

As your test suite grows, it is also useful to replace the above `and:` block with a utility method that opens a socket and forwards to latch all in one go. You don't want boilerplate code like this distracting you from the logic under test:

```java
def openSocket(latch) {
    openSocket().addListener(new WebSocketAdapter() {
        @Override
        void onTextMessage(WebSocket websocket, 
                           String text) {
            latch.set(text)
        }
    })
}
```

So there you have it! Once again, feel free to use the source code [available on GitHub](https://github.com/jjarzynski/spring-ws-spock-tests) and happy testing!
