# Spooled Trino protocol (protocol v1+)

> [!IMPORTANT]
> This document is a design proposal and can change in the future.

## Background

The existing [Trino client](https://trino.io/docs/current/develop/client-protocol.html) protocol is over 10 years old and served different purpose upon its inception than the use cases that we want to support today. These new requirements that we want to address include:

- Ability to retrieve data in parallel for faster result retrieval,
- Support for other than JSON encoding formats (both row and column-oriented),
- Moving data encoding from coordinator to workers.
  

To address these challenges, we are going to introduce series of changes to the existing protocol v1 called <u>spooled protocol extension</u>, which do not change the structure and flow of the existing protocol but extend its semantics in a backward-compatible fashion.

## Existing protocol v1

The current client protocol consists of two endpoints that are used to submit queries and retrieve partial results:

- submission (**POST /v1/statement**)
- partial result set retrieval (**GET /v1/statement/executing**)

Both endpoints share the same *QueryResults* definition:

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
> Some of the query results fields were omitted for brevity.

The most important fields related to result set retrieval are:

- **`data`** - contains encoded JSON data as list of row values. Individual values are encoded as a result of the *Type.getObjectValue* call (important: these are low-level, raw representations that are interpreted on the client side (see: AbstractTrinoResultSet).
- **`columns`** - contains list of columns and types descriptors. It's worth to note that `data` field can't have non-null value without `column` information present,
- **`nextUri`** - used to retrieve next partial result set.
  

## Spooled protocol extension

To express the partial result sets having a different encoding, spooled protocol extension is introduced, which contains **two** backward and forward-compatible changes to the existing set of resources and `QueryResults` object. 

These two changes are:

- Extended **data** field on-the-wire representation and meaning,
- Additional header (**Trino-Query-Data-Encoding**)

### Trino-Query-Data-Encoding

The header is used by the client to specify a supported encodings in the order of preference (comma separated). If any of the encodings is supported by the server, the client can expect the **`QueryResults.data`** in a new format. If it's not supported, server fallbacks to the existing behaviour for compatibility.

### EncodedQueryData

**`EncodedQueryData`** is the extension of the **`QueryResults.data`** field that has following on-the-wire representation:

```json
{
  "id": "20160128_214710_00012_rk68b",
  "infoUri": "http://coordinator/query.html?20160128_214710_00012_rk68b",
  "nextUri": null,
  "columns": [...],
  "data": {
    "encodingId": "json-ext",
    "segments": [
      {
        "type": "inline",
        "data": "c3VwZXI=",
        "metadata": {
          "offset": 0,
          "rowsCount": 100,
          "byteSize": 5
        }
      },
      {
        "type": "spooled",
        "dataUri": "http://localhost:8080/v1/download/20160128_214710_00012_rk68b/segments/2",
        "metadata": {
          "offset": 200,
          "rowsCount": 100,
          "byteSize": 1024
        }
      }
    ]
  }
}
```

Meaning of the fields is following:

- **`encodingId`** - the id of the encoding format as requested by the client in the `Trino-Query-Data-Encoding` header. These are known and shared by both the client and the server and are part of the spooled protocol extension,
- **`metadata`** - the Map<String, Object> that represents metadata of the partial result set, i.e. decryption key used to decrypt encoded data,
- **`segments`** - list of data segments representing partial result set. Single response can return arbitrary number of segments.
  

### DataSegment

**`DataSegment`** is a representation of the encoded data and has two distinct types: `inline` and `spooled` with following semantics:
- **`inline`** segment holds encoded, partial result set data in the `byte[] data` field base64-encoded,
- **`spooled`** segment type points to the encoded partial result set spooled in the configured storage location. Location designated by the `URI dataUri` field value is used to retrieve spooled `byte[]` data from the spooling storage. `dataUri` is opaque and contains an authentication information which means that client implementation can retrieve spooled segment data by doing an ordinary `GET` HTTP call without any processing of the URI. It's worth to note that URI can point to arbitrary location including endpoints exposed on the coordinator or storage used for spooling (i.e. presigned URIs on S3). This depends on the actual implementation of the spooling manager and server configuration.
  

> [!IMPORTANT]
> In order to support the spooled protocol, client implementations need to support both inline and spooled representations as server can use these types interchangeably.

> [!CAUTION]
> Client implementation must not send any additional information when retrieving spooled data segment, particularly the authentication headers used to authenticate to a Trino.

**`DataSegment`** contains a **`metadata`** field of type `Map<String, Object>` with attributes describing a data segment.

Following metadata attributes are always present:

- **`offset`** of the data segment in relation to the whole result set (`long`),
- **`rowsCount`** number of the rows in the data segment (`long`),
- **`byteSize`** size of the encoded data segment (`long`).
  

Optional metadata attributes are part of the encoding definition shared between the client and server implementations.

### Implementation considerations

#### Encodings

Encoding describes the serialization format (like JSON) and other information required to both write (encode) and read (decode) result set as data segments. Example of encodings are:

- `json-ext+zstd` which reads as JSON serialization with ZSTD compression,
- `parquet+snappy` which reads as parquet encoding with Snappy compression.
  

Definition and meaning of the encoding is a contract between client and the server.

#### Spooling

Spooling by definition means buffering data in a separate location by the server and retrieving it later by the client. In order to support `spooled protocol extension` servers (coordinator and workers) are required to be configured to use the spooling. In the initial implementation we plan to add a plugin SPI for the `SpoolingManager` and implementation based on the `native file-system API`. `SpoolingManager` is responsible for both storing and retrieving `spooled` data segments from the external storage.

#### Segments

It's up to the server implementation whether the partial result set will be `inlined`, `spooled`, either or both. It's required that client requesting a `spooled protocol extensions` supports both types of the data segments.

#### Plugabillity

An encoding is a shared definition between the client and the server, therefore it can't be a Plugin SPI interface. In the initial implementation, we do not plan to provide an external SPI for adding new encodings. These will be shipped as part of the implementation of the client and server libraries. Spooling process on the server-side is pluggable with the new `SpoolingManager` SPI.

#### Backward compatibility

Above and for all, the new spooled protocol can't break existing one as it's an opt-in client/server feature. At any given moment, the client or the server can fall back to the original, unmodified v1 protocol to maintain backward compatibility.

#### Performance

For small result sets, spooling on the storage adds an additional overhead which can be avoided by inlining the data in the response. For larger ones, spooling allows faster, parallel data retrieval as the result set can be fully spooled and query finished as fast as possible and then retrieved, decoded and processed by the client after the query has already completed.

We plan to implement spooling on the worker side which means that when the result set is partially or fully spooled, coordinator node is not involved in encoding of the data to a requested format and spooling it in the storage. According to our benchmarks, JSON encoding in the existing format accounts for significant amount of the CPU time during output processing. Initially we plan to support existing JSON format but the encoding will be moved to worker nodes which will reduce the load on the coordinator.

#### Security

We plan to add a support for data segment encryption at the later stage with the per-query, ephemeral encryption key. This is yet to be defined.
