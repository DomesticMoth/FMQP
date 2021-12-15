# FMQP specification

## Abstract
**FMQP** (which stands for **F**eedback **M**essage **Q**ueue **P**rotocol) is a Client Server publish/subscribe messaging transport protocol. It is designed to be used in low-latency environments to work with large streams of rapidly aging data. FMQP is designed to be easy to implement and use.  
It works over protocols that provide ordered, lossless bidirectional connections, such as TCP/IP, TLS, UNIX DOMAIN SOCKET, etc.  
The main feature of this protocol is the ability of message publishers to receive feedback from the broker. This allows publishers to perform labor-intensive work "on demand" and publish only those messages to which someone is subscribed.  

## Licensing 
<p xmlns:dct="http://purl.org/dc/terms/">
  <a rel="license"
     href="http://creativecommons.org/publicdomain/zero/1.0/">
    <img src="http://i.creativecommons.org/p/zero/1.0/88x31.png" style="border-style: none;" alt="CC0" />
  </a>
  <br />
  To the extent possible under law,
  <a rel="dct:publisher"
     href="https://github.com/DomesticMoth/FMQP">
    <span property="dct:title">DomesticMoth</span></a>
  has waived all copyright and related or neighboring rights to
  <span property="dct:title">FMQP specification</span>.
</p>
Free as beer and free as freedom. 

## Terminology
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this specification are to be interpreted as described in [IETF RFC 2119](#rfc2119).  

**Client** - a program or device that uses **FMQP** and initializes the connection to the server.  

**Broker** (aka **server**) - a program or device that accepts client connections via the **FMQP** protocol, working as an intermediary between connected **clients** by transmitting **messages** between them.  

**Packet** - a fragment of binary data of finite length transmitted between clients and a broker.  

**Message** - the semantic content of the **packet**. The terms "**packet**" and "**message**" MAY be used interchangeably.  

**Payload** - a fragment of binary data of finite length that can be part of a **packet** (**message**). **Payload** transfer between **clients** via a **broker** is the main purpose of the **FMQP** protocol.  

**Topic** - a string included in the **message** and used for routing **messages** by the **broker**. Also, a **topic** is a collection of all **messages** that include equivalent **topics**.  

**Routing** - a set of actions taken by a **broker** to transfer **messages** between **clients**.  

**Topic system** - an abstraction in the logic of the **broker's** work through which **messages** are **routed**. The **FMQP** protocol describes two separate **topic systems**.  

## Interaction between the client and the broker
**Client** can act as a publisher and subscriber (including simultaneously).  
As a publisher, the **client** can publish **messages** in **topics**.  
As a subscriber, the **client** can subscribe/unsubscribe to certain **topics** and receive **messages** from the **broker** that are published in these **topics**.  

## Authentication and access control
**FMQP** does not include authentication and access control mechanisms.  
If necessary, this functionality SHOULD be provided by underlying protocols, such as TLS.  

## Caching
By default, the **broker** SHOULD NOT store **messages** after passes them to subscribers.  
However, **FMQP** includes a flag that allows the publisher to inform the **broker** about the need for caching for the specified message.  
If this flag is enabled, this **message** MUST be stored by the **broker**. Whenever a new **client** subscribes, the **broker** must transfer to him all the cached **messages** published under the topics of interest to the **client**.  
The cache size for each individual **topic** is equal to one **message**. Thus, publishing a new **message** with caching enabled will displace the previous one from the cache.  

## Last will
**FMQP** includes a flag that allows the publisher to mark the published **message** as a last will. In this case, the **broker** MUST NOT publish it immediately, but instead MUST publish it after the **client** who sent it disconnects.  

## Empty messages
The **client** MAY send **messages** with an empty **payload** to the **broker**. The **broker** MUST NOT forward this **messages** to other **clients**.  

## Topics
First of all, the **topic** is represented by a set of [UTF-8](#rfc3629) encoded characters, the size of the **topic** MUST NOT exceed 255 bytes (the NULL character "\0" MUST NOT and, accordingly, the size of the **topic** is not taken into account).  
Semantically, **topics** are divided into several levels separated by a "/" sign, similar to paths in *nix systems or in URIs.  
If the "/" symbol is at the beginning or end of the **topic**, it MUST be ignored.  
A **topic** MUST NOT have zero length or consist of only one or more "/" characters.  
Examples of valid **topics**:  
+ "home/living-space/living-room1/temperature"
+ "home/living-space/living-room1/temperature/"
+ "/home/living-space/living-room1/temperature"
+ "/home/living-space/living-room1/temperature/"
+ "/home/"
+ "/home/"
+ "home/"
+ "home"

Examples of invalid **topics**:  
+ "/"
+ "////////////////////"
+ ""
### Equivalence of topics
Two **topics** are considered equivalent if they consist of the same number of levels and all the levels included in them in the same positions are equivalent.  
In addition, there is a special level "*" which is considered equivalent to any other.  
Thus, the following **topics** are considered equivalent to each other:  
+ "home/living-space/living-room1/temperature"
+ "home/living-space/living-room1/*"
+ "home/living-space/*/temperature"
+ "home/*/living-room1/temperature"
+ "*/living-space/living-room1/temperature"
+ "\*/\*/\*/\*"

## Message routing
When receiving a message from the publisher, the **broker** MUST forward it to all connected **clients** who have subscribed to **topics** equivalent to the topic under which this **message** was published. 

## Service topics and subscriptions
There are predefined **topics** to which the **broker** himself can send **messages**. They are called service ones.  
Service **topics** are distinguished by the fact that they start at the "$" level, for example, "$/signals/stop".  
Service **topics** exist for **clients** to communicate directly with the **broker** himself for management purposes, statistics collection, etc.  
The **broker** MUST NOT forward **messages** published in service **topics** unless it is explicitly stated in the description of their purpose.  
There are the following **topics** predefined by the protocol specification:  
+ "$/signals/terminate" - when any **client** publishes any **message** to this **topic**, the **broker** MUST immediately stop its work
+ "$/signals/stop" - when any **client** publishes any **message** in this **topic**, the **broker** MUST proceed to a "soft" shutdown by sequentially performing the following steps:
  1) Stop accepting new **clients**
  2) Stop accepting new **messages** from **clients**
  3) Wait until subscribers receive all **messages** that have already been published by this time
  4) Forcibly send all "last wills"
  5) Wait until all the "wills" are received by the **clients** who have subscribed to them
  6) Close all **client** connections
  7) Stop
+ "$/info/clients" - every time the number of **clients** changes, the **broker** MUST send a **message** to this **topic** in which a 64-bit unsigned integer in the Be format is located as a **payload*** - the current number of **clients**. This **message** must be cached
+ "$/info/messages/second" - every second the **broker** must publish a **message** to this **topic** in which a 64-bit unsigned integer in the Be format is located as a **payload** - the number of **messages** received by the **broker** during this second

In addition, specific **broker** implementations can add their own service **topics** to this list.  

## Feedback topics
**FMQP** describes, in addition to the main **topic system**, an additional system of feedback **topics** that exists in parallel with it.  
To distinguish the main **topic** system, a special flag is provided in the **FMQP** **message** structure (more on this later).  
In this **system of topics**, only the **broker** himself can publish **messages**, any **messages** published in it by **clients** MUST be discarded by the **broker** as incorrect.  
Whenever any of the **clients** subscribes/unsubscribes to a regular **topic**, the **broker** MUST publish a **message** with the same **topic** and a raised feedback flag.  
These **messages** MUST be cached.  
This mechanism allows publishers to track which **topics** subscribers are interested in.  

## Debugging messages
**FMQP** includes a flag that allows to mark the subscription as debug.  
In this case, information about it should not be transmitted through the feedback **topics system**.  

## Processing of incorrect messages
Any incorrectly formed **messages** MUST be discarded by the recipient, whether it is a **broker** or a **client**.  

## Packets format
**Messages** transmitted between the **broker** and **clients** consist of a header, topic and body.  
The header consists of a set of flags that fit together in one byte, a single-byte field of topic length and a 4-byte field of body length.  
The topic and body lengths are unsigned integers in the Be format.  
Flags are represented by single bits taking Boolean values 0-false and 1-true.  
The list of flags included in the header and going in the following order from the beginning of the zero byte of the message:  
0) Package type 0-publication 1-subscription
1) Subscription type 0-subscription 1-unsubscribe
2) Topic system type 0-normal 1-feedback
3) Debugging flag
4) Last will flag
5) Caching flag

The header is followed by the **topic** and the body of the **message** length, respectively.  
The format of the **message** in graphic form:  
```
0        1        2        3        4        5        6  ~     ~
+--------+--------+--------+--------+--------+--------+- ~ -+- ~ -+
|01234567|topic ln|            body length            |  ~  |  ~  |
+||||||--+--------+--------+--------+--------+--------+- ~ -+- ~ -+
 ||||||                                                  ~     ~
 ||||||                                                  |     |
 ||||||                        $(topic ln) bytes topic --+     |
 ||||||                            $(body length) bytes body --+
 |||||+-- cache flag (makes sense only for regular topics publications)
 ||||+-- last will flag (makes sense only for regular topics publications)
 |||+-- debug flag (makes sense only for regular topics subscriptions)
 ||+-- topic system type 0-regular 1-feedback
 |+-- subscribe type 0-sub 1-unsub
 +-- package type 0-pub 1-sub

Legend:

0 <---------- byte nomber        ~
+--------+                    +- ~ -+
|0       | <- byte field      |  ~  | <- variable number of bytes
+|-------+                    +- ~ -+
 +-------- <- bit                ~

+--------+--------+--------+--------+
|             somename              | <- a field of several bytes long
+--------+------|-+--------+--------+
                |
                +---- <- field names
                | 
             +--|-----+
             |somename|
             +--------+
```


## Normative references
### RFC2119
Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.  
http://www.ietf.org/rfc/rfc2119.txt  
### RFC3629
Yergeau, F., "UTF-8, a transformation format of ISO 10646", STD 63, RFC 3629, November 2003   
http://www.ietf.org/rfc/rfc3629.txt  
