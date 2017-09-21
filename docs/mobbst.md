# Mobbst (tentative name)
**Mo**dern **BBS** **T**ransfer

A transfer protocol for modern (lol!) Bulletin Board Systems.

**THIS IS A WORKING DRAFT**

## Motivation
Retro transfer protocols such as X/Y/Z modem were designed with [POTS](https://en.wikipedia.org/wiki/Plain_old_telephone_service) in mind where error checking had to be done manually. In the modern day where a connection resides over TCP, most of this is done for us. Additionally, many systems already have great ways to transfer binary data... let's use 'em!

## Protocol Overview
In the following documentation `>>` represnets a client while `<<` represents the server or remote end (the BBS). `<>` Represents a mix.

### Basic Flow

**Download**
```
<< MAGIC
<< HEADER
>> READY
>> CAPS
<< SELECTED_CAPS | CAPS_FAIL
<< BATCH_SIZE
<> FILE_TRANFER [FILE_TRANSFER...]
...
<< BATCH_END
>> END
```

**Upload**
```
>> MAGIC
>> HEADER
<< READY
>> CAPS
<< SELECTED_CAPS | CAPS_FAIL
>> BATCH_SIZE
<> FILE_TRANFER [FILE_TRANSFER...]
>> BATCH_END
<< END
```

### Magic
`MAGIC` is defined as the following bytes:
```
0x1b 0x01 0x02 MBBST 0x03 0x1c
```

### Header
`HEADER` is defined as the following
* Version: Byte. Currently `0x01`
* Reserved: 32 bytes

### Ready
`READY` is defined as the following bytes:
```
0x1b 0x06 0x06 0x06
```

### Capabilities
Shown as `CAPS` above. Specifies one or more capabilities in the following form:
```
0x02 <capName>:<optionalCapValue>; [<CapName>:<OptionalCapValue>;...] 0x03
```

Capabilities are always sent by the **client**. All `capName` and `optionalCapValue` entries are in string form.

#### Registered Capabilities
* `PROT` (required; multi): Sub Protocol. Possible values:
  * `HTTP`: Hyper Text Transfer Protocol hand off.
  * `HTTPS`: Hyper Text Transfer Protocol over TLS/SSL hand off.
  * `BB`: BigBlock
* `HASH` (required; multi): Supported hash. Possible values:
  * `SHA1`: Preferred hashing method
  * `MD5`:
  * `ZCRC32`: Z-Modem style CRC32
* `MBBZ`: Max BigBlock size in bytes. Must be `2048` or higher. `1048576` (1M) recommended.
  
Capabilities marked "multi" may appear more than once, with different values.

An example capabilities block advertising HTTP, HTTPS, and BigBlock protocols, and SHA1 hash:
```
0x02PROT:HTTP;PROT:HTTPS;PROT:BB;HASH:SHA1;0x03
```

#### Selected Capabilities
`SELECTED_CAPS` is a response of **server selected** capabilities chosen from the capabilities advertised from the client. The form is the same as the client `CAPS` with the exception that only single values may be returned and any optionals may be omitted:

```
0x02 <CapName>:<OptionalCapValue>; [<CapName>:<OptionalCapValue>;...] 0x03
```

Example in response to `CAPS` example above:
```
0x02PROT:HTTPS;HASH:SHA1;0x03
```

If a set of capabilities could not be agreed upon, a `NO_CAPS` response is given:
```
0x02 0x03
```

### Batch Size
`BATCH_SIZE` simply specifies the number of items in the batch as a single 16-bit unsigned integer in little endian format.

Example batch size of 7: `0x07 0x00`

### File Transfer
`FILE_TRANSFER` details vary based on the selected `PROT` capability. However, all file transfters start with a `FILE_HASH` header:
```
0x01 <hashDigestBytes> 0x1e
```

`hashDigestBytes` is an array of _1:n_ bytes. The size is fixed by that of the selected `HASH` capability.


#### HTTP or HTTPS Transfers
The `HTTP` and `HTTPS` protocols the same at a basic level.

* The scheme of URL **must** match of `http://` for `HTTP` or `https://` for `HTTPS`. URL encoding must conform to spec and is terminated with `0x00`.
* HTTP(S) POSTs and 200 OK responses **must** contain a valid `Content-Length` header.


##### Upload
```
>> FILE_HASH
<< URL
>> (HTTP(S) POST to URL)
```

##### Download
```
<< FILE_HASH
<< URL
>> (HTTPS(S) download from URL)
```

#### BigBlock
BigBlock transfers require some additional negotiations. A complete `FILE_HASH` is sent, followed by a `FILE_SIZE`, then one or more tuples of `BLOCK_SIZE`, `BLOCK_HASH`, and the block data itself. Each complete block is acknowledged by the other end with a `HASH_OK` or `HASH_FAIL`. Blocks must be less than or equal (<=) to that of the negotiated `MBBZ` value.

* `FILE_SIZE` and `BLOCK_SIZE` take the same form: A single 32-bit unsigned integer in little endian format
* `BLOCK_HASH` takes the same form as `FILE_HASH`
* `HASH_OK`: `0x03`
* `HASH_FAIL`: `0x13`

##### Upload
```
>> FILE_HASH
>> FILE_SIZE
>> BLOCK_SIZE
>> BLOCK_HASH
>> ...binary data...
<< HASH_OK | HASH_FAIL
...
```

##### Download
```
<< FILE_HASH
<< FILE_SIZE
<< BLOCK_SIZE
<< BLOCK_HASH
<< ...binary data...
>> HASH_OK | HASH_FAIL
...
```