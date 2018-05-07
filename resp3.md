# RESP3 specification

Versions history:
* 1.0, 2 May 2018, Initial draft to get community feedbacks.

## Background

The Redis protocol has served us well in the past years, showing that, if carefully designed, a simple human readable protocol is not the bottleneck in the client server communication, and that the simplicity of the design is a major advantage in creating a healthy client libraries ecosystem.

Yet the Redis experience has shown that after about six years from its introduction (when it replaced the initial Redis protocol), the current RESP protocol could be improved, especially in order to make client implementations simpler and to support new features.

At the same time the basic structure is tested and sounding, and there are several things to save from the old design:

* RESP is human readable. This confers to the protocol the important quality of being simple to observe and debug.
* Despite being human readable, RESP is no slower than a binary protocol with fixed length fields, and in many cases can be more compact. Often Redis commands have few arguments composed of few bytes each. Similarly many replies are short. The protocol prefixed lengths represented as decimal digits will save space on the wire, compared to 64 bit integers, and will not be slower to parse. It is possible to design a more efficient binary protocol only introducing the complexity of a variable length encoding, defeating the goal of simplicity.
* The design is very simple: this makes the specification and the resulting implementations short and easy to understand.

At the same time, RESP has flaws. One of the most obvious is the fact that it has not enough semantic power to allow the client to implicitly understand what kind of conversion is appropriate. For instance the Redis commands `LRANGE`, `SMEMBERS` and `HGETALL` will all reply an array, called in RESP v2 terms a *multi bulk reply*. However the three commands actually return an array, a set and a map.
Currently, from the point of view of the client, the conversion of the reply to the programming language type is command-dependent. Clients need to remember what command was called in order to turn the Redis reply into a reply object of some kind. What is worse is that clients need to know about each command, or alternatively provide a lower level interface letting users select the appropriate conversion.

Similarly RESP lacks important data types: floating point numbers and boolean values were returned respectively as strings and integers. Null values had a double representation, called *null bulk* and *null multi bulk*, which is useless, since the semantic value of telling apart null arrays from null strings is non existent.

Finally there was no way to return binary safe errors. When implementing generic APIs producing the protocol, implementations must check and remove potential newlines from the error string.

The limitations stated so far are the main motivations for a new version of RESP. However the redesign gave us a chance to consider different other improvements that may be worthwhile and make the protocol both more powerful and less dependent on the implicit state of the connection.

The gist of such additional features are to provide the protocol with the ability to support a more generic *push mode* compared to the Pub/Sub mode used by Redis, which is not really built-in in the protocol, but is an agreement between the server and the client about the fact that the connection will consume replies as they arrive. Moreover it was useful to modify the protocol to return *out of band* data, such as attributes that augment the reply. Also the protocol was sometimes abused by the internals of Redis, like for example in the case of *slaveless replication*, in order to support streaming of large strings whose length is initially not known. This specification makes this special mode part of the RESP protocol itself.

Finally, in order to follow the standard line termination, in RESP v2 all the protocol tokes are terminated with the two bytes sequence CRLF. Now this choice looks like just a waste of space, so RESP3 is designed in order to terminate everything with just LF.

This specification describes RESP3 from scratch, and not just as a change from RESP v2. However differences between the two will be noted while describing the new protocol.

## Why existing serialization protocols are not good enough?

In theory instead of improving RESP v2, we could use an existing serialization
protocol like MessagePack, BSON or others, which are widely available
and implemented. This is a viable approach, and could be a potential solution,
however there are certain design goals that RESP has that are not aligned
with using such serialization formats.

The first and most important is the fact that such serialization protocols
are not specifically designed for a request-response server-client chat, so
clients and servers would be required to agree on an additional protocol on
top of the underlying serialization protocol. Basically it is not possible
to escape the problem of designing a Redis specific protocol, so
to bind together the serialization and the semantic looks a more effective
way to reach the goals of RESP3.

A different problem is the fact that such serialization protocols are more
complex than RESP, so client libraries would have to rely on a separated
library implementing the serialization protocol. The library may not have
support for streaming directly to a socket, and may return a buffer instead,
leading to potential inefficiencies. Relying on a library in order to perform
the serialization has the other disadvantage of having more moving parts
where things may go wrong, or where different versions of Redis and
serialization libraries may not work well together.

Finally certain features like transferring large strings with initially
unknown length, or streaming of arrays, two features that RESP3 supports
and that this specification describes, are not easy to found among
existing serialization protocols.

In short this specification was written believing that designing a good
serialization format is different compared to designing a protocol
specifically suited in order to support the chat between a client and
its server. RESP3 aims to create a protocol which is not just suitable
for Redis, but more broadly to solve the general problem of client-server
communication in many scenarios even outside Redis and outside the
database space.

## Conventions used in this document

In order to illustrate the RESP3 protocol this specification will show
fragments of the protocol in many sections. In addition of using the escaped
string format, like "foo\n", we'll use a more readable format where
"\n" will be presented as `<LF>` followed by an actual newline. Other
special characters will be displayed as `<\xff>`, where `ff` is the
hex code of the byte.

So for instance the string `"*1\n$1\nA\n" representing a RESP3 array with
a single string `"A"` as unique element, will be presented like this:

    *1<LF>
    $1<LF>
    A<LF>

However for the first part of this specification, both this format and
the escaped string format will be used in order to make sure there is
no confusion. In the latter sections however only one or the other
will be used.

When nested aggregate data structures are shown, indentation is used in
order to make it more clear the actual structure of the protocol. For
instance instead of writing:

    *2<LF>
    *2<LF>
    :1<LF>
    :2<LF>
    #t<LF>

We'll write:

    *2<LF>
        *2<LF>
            :1<LF>
            :2<LF>
        #t<LF>

## RESP3 overview

RESP3 is an updated version of RESP v2, which is the protocol used in Redis
starting with roughly version 2.0 (1.2 already supported it, but Redis 2.0
was the first version to talk only this protocol). The name of this protocol
is just **RESP3** and not RESP v3 or RESP 3.0.

The protocol is designed to handle request-response chats between clients
and servers, where the client performs some kind of request, and the server
replies with some data. The protocol is especially suitable for databases due
to its ability to return complex data types and associated informations to
augment the returned data (for instance the popularity index of a given
information).

The RESP3 protocol is asymmetrical: only a subset can be sent by the client
to the server, while the server can return the full set of types available.
This is due to the fact that RESP is designed to send non structured commands
like `SET mykey somevalue` or `SADD myset a b c d`. Such commands can be
represented as arrays, where each argument is an array element, so this is the
only type the client can send to a server. However different applications
willing to use RESP3 for other goals may just allow the protocol to be used
in a "full duplex" fashion where both the ends can use the full set of types
available.

Not all parts of RESP3 are mandatory for clients and servers. In the specific
case of Redis, RESP3 describes certain functionalities that will be useful
in the future and likely will not be initially implemented. Other optional parts
of RESP3 may be implemented by Redis only in specific situations, like the
link between the primary database and its replicas, or client connections in
a specific state.

## RESP3 types

RESP3 abandons the confusing wording of the second version of RESP, and uses
a much simpler to understand name for types, so you'll see no mention of
*bulk reply* or *multi bulk reply* in this document.

The following are the types implemented by RESP3:

**Types equivalent to RESP version 2**

* Array: an ordered collection of N other types
* Blob string: binary safe strings
* Simple string: a space efficient non binary safe string
* Simple error: a space efficient non binary safe error code and message
* Number: an integer in the signed 64 bit range

**Types introduced by RESP3**

* Null: a single null value replacing RESP v2 `*-1` and `$-1` null values.
* Double: a floating point number
* Boolean: true or false
* Blob error: binary safe error code and message.
* Verbatim string: a binary safe string that should be displayed to humans without any escaping or filtering. For instance the output of `LATENCY DOCTOR` in Redis.
* Map: an ordered collection of key-value pairs. Keys and values can be any other RESP3 type.
* Set: an unordered collection of N other types.
* Attribute: Like the Map type, but the client should keep reading the reply ignoring the attribute type, and return it to the client as additional information.
* Push: Out of band data. The format is like the Array type, but the client should just check the first string element, stating the type of the out of band data, a call a callback if there is one registered for this specific type of push information. Push types are not related to replies, since they are informations that the server may push at any time in the connection, so the client should keep reading if it is reading the reply of a command.
* Hello: Like the Map type, but is sent only when the connection between the client and the server is established, in order to welcome the client with different information like the name of the server, its version, and so forth.
* Big number: a large number non representable by the Number type

## Simple types

This section describes all the RESP3 types which are not aggregate types.
They consist of just a single typed element.

**Blob string**

The general form is `$<length>\n<bytes>\n`. It is basically exactly like
in the previous version of RESP, but instead of using CRLF, only LF is used
to terminate each token.

The string `"hello world"` is represented by the following protocol:

    $11<LF>
    helloworld<LF>

Or as an escaped string:

    "$11\nhelloworld\n"

The length field is limited to the range of an unsigned 64 bit
integer. Zero is a valid length, so the empty string is represented by:

    "$0\n\n"

**Simple string**

The general form is `+<string>\n`, so "hello world" is encoded as

    +hello world<LF>

Or as an escaped string:

    "+hello world\n"

Simple strings cannot contain `<LF>` characters inside.

**Simple error**

This is exactly like a simple string, but the initial byte is `-` instead
of `+`:

    -ERR this is the error description<LF>

Or as an escaped string:

    "-ERR this is the error description\n"

The first word in the error is in upper case and describes the error
code. The remaining string is the error message itself.
The `ERR` error code is the generic one. The error code is useful for
clients to distinguish among different error conditions without having
to do pattern matching in the error message, that may change.

**Number**

The general form is `:<number>\n`, so the number 1234 is encoded as

    :1234<LF>

Or as an escaped string:

    ":1234\n"

Valid numbers are in the range of the signed 64 bit integer.
Larger numbers should use the Big Number type instead.

**Null**

The null type is encoded just as `_\n`, which is just the underscore
character followed by the `LF` character.

**Double**

The general form is `,<floating-point-number>\n`. For instance 1.23 is
encoded as:

    ,1.23<LF>

Or as an escaped string:

    ",1.23\n"

To just start with `.` assuming an initial zero is invalid.
Exponential format is invalid.
To completely miss the decimal part, that is, the point followed by other
digits, is valid, so the number 10 may be returned both using the number
or double format:

    ":10\n"
    ",10\n"

However the client library should return a floating point number in one
case and an integer in the other case, if the programming language in which
the client is implemented has a clear distinction between the two types.

**Boolean**

True and false values are just represented using `#t\n` and `#f\n`
sequences. Client libraries implemented in programming languages without
the boolean type should return to the client the canonical values used
to represent true and false in such languages. For instance a C program
should likely return an integer type with a value of 0 or 1.

**Blob error**

The general form is `!<length>\n<bytes>\n`. It is exactly like the String
type. However like the Simple error type, the first uppercase word represents
the error code.

The error `"SYNTAX invalid syntax"` is represented by the following protocol:

    !21<LF>
    SYNTAX invalid syntax<LF>

Or as an escaped string:

    "*21\nSYNTAX invalid syntax\n"

**Verbatim string**

This is exactly like the String type, but the initial byte is `=` instead
of `*`. Moreover the first three bytes provide information about the format
of the following string, which can be `txt` for plain text, or `mkd` for
markdown. The forth byte is always `<LF>`. Then the real string follows.

For instance this is a valid verbatim string:

    =15<LF>
    txt<LF>
    Some string<LF>

Normal client libraries may ignore completely the difference between this
type and the String type, and return a string in both cases. However interactive
clients such as command line interfaces (for instance `redis-cli`), knows that
the output must be presented to the human user as it is, without quoting
the string.

For example the Redis command `LATENCY DOCTOR` outputs a report that includes
newlines. It's basically a plain text document. However currently `redis-cli`
requires special handling to avoid quoting the resulting string as it normally
does when a string is received.

From the `redis-cli` source code:

```c
output_raw = 0;
if (!strcasecmp(command,"info") ||
    ... [snip] ...
    (argc == 3 && !strcasecmp(command,"latency") &&
                   !strcasecmp(argv[1],"graph")) ||
    (argc == 2 && !strcasecmp(command,"latency") &&
                   !strcasecmp(argv[1],"doctor")))
{
    output_raw = 1;
}
```

With the introduction of verbatim strings clients can be simplified not
having to remember if the output must be escaped or not.

**Big number**

This type represents very large numbers that are out of the range of
the signed 64 bit numbers that can be represented by the Number type.

The general form is `(<big number>\n`, like in the following example:

    (3492890328409238509324850943850943825024385<LF>

Or as an escaped string:

    "(3492890328409238509324850943850943825024385\n"

Big numbers can be positive or negative, but they must not include a
decimal part. Client libraries written in languages with support for big
numbers should just return a big number. When big numbers are not available
the client should return a string, signaling however that the reply is
a big integer when possible (it depends on the API used by the client
library).

## Aggregate data types overview

The types described so far are simple types that just define a single
item of a given type. However the core of RESP3 is the ability to represent
different kinds of aggregate data types having different semantic meaning,
both from the type perspective, and from the protocol perspective.

In general an aggregate type has a given format stating what is the type
of the aggregate, and how many elements there are inside the aggregate.
Then the single elements follow. Elements of the aggregate type can be, in turn,
other aggregate types, so it is possible to have an array of arrays, or
a map of sets, and so forth. Normally Redis commands will use just a subset
of the possibilities. However with Lua scripts or using Redis modules any
combination is possible. From the point of view of the client library however
this is not complex to handle: every type fully specifies how the client
should translate it to report it to the user, so all the aggregated data types
are implemented as recursive functions that then read N other types.

The format for every aggregate type in RESP3 is always the same:

    <aggregate-type-char><numelements><LF> ... numelements other types ...

The aggregate type char for the Array is `*`, so to represent an array
of three Numbers 1, 2, 3, the following protocol will be emitted:

    *3<LF>
    :1<LF>
    :2<LF>
    :3<LF>

Or as an escaped string:

    "*3\n:1\n:2\n:3\n"

Of course an array can also contain other nested arrays:

    *2<LF>
        *3<LF>
            :1<LF>
            $5<LF>
            hello<LF>
            :2<LF>
        #f<LF>

The above represents the array `[[1,"hello",2],false]`

Client libraries should return the arrays with a sensible type representing
ordered sequences of elements, accessible at random indexes in constant
or logarithmic time. For instance a Ruby client should return a
*Ruby array* type, while Python should use a *Python list*, and so forth.

## Map type

Maps are represented exactly as arrays, but instead of using the `*`
byte, the encoded value starts with a `%` byte. Moreover the number of
following elements must be even. Maps represent a sequence of field-value
items, basically what we could call a dictionary data structure, or in
other terms, an hash.

For instance the dictionary represented in JSON by:

    {
        "first":1,
        "second":2
    }

Is represented in RESP3 as:

    %2<LF>
    +first<LF>
    :1<LF>
    +second<LF>
    :2<LF>

Note that after the `%` character, what follow is not, like in the array,
the number of single items, but the number of field-value pairs.

Maps can have any other type as field and value, however Redis will use
only a subset of the available possibilities. For instance it is very unlikely
that Redis commands would return an Array as a key, however Lua scripts
and modules will likely be able to do so.

Client libraries should return Maps using the idiomatic dictionary type
available. However low level languages like C will likely still return
an array of items, but with type informations so that the user can tell
the reply is actually a dictionary.

## Set reply

Sets are exactly like the Array type, but the first byte is `~` instead
of `*`:

    ~5<LF>
    +orange<LF>
    +apple<LF>
    #t<LF>
    :100<LF>
    :999<LF>

However they are semantically different because the represented
items are *unordered* collections of elements, so the client library should
return a type that, while not necessarily ordered, has a *test for existence*
operation running in constant or logarithmic time.

Since many programming languages lack a native set type, it a sensible
choice is to return an Hash where the fields are the elements inside the
Set type, and the values are just *true* values or any other value.

In lower level programming languages such as C, the type should be still
reported as a linear array, together with type information to signal the
user it is a Set type.

## Attribute type

The metadata type is exactly like the Map type, but instead of the `%`
first byte, the `|` byte is used. Attributes describe a dictionary exactly
like the Map type, however the client should not consider such a dictionary
part of the reply, but *just auxiliary data* that is used in order to
augment the reply.

For instance newer versions of Redis may include the ability to report, for
every executed command, the popularity of keys. So the reply to the command
`MGET a b` may be the following:

    |1<LF>
        +key-popularity<LF>
        %2
            :a
            ,0.1923
            :b
            ,0.0012
    *2<LF>
        :2039123
        :9543892

The actual reply to `MGET` was just the two items array `[2039123,9543892]`,
however the attributes specify the popularity (frequency of requests) of the
keys mentioned in the original command, as floating point numbers from 0
to 1 (at least in the example, the actual Redis implementation may differ).

When a client reads a reply and encounters an attribute type, it should read
the attribute, and continue reading the reply. The attribute reply should
be accumulated separately, and the user should have a way to access such
attributes. For instance, if we imagine a session in an higher level language,
something like that could happen:

    > r = Redis.new
    #<Redis client>

    > r.mget("a","b")
    #<Redis reply>

    > r
    [2039123,9543892]

    > r.attribs
    {:key-popularity => {:a => 0.1923, :b => 0.0012}}

Attributes can appear anywhere before a valid part of the protocol identifying
a given type, and will inform only the part of the reply that immediately
follows, like in the following example:

    *3<LF>
        :1<LF>
        :2<LF>
        |1<LF>
            +ttl
            :3600
        :3<LF>

In the above example the third element of the array has an associated
auxiliary information of `{ttl:3600}`. Note that it's not up to the client
library to interpret the attributes, they'll just be passed to the caller
in a sensible way.

## Push type

A push connection is once where the usual *request-response* mode of the
protocol is no longer true, and the server may send to the client asynchronous
data which was not explicitly requested.

In Redis there is already the concept of connection pushing data in at least
three different parts of the Redis protocol:

1. Pub/Sub is a push-mode connection, where clients receive published data.
2. The `MONITOR` command implements an ad-hoc push mode with a kinda unspecified protocol which is obvious to parse, but still, unspecified.
3. The Master-Slave link may, at a first glance, be considered a push mode connection. However actually in this case the client (which is the slave), even if is the entity starting the connection, will configure the connection like if the master is a client sending commands, so in practical terms, it is unfair to call this a push mode connection.

Let's ignore the master-slave link since it is an internal protocol, and
as already noted, is an edge case, and focus on the Pub/Sub and `MONITOR`
modes. They have a common problem:

1. The fact that the connection is in push mode is a private *state* of the connection. Otherwise the data we get from Pub/Sub do not contain anything that at the protocol level to make them distinguishable from other replies.
2. The connection can only be used for Pub/Sub or `MONITOR` once setup in this way, because there is no way (because of the previous problem) in order to tell apart replies from commands and push data.

Moreover a connection for Pub/Sub cannot be used also for `MONITOR` or any other kind of push notifications. For this reasons RESP3 introduces an explicit
push data type, attempting to solve the above issues.

RESP3 push data is represented from the point of view of the protocol exactly
like the Array type. However the first byte is `>` instead of `@`, and the
first element of the array is always a String item, representing the kind
of push data the server is sending to the client. All the other fields in the
push array are type dependent, which means that depending on the type string
as first argument, the remaining items will be interpreted following different
conventions. The existing push data in RESP version 2 will be represented
in RESP3 by the push types `pubsub` and `monitor`.

This is an example of push data:

    >4<LF>
    :pubsub<LF>
    :message<LF>
    :somechannel<LF>
    :this is the message<LF>

*Note that the above format uses simple strings for simplicity, the
actual Redis implementation would use blob strings instead*

In the above example the push data type is `pubsub`. For this type, if
the next element is `message` we know that it's a Pub/Sub message (other
sub types may be subscribe, unsubscribe, and so forth), which is followed
by the channel name and the message itself.

Push data may be interleaved with any protocol data, but always at the top
level, so the client will never find push data in the middle of a Map reply
for instance.

Clients should normally react to the publication of a push data by invoking
a callback associated with the push data type. Synchronous clients may
instead enter a loop and return every new message or command reply.

Note that in this mode it is possible to get both replies and push messages
at the same time, however the order of the commands and their replies is
not affected: if a command is called, the next reply received will be the
one relative of this command, and so forth, even if it is possible that there
will be push data items to consume before. However synchronous clients
may of course miss for a long time that there is something to read in
the socket, because they only read after a command is executed, so the
client library should still allow the user to specify if the connection
should be monitored for new messages in some way (usually by entering
some loop) or not. For asynchronous clients the implementation is a lot more
obvious.

## Hello replies

Hello replies are exactly like Map replies, but they start with the
byte `@` instead of `%`. They are sent immediately by the server after
the connection with the client, in order to inform the client about the
exact object the server represents. The dictionary in the Hello reply is
server dependent, but there is one item which is mandatory: a field
named `server` must tell the client the name of the service that implements
the RESP3 protocol. In the case of Redis the value will just be `redis`.
All the other fields are service dependent. Redis will likely emit
the following fields:

    * server: "redis"
    * version: the Redis version
    * mode: standalone, sentinel, cluster
    * role: master or slave
    * modules: list of loaded modules

The exact number and value of fields emitted by Redis is however currently
a work in progress, you should not rely on the above list.

Note that the Hello reply still allows a client that wants to be compatible
with older versions of Redis to detect the version of the protocol. Upon
connection the client should send a `PING` command. If the first byte read
will be `@`, the version of the protocol is RESP3, the Hello reply will
be processed, and later the `PING` reply will be read. Instead if the first
byte is not `@` we just read the `PING` reply and switch to RESP version 2
mode.

## Acknowledgements

This specification was written by Salvatore Sanfilippo, however the design was informed by multiple people that contributed worthwhile ideas and improvements. A special thank to Dvir Volk and Yao Yue for the conversation and ideas around
having metadata in requests.

## TODOs in this specification

* Document streaming of big strings.
* Document streaming of arrays.
* Document the optional "inline" protocol.
* Document pipelining
