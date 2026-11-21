# httparse

[![crates.io](https://img.shields.io/crates/v/httparse.svg)](https://crates.io/crates/httparse)
[![Released API docs](https://docs.rs/httparse/badge.svg)](https://docs.rs/httparse)
[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE-MIT)
[![CI](https://github.com/seanmonstar/httparse/workflows/CI/badge.svg)](https://github.com/seanmonstar/httparse/actions?query=workflow%3ACI)
[![Discord chat][discord-badge]][discord-url]

A push parser for the HTTP 1.x protocol. Avoids allocations. No copy. **Fast.**

Works with `no_std`, simply disable the `std` Cargo feature.

[Changelog](https://github.com/seanmonstar/httparse/releases)


[discord-badge]: https://img.shields.io/discord/500028886025895936.svg?logo=discord
[discord-url]: https://discord.gg/kkwpueZ

## Usage

```rust
let mut headers = [httparse::EMPTY_HEADER; 128];
let mut req = httparse::Request::new(&mut headers);

let buf = b"GET /index.html HTTP/1.1\r\nHost";
assert!(req.parse(buf)?.is_partial());

// a partial request, so we try again once we have more data

let buf = b"GET /index.html HTTP/1.1\r\nHost: example.domain\r\n\r\n";
assert!(req.parse(buf)?.is_complete());
```

## Header Buffer Sizing

When parsing HTTP requests, you must allocate a header buffer. The size of this buffer determines how many headers can be parsed.

### Recommended Sizes

Choose a buffer size based on your use case:

| Use Case | Headers | Rationale |
|----------|---------|-----------|
| **Minimal** (8-16) | Controlled environments | IoT devices, embedded systems, internal APIs |
| **Standard** (32-64) | Most applications | Typical client-server communication |
| **Web Server** (64-128) | Production services | Public-facing APIs, services behind load balancers |
| **Gateway/Proxy** (128-256) | High complexity | Services aggregating headers from multiple sources |

### Example

```rust,no_run
use httparse::{Request, EMPTY_HEADER};

// For a production web server, use 128 headers
const MAX_HEADERS: usize = 128;
let mut headers = [EMPTY_HEADER; MAX_HEADERS];
let mut req = Request::new(&mut headers);

let buf = b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n";
match req.parse(buf) {
    Ok(status) => { /* ... */ },
    Err(httparse::Error::TooManyHeaders) => {
        // Request exceeded MAX_HEADERS
        eprintln!("Request has more than {} headers", MAX_HEADERS);
    }
    Err(e) => { /* handle other errors */ }
}
```

### Memory Impact

Each header requires approximately 40 bytes (two pointers + two lengths):

| Headers | Memory | Use Case |
|---------|--------|----------|
| 8 | ~320 B | Minimal |
| 16 | ~640 B | Embedded |
| 32 | ~1.3 KB | Basic |
| 64 | ~2.6 KB | Standard |
| **128** | **~5.1 KB** | **Production (Recommended)** |
| 256 | ~10.2 KB | Gateway |

### Production Considerations

Modern HTTP requests commonly have 40-60 headers due to:

- **Load Balancers**: `X-Forwarded-For`, `X-Forwarded-Proto`, `X-Real-IP`, etc.
- **Security Headers**: CORS, CSP, HSTS, `X-Frame-Options`, etc.
- **Observability**: `X-Request-ID`, `X-Trace-ID`, `X-Correlation-ID`, etc.
- **Authentication**: `Authorization`, `X-API-Key`, `Cookie`, `X-CSRF-Token`, etc.
- **CDNs/Proxies**: `Via`, `X-Cache`, `CF-*`, etc.
- **Browser Headers**: `User-Agent`, `Accept-*`, `Referer`, etc.

**Industry Standards**:
- **Nginx**: 100 headers (default `large_client_header_buffers`)
- **Apache**: Unlimited (configurable via `LimitRequestFields`)
- **Node.js http**: 2000 headers (default `maxHeadersCount`)
- **Rust hyper**: 100 headers
- **Rust actix-web**: 32 headers

If your service sits behind multiple proxies, load balancers, or in Kubernetes, consider using **128+ headers** to avoid `TooManyHeaders` errors in production.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or https://apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or https://opensource.org/licenses/MIT)

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
