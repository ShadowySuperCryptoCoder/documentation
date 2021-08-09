---
layout: nodes.liquid
date: Last Modified
title: "Jobs"
permalink: "docs/jobs/migration-v1-v2/"
---

## Comparison between v1 and v2 jobs

v1 jobs were intended for extremely targeted use cases, and as such, they opted for simplicity in the job spec over explictness.

v2 jobs were designed with an awareness of the rapid expansion of functionality supported by the Chainlink node, as well as the ever-increasing complexity of jobs. As such, they prefer explicitness.

### DAG dependencies and variables

v2 jobs require the spec author to specify dependencies using DOT syntax. If a task needs data produced by another task, this must be explicitly specified using DOT.

Additionally, to facilitate explicitness, v2 jobs require the author to specify inputs to tasks using `$(variable)` syntax.

For example, if an `http` task should feed data into a `jsonparse` task, it should be specified as such:

```dot
fetch [type=http method=get url="http://chain.link/price_feeds/ethusd"]

# This task consumes the output of the 'fetch' task in its 'data' parameter
parse [type=jsonparse path="data,result" data="$(fetch)"]

# This is the specification of the dependency
fetch -> parse
```

- Each job type provides a particular set of variables to its pipeline.  See the documentation for each job type to understand which variables are provided.
- Each task type provides a certain kind of output variable to other tasks that consume it.  See the documentation for each task type to understand their output types.

## Example Migrations

### Runlog with ETH ABI encoding

#### v1 spec

This spec relies on CBOR encoded on-chain values for the `httpget` URL and `jsonparse` path.

```json
{ 
  "name": "Get > Bytes32",
  "initiators": [
    {
      "type": "runlog",
      "params": {
        "address": "YOUR_ORACLE_CONTRACT_ADDRESS"
      }
    }
  ],
  "tasks": [
    {
      "type": "httpget"
    },
    {
      "type": "jsonparse"
    },
    {
      "type": "ethbytes32"
    },
    {
      "type": "ethtx"
    }
  ]
}
```

Notes:
- In v1, the job ID is randomly generated at creation time. In v2 it can either be automatically generated, or manually specified. 
- The `ethbytes32` task (or any of the other ABI-encoding tasks) is now encapsulated within the `ethabiencode` task, with much more flexibility. Please see [the docs for this task](/docs/jobs/task-types/eth-abi-encode/).



#### Equivalent v2 spec:


```toml
ype                = "directrequest"
schemaVersion       = 1
name                = "Get > Bytes32"
contractAddress     = "0x613a38AC1659769640aaE063C651F48E0250454C"
externalJobID       = "0EEC7E1D-D0D2-476C-A1A8-72DFB6633F47" # OPTIONAL - if left unspecified, a random value will be automatically generated
observationSource   = """
    decode_log   [type=ethabidecodelog
                  abi="OracleRequest(bytes32 indexed specId, address requester, bytes32 requestId, uint256 payment, address callbackAddr, bytes4 callbackFunctionId, uint256 cancelExpiration, uint256 dataVersion, bytes data)"
                  data="$(jobRun.logData)"
                  topics="$(jobRun.logTopics)"]

    decode_cbor  [type=cborparse data="$(decode_log.data)"]
    fetch        [type=http method=get url="$(decode_cbor.url)"]
    parse        [type=jsonparse path="$(decode_cbor.path)"]
    encode_data  [type=ethabiencode abi="(uint256 value)" data=<{ "value": $(parse) }>]
    encode_tx    [type=ethabiencode
                  abi="fulfillOracleRequest(bytes32 requestId, uint256 payment, address callbackAddress, bytes4 callbackFunctionId, uint256 expiration, bytes32 data)"
                  data=<{
                      "requestId": $(decode_log.requestId),
                      "payment": $(decode_log.payment),
                      "callbackAddress": $(decode_log.callbackAddr),
                      "callbackFunctionId": $(decode_log.callbackFunctionId),
                      "expiration": $(decode_log.cancelExpiration),
                      "data": $(encode_data)
                  }>]
    submit       [type=ethtx to="$(jobSpec.contractAddress)" data="$(encode_tx)"]

    decode_log -> decode_cbor -> fetch -> parse -> encode_data -> encode_tx -> submit
"""
```

### Simple fetch (runlog)

#### v1 spec:

```json
{
  "initiators": [
    {
      "type": "RunLog",
      "params": { "address": "0x51DE85B0cD5B3684865ECfEedfBAF12777cd0Ff8" }
    }
  ],
  "tasks": [
    {
      "type": "HTTPGet",
      "confirmations": 0,
      "params": { "get": "https://bitstamp.net/api/ticker/" }
    },
    {
      "type": "JSONParse",
      "params": { "path": [ "last" ] }
    },
    {
      "type": "Multiply",
      "params": { "times": 100 }
    },
    { "type": "EthUint256" },
    { "type": "EthTx" }
  ],
  "startAt": "2020-02-09T15:13:03Z",
  "endAt": null,
  "minPayment": "1000000000000000000"
}
```

#### v2 spec:

```toml
type                = "directrequest"
schemaVersion       = 1
name                = "Get > Bytes32"
contractAddress     = "0x613a38AC1659769640aaE063C651F48E0250454C"
externalJobID       = "0EEC7E1D-D0D2-476C-A1A8-72DFB6633F47"
observationSource   = """
    ds          [type=http method=get url="https://bitstamp.net/api/ticker/"]
    parse       [type=jsonparse path="last"]
    multiply    [type=multiply times=100]
    encode_data [type=ethabiencode abi="(uint256 value)" data=<{ "value": $(multiply) }>]
    encode_tx   [type=ethabiencode
                 abi="fulfillOracleRequest(bytes32 requestId, uint256 payment, address callbackAddress, bytes4 callbackFunctionId, uint256 expiration, bytes32 data)"
                 data=<{
                     "requestId": $(decode_log.requestId),
                     "payment": $(decode_log.payment),
                     "callbackAddress": $(decode_log.callbackAddr),
                     "callbackFunctionId": $(decode_log.callbackFunctionId),
                     "expiration": $(decode_log.cancelExpiration),
                     "data": $(encode_data)
                 }>]
    send_tx     [type=ethtx to="$(jobSpec.contractAddress)" data="$(encode_tx)"]

    ds -> parse -> multiply -> encode_data -> encode_tx -> send_tx
"""
```

### Cron

#### v1 spec:

```json
{
  "initiators": [{ "type": "cron", "params": { "schedule": "CRON_TZ=UTC * */20 * * * *" }}],
  "tasks": [{ "type": "HttpGet", "params": { "get": "https://example.com/api" } }]
}
```

#### v2 spec:

```toml
type            = "cron"
schemaVersion   = 1
schedule        = "CRON_TZ=UTC * */20 * * * *"
externalJobID   = "0EEC7E1D-D0D2-476C-A1A8-72DFB6633F46"
observationSource   = """
    ds          [type=http method=GET url="https://example.com/api"];
    ds_parse    [type=jsonparse path="data,price"];
    ds_multiply [type=multiply times=100];
    encode_tx   [type=ethtxabiencode
                 abi="submit(uint256 value)"
                 data=<{ "value": $(ds_multiply) }>]
    submit_tx   [type=ethtx to="0x859AAa51961284C94d970B47E82b8771942F1980" data="$(encode_tx)"]

    ds -> ds_parse -> ds_multiply -> encode_tx -> submit_tx;
"""
```