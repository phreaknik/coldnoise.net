+++
date = '2020-11-10T10:00:00-05:00'
draft = false
title = 'EasyTLV: A Lightweight TLV Serialization Library'
+++

[EasyTLV](https://github.com/phreaknik/easytlv) is a TLV (Tag-Length-Value) serialization library I built while developing smartcard drivers for the [GridPlus hardware wallet](https://gridplus.io/). TLV is a simple binary encoding format where each data element is represented as a tag identifying the type, a length field indicating the size, and the actual value bytes.

## Why Am I Working With TLV?

TLV is an older binary serialization format that does not see much use in new applications today. In fact, with modern binary serialization schemes like Protobuf or CBOR, which offer much better tradeoffs for most applications, it's a wonder people still use TLV. 

There is a niche still, however, where TLV is unlikely to be replaced. Particularly with incredibly resource constrained applications or security sensitive systems that place a high priority on code auditability.

That last case is where I met TLV. I discovered TLV while working on java applet code to run in a secure element (think smart-cards: chip & pin bank card, tap-to-pay credit card, mobile SIM cards, etc). These environments are VERY resource constrained: ROM size *O(10KB)* and memory size *O(100B)*. There is no room for fancy serialization libraries. Additionally, these applications prioritize auditability, often requiring various banking industry certifications, which means every extra line of code is a liability. Here, TLV is king. Dead simple. No-code (de)serialization is possible (even if it creates ugly hard-coded patterns).

## The Problem with Existing Solutions

Most TLV implementations fall into two categories:
1. Vendor-specific libraries locked behind NDAs
2. Application-specific code tightly coupled to particular use cases

In practice, it's fairly common for TLV parsing to look like this:

```c
// Brittle and error-prone
if(rawData[0] == MY_TAG) {           
    int len = rawData[1];            
    memcpy(dest, &rawData[2], len);  
}
```

This breaks when:
- Tags require multiple bytes
- Length exceeds 127 (requiring extended length encoding)
- Packets contain nested structures
- Buffer boundaries aren't checked properly

Even without those limitations, this code is not very ergonomic. Maintaining an application built like this can be a challenge, and the code complexity seems to grow quadratically with the complexity of the message protocol.

## EasyTLV's Approach

I strove to find a middle ground. Especially on the client-side (the terminal talking to the smart card, in my case) where resource constraints are somewhat relaxed, it makes no sense to build applications this way. EasyTLV is still very resource friendly, but provides an improved interface for defining message types and handling (de)serialization of arbitrarily complex messaging protocols.

The library provides a simple token-based API:

**Parsing:**
```c
ETLVToken tokens[10];
int nTokens = 10;
etlv_parse(tokens, &nTokens, rawData, dataLen);

// Access data through token structures
uint32_t value = ntohl(*(uint32_t*)tokens[0].val);
```

**Serialization:**
```c
ETLVToken tokens[] = {
    {.tag = 0x20, .len = 4, .val = &myInt},
    {.tag = 0x21, .len = dataLen, .val = dataPtr}
};

uint8_t output[256];
int outLen = sizeof(output);
etlv_serialize(output, &outLen, tokens, 2);
```

## Key Design Decisions

- **No dynamic memory**: All operations use caller-provided buffers
- **No dependencies**: Not even libc beyond memcpy
- **Predictable performance**: O(n) parsing and serialization with minimal overhead - typically under 1KB of code size
- **Clear error handling**: All functions return explicit error codes
- **Support for extended length encoding**: Handles payloads larger than 127 bytes
- **Nested TLV support**: Can parse hierarchical structures

## Usage Examples

Let's walk through some practical examples to see how EasyTLV simplifies working with TLV data.

### Basic Parsing

When you receive TLV-encoded data, parsing it is straightforward:

```c
// Example TLV data, encoding two 32 bit numbers:
const uint8_t tlvRaw[] = {
    0x02, 0x04, 0x00, 0x00, 0x00, 0x2A, // Number 42 (network byte order)
    0x02, 0x04, 0x00, 0x00, 0x01, 0x01, // Number 257 (network byte order)
};

// Parse the two numbers
ETLVToken t[2];
int nTok = sizeof(t)/sizeof(t[0]);
int err = etlv_parse(t, &nTok, tlvRaw, sizeof(tlvRaw));
assert(err >= 0);

// Cast the numbers to uint32_t's and correct endianness
uint32_t num0 = ntohl(*(uint32_t *)t[0].val);
uint32_t num1 = ntohl(*(uint32_t *)t[1].val);
assert(num0 == 42);
assert(num1 == 257);
```

### Building Messages

Creating TLV messages for transmission is equally simple:

```c
// Define your tags
enum {
  TAG_UID   = 0x10,
  TAG_UNAME,
  TAG_FLAGS,
};

// Data to send
uint32_t userId = htonl(12345);
char userName[] = "Alice";
uint8_t flags = 0x03;

// Define the message structure
ETLVToken message[] = {
    {.tag = TAG_UID,   .len = sizeof(userId), .val = &userId},
    {.tag = TAG_UNAME, .len = strlen(userName), .val = userName},
    {.tag = TAG_FLAGS, .len = sizeof(flags), .val = &flags}
};

// Serialize to buffer
uint8_t output[256];
int outputLen = sizeof(output);
etlv_serialize(output, &outputLen, message, 3);

// output now contains the TLV-encoded message
// outputLen contains the actual size used
```

### Handling Complex Protocols

Consider a more realistic example - a command-response protocol for a secure element:

```c
// Define your tags
enum {
  TAG_AUTH_CMDID = 0x80,
  TAG_AUTH_SESID,
  TAG_AUTH_PIN,
  TAG_STATUS = 0x90,
  TAG_SESTOK,
};

// Command: Authenticate User
typedef struct {
    uint8_t commandId;
    uint32_t sessionId;
    uint8_t pin[8];
} AuthCommand;

AuthCommand authCmd = {
    .commandId = 0xA4,
    .sessionId = htonl(0x1234ABCD),
    .pin = {1, 2, 3, 4, 5, 6, 7, 8}
};

// Build the TLV command
ETLVToken cmdTokens[] = {
    {.tag = TAG_AUTH_CMDID, .len = 1, .val = &authCmd.commandId},
    {.tag = TAG_AUTH_SESID, .len = 4, .val = &authCmd.sessionId},
    {.tag = TAG_AUTH_PIN,   .len = 8, .val = authCmd.pin}
};

uint8_t cmdBuffer[64];
int cmdLen = sizeof(cmdBuffer);
etlv_serialize(cmdBuffer, &cmdLen, cmdTokens, 3);

// Send command and receive response...
uint8_t response[256];
int respLen = receiveResponse(response, sizeof(response));

// Parse the response
ETLVToken respTokens[5];
int respTokenCount = 5;
etlv_parse(respTokens, &respTokenCount, response, respLen);

// Process response based on tags
for (int i = 0; i < respTokenCount; i++) {
    switch (respTokens[i].tag) {
        case TAG_STATUS:  // Status code
            uint8_t status = *(uint8_t*)respTokens[i].val;
            if (status == 0x00) {
                printf("Authentication successful\n");
            }
            break;
        case TAG_SESTOK:  // Session token
            // Process session token...
            break;
    }
}
```

### Nested TLV Structures

EasyTLV also handles nested TLV structures, common in smartcard applications:

```c
// Parse outer container
ETLVToken outer[1];
int outerCount = 1;
etlv_parse(outer, &outerCount, rawData, dataLen);

// Parse inner TLV data from the outer container's value
ETLVToken inner[10];
int innerCount = 10;
etlv_parse(inner, &innerCount, outer[0].val, outer[0].len);

// Now you can access the nested data through the inner tokens
```

## Availability

EasyTLV is available under the MIT license at [github.com/phreaknik/easytlv](https://github.com/phreaknik/easytlv). It's a single-file library that can be dropped into any C project. The repository includes comprehensive examples demonstrating common use cases like nested TLV parsing and working with extended length fields.

## Conclusion

Despite the dominance of Protobuf and other modern serialization formats, TLV still has its place in resource-constrained and security-critical applications. EasyTLV bridges the gap between the simplicity TLV promises and the robustness modern applications demand. It provides a clean, safe API without sacrificing the minimal footprint that makes TLV attractive in the first place. If you're working with smartcards, secure elements, or other constrained environments where TLV is king, EasyTLV can significantly simplify your codebase while maintaining the auditability and efficiency these systems require.
