# Trino protocol v1 extensions (protocol v1.5)

> [!IMPORTANT]
> This is a draft document and can change in the future.

## Background

The existing [Trino client](https://trino.io/docs/current/develop/client-protocol.html) protocol is over 10 years old and served different purposes upon its conception than the use cases that we want to support today. These new use cases include:

- Ability to retrieve data in parallel by multiple threads for faster data movement,
  
- Ability to retrieve data from multiple clients for faster data processing,
  
- Support for other than JSON serialization formats (both row and column-oriented).
  

To address these challenges, we are going to introduce intermediate changes to the existing protocol v1 called <u>protocol extensions</u> which do not change the structure and flow of the existing protocol but extend its semantics in a backward-compatible way.

## Existing protocol v1

The current client protocol consists of two endpoints that are used to submit queries and retrieve partial results:

- QueuedStatementResource (**POST /v1/statement**)
- ExecutingStatementResource (**GET /v1/statement/executing**)

Both endpoints share the *QueryResults* object:

```json
{
  "id": "20160128_214710_00012_rk68b",
  "infoUri": "http://coordinator/query.html?20160128_214710_00012_rk68b",
  "nextUri": null,
  "columns": [
    {
      "name": "_col0",
      "type": "bigint",
      "typeSignature": {
        "rawType": "bigint",
        "arguments": []
      }
    }
  ],
  "data": [[1],[5]]
  ...
}
```

> [!NOTE]
> Some of the result fields were omitted for brevity.

The most important fields related to data transmission are:

- **`data`** contains serialized data in the array of arrays of object format. Values are encoded as a result of the *Type.getObjectValue* call (important: these are low-level, raw representations that are decoded on the client side (see: AbstractTrinoResultSet).
  
- **`columns`** contain a list of columns and types descriptors. data cannot be present without column information.
  
- **`nextUri`** is used to point a client to the next endpoint to retrieve.
  

## Protocol extensions v1.5

To express the partial/final results in a different format, protocol extensions are introducing two backward and forward-compatible changes to the existing set of resources and QueryResults object. 

These two changes are:

- New QueryResults.**data** field representation and meaning,
- Additional header (**Trino-Query-Data-Extension**)

### Trino-Query-Data-Extension

The header is used by the client to request a given protocol extension. If it is supported, the client can expect the `QueryResults.data` in a new format. If it's not supported, `USER_ERROR` is raised.

### ExtQueryData

`ExtQueryData` is an alternative representation of the `QueryResults.data` field that carries over the following subfields:

- **`extension`** - the id of the extension format as requested by the client in the `Trino-Query-Data-Extension` header. These are known to both the client and the server and are part of the protocol extensions.
  
- **`metadata`** - the Map<String, Object> that allows attaching additional metadata to the results, i.e. encoded schema, number of rows so far, size of the result, etc.
  
- **`segments`** - List<DataSegment> that allows returning one or more `DataSegment` objects
  

### DataSegment

`DataSegment` is a pointer to the data and has two types - `inline` and `spooled`.

- **`inline`** data segment type encodes a partial result in the `byte[] values` field.
  
- **`spooled`** data segment type encodes a partial result as a spooled file in the URI location stored in the `URI uri` field. URIs are opaque and can point to either coordinator resource, worker resource, or directly to the storage in the form of the signed URI. This will depend on the actual implementation and runtime configuration.
  

`DataSegment` contains a `metadata` field of type Map<String, Object> that allows attaching additional metadata to the results, i.e. number of rows in the segment, uncompressed size, encryption key, etc.

### Extended QueryResults

```json
{
  ...
  "columns": [
    {
      "name": "_col0",
      "type": "bigint",
      "typeSignature": {
        "rawType": "bigint",
        "arguments": []
      }
    }
  ],
  "data": {
    "extension": "json-v2",
    "metadata": {
        "encryption-key": "SecretEncryptionKey",
        "compression": "snappy"
    },
    "segments": [
      {
        "type": "inline",
        "values": "W1sxMF0sIFsyMF1d", // base64 encoded byte[]
        "metadata": {
          "offset": 0,
          "rows": 12,
          "size": 1203
      },
      {
        "type": "spooled",
        "uri": "http://coordinator/v1/segments/segment-id", // base64 encoded byte[]
        "metadata": {
          "offset": 12,
          "rows": 5,
          "size": 1503
        }
      }
    ]
  },
  ...
}
```

## Implementation considerations

### Extension

Protocol extension describes the serialization format (like JSON), type encoding, and data retrieval requirements (compression, encryption) and must be treated as a whole which means that both server and client understand the extension in the same way and supports control metadata fields.

### Segments

It's up to the engine to decide whether the result set will be inlined, spooled, or both (depending on the result size). For small results, spooling on the storage adds overhead but for big result sizes, spooling can complete queries faster and allow the client to move at its own pace or distribute work to multiple threads.

The engine can also decide whether it will return a partial result set (consecutive `segments` are returned when `nextURI` is fetched) or all of the segments will be returned at once when the engine has finished processing the query.

### Plugabillity

An extension is a shared definition between the client and the server, therefore it can't be a Plugin SPI interface. In the initial implementation, we do not plan to provide an external SPI that will allow us to supplement the client and the server with new extensions. These new extensions will be a part of the distribution for both client and server.

### Backward compatibility

Above and for all, the new extensions can't break existing protocols as these are opt-in client/server features. At any given moment, the client can fall back to the original, unmodified v1 protocol to maintain backward compatibility.

### Class hierarchy

![EncodedQueryData (1)](https://github.com/wendigo/wendigo/assets/66972/d4409719-5881-4091-a070-217ce822e5d8)
