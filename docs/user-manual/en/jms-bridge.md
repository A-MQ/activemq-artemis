# The JMS Bridge

ActiveMQ includes a fully functional JMS message bridge.

The function of the bridge is to consume messages from a source queue or
topic, and send them to a target queue or topic, typically on a
different server.

The source and target servers do not have to be in the same cluster
which makes bridging suitable for reliably sending messages from one
cluster to another, for instance across a WAN, and where the connection
may be unreliable.

A bridge can be deployed as a standalone application, with ActiveMQ
standalone server or inside a JBoss AS instance. The source and the
target can be located in the same virtual machine or another one.

The bridge can also be used to bridge messages from other non ActiveMQ
JMS servers, as long as they are JMS 1.1 compliant.

> **Note**
>
> Do not confuse a JMS bridge with a core bridge. A JMS bridge can be
> used to bridge any two JMS 1.1 compliant JMS providers and uses the
> JMS API. A core bridge (described in [Core Bidges](core-bridges.md)) is used to bridge any two
> ActiveMQ instances and uses the core API. Always use a core bridge if
> you can in preference to a JMS bridge. The core bridge will typically
> provide better performance than a JMS bridge. Also the core bridge can
> provide *once and only once* delivery guarantees without using XA.

The bridge has built-in resilience to failure so if the source or target
server connection is lost, e.g. due to network failure, the bridge will
retry connecting to the source and/or target until they come back
online. When it comes back online it will resume operation as normal.

The bridge can be configured with an optional JMS selector, so it will
only consume messages matching that JMS selector

It can be configured to consume from a queue or a topic. When it
consumes from a topic it can be configured to consume using a non
durable or durable subscription

Typically, the bridge is deployed by the JBoss Micro Container via a
beans configuration file. This would typically be deployed inside the
JBoss Application Server and the following example shows an example of a
beans file that bridges 2 destinations which are actually on the same
server.

    <?xml version="1.0" encoding="UTF-8"?>
    <deployment xmlns="urn:jboss:bean-deployer:2.0">
       <bean name="JMSBridge" class="org.apache.activemq.api.jms.bridge.impl.JMSBridgeImpl">
          <!-- ActiveMQ must be started before the bridge -->
          <depends>ActiveMQServer</depends>
          <constructor>
             <!-- Source ConnectionFactory Factory -->
             <parameter>
                <inject bean="SourceCFF"/>
             </parameter>
             <!-- Target ConnectionFactory Factory -->
             <parameter>
                <inject bean="TargetCFF"/>
             </parameter>
             <!-- Source DestinationFactory -->
             <parameter>
                <inject bean="SourceDestinationFactory"/>
             </parameter>
             <!-- Target DestinationFactory -->
             <parameter>
                <inject bean="TargetDestinationFactory"/>
             </parameter>
             <!-- Source User Name (no username here) -->
             <parameter><null /></parameter>
             <!-- Source Password (no password here)-->
             <parameter><null /></parameter>
             <!-- Target User Name (no username here)-->
             <parameter><null /></parameter>
             <!-- Target Password (no password here)-->
             <parameter><null /></parameter>
             <!-- Selector -->
             <parameter><null /></parameter>
             <!-- Failure Retry Interval (in ms) -->
             <parameter>5000</parameter>
             <!-- Max Retries -->
             <parameter>10</parameter>
             <!-- Quality Of Service -->
             <parameter>ONCE_AND_ONLY_ONCE</parameter>
             <!-- Max Batch Size -->
             <parameter>1</parameter>
             <!-- Max Batch Time (-1 means infinite) -->
             <parameter>-1</parameter>
             <!-- Subscription name (no subscription name here)-->
             <parameter><null /></parameter>
             <!-- Client ID  (no client ID here)-->
             <parameter><null /></parameter>
             <!-- Add MessageID In Header -->
             <parameter>true</parameter>
             <!-- register the JMS Bridge in the AS MBeanServer -->
             <parameter>
                <inject bean="MBeanServer"/>
             </parameter>
             <parameter>org.apache.activemq:service=JMSBridge</parameter>
          </constructor>
          <property name="transactionManager">
             <inject bean="RealTransactionManager"/>
          </property>
       </bean>

       <!-- SourceCFF describes the ConnectionFactory used to connect to the source destination -->
       <bean name="SourceCFF"
           class="org.apache.activemq.api.jms.bridge.impl.JNDIConnectionFactoryFactory">
          <constructor>
             <parameter>
                <inject bean="JNDI" />
             </parameter>
             <parameter>/ConnectionFactory</parameter>
          </constructor>
       </bean>

       <!-- TargetCFF describes the ConnectionFactory used to connect to the target destination -->
       <bean name="TargetCFF"
           class="org.apache.activemq.api.jms.bridge.impl.JNDIConnectionFactoryFactory">
          <constructor>
             <parameter>
                <inject bean="JNDI" />
             </parameter>
             <parameter>/ConnectionFactory</parameter>
          </constructor>
       </bean>

       <!-- SourceDestinationFactory describes the Destination used as the source -->
       <bean name="SourceDestinationFactory" class="org.apache.activemq.api.jms.bridge.impl.JNDIDestinationFactory">
          <constructor>
             <parameter>
                <inject bean="JNDI" />
             </parameter>
             <parameter>/queue/source</parameter>
          </constructor>
       </bean>

       <!-- TargetDestinationFactory describes the Destination used as the target -->
       <bean name="TargetDestinationFactory" class="org.apache.activemq.api.jms.bridge.impl.JNDIDestinationFactory">
          <constructor>
             <parameter>
                <inject bean="JNDI" />
             </parameter>
             <parameter>/queue/target</parameter>
          </constructor>
       </bean>

       <!-- JNDI is a Hashtable containing the JNDI properties required -->
       <!-- to connect to the sources and targets JMS resrouces         -->
       <bean name="JNDI" class="java.util.Hashtable">
          <constructor class="java.util.Map">
             <map class="java.util.Hashtable" keyClass="String"
                                              valueClass="String">
                <entry>
                   <key>java.naming.factory.initial</key>
                   <value>org.jnp.interfaces.NamingContextFactory</value>
                </entry>
                <entry>
                   <key>java.naming.provider.url</key>
                   <value>jnp://localhost:1099</value>
                </entry>
                <entry>
                   <key>java.naming.factory.url.pkgs</key>
                   <value>org.jboss.naming:org.jnp.interfaces"</value>
                </entry>
                <entry>
                   <key>jnp.timeout</key>
                   <value>5000</value>
                </entry>
                <entry>
                   <key>jnp.sotimeout</key>
                   <value>5000</value>
                </entry>
             </map>
          </constructor>
       </bean>

       <bean name="MBeanServer" class="javax.management.MBeanServer">
          <constructor factoryClass="org.jboss.mx.util.MBeanServerLocator" factoryMethod="locateJBoss"/>
       </bean>
    </deployment>

## JMS Bridge Parameters

The main bean deployed is the `JMSBridge` bean. The bean is configurable
by the parameters passed to its constructor.

> **Note**
>
> To let a parameter be unspecified (for example, if the authentication
> is anonymous or no message selector is provided), use `<null
>                         />` for the unspecified parameter value.

-   Source Connection Factory Factory

    This injects the `SourceCFF` bean (also defined in the beans file).
    This bean is used to create the *source* `ConnectionFactory`

-   Target Connection Factory Factory

    This injects the `TargetCFF` bean (also defined in the beans file).
    This bean is used to create the *target* `ConnectionFactory`

-   Source Destination Factory Factory

    This injects the `SourceDestinationFactory` bean (also defined in
    the beans file). This bean is used to create the *source*
    `Destination`

-   Target Destination Factory Factory

    This injects the `TargetDestinationFactory` bean (also defined in
    the beans file). This bean is used to create the *target*
    `Destination`

-   Source User Name

    this parameter is the username for creating the *source* connection

-   Source Password

    this parameter is the parameter for creating the *source* connection

-   Target User Name

    this parameter is the username for creating the *target* connection

-   Target Password

    this parameter is the password for creating the *target* connection

-   Selector

    This represents a JMS selector expression used for consuming
    messages from the source destination. Only messages that match the
    selector expression will be bridged from the source to the target
    destination

    The selector expression must follow the [JMS selector
    syntax](http://docs.oracle.com/javaee/6/api/javax/jms/Message.html)

-   Failure Retry Interval

    This represents the amount of time in ms to wait between trying to
    recreate connections to the source or target servers when the bridge
    has detected they have failed

-   Max Retries

    This represents the number of times to attempt to recreate
    connections to the source or target servers when the bridge has
    detected they have failed. The bridge will give up after trying this
    number of times. `-1` represents 'try forever'

-   Quality Of Service

    This parameter represents the desired quality of service mode

    Possible values are:

    -   `AT_MOST_ONCE`

    -   `DUPLICATES_OK`

    -   `ONCE_AND_ONLY_ONCE`

    See Quality Of Service section for a explanation of these modes.

-   Max Batch Size

    This represents the maximum number of messages to consume from the
    source destination before sending them in a batch to the target
    destination. Its value must `>= 1`

-   Max Batch Time

    This represents the maximum number of milliseconds to wait before
    sending a batch to target, even if the number of messages consumed
    has not reached `MaxBatchSize`. Its value must be `-1` to represent
    'wait forever', or `>= 1` to specify an actual time

-   Subscription Name

    If the source destination represents a topic, and you want to
    consume from the topic using a durable subscription then this
    parameter represents the durable subscription name

-   Client ID

    If the source destination represents a topic, and you want to
    consume from the topic using a durable subscription then this
    attribute represents the the JMS client ID to use when
    creating/looking up the durable subscription

-   Add MessageID In Header

    If `true`, then the original message's message ID will be appended
    in the message sent to the destination in the header
    `ACTIVEMQ_BRIDGE_MSG_ID_LIST`. If the message is bridged more than
    once, each message ID will be appended. This enables a distributed
    request-response pattern to be used

    > **Note**
    >
    > when you receive the message you can send back a response using
    > the correlation id of the first message id, so when the original
    > sender gets it back it will be able to correlate it.

-   MBean Server

    To manage the JMS Bridge using JMX, set the MBeanServer where the
    JMS Bridge MBean must be registered (e.g. the JVM Platform
    MBeanServer or JBoss AS MBeanServer)

-   ObjectName

    If you set the MBeanServer, you also need to set the ObjectName used
    to register the JMS Bridge MBean (must be unique)

The "transactionManager" property points to a JTA transaction manager
implementation. ActiveMQ doesn't ship with such an implementation, but
one is available in the JBoss Community. If you are running ActiveMQ in
standalone mode and wish to use a JMS bridge simply download the latest
version of JBossTS from http://www.jboss.org/jbosstm/downloads and add
it to ActiveMQ's classpath. If you are running ActiveMQ with JBoss AS
then you won't need to do this as JBoss AS ships with a JTA transaction
manager already. The bean definition for the transaction manager would
look something like this:

    <bean name="RealTransactionManager" class="com.arjuna.ats.internal.jta.transaction.arjunacore.TransactionManagerImple"/>

## Source and Target Connection Factories

The source and target connection factory factories are used to create
the connection factory used to create the connection for the source or
target server.

The configuration example above uses the default implementation provided
by ActiveMQ that looks up the connection factory using JNDI. For other
Application Servers or JMS providers a new implementation may have to be
provided. This can easily be done by implementing the interface
`org.apache.activemq.jms.bridge.ConnectionFactoryFactory`.

## Source and Target Destination Factories

Again, similarly, these are used to create or lookup up the
destinations.

In the configuration example above, we have used the default provided by
ActiveMQ that looks up the destination using JNDI.

A new implementation can be provided by implementing
`org.apache.activemq.jms.bridge.DestinationFactory` interface.

## Quality Of Service

The quality of service modes used by the bridge are described here in
more detail.

### AT_MOST_ONCE

With this QoS mode messages will reach the destination from the source
at most once. The messages are consumed from the source and acknowledged
before sending to the destination. Therefore there is a possibility that
if failure occurs between removing them from the source and them
arriving at the destination they could be lost. Hence delivery will
occur at most once.

This mode is available for both durable and non-durable messages.

### DUPLICATES_OK

With this QoS mode, the messages are consumed from the source and then
acknowledged after they have been successfully sent to the destination.
Therefore there is a possibility that if failure occurs after sending to
the destination but before acknowledging them, they could be sent again
when the system recovers. I.e. the destination might receive duplicates
after a failure.

This mode is available for both durable and non-durable messages.

### ONCE_AND_ONLY_ONCE

This QoS mode ensures messages will reach the destination from the
source once and only once. (Sometimes this mode is known as "exactly
once"). If both the source and the destination are on the same ActiveMQ
server instance then this can be achieved by sending and acknowledging
the messages in the same local transaction. If the source and
destination are on different servers this is achieved by enlisting the
sending and consuming sessions in a JTA transaction. The JTA transaction
is controlled by JBoss Transactions JTA \* implementation which is a
fully recovering transaction manager, thus providing a very high degree
of durability. If JTA is required then both supplied connection
factories need to be XAConnectionFactory implementations. This is likely
to be the slowest mode since it requires extra persistence for the
transaction logging.

This mode is only available for durable messages.

> **Note**
>
> For a specific application it may possible to provide once and only
> once semantics without using the ONCE\_AND\_ONLY\_ONCE QoS level. This
> can be done by using the DUPLICATES\_OK mode and then checking for
> duplicates at the destination and discarding them. Some JMS servers
> provide automatic duplicate message detection functionality, or this
> may be possible to implement on the application level by maintaining a
> cache of received message ids on disk and comparing received messages
> to them. The cache would only be valid for a certain period of time so
> this approach is not as watertight as using ONCE\_AND\_ONLY\_ONCE but
> may be a good choice depending on your specific application.

### Time outs and the JMS bridge

There is a possibility that the target or source server will not be
available at some point in time. If this occurs then the bridge will try
`Max Retries` to reconnect every `Failure Retry Interval` milliseconds
as specified in the JMS Bridge definition.

However since a third party JNDI is used, in this case the JBoss naming
server, it is possible for the JNDI lookup to hang if the network were
to disappear during the JNDI lookup. To stop this from occurring the
JNDI definition can be configured to time out if this occurs. To do this
set the `jnp.timeout` and the `jnp.sotimeout` on the Initial Context
definition. The first sets the connection timeout for the initial
connection and the second the read timeout for the socket.

> **Note**
>
> Once the initial JNDI connection has succeeded all calls are made
> using RMI. If you want to control the timeouts for the RMI connections
> then this can be done via system properties. JBoss uses Sun's RMI and
> the properties can be found
> [here](http://docs.oracle.com/javase/6/docs/technotes/guides/rmi/sunrmiproperties.html).
> The default connection timeout is 10 seconds and the default read
> timeout is 18 seconds.

If you implement your own factories for looking up JMS resources then
you will have to bear in mind timeout issues.

### Examples

Please see ? which shows how to configure and use a JMS Bridge with
JBoss AS to send messages to the source destination and consume them
from the target destination.

Please see ? which shows how to configure and use a JMS Bridge between
two standalone ActiveMQ servers.