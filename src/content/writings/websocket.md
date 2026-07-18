
---
title: "WebSocket in Spring Boot, Explained Simply"
date: 2026-07-03
description: "Kicking off the writing feed."
tags: ["websocket", "springboot", "backend"]
---


### What Problem Does WebSocket Solve?

My always go-to learning technique is asking "what problem does this solve?"

I think this is the best method to learn any new topic  when we learn this way, we eventually get the curiosity to dig deeper into that topic and explore more.

So, what problem does WebSocket solve?

In a traditional website, if a client wants to get information from the server, the client sends an HTTP request asking for that specific thing. Upon receiving the request, the server processes it (this may include querying the database, performing business logic, etc.) and sends back an HTTP response containing the requested data along with an appropriate status code.

Here's the catch HTTP is stateless, meaning the server doesn't remember the previous request. So every single request must carry all the information the server needs to process it.

### The Chat App Example

Imagine there are two users chatting with each other. Client 1 sends a message, and the server stores it. But the problem here is Client 2 currently cannot see the message. Instead, Client 2 has to keep sending requests at fixed intervals, asking "Do I have any new messages?" This technique of repeatedly sending requests to check for new data is called **Polling**.

Now imagine there are thousands of users online, all continuously sending requests just to check if there's a new message  even when there isn't one. This puts unnecessary load on the server.

This is where **WebSockets** provide a much better solution.

### What Is a WebSocket?

WebSockets are a different protocol that starts with an **HTTP** request. Once the connection is established, it remains open until either the client or the server closes it. Since the connection stays open, there's no need to create a new connection for every request.

Think of it like having a door between two houses. Once the door is unlocked and opened, both people can walk through it whenever they want, without having to knock every single time.

I won't go too deep into WebSockets because that's not the goal of this article  we're here to build one with Spring Boot. If you're curious how WebSockets work behind the scenes, I'd recommend giving the **WebSocket Handbook** by Ably a read:

[https://pages.ably.com/hubfs/the-websocket-handbook.pdf](https://pages.ably.com/hubfs/the-websocket-handbook.pdf)

With that out of the way, let's jump into the implementation.

---

## Setting Up WebSocket in Spring Boot

### Add the Dependency

Tbh, this is one of the best parts about using Spring Boot  it handles most of the heavy lifting for us, so we don't have to implement the core WebSocket functionality from scratch.

```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-websocket</artifactId>  
</dependency>
```

If you created your project using Spring Initializr, you can simply select the WebSocket dependency while creating the project. If you already have an existing project, just add this to your `pom.xml` and run `mvn clean install` (or let your IDE reload the Maven project).

### What We're Building

For this tutorial, let's build a simple dashboard where multiple users can post something. The goal: once **User 1** posts something, it should automatically reflect in **User 2's** browser without refreshing the page.

To make this work, we obviously need to set up WebSocket communication in our Spring Boot app. And that starts with configuration.

### Why We Need a WebSocketConfig Class

By default, Spring Boot automatically configures REST APIs. That's why we can simply declare REST endpoints using annotations like `@RequestMapping()` or `@GetMapping()` without any additional setup when the application starts, Spring Boot starts the embedded Tomcat server and configures HTTP request handling automatically.

However, Spring does **not** know how we want to configure WebSockets. We need to specify things like the WebSocket endpoint, the message broker, and the destination prefixes ourselves. That's why we create a separate `WebSocketConfig` class. When the application starts, Spring scans this configuration class, recognizes that the application uses WebSockets, and sets up the WebSocket infrastructure based on the settings we provide.

Here's what that class looks like:

```java

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("http://localhost:5173")
                .withSockJS(); 
            // .withSockJS() is for compatibility , if the browser doesn't support web socket,
               // then roll back to default polling or streaming method .
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
        registry.setApplicationDestinationPrefixes("/app");
    }
}
```

---

### What is registerStompEndpoints()?

This method is used to define the endpoint for WebSocket communication. But you might wonder "why do we need a separate endpoint for WebSockets?", Why can't we just use `@RequestMapping` like we do for REST APIs?

The reason is simple. `@RequestMapping` is designed only for HTTP which is a request-response protocol. The client asks, the server answers, and the connection closes. That's it. It's one-directional per request.

WebSockets are different. Once the connection is established, both the client and the server can send messages to each other at any time without waiting for a request. That's bi-directional communication and HTTP simply wasn't designed for that. So we need a separate endpoint that upgrades the HTTP connection into a WebSocket connection.

You might also wonder about the `registry` argument this method receives — it's of type `StompEndpointRegistry`. Here's what's happening behind the scenes: when Spring starts up, it sees the `@EnableWebSocketMessageBroker` annotation, builds the entire WebSocket infrastructure internally, and then passes that `registry` object into your method. Think of it like Spring handing you a blank blueprint and saying _"here, fill in the details."_ You then use methods like `addEndpoint()` to define where clients can connect — building the configuration piece by piece. That's exactly what `registerStompEndpoints()` does it tells Spring "this is the URL where clients can connect to open a WebSocket connection."

---

### **What is `configureMessageBroker()`?**

Now that we've defined where clients connect, we need to tell Spring how to route messages once they're inside. That's exactly what `configureMessageBroker()` does.

Think of it like a post office. When a message arrives, the post office needs rules  which messages go where, and how to deliver them. `configureMessageBroker()` defines those rules.

Similar to `registerStompEndpoints()`, Spring builds the `MessageBrokerRegistry` internally and hands it to you same idea, you just fill in the details.

Now let's look at the two lines inside:

```java
registry.enableSimpleBroker("/topic");
```

This enables Spring's built-in message broker and tells it to handle any destination that starts with `/topic`. Think of `/topic/activities` like a YouTube channel  any client can subscribe to it, and whenever a message is sent to that channel, all subscribers receive it instantly.

```java
registry.setApplicationDestinationPrefixes("/app");
```

This defines the prefix for messages that are sent FROM the client TO the server. So if a client sends a message to `/app/activity`, Spring knows to route it to a controller method annotated with `@MessageMapping("/activity")`. Without this prefix, Spring wouldn't know whether an incoming message is meant for the broker or for your controller.

In short `/topic` is for server-to-client delivery, `/app` is for client-to-server routing.

---

### **What is STOMP?**

We already know what WebSockets do they maintain a persistent connection between the client and the server, allowing both sides to send data to each other at any time.

But raw WebSockets are just an open pipe. You can put anything in that pipe there's no structure, no rules, no way to say "this message goes to this person" or "this is a chat message vs a notification." It's just raw data flying back and forth.

That's where **STOMP** (Simple Text Oriented Messaging Protocol) comes in. STOMP works on top of WebSockets and adds structure to the flow of data. It introduces three simple concepts:

- **Destinations** — like addresses, e.g. `/topic/activities`
- **Subscribe** — a client says "I want messages sent to this destination"
- **Send** — the server or client pushes a message to a destination

Think of it like this WebSocket is the road, STOMP is the traffic rules. Without traffic rules, cars go everywhere and crash into each other. With rules, everyone knows where to go.

In Spring Boot, when you use `@EnableWebSocketMessageBroker`, you're enabling STOMP support on top of WebSockets. That's why your endpoints and broker configuration use STOMP concepts like destinations and prefixes.

---

### **Broadcasting from the Server `SimpMessagingTemplate`**

Okay so we've set up WebSockets, configured STOMP, defined our endpoints and broker. But here's the question how does our Spring controller actually _push_ a message to all the connected clients? Like who does the actual sending?

That's where `SimpMessagingTemplate` comes in.

Spring gives you this object out of the box you don't need to create it, you don't need to configure it. Just autowire it like any other bean and it's ready to use.

```java
@Autowired
private SimpMessagingTemplate smt;
```

Now whenever you want to push something to all subscribers, you just call one method:

```java
smt.convertAndSend("/topic/activities", savedActivity);
```

That's it. One line. Here's what it does under the hood:

- Takes your Java object (`savedActivity`)
- Converts it to JSON automatically
- Pushes it to everyone currently subscribed to `/topic/activities`

Now let's see it in context. Here's what the full controller method looks like:

```java
@PostMapping
public Activity insertActivity(@RequestBody Activity activity) {
    Activity savedActivity = activityRepository.save(activity);
    smt.convertAndSend("/topic/activities", savedActivity);
    return savedActivity;
}
```

Two things are happening here. First, the activity gets saved to the database. Second, that same activity gets broadcast to all connected clients instantly over WebSocket. The person who made the POST request gets a normal HTTP response back. Everyone else watching the dashboard gets it pushed to them in real time.

### **Testing It The HTML Client**

So after configuring everything, let's look at how we can test it. Just a heads up I'm not really focused on JS or frontend right now, so I won't go deep into how this code works internally. I know it at a high level, and that's good enough for this test.

```html
<!DOCTYPE html>
<html>
<head>
<title>WebSocket Test</title>
<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
</head>
<body>
<h2>WebSocket Test</h2>
<div id="messages"></div>
<script>
var socket = new SockJS('http://localhost:8080/ws');
var stompClient = Stomp.over(socket);
stompClient.connect({}, function(frame) {
    console.log('Connected: ' + frame);
    stompClient.subscribe('/topic/activities', function(message) {
        document.getElementById('messages').innerHTML += '<p>' + message.body + '</p>';
    });
});
</script>
</body>
</html>
```

This is just a small HTML page that connects to our `/ws` endpoint, subscribes to `/topic/activities`, and displays whatever comes through. That's it nothing fancy, just enough to prove the WebSocket connection actually works end to end.

#### **The test:**

1. Open this HTML file in the browser.
2. Hit the `POST /activity` endpoint with some activity data.
3. Watch the page  the new activity shows up instantly, no refresh needed.

That confirms the full flow save to DB, broadcast over WebSocket, client receives it live.

---

### **Conclusion**

And that's it that's a full WebSocket setup with Spring Boot, from the problem it solves all the way to seeing it work live in the browser.

To recap what we built: we configured a WebSocket endpoint, set up STOMP for structured messaging, used `SimpMessagingTemplate` to broadcast from the server, and tested it with a simple HTML client all to get one thing working: **when User 1 posts something, User 2 sees it instantly, without refreshing.**

This is the exact same foundation used for things like chat apps, live notifications, live dashboards, and real-time tracking apps so once you're comfortable with this, you can build a lot on top of it.