# Thread management

This chapter describes how ActiveMQ uses and pools threads and how you
can manage them.

First we'll discuss how threads are managed and used on the server side,
then we'll look at the client side.

## Server-Side Thread Management

Each ActiveMQ Server maintains a single thread pool for general use, and
a scheduled thread pool for scheduled use. A Java scheduled thread pool
cannot be configured to use a standard thread pool, otherwise we could
use a single thread pool for both scheduled and non scheduled activity.

A separate thread pool is also used to service connections. ActiveMQ can
use "old" (blocking) IO or "new" (non-blocking) IO also called NIO. Both
of these options use a separate thread pool, but each of them behaves
uniquely.

Since old IO requires a thread per connection its thread pool is
unbounded. The thread pool is created via `
            java.util.concurrent.Executors.newCachedThreadPool(ThreadFactory)`.
As the JavaDoc for this method states: “Creates a thread pool that
creates new threads as needed, but will reuse previously constructed
threads when they are available, and uses the provided ThreadFactory to
create new threads when needed.” Threads from this pool which are idle
for more than 60 seconds will time out and be removed. If old IO
connections were serviced from the standard pool the pool would easily
get exhausted if too many connections were made, resulting in the server
"hanging" since it has no remaining threads to do anything else.
However, even an unbounded thread pool can run into trouble if it
becomes too large. If you require the server to handle many concurrent
connections you should use NIO, not old IO.

When using new IO (NIO), ActiveMQ will, by default, cap its thread pool
at three times the number of cores (or hyper-threads) as reported by `
            Runtime.getRuntime().availableProcessors()` for processing
incoming packets. To override this value, you can set the number of
threads by specifying the parameter `nio-remoting-threads` in the
transport configuration. See the ? for more information on this.

There are also a small number of other places where threads are used
directly, we'll discuss each in turn.

### Server Scheduled Thread Pool

The server scheduled thread pool is used for most activities on the
server side that require running periodically or with delays. It maps
internally to a `java.util.concurrent.ScheduledThreadPoolExecutor`
instance.

The maximum number of thread used by this pool is configure in
`activemq-configuration.xml` with the `scheduled-thread-pool-max-size`
parameter. The default value is `5` threads. A small number of threads
is usually sufficient for this pool.

### General Purpose Server Thread Pool

This general purpose thread pool is used for most asynchronous actions
on the server side. It maps internally to a
`java.util.concurrent.ThreadPoolExecutor` instance.

The maximum number of thread used by this pool is configure in
`activemq-configuration.xml` with the `thread-pool-max-size` parameter.

If a value of `-1` is used this signifies that the thread pool has no
upper bound and new threads will be created on demand if there are not
enough threads available to satisfy a request. If activity later
subsides then threads are timed-out and closed.

If a value of `n` where `n`is a positive integer greater than zero is
used this signifies that the thread pool is bounded. If more requests
come in and there are no free threads in the pool and the pool is full
then requests will block until a thread becomes available. It is
recommended that a bounded thread pool is used with caution since it can
lead to dead-lock situations if the upper bound is chosen to be too low.

The default value for `thread-pool-max-size` is `30`.

See the [J2SE
javadoc](http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/ThreadPoolExecutor.htm)
for more information on unbounded (cached), and bounded (fixed) thread
pools.

### Expiry Reaper Thread

A single thread is also used on the server side to scan for expired
messages in queues. We cannot use either of the thread pools for this
since this thread needs to run at its own configurable priority.

For more information on configuring the reaper, please see ?.

### Asynchronous IO

Asynchronous IO has a thread pool for receiving and dispatching events
out of the native layer. You will find it on a thread dump with the
prefix ActiveMQ-AIO-poller-pool. ActiveMQ uses one thread per opened
file on the journal (there is usually one).

There is also a single thread used to invoke writes on libaio. We do
that to avoid context switching on libaio that would cause performance
issues. You will find this thread on a thread dump with the prefix
ActiveMQ-AIO-writer-pool.

## Client-Side Thread Management

On the client side, ActiveMQ maintains a single static scheduled thread
pool and a single static general thread pool for use by all clients
using the same classloader in that JVM instance.

The static scheduled thread pool has a maximum size of `5` threads, and
the general purpose thread pool has an unbounded maximum size.

If required ActiveMQ can also be configured so that each
`ClientSessionFactory` instance does not use these static pools but
instead maintains its own scheduled and general purpose pool. Any
sessions created from that `ClientSessionFactory` will use those pools
instead.

To configure a `ClientSessionFactory` instance to use its own pools,
simply use the appropriate setter methods immediately after creation,
for example:

``` java
ServerLocator locator = ActiveMQClient.createServerLocatorWithoutHA(...)

ClientSessionFactory myFactory = locator.createClientSessionFactory();

myFactory.setUseGlobalPools(false);

myFactory.setScheduledThreadPoolMaxSize(10);

myFactory.setThreadPoolMaxSize(-1); 
```

If you're using the JMS API, you can set the same parameters on the
ClientSessionFactory and use it to create the `ConnectionFactory`
instance, for example:

``` java
ConnectionFactory myConnectionFactory = ActiveMQJMSClient.createConnectionFactory(myFactory);
```

If you're using JNDI to instantiate `ActiveMQConnectionFactory`
instances, you can also set these parameters in the JNDI context
environment, e.g. `jndi.properties`. Here's a simple example using the
"ConnectionFactory" connection factory which is available in the context
by default:

    java.naming.factory.initial=org.apache.activemq.jndi.ActiveMQInitialContextFactory
    
    java.naming.provider.url=tcp://localhost:5445
    
    connection.ConnectionFactory.useGlobalPools=false
    
    connection.ConnectionFactory.scheduledThreadPoolMaxSize=10
    
    connection.ConnectionFactory.threadPoolMaxSize=-1

