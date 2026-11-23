# mssql-python Lambda Layer

An AWS Lambda layer that packages [`mssql-python`](https://pypi.org/project/mssql-python/) (Microsoft's official SQL Server driver for Python) together with the Linux system libraries it needs. The Lambda runtime doesn't ship those libraries.

Without this layer, importing and connecting with `mssql-python` on Lambda fails with:

```
mssql_python.exceptions.OperationalError: Driver Error: Connection operation failed;
DDBC Error: Failed to load the driver.
```

## Compatibility

| Setting        | Value         |
|----------------|---------------|
| Runtime        | Python 3.13   |
| Architecture   | x86_64        |
| mssql-python   | 1.18.0        |
| Zipped size    | ~24 MB        |
| Unzipped size  | ~66 MB        |

The function using the layer must use the same runtime and architecture.

## Layer contents

```
layer.zip
├── python/          # mssql-python and its Python dependencies → /opt/python
└── lib/             # native system libraries                   → /opt/lib
    ├── libltdl.so.7
    ├── libkrb5.so.3
    ├── libgssapi_krb5.so.2
    ├── libk5crypto.so.3
    ├── libkrb5support.so.0
    ├── libcom_err.so.2
    └── libkeyutils.so.1
```

Lambda extracts layers into `/opt`, and `/opt/lib` is already on `LD_LIBRARY_PATH`. The driver finds the libraries automatically, so no environment variables are needed.

### Why these libraries

- `libltdl.so.7` is needed by the bundled `libodbcinst.so.2`. A missing `libltdl` is the cause of the "Failed to load the driver" error.
- The Kerberos libraries (`libkrb5`, `libgssapi_krb5` and their dependencies) are needed by the bundled Microsoft ODBC Driver 18 (`libmsodbcsql-18.6.so`).

### Why Ubuntu 20.04 libraries

The libraries come from Ubuntu 20.04 (focal) packages. They require glibc 2.27 or older, so they load on Lambda's Python 3.13 runtime (Amazon Linux 2023). Libraries from newer distributions, such as Ubuntu 24.04 or recent WSL images, need a newer glibc and fail to load on Lambda.

## Deploying

### AWS Console

1. Go to **Lambda → Layers → Create layer**.
2. Upload `layer.zip`.
3. Choose compatible runtime **Python 3.13** and architecture **x86_64**.
4. Open your function, go to **Layers → Add a layer → Custom layers**, and select this layer.

### AWS CLI

```bash
aws lambda publish-layer-version \
  --layer-name mssql-python \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.13 \
  --compatible-architectures x86_64
```

## Usage

```python
import mssql_python

def lambda_handler(event, context):
    conn = mssql_python.connect(
        "Server=tcp:<host>,1433;"
        "Database=<database>;"
        "UID=<user>;"
        "PWD=<password>;"
        "Encrypt=yes;"
    )
    cursor = conn.cursor()
    cursor.execute("SELECT TOP 5 * FROM my_table WHERE id = ?", (1,))
    rows = cursor.fetchall()
    conn.close()
    return {"rows": len(rows)}
```

Notes:
- Parameters use `?` placeholders.
- Keep credentials out of code. Use environment variables, Secrets Manager or Parameter Store.
- If the server uses a self-signed certificate, add `TrustServerCertificate=yes;` to the connection string.
- `Connection Timeout=` isn't a recognized keyword in the connection string, and the driver raises a parse error if you include it.
- The function needs network access to the database: VPC configuration, security groups, and port 1433 open.

## Known limitations

- **Bulk copy (`cursor.bulkcopy`) and the async API don't work.** They use a separate native module (`mssql_py_core`) that requires OpenSSL 1.1, which Lambda's runtime doesn't include. Regular `connect`, `execute` and `fetch` calls are unaffected.
- **Only Python 3.13 on x86_64 is supported.** Rebuild for other versions or for arm64 (see below).
- The SUSE driver variant is removed to save space. The RHEL and Debian/Ubuntu variants are both kept and tested, since the driver picks one based on OS release files.


## Troubleshooting

| Error | Likely cause |
|-------|--------------|
| `No module named 'mssql_python'` | The layer isn't attached, or the zip doesn't have `python/` at its root |
| `Failed to load the driver` | `lib/` is missing from the zip, or the runtime or architecture doesn't match |
| `TCP Provider: Error code 0x2749` / timeout | The driver loaded fine; this is a network problem (VPC, security group, host or port) |
| `GLIBC_2.xx not found` | Libraries were copied from a distribution that's too new. Use the Ubuntu 20.04 packages above |
