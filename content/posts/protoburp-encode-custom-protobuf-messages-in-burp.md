---
title: "ProtoBurp: Encode and Fuzz Custom Protobuf Messages in Burp Suite"
date: 2023-08-02T14:22:41-07:00
cover:
    image: "/introducing-protoburp.png"
---

# Background

Protocol Buffers (Protobufs) are a language agnostic data serialization format that allow data to be safely and efficiently trasmitted or stored. Protobuf usage has exploded within the past several years. When testing web applications, mobile applications, and embedded devices alike, it's increasingly likely you'll encounter Protobuf data within requests like this:

{{< figure src="/normal-protobuf-binary-request.png" alt="Normal binary Protobuf request" caption="A normal binary Protobuf request" >}}

You might have logically tried to fuzz these inputs as you would any other parameter, only to realize that things weren't as simple as they appeared:

{{< figure src="/naively-fuzzing-protobuf-binary-data.png" alt="Naively fuzzing binary Protobuf data" caption="Naively fuzzing binary protobuf data doesn't work" >}}

Additionally, you might have discovered a Protobuf definition (`.proto`) file like the one below and wondered how you could use such a file to send valid, fuzzable requests. If so, `ProtoBurp` was made for you ❤️

```protobuf
syntax = "proto3";

package tutorial;

message Person {
  string name = 1;
  int32 id = 2;
  string email = 3;

  enum PhoneType {
    PHONE_TYPE_UNSPECIFIED = 0;
    PHONE_TYPE_MOBILE = 1;
    PHONE_TYPE_HOME = 2;
    PHONE_TYPE_WORK = 3;
  }

  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
  }

  repeated PhoneNumber phones = 4;
}

message AddressBook {
  repeated Person people = 1;
}
```

# Introducing ProtoBurp

`ProtoBurp` is a Burp Suite extension that enables security researchers to encode and fuzz custom Protobuf messages. It allows users to automatically convert JSON data into a Protobuf message based on a provided protobuf definition file. This opens up opportunities for fuzzing inputs using Burp's Repeater and Intruder tools, as well as proxy traffic from other tools (e.g. `sqlmap`).

{{< figure src="/protoburp-transformation.png" alt="Protoburp's functionality" caption="ProtoBurp converts JSON data into a valid Protobuf" >}}

## How do I use ProtoBurp?

The magic behind `ProtoBurp` lies in its ability to dynamically create Protobuf messages based on provided JSON input and a Protobuf definition file.

{{< figure src="/protoburp-extension-tab.png" alt="Protoburp extension tab in Burp Suite" caption="The ProtoBurp extension tab in Burp Suite" >}}

1. Install ProtoBurp and its dependencies (see the [GitHub repository](https://github.com/dillonfranke/protoburp))
2. Create or obtain a `.proto` file you'd like to use to serialize Protobuf messages (examples are in the `test_app` folder).
3. Use the `protoc` utility to compile your `.proto` file into Python format

```bash
protoc --python_out=./ MyMessage.proto
```

4. Click the 'Choose File' button to select your compiled protobuf file.
5. Check the 'Enable ProtoBurp' checkbox.
6. All requests sent with the header `ProtoBurp: True` will then be converted from JSON to a Protobuf!

### Usage with Intruder
Once you have `ProtoBurp` set up, you can fuzz Protobuf inputs with Burp just like you would any other parameter! For example, to fuzz the `number` field we've seen in the previous examples, we can just send our JSON payload request to Intruder:

{{< figure height=600 src="/burp-intruder-usage.png" alt="Burp Suite Intruder usage" caption="Burp Suite Intruder usage" >}}

For each fuzzed request Intruder makes, `ProtoBurp` will handle the conversion from JSON into Protobuf format:

{{< figure height=600 src="/burp-intruder-usage-requests.png" alt="Converted Intruder requests" caption="Intruder requests converted into Protobuf data" >}}

> **Note:** The modified requests won't show up in Burp's proxy history. You'll need to view the outgoing requests using Burp's `Logger` or an extension like `Logger++`, as I did above.

### Usage with sqlmap (or any external tool)
Possibly the largest motivator for creating `ProtoBurp` was to generate a way to seamlessly use external security tools on Protobuf inputs. For example, during one web application assessment, I identified an OR-based SQL injection in a Protobuf field, as shown below.

{{< figure src="/or-based-sql-injection.png" alt="Successful OR-based SQLi" caption="Successful OR-based SQL injection" >}}

Normally, my next step would be to pass this request to `sqlmap` and have the database fully dumped within 30 minutes. However, I couldn't do this because of the pesky Protobufs.

But now enter `ProtoBurp`. It is now possible to give `sqlmap` a JSON payload to test its payloads and let `ProtoBurp` handle the Protobuf encoding. We can now use the following command:

```shell
sqlmap -u http://chickenblasters.com:5000 --force-ssl --level 5 --risk 3 -H "ProtoBurp: true" --batch --proxy=http://localhost:8080 --data='{"people":[{"name":"JohnDoe","id":1,"email":"john.doe@example.com","phones":[{"number":"1234567890*","type":"PHONE_TYPE_MOBILE"},{"number":"0987654321","type":"PHONE_TYPE_HOME"}]}]}'
```

> **Note**: Make sure to set pass the `ProtoBurp: True` header!

All proxy traffic from `sqlmap` is then converted by `ProtoBurp`:

{{< figure height=600 src="/sqlmap-protoburp-success.png" alt="Sqlmap ProtoBurp success" caption="Sqlmap payloads converted by ProtoBurp" >}}

### Generating a JSON payload
You might be wondering: "How can I generate a JSON object from a `.proto` file to use with `ProtoBurp`?"

Easy, I wrote a script that, given a `.proto` file, will fill in placeholder values to generate a JSON payload. You can then use the JSON payload with `ProtoBurp`. Here's how you use the script:

```bash
❯ python3 json-generator.py
Usage: python3 json-generator.py <compiled_proto_definition_pb2.py> <MessageName>
```
```bash
❯ python3 json-generator.py test_app/addressbook_pb2.py AddressBook
{
  "people": [
    {
      "name": "example",
      "id": 1,
      "email": "example",
      "phones": [
        {
          "number": "example",
          "type": "PHONE_TYPE_UNSPECIFIED"
        },
        {
          "number": "example",
          "type": "PHONE_TYPE_UNSPECIFIED"
        }
      ]
    },
    {
      "name": "example",
      "id": 1,
      "email": "example",
      "phones": [
        {
          "number": "example",
          "type": "PHONE_TYPE_UNSPECIFIED"
        },
        {
          "number": "example",
          "type": "PHONE_TYPE_UNSPECIFIED"
        }
      ]
    }
  ]
}
```

## What about existing tooling?

I've used the `blackboxprotobuf` [Burp extension](https://github.com/nccgroup/blackboxprotobuf) for years, which aids in the decoding, editing, and re-encoding of Protobuf messages without known `.proto` files. For example, the figure below shows the Protobuf from above decoded by `blackboxprotobuf`.

{{< figure height=400 src="/blackbox-protobuf-result.png" alt="Blackboxprotobuf in action" caption="The Blackboxprotobuf extension in action" >}}

While it is an excellent tool, one of `blackboxprotobuf`'s notable limitations is the fact that it can only send protobuf messages it has previously seen through Burp Suite's proxy. Sometimes, however, researchers want to send new protobuf messages, either because we want to fuzz them, or we know the format of a specific message.

`ProtoBurp` solves two novel problems:
1. The ability to send new protobuf messages that weren't already seen through proxy traffic
2. Operability with Burp Intruder and other extensions to fuzz protobuf inputs

# ProtoBurp GitHub Repository

You can access `ProtoBurp`, the JSON generator script, and a test application that accepts Protobufs [here](https://github.com/dillonfranke/protoburp). Detailed installation and usage instructions are also included.

Please submit any bugs or feature requests to `ProtoBurp`'s [issue tracker](https://github.com/dillonfranke/protoburp/issues).

Thanks for reading, and I wish you much success testing Protobufs!