# Plan B: ETag, Compression, and URL Digests for Embedded Assets (Simple + LazyLock)

## Goals
1. ETag handling for cache validation.
2. Compression (gzip or gzip+br).
3. URL digests for long-lived caching, without breaking unhashed URLs.

## Key Choices (Simplicity First)
- Use rust-embed's built-in SHA-256 metadata for ETags and digests.
- Use `std::sync::LazyLock` to build an in-memory manifest once at startup.
- Rewrite `index.html` once to reference digested asset URLs.
- Avoid new dependencies by using a tiny local hex encoder.

## High-Level Approach
- Build a static manifest of embedded assets when the server starts.
- Serve assets through a single handler that supports:
  - ETag checks
  - digest and non-digest URLs
  - cache-control based on URL type
- Add `CompressionLayer` to the router.

## Implementation Details

### 1. Manifest built with LazyLock
Create a manifest that indexes all embedded files and precomputes:
- Raw bytes
- MIME type
- ETag (SHA-256 hex)
- Digest URL (insert hash into filename)
- Reverse lookup for digest URLs
- Pre-rewritten `index.html`

Sketch:
```rust
use std::collections::HashMap;
use std::sync::LazyLock;
use bytes::Bytes;

struct AssetInfo {
    bytes: Bytes,
    mime: &'static str,
    etag: String,
    digest_path: Option<String>,
}

struct AssetManifest {
    by_path: HashMap<String, AssetInfo>,
    by_digest: HashMap<String, String>,
    index_html: Bytes,
}

static MANIFEST: LazyLock<AssetManifest> = LazyLock::new(|| {
    let mut by_path = HashMap::new();
    let mut by_digest = HashMap::new();

    for path in Assets::iter() {
        if let Some(file) = Assets::get(&path) {
            let bytes = Bytes::from(file.data.into_owned());
            let mime = mime_from_path(&path);
            let hash_hex = hex_encode(file.metadata.sha256_hash());
            let etag = format!("\"{}\"", hash_hex);
            let digest_path = digest_path(&path, &hash_hex);

            if let Some(d) = &digest_path {
                by_digest.insert(d.clone(), path.to_string());
            }

            by_path.insert(
                path.to_string(),
                AssetInfo {
                    bytes,
                    mime,
                    etag,
                    digest_path,
                },
            );
        }
    }

    let index_html = rewrite_index_html(&by_path);

    AssetManifest {
        by_path,
        by_digest,
        index_html,
    }
});
```

### 2. Hex encoding (no new crate)
Add a small helper:
```rust
fn hex_encode(bytes: &[u8]) -> String {
    const LUT: &[u8; 16] = b"0123456789abcdef";
    let mut out = String::with_capacity(bytes.len() * 2);
    for &b in bytes {
        out.push(LUT[(b >> 4) as usize] as char);
        out.push(LUT[(b & 0x0f) as usize] as char);
    }
    out
}
```

### 3. Digest URL format
Insert the hash before the extension, e.g.:
- `asciinema-player.css` -> `asciinema-player.<hash>.css`

Skip digesting `index.html`.

```rust
fn digest_path(path: &str, hash: &str) -> Option<String> {
    if path == "index.html" {
        return None;
    }
    match path.rsplit_once('.') {
        Some((base, ext)) => Some(format!("{base}.{hash}.{ext}")),
        None => Some(format!("{path}.{hash}")),
    }
}
```

### 4. Rewrite index.html once
Replace asset references with digested versions using the manifest:
```rust
fn rewrite_index_html(by_path: &HashMap<String, AssetInfo>) -> Bytes {
    let Some(asset) = by_path.get("index.html") else {
        return Bytes::new();
    };

    let mut html = String::from_utf8_lossy(&asset.bytes).into_owned();

    for (path, info) in by_path {
        if let Some(digest) = &info.digest_path {
            html = html.replace(path, digest);
        }
    }

    Bytes::from(html)
}
```

### 5. Static handler changes
- Accept `HeaderMap` to read `If-None-Match`.
- Resolve digest URLs via the reverse map.
- Serve rewritten `index.html` from manifest.
- Add ETag and Cache-Control headers.
- Return `304` when ETag matches.

Cache-Control rules:
- Digest URL: `public, max-age=31536000, immutable`
- Non-digest: `public, max-age=0, must-revalidate`

Add `Vary: Accept-Encoding` to all asset responses.

### 6. Compression
Add `CompressionLayer` and enable the required features in `tower-http`.

Cargo.toml:
```toml
tower-http = { version = "0.6", features = ["trace", "compression-gzip"] }
```

Server router:
```rust
use tower_http::compression::CompressionLayer;

let app = Router::new()
    .route("/ws", get(ws_handler))
    .with_state(state)
    .fallback(static_handler)
    .layer(CompressionLayer::new())
    .layer(trace);
```

## Verification
- ETag:
  - `curl -I http://localhost:port/asciinema-player.css` includes `ETag`
  - `curl -H 'If-None-Match: "<etag>"' -I ...` returns 304
- Compression:
  - `curl -H 'Accept-Encoding: gzip' --compressed ...` includes `Content-Encoding: gzip`
- Digest URLs:
  - `index.html` contains `asciinema-player.<hash>.css`
  - Digest URL returns asset with `Cache-Control: immutable`
  - Non-digest URL still works
