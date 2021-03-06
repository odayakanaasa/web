---
path: /docs/ifaces/basics/
---

# Protocol Basics

## Main interface parts

Each FutoIn interface must have three mandatory fields:

1. `iface` - globally unique identifier:
    - e.g. `com.example.project.component`,
    - it should follow Java-style with company domain in reverse order,
    - `futoin.` prefix is used for official FutoIn specs.
1. `version` - SemVer version of the interface
    - only MAJOR and MINOR part
3. `ftn3rev` - minimal version of [FTN3][] specification to use.
    - important to enable new features,
    - still consider to use older versions to be compatible with older
        implementations of Invoker and Executor.

Optional fields:

* `imports` - list of mixin interfaces in `{name}:{version}` format:
    - `types` and `funcs` get copied.
* `inherit` - name of base interface:
    - the derived interface can be called via base interface,
    - Executor forbids registration with conflict in handling.
* `funcs` - map of function identifiers to their definitions:
    - identifiers must be in `camelCase` starting from lower case,
    - details are in the following chapters.
* `types` - map of custom type identifiers to their definitions:
    - identifier must be in `CamcelCase` starting from upper case,
    - details are in the following chapters.
* `requires` - list of constraints:
    - `AllowAnonymous` - allow anonymous calls.
    - `SecureChannel` - requires secure channel.
    - `BiDirectChannel` - required bi-directional channel.
    - `MessageSignature` - require Message Authentication Code.
    - `BinaryData` - require binary data support (codec).
* `desc` - arbitrary description string

## Message format

All messages are represented in tree structure and can be coded in various formats.
The default one is JSON as it's the fastest to process.

### Request message fields

* `f` - function to call:
    - format: `{interface}:{version}:{function}`,
    - e.g. `futoin.db.l1:1.0:query`
* `p` - optional call parameters, if required by spec:
    - type: key-value map
* `rid` - optional request ID counter for multiplexing purposes:
    - added only for bi-directional channels,
    - it is always auto-generated by implementation,
    - no code must depend on that value.
- `forcersp` - optional indicator to force response message:
    - it's very special for `SimpleCCM` and to be defined in other chapters.
- `sec` - optional security field:
    - see below
- `obf` - optional on-behalf-of indicator:
    - see below

### Response message fields

* `r` - result according to definition in spec.
* `rid` - copy of request `rid`, if present.
* `sec` - present, if request uses Message Authentication Code.

### Error response message fields

* `e` - associative error code as string.
* `edesc` - optional, error description.
* `sec` - present, if request uses verified Message Authentication Code

## Security

Security data in transmitted in special `sec` field.

Base security is derived from HTTP Basic Authentication with `{user}:{password}` pair.
Basic Auth can be extracted from `Authorization` HTTP header.

More secure is **HMAC** approach `-hmac:{user}:{type}:{hash}`.

Basic Auth is very weak protection and should be avoided. HMAC adds security, but still
has many flaws with secret key management. Advanced Security concept is a separate FutoIn
specification.

## On-behalf-of calls

FutoIn requests support a special `obf` field to allows calls on behalf of another user
what is typical in cases when one Services makes calls to another Services with authorization
of some particular user. Details are also covered in dedicated spec.

## Description inside spec

Types, functions, parameters, result variables and map fields may have optional `desc` field
inside the spec to add arbitrary description.

However, it's use should be minimized as the spec itself must be self-explanatory.

## Examples spec

FutoIn Database interface Level 1.

```json
{
  "iface": "futoin.db.l1",
  "version": "1.0",
  "ftn3rev": "1.7",
  "imports": [
    "futoin.ping:1.0"
  ],
  "types": {
    "Query": {
      "type": "string",
      "minlen": 1,
      "maxlen": 10000
    },
    "Identifier": {
      "type": "string",
      "maxlen": 256
    },
    "Row": "array",
    "Rows": {
      "type": "array",
      "elemtype": "Row",
      "maxlen": 1000
    },
    "Field": {
      "type": "string",
      "maxlen": 256
    },
    "Fields": {
      "type": "array",
      "elemtype": "Field",
      "desc": "List of field named in order of related Row"
    },
    "Flavour": {
      "type": "Identifier",
      "desc": "Actual actual database driver type"
    },
    "QueryResult": {
      "type": "map",
      "fields": {
        "rows": "Rows",
        "fields": "Fields",
        "affected": "integer"
      }
    }
  },
  "funcs": {
    "query": {
      "params": {
        "q": "Query"
      },
      "result": "QueryResult",
      "throws": [
        "InvalidQuery",
        "Duplicate",
        "OtherExecError",
        "LimitTooHigh"
      ]
    },
    "callStored": {
      "params": {
        "name": "Identifier",
        "args": "Row"
      },
      "result": "QueryResult",
      "throws": [
        "InvalidQuery",
        "Duplicate",
        "OtherExecError",
        "LimitTooHigh",
        "DeadLock"
      ]
    },
    "getFlavour": {
      "result": "Flavour"
    }
  }
}
```

## Example messages

Canonical formatted JSON examples.

Request:

```json
{
    "f" : "futoin.db.l1:1.0:query",
    "p" : {
        "q" : "SELECT 1 AS N"
    },
    "sec" : "-hmac:user:SHA-256:abcd...efgh"
}
```

Response:

```json
{
    "r" : {
        "rows" : [
            [ "1" ]
        ],
        "fields" : [ "N" ]
    },
    "sec" : "-hmac:user:SHA-256:abcd...efgh"
}
```

Error:

```json
{
    "e" : "SecurityError",
    "edesc" : "Invalid user or HMAC"
}
```

[FTN3]: https://specs.futoin.org/final/preview/ftn3_iface_definition.html
