# RESP3 specification

Versions history:
* 1.0, 2 May 2018, Initial draft to get community feedbacks.
* 1.1, 5 Nov 2018, Leave CRLF as terminators + improved "hello reply" section.
* 1.2, 5 Nov 2018, A few things are now better specified in the document
                   thanks to developers reading the specification and
                   sending their feedbacks. No actual change to the protocol
                   was made.
* 1.3, 11 Mar 2019, Streamed strings and streamed aggregated types.

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

The gist of such additional features are to provide the protocol with the ability to support a more generic *push mode* compared to the Pub/Sub mode used by Redis, which is not really built-in in the protocol, but is an agreement between the server and the client about the fact that the connection will consume replies as they arrive. Moreover it was useful to modify the protocol to return *out of band* data, such as attributes that augment the reply. Also the protocol was sometimes abused by the internals of Redis, like for example in the case of *replicaless replication*, in order to support streaming of large strings whose length is initially not known. This specification makes this special mode part of the RESP protocol itself.

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
string format, like "foo\r\n", we'll use a more readable format where
"\r\n" will be presented as `<CR><LF>` followed by an actual newline. Other
special characters will be displayed as `<\xff>`, where `ff` is the
hex code of the byte.

So for instance the string `"*1\r\n$1\r\nA\r\n"` representing a RESP3 array with
a single string `"A"` as unique element, will be presented like this:

    *1<CR><LF>
    $1<CR><LF>
    A<CR><LF>

However for the first part of this specification, both this format and
the escaped string format will be used in order to make sure there is
no confusion. In the latter sections however only one or the other
will be used.

When nested aggregate data structures are shown, indentation is used in
order to make it more clear the actual structure of the protocol. For
instance instead of writing:

    *2<CR><LF>
    *2<CR><LF>
    :1<CR><LF>
    :2<CR><LF>
    #t<CR><LF>

We'll write:

    *2<CR><LF>
        *2<CR><LF>
            :1<CR><LF>
            :2<CR><LF>
        #t<CR><LF>

Both the indentation and the newlines are hence only used in order to improve
human readability and are not semantical in respect of the actual protocol.

## RESP3 overview

RESP3 is an updated version of RESP v2, which is the protocol used in Redis
starting with roughly version 2.0 (1.2 already supported it, but Redis 2.0
was the first version to talk only this protocol). The name of this protocol
is just **RESP3** and not RESP v3 or RESP 3.0.

The protocol is designed to handle request-response chats between clients
and servers, where the client performs some kind of request, and the server
replies with some data. The protocol is especially suitable for databases due
to its ability to return complex data types and associated information to
augment the returned data (for instance the popularity index of a given
information).

The RESP3 protocol can be used asymmetrically, as it is in Redis: only a subset
can be sent by the client to the server, while the server can return the full set
of types available. This is due to the fact that RESP is designed to send non
structured commands like `SET mykey somevalue` or `SADD myset a b c d`. Such
commands can be represented as arrays, where each argument is an array element,
so this is the only type the client needs to send to a server. However different
applications willing to use RESP3 for other goals may just allow the protocol
to be used in a "full duplex" fashion where both the ends can use the full set
of types available.

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
* Push: Out of band data. The format is like the Array type, but the client should just check the first string element, stating the type of the out of band data, a call a callback if there is one registered for this specific type of push information. Push types are not related to replies, since they are information that the server may push at any time in the connection, so the client should keep reading if it is reading the reply of a command.
* Hello: Like the Map type, but is sent only when the connection between the client and the server is established, in order to welcome the client with different information like the name of the server, its version, and so forth.
* Big number: a large number non representable by the Number type

## Simple types

This section describes all the RESP3 types which are not aggregate types.
They consist of just a single typed element.

**Blob string**

The general form is `$<length>\r\n<bytes>\r\n`. It is basically exactly like
in the previous version of RESP.

The string `"hello world"` is represented by the following protocol:

    $11<CR><LF>
    helloworld<CR><LF>

Or as an escaped string:

    "$11\r\nhelloworld\r\n"

The length field is limited to the range of an unsigned 64 bit
integer. Zero is a valid length, so the empty string is represented by:

    "$0\r\n\r\n"

**Simple string**

The general form is `+<string>\r\n`, so "hello world" is encoded as

    +hello world<CR><LF>

Or as an escaped string:

    "+hello world\r\n"

Simple strings cannot contain the `<CR>` nor the `<LF>` characters inside.

**Simple error**

This is exactly like a simple string, but the initial byte is `-` instead
of `+`:

    -ERR this is the error description<CR><LF>

Or as an escaped string:

    "-ERR this is the error description\r\n"

The first word in the error is in upper case and describes the error
code. The remaining string is the error message itself.
The `ERR` error code is the generic one. The error code is useful for
clients to distinguish among different error conditions without having
to do pattern matching in the error message, that may change.

**Number**

The general form is `:<number>\r\n`, so the number 1234 is encoded as

    :1234<CR><LF>

Or as an escaped string:

    ":1234\r\n"

Valid numbers are in the range of the signed 64 bit integer.
Larger numbers should use the Big Number type instead.

**Null**

The null type is encoded just as `_\r\n`, which is just the underscore
character followed by the `CR` and `LF` characters.

**Double**

The general form is `,<floating-point-number>\r\n`. For instance 1.23 is
encoded as:

    ,1.23<CR><LF>

Or as an escaped string:

    ",1.23\r\n"

To just start with `.` assuming an initial zero is invalid.
Exponential format is invalid.
To completely miss the decimal part, that is, the point followed by other
digits, is valid, so the number 10 may be returned both using the number
or double format:

    ":10\r\n"
    ",10\r\n"

However the client library should return a floating point number in one
case and an integer in the other case, if the programming language in which
the client is implemented has a clear distinction between the two types.

In addition the double reply may return positive or negative infinity
as the following two stings:

    ",inf\r\n"
    ",-inf\r\n"

So client implementations should be able to handle this correctly.

**Boolean**

True and false values are just represented using `#t\r\n` and `#f\r\n`
sequences. Client libraries implemented in programming languages without
the boolean type should return to the client the canonical values used
to represent true and false in such languages. For instance a C program
should likely return an integer type with a value of 0 or 1.

**Blob error**

The general form is `!<length>\r\n<bytes>\r\n`. It is exactly like the String
type. However like the Simple error type, the first uppercase word represents
the error code.

The error `"SYNTAX invalid syntax"` is represented by the following protocol:

    !21<CR><LF>
    SYNTAX invalid syntax<CR><LF>

Or as an escaped string:

    "!21\r\nSYNTAX invalid syntax\r\n"

**Verbatim string**

This is exactly like the Blob string type, but the initial byte is `=` instead
of `$`. Moreover the first three bytes provide information about the format
of the following string, which can be `txt` for plain text, or `mkd` for
markdown. The fourth byte is always `:`. Then the real string follows.

For instance this is a valid verbatim string:

    =15<CR><LF>
    txt:Some string<CR><LF>

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

The general form is `(<big number>\r\n`, like in the following example:

    (3492890328409238509324850943850943825024385<CR><LF>

Or as an escaped string:

    "(3492890328409238509324850943850943825024385\r\n"

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

    <aggregate-type-char><numelements><CR><LF> ... numelements other types ...

The aggregate type char for the Array is `*`, so to represent an array
of three Numbers 1, 2, 3, the following protocol will be emitted:

    *3<CR><LF>
    :1<CR><LF>
    :2<CR><LF>
    :3<CR><LF>

Or as an escaped string:

    "*3\r\n:1\r\n:2\r\n:3\r\n"

Of course an array can also contain other nested arrays:

    *2<CR><LF>
        *3<CR><LF>
            :1<CR><LF>
            $5<CR><LF>
            hello<CR><LF>
            :2<CR><LF>
        #f<CR><LF>

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

    %2<CR><LF>
    +first<CR><LF>
    :1<CR><LF>
    +second<CR><LF>
    :2<CR><LF>

Note that after the `%` character, what follows is not, like in the array,
the number of single items, but the number of field-value pairs.

Maps can have any other type as field and value, however Redis will use
only a subset of the available possibilities. For instance it is very unlikely
that Redis commands would return an Array as a key, however Lua scripts
and modules will likely be able to do so.

Client libraries should return Maps using the idiomatic dictionary type
available. However low level languages like C will likely still return
an array of items, but with type information so that the user can tell
the reply is actually a dictionary.

## Set reply

Sets are exactly like the Array type, but the first byte is `~` instead
of `*`:

    ~5<CR><LF>
    +orange<CR><LF>
    +apple<CR><LF>
    #t<CR><LF>
    :100<CR><LF>
    :999<CR><LF>

However they are semantically different because the represented
items are *unordered* collections of elements, so the client library should
return a type that, while not necessarily ordered, has a *test for existence*
operation running in constant or logarithmic time.

Since many programming languages lack a native set type, a sensible
choice is to return an Hash where the fields are the elements inside the
Set type, and the values are just *true* values or any other value.

In lower level programming languages such as C, the type should be still
reported as a linear array, together with type information to signal the
user it is a Set type.

Normally Set replies should not contain the same element emitted multiple
times, but the protocol does not enforce that: client libraries should try
to handle such case, and in case of repeated elements, do some effort to
avoid returning duplicated data, at least if some form of hash is used in
order to return the reply. Otherwise when returning an array just reading
what the protocol contains, duplicated items if present could be passed
by client libraries to the caller. Many implementations will find it very
natural to avoid duplicates. For instance they'll try to add every read
element in some Map or Hash or Set data type, and adding the same element
again will either replace the old copy or will fail silently, retaining the
old copy.

## Attribute type

The attribute type is exactly like the Map type, but instead of the `%`
first byte, the `|` byte is used. Attributes describe a dictionary exactly
like the Map type, however the client should not consider such a dictionary
part of the reply, but *just auxiliary data* that is used in order to
augment the reply.

For instance newer versions of Redis may include the ability to report, for
every executed command, the popularity of keys. So the reply to the command
`MGET a b` may be the following:

    |1<CR><LF>
        +key-popularity<CR><LF>
        %2<CR><LF>
            $1<CR><LF>
            a<CR><LF>
            ,0.1923<CR><LF>
            $1<CR><LF>
            b<CR><LF>
            ,0.0012<CR><LF>
    *2<CR><LF>
        :2039123<CR><LF>
        :9543892<CR><LF>

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

    *3<CR><LF>
        :1<CR><LF>
        :2<CR><LF>
        |1<CR><LF>
            +ttl
            :3600
        :3<CR><LF>

In the above example the third element of the array has an associated
auxiliary information of `{ttl:3600}`. Note that it's not up to the client
library to interpret the attributes, they'll just be passed to the caller
in a sensible way.

## Push type

A push connection is one where the usual *request-response* mode of the
protocol is no longer true, and the server may send to the client asynchronous
data which was not explicitly requested.

In Redis there is already the concept of a connection pushing data in at least
three different parts of the Redis protocol:

1. Pub/Sub is a push-mode connection, where clients receive published data.
2. The `MONITOR` command implements an *ad-hoc* push mode with a kinda unspecified protocol which is obvious to parse, but still, unspecified.
3. The Master-Replica link may, at a first glance, be considered a push mode connection. However actually in this case the client (which is the replica), even if is the entity starting the connection, will configure the connection like if the master is a client sending commands, so in practical terms, it is unfair to call this a push mode connection.

Let's ignore the master-replica link since it is an internal protocol, and
as already noted, is an edge case, and focus on the Pub/Sub and `MONITOR`
modes. They have a common problem:

1. The fact that the connection is in push mode is a private *state* of the connection. Otherwise the data we get from Pub/Sub do not contain anything that at the protocol level to make them distinguishable from other replies.
2. The connection can only be used for Pub/Sub or `MONITOR` once setup in this way, because there is no way (because of the previous problem) in order to tell apart replies from commands and push data.

Moreover a connection for Pub/Sub cannot be used also for `MONITOR` or any other kind of push notifications. For this reasons RESP3 introduces an explicit
push data type, attempting to solve the above issues.

RESP3 push data is represented from the point of view of the protocol exactly
like the Array type. However the first byte is `>` instead of `*`, and the
first element of the array is always a String item, representing the kind
of push data the server is sending to the client. All the other fields in the
push array are type dependent, which means that depending on the type string
as first argument, the remaining items will be interpreted following different
conventions. The existing push data in RESP version 2 will be represented
in RESP3 by the push types `pubsub` and `monitor`.

This is an example of push data:

    >4<CR><LF>
    +pubsub<CR><LF>
    +message<CR><LF>
    +somechannel<CR><LF>
    +this is the message<CR><LF>

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
at the same time, interleaved in any way, however the order of the commands
and their replies is not affected: if a command is called, the next reply
(that is not a push reply) received will be the one relative of this command,
and so forth, even if it is possible that there will be push data items to
consume before.

For instance after a `GET key` command, it is possible to get the two following
valid replies:

    >4<CR><LF>
    +pubsub<CR><LF>
    +message<CR><LF>
    +somechannel<CR><LF>
    +this is the message<CR><LF>
    $9<CR><LF>
    Get-Reply<CR><LF>

Or in inverse order:

    $9<CR><LF>
    Get-Reply<CR><LF>
    >4<CR><LF>
    +pubsub<CR><LF>
    +message<CR><LF>
    +somechannel<CR><LF>
    +this is the message<CR><LF>

Still the client will know that the first non push type reply processed
will be the actual reply to GET.

However synchronous clients may of course miss for a long time that there is
something to read in the socket, because they only read after a command is
executed, so the client library should still allow the user to specify if the
connection should be monitored for new messages in some way (usually by
entering some loop) or not. For asynchronous clients the implementation is a
lot more obvious.

## Streamed strings

Normally RESP strings have a prefixed length:

    $1234<CR><LF>
    .... 1234 bytes of data here ...<CR><LF>

Unfortunately this is not always optimal.

Sometimes it is very useful to transfer a large string from the server
to the client, or the other way around, without knowing in advance the
size of such string. Redis already uses this feature internally, however it
is a private extension of the protocol in the case of RESP2. For RESP3
we want it to be part of the specification, because we found other uses
for the feature where it is crucial that the client has support for it.

For instance in diskless replication the Redis master sends the RDB file
for the first synchronization to its replica without generating the file
to disk: instead the output of the RDB file, that is incrementally generated
from the keyspace data in memory, is directly sent to the socket. In this
case we don't have any way to know in advance the final length of the
string we are transferring.

The protocol we used internally was something like that:

    $EOF:<40 bytes marker><CR><LF>
    ... any number of bytes of data here not containing the marker ...
    <40 bytes marker>

As we already specified, this was just a *private extension* only known
by the server itself. It uses an EOF marker that is generated in a pseudo
random way, and is practically impossible to collide with normal data. However
such approach, we found, have certain limits when extended to be a known,
well documented mechanism that Redis can use when talking with other clients.
We were worried expecially by the following issues:

1. Generating the EOF: failing at that makes the protocol very fragile, and often even experienced developers don't know much about probability, counting, and randomness.
2. Parsing the EOF: while not so hard, is non trivial. The client need to implement an algorithm that can detect the EOF even while reading it in two separated calls.

For this reason instead the final version of this specification proposes
a chunked encoding approach, that is often used in order protocols.

The protocol can be easily explained by a small example, in which the
string "Hello world" is transmitted without knowing its size in advance:

    $?<CR><LF>
    ;4<CR><LF>
    Hell<CR><LF>
    ;5<CR><LF>
    o wor<CR><LF>
    ;1<CR><LF>
    d<CR><LF>
    ;0<CR><LF>

Basically the transfer starts with `$?`. We use the same prefix as normal
strings, that is `$`, but later instead of the count we use a question mark
in order to communicate the client that this is a chunked encoding transfer,
and we don't know the final size yet.

Later the differnet parts are transferred like that:

    ;<count><CR><LF>
    ... coun bytes of data ...<CR><LF>

The transferring program can send as many parts as it likes, there are no
limits. Finally in order to signal that the transfer has ended, a part
with length zero, without any data, is transferred:

    ;0<CR><LF>

Note that right now the Redis server does not implement such protocol, that
is, there is no command that will reply like that, however it is likely that
we'll soon implement this ability at least for modules. However it is currently
not planned to have support to send streamed strings to the server, as
part of a command.

## Streamed aggregated data types

Sometimes it is useful to send an aggregated data type whose size is not
known in advance. Imagine a search application written as a Redis module
that collects the data it finds in the inverted index using multiple threads
and, as it finds results, it sends such results to the client.

In this, and many other situations, the size of the result set is not known
in advance. So far the only possibility to solve the issue has been to
buffer the result in memory, and at the end send everything, because the
old RESP2 protocol had no mechanism in order to send aggregated data types
without specifying the final number of elements immediately.

For instance the Array type is like that:

    *123<CR><LF>      (number of items)
    :1<CR><LF>        (items...)
    :2<CR><LF>
    ...

RESP3 extends this mechanism, allowing to send all the aggregated data
types of type Array, Set and Map, not specifying the length, but instead
using an explicit terminator. The transfer is initiated like that
(in the case of the Array type):

    *?<CR><LF>

So instead of the length, we just use a '?' character. Then we can
continue to reply with other RESP3 types:

    :1<CR><LF>
    :2<CR><LF>
    :3<CR><LF>

Finally we can terminate the Array using a special **END type**, that
has the following format:

    .<CR><LF>

Unbound Sets are exactly like Arrays with the difference that the transfer
wills tart with `~` instead of `*` character. However with the Map type
things are marginally different: the program emitting the protocol *must*
make sure that it emits an even number of elements, since every couple
represents a field-value pair:

    %?<CR><LF>
    +a<CR><LF>
    :1<CR><LF>
    +b<CR><LF>
    :2<CR><LF>
    .<CR><LF>

Currently there is no Redis 6 command that uses such extension to the protocol,
however it is possible that modules running in Redis 6 will use such feature
so it is suggested for client libraries to implement this part of
the specification ASAP, and if this is not the case, to clearly document that
this part of RESP3 is not supported.

## The HELLO command and connection handshake

RESP connections should always start sending a special command called HELLO.
This step accomplishes two things:

1. It allows servers to be backward compatible with RESP2 versions. This is needed in Redis in order to switch more gently to the version 3 of the protocol.
2. The `HELLO` command is useful in order to return information about the server and the protocol, that the client can use for different goals.

The HELLO command has the following format and arguments:

    HELLO <protocol-version> [AUTH <username> <password>]

Currently only the AUTH option is available, in order to authenticate the client
in case the server is configured in order to be password protected. Redis 6
will support ACLs, however in order to just use a general password like in
Redis 5 clients should use "default" as username (all lower case).

The first argument of the command is the protocol version we want the connection
to be set. By default the connection starts in RESP2 mode. If we specify a
connection version which is too big and is not supported by the server, it should
reply with a -NOPROTO error. Example:

    Client: HELLO 4
    Server: -NOPROTO sorry this protocol version is not supported

Then the client may retry with a lower protocol version.

Similarly the client can easily detect a server that is only able to speak
RESP2:

    Client: HELLO 3 AUTH default mypassword
    Server: -ERR unknown command 'HELLO'

It can then proceed by sending the AUTH command directly and continue talking
RESP2 to the server.

Note that even if the protocol is supported, the HELLO command may return an
error and perform no action (so the connection will remain in RESP2 mode) in case
the authentication credentials are wrong:

    Client: HELLO 3 AUTH default mypassword
    Server: -ERR invalid password
    (the connection remains in RESP2 mode)

The successful reply to the HELLO command is just a map reply.
The information in the Hello reply is in part server dependent, but there are
certain fields that are mandatory for all the RESP3 implementations:

    * server: "redis" (or other software name)
    * version: the server version
    * proto: the maximum version of the RESP protocol supported

In addition, in the case of the RESP3 implementation of Redis, the following
fields will also be emitted:

    * id: the client connection ID
    * mode: "standalone", "sentinel" or "cluster"
    * role: "master" or "replica"
    * modules: list of loaded modules as an array of strings

The exact number and value of fields emitted by Redis is however currently
a work in progress, you should not rely on the above list.

## Acknowledgements

This specification was written by Salvatore Sanfilippo, however the design was informed by multiple people that contributed worthwhile ideas and improvements. A special thank to:

    * Dvir Volk
    * Yao Yue
    * Yossi Gottlieb
    * Marc Gravell
    * Nick Craver

For the conversation and ideas to make this specification better.

## FAQ

* **Why the RESP3 line break was not changed to a single character?**

Because that would require a client to send command, and parse replies, in
a different way based on the fact the client is in RESP v2 or v3 mode.
Even a client supporting only RESP3, would start sending the `HELLO` command
with CRLF as separators, since initially the connection is in RESP v2 mode.
The parsing code would also be designed in order to accept the different line
break based on the conditions. All in all the saving of one byte did not made
enough sense in light of a more complex client implementation, especially since
the way RESP3 is designed, it is mostly a superset of RESP2, so most clients
will just have to add the parsing of the new data types supported without
touching the implementation of the old ones.

## TODOs in this specification

* Document the optional "inline" protocol.
* Document pipelining
