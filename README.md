# Java NIO - Reactor

## Why non-blocking IO
A typical server application, such as a web server, needs to process thousands of request concurrently. Therefore, the modern web server needs to meet the following requirements.
Handling of thousands of connections simultaneously (significant number of connections may be in idle state as well)
Handling high latency connections
- Request/response handling needs to be decoupled. 
- Minimize latency, maximize throughput and avoiding unnecessary CPU cycles. 

Hence, to cater to such requirements, there can be several possibilities in the server application architecture.
- **Having a pool of sockets for each client, and periodically polling them**: This is the most straightforward approach; however, it is more or less impractical without non-blocking sockets. This is extremely inefficient and never scales with the increasing number of connections.
- **Thread per socket**: This is the conventional approach and was initially used in some applications and this is the only practical solution with blocking sockets. Having thread per each client connection has several drawbacks, such as a large amount of overhead with the thread scheduling. Another flaw in this approach is that as the number of connections/client increases, we need to increase the number of threads as well. Therefore, this too hardly scales.
- **Readiness selection**: It is the ability to choose a socket that will not block when read or written. This is a very efficient way of handling thousands of concurrent clients and scales well.

The readiness selection approach was originally presented as an object behavioral pattern by under [‘Reactor: An Object Behavioral Pattern for Demultiplexing and Dispatching Handles for Synchronous Events’](http://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf) paper. Therefore, in order to understand the readiness selection better, we need to take a closer look at the Reactor pattern.

## Reactor Pattern

The reactor design pattern was introduced as a general architecture to implement event-driven systems. To solve our original problem of implementing a server application that can handle thousands of simultaneous client connections, Reactor pattern provides a way in which we can listen to the events (incoming connections/requests) with a synchronous demultiplexing strategy so that when an incoming event occurs, it is dispatched to a service provider (handler) that can handle this event.

Let's have a detailed look at each key participant in the Reactor pattern, which is depicted in the following class diagram. 

![Reactor Pattern](https://wso2.com/files/ESB_1.png)

In the reactor pattern, the initiation dispatcher is the most crucial component. Often this is also known as the ‘Reactor’. For each type of service that the server application offers, it introduces a separate event handler that can process that particular event type. All these event handlers are registered with the initiation dispatcher. Initiation dispatcher uses a demultiplexer that can listen to all the incoming events and notify the initiation dispatcher accordingly. The demultiplexer uses ‘handles’ to identify the events that occur on a given resource, such as network connection. Handles are often used to identify OS managed resources, such as network connections, open files, etc.
The behavior of demultiplexer is synchronous such that it blocks on waiting for events to occur on a set of handlers. However, once the event occurs, it simply notifies the initiation dispatcher and it will hand over the event to the respective concrete event handler type.

### Handle
Handle identifies resources that are managed by the operating system, such as network connections, open files, etc. Handles are used by demultiplexer to wait on the events to occur on handles.

### Demultiplexer
Demultiplexer works in synchronous mode to waiting on the events to occurs on the handlers. This is a synchronous blocking behavior, but this only blocks when we do not have events queued up at the handles. In all other cases, when there is an event for a given handle, demultiplexer notifies the initiation dispatcher to call-back the appropriate event handler.
A very common realization of a demultiplexer in Unix is the ‘select(.)’ system call, which is used to examine the status of file descriptors in Unix.

### Initiation dispatcher/Reactor
Initiation dispatcher provides an API for registering, removing, and dispatching of event handler objects. Therefore, various types of event handlers are registered with the initiation dispatcher and it also initiates the demultiplexer so that it can receive notifications when the demultiplexer detects any event.
When events such as connection acceptance, data input/output, timeout, etc. have occurred at the demultiplexer, it then notifies the initiation dispatcher. Thereafter, the initiation dispatcher invokes the respective concrete event handler.

### Event handler
This is merely an interface that represents dispatching operation for a specific event.

### Concrete event handler
This is derived from the abstract event handler and each implementation comprises a specific method of processing a specific event type. It is important to keep in mind that these concrete event handlers are often run on a dedicated thread pool, which is independent from the initiation dispatcher and the demultiplexer.


## Reactor Pattern in Java NIO

When it comes to developing server applications with Java, we need to have an underlying framework that supports a realization of the Reactor pattern. With the Java NIO framework, jdk provides the necessary building blocks to implement Reactor pattern with Java.

![Reactor Pattern Java](https://wso2.com/files/ESB_2.png)

### Non-blocking echo server with Java NIO
In order to understand how Reactor pattern can be implemented with Java NIO, let's take an example of a simple echo server and a client, in which the server is implemented based on the readiness selection strategy.

### Selector (demultiplexer)
Selector is the Java building block, which is analogous to the demultiplexer in the Reactor pattern. Selector is where you register your interest in various I/O events and the objects tell you when those events occur.

### Reactor/initiation dispatcher
We should use the Java NIO Selector in the Dispatcher/Reactor. For this, we can introduce our own Dispatcher/Reactor implementation called ‘Reactor’. The reactor comprises java.nio.channels.Selector and a map of registered handlers. As per the definition of the Dispatcher/Reactor, ‘Reactor’ will call the Selector.select() while waiting for the IO event to occur.

### Handle
In the Java NIO scope, the Handle in the Reactor pattern is realized in the form of a SelectionKey.

### Event
The events that trigger from various IO events are classified as - SlectionKey.OP_READ etc.

### Handler
A handler is often implemented as runnable or callable in Java.

### Structure of the sample scenario
In our sample scenario, we have introduced an abstract event Handler with the abstract method ‘public void handleEvent(SelectionKey handle)’ which is implemented by the respective concrete event handlers. AcceptEventHandler, ReadEventHandler, and WriteEventHandler are the concrete handler implementations.
ReactorManager - This is the place where we initialize the Reactor and execute the server side operations.
ReactorManager#startReactor
Create the ServerSocketChannel, bind with a port and configure non-blocking behavior. ServerSocketChannel server = ServerSocketChannel.open(); server.socket().bind(new InetSocketAddress(port)); server.configureBlocking(false);
Initialize Reactor and register channels Reactor initiationDispatcher = new Reactor(); initiationDispatcher.registerChannel(SelectionKey.OP_ACCEPT, server);
Register all the concrete event handlers with the Reactor.


```java 
ServerSocketChannel server = ServerSocketChannel.open();

server.socket().bind(new InetSocketAddress(port));

server.configureBlocking(false);


Reactor reactor = new Reactor();

reactor.registerChannel(SelectionKey.OP_ACCEPT, server);


reactor.registerEventHandler(

        SelectionKey.OP_ACCEPT, new AcceptEventHandler(

        reactor.getDemultiplexer()));


reactor.registerEventHandler(

        SelectionKey.OP_READ, new ReadEventHandler(

        reactor.getDemultiplexer()));


reactor.registerEventHandler(

        SelectionKey.OP_WRITE, new WriteEventHandler());


reactor.run();
``` 

Invoke run() method of the Reactor. This method is responsible for calling select() method of the Selector/Demultiplexer on an indefinite loop and as the new event occurs they are retained with selectedKeys() method of the Selector. Then for each selected key, it invokes the respective event handler.

```java 
package org.panorama.kasun;


import java.nio.channels.SelectableChannel;

import java.nio.channels.SelectionKey;

import java.nio.channels.Selector;

import java.util.Iterator;

import java.util.Map;

import java.util.Set;

import java.util.concurrent.ConcurrentHashMap;



public class Reactor {

    private Map<integer, eventhandler=""> registeredHandlers = new ConcurrentHashMap<integer, eventhandler="">();

    private Selector demultiplexer;


    public Reactor() throws Exception {

        demultiplexer = Selector.open();

    }


    public Selector getDemultiplexer() {

        return demultiplexer;

    }


    public void registerEventHandler(

            int eventType, EventHandler eventHandler) {

        registeredHandlers.put(eventType, eventHandler);

    }


    public void registerChannel(

            int eventType, SelectableChannel channel) throws Exception {

        channel.register(demultiplexer, eventType);

    }


    public void run() {

        try {

            while (true) { // Loop indefinitely

                demultiplexer.select();

      Set<selectionkey> readyHandles =

                        demultiplexer.selectedKeys();

                Iterator<selectionkey> handleIterator =

                        readyHandles.iterator();


                while (handleIterator.hasNext()) {

                    SelectionKey handle = handleIterator.next();


                    if (handle.isAcceptable()) {

                        EventHandler handler =

                                registeredHandlers.get(SelectionKey.OP_ACCEPT);

                        handler.handleEvent(handle);

                    }


                    if (handle.isReadable()) {

                        EventHandler handler =

                                registeredHandlers.get(SelectionKey.OP_READ);

                        handler.handleEvent(handle);

                        handleIterator.remove();

                    }


                    if (handle.isWritable()) {

                        EventHandler handler =

                                registeredHandlers.get(SelectionKey.OP_WRITE);

                        handler.handleEvent(handle);

                        handleIterator.remove();

                    }

                }

            }

        } catch (Exception e) {

            e.printStackTrace();

        }

    }

}
``` 

Handle is represented from SlectionKey and Event are represented with SelectionKey.OP_ACCEPT, SelectionKey.OP_READ, SelectionKey.OP_WRITE etc.


### References 
- This content is originally published in my [blog](http://kasunpanorama.blogspot.com/2015/04/understanding-reactor-pattern-with-java.html) back in April 2015. 
- [Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf) by Doug Lea 