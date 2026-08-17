# OssTables Lance Connector

A Lance Namespace implementation for [Alibaba Cloud OssTables](https://www.alibabacloud.com/product/oss)
that adds **AWS SigV4 request signing** to the standard
[Lance REST Namespace](https://lance.org/docs/namespace/) protocol.

This allows customers to use OssTables' Lance catalog with the released Lance SDK
(Python, Java, Spark, Trino, Ray) without waiting for native SigV4 support upstream.

## Installation

**Python**

```bash
pip install osstables-lance-connector
```

**Java (Maven)**

```xml
<dependency>
    <groupId>com.aliyun.lance</groupId>
    <artifactId>osstables-lance-connector</artifactId>
    <version>0.1.0</version>
</dependency>
```

For Spark / Trino classpath isolation, build with the `shaded` profile:
```bash
mvn package -Pshaded
```

## Quick Start

### Python

```python
import lance
import lance_namespace

ns = lance_namespace.connect("osstables", {
    "osstables.uri": "https://<bucket>.<region>.oss-tables.aliyuncs.com/lance",
    "osstables.region": "<region>",
    "osstables.service": "osstables",
    "osstables.access_key_id": "<catalog-ak>",
    "osstables.secret_access_key": "<catalog-sk>",
})

storage_options = {
    "access_key_id": "<oss-ak>",
    "access_key_secret": "<oss-sk>",
    "endpoint": "https://oss-<region>-internal.aliyuncs.com",
    "region": "<region>",
}

data = pa.table({"id": [1, 2, 3]})
lance.write_dataset(data, namespace_client=ns,
                    table_id=["my_db", "my_table"],
                    mode="create", storage_options=storage_options)

ds = lance.dataset(namespace_client=ns,
                   table_id=["my_db", "my_table"],
                   storage_options=storage_options)
print(ds.count_rows())
```

### Java

```java
import org.lance.namespace.LanceNamespace;

Map<String, String> props = Map.of(
    "osstables.uri", "https://<bucket>.<region>.oss-tables.aliyuncs.com/lance",
    "osstables.region", "<region>",
    "osstables.service", "osstables",
    "osstables.access_key_id", "<catalog-ak>",
    "osstables.secret_access_key", "<catalog-sk>");

LanceNamespace ns = LanceNamespace.connect(
    "com.aliyun.lance.osstables.namespace.OssTablesNamespace", props, allocator);
```

## Configuration Properties

All properties use the `osstables.` prefix.

| Property | Required | Description |
|----------|----------|-------------|
| `osstables.uri` | Yes | Catalog endpoint URI |
| `osstables.region` | Yes | SigV4 signing region (e.g. `cn-hangzhou`) |
| `osstables.service` | No | SigV4 service name (default: `osstables`) |
| `osstables.access_key_id` | No | Explicit access key (falls back to env vars) |
| `osstables.secret_access_key` | No | Explicit secret key |
| `osstables.session_token` | No | STS temporary session token |
| `osstables.delimiter` | No | Multi-level ID delimiter (default: `$`) |
| `osstables.verify_ssl` | No | TLS verification (default: `true`) |

When `access_key_id` / `secret_access_key` are not provided, credentials are resolved
from environment variables: `OSSTABLES_ACCESS_KEY_ID`, `OSSTABLES_SECRET_ACCESS_KEY`,
`OSSTABLES_SESSION_TOKEN`.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Lance SDK / Spark / Trino / Ray                    │
│  (namespace_client / spark.sql.catalog.impl / ...)  │
└───────────────────────┬─────────────────────────────┘
                        │
           ┌────────────▼────────────┐
           │  OssTablesNamespace     │
           │  (this package)         │
           │  ┌───────────────────┐  │
           │  │ SigV4 Signer      │  │ Signs each HTTP request
           │  └───────────────────┘  │
           └────────────┬────────────┘
                        │ Lance REST protocol (signed)
                        ▼
           ┌─────────────────────────┐
           │  OssTables REST Server  │  Catalog (metadata)
           └─────────────────────────┘

Data plane (read/write files) goes directly to OSS
via storage_options — independent of catalog auth.
```

## Engine Integration

**Spark**
```
spark.sql.catalog.lance.impl = com.aliyun.lance.osstables.namespace.OssTablesNamespace
spark.sql.catalog.lance.osstables.uri = ...
spark.sql.catalog.lance.osstables.region = ...
spark.sql.catalog.lance.storage.access_key_id = ...
```

**Trino** (`etc/catalog/lance.properties`)
```
lance.impl = com.aliyun.lance.osstables.namespace.OssTablesNamespace
lance.osstables.uri = ...
lance.osstables.region = ...
lance.storage.access_key_id = ...
```

## Development

```bash
# Python
cd python && pip install -e ".[dev]" && pytest

# Java
cd java && mvn verify
```

All tests run offline with zero credentials (using AWS public SigV4 test vectors
and local mock gateways).

## License

[MIT](../LICENSE)
