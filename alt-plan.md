# Plan: ETag, Compression, and URL Digests for Embedded Assets

## Current State
- `src/server.rs` serves embedded assets from `assets/` folder using rust-embed
- Simple `static_handler` returns files with MIME type only
- No caching headers, no compression, no content hashing

## Goals
1. **ETag handling** - Enable browser caching with conditional requests
2. **Compression** - Gzip/brotli for text assets
3. **URL digests** - Content-hashed URLs for aggressive caching

## Design Decisions
- **Fallback URLs**: Both hashed and non-hashed URLs work (hashed get immutable caching)
- **Rust version**: 1.80+ (use `std::sync::LazyLock`, no `once_cell` dependency)

---

## Implementation Plan

### 1. ETag Handling

**Approach**: rust-embed provides a `hash()` method on embedded files returning the file's SHA256 hash. Use this as the ETag value.

**Changes to `src/server.rs`**:
- Check `If-None-Match` request header against file hash
- Return `304 Not Modified` when ETag matches
- Add `ETag` header to all asset responses
- Add `Cache-Control` header with appropriate directives

```rust
async fn static_handler(
    headers: HeaderMap,  // Add header extraction
    uri: Uri,
) -> impl IntoResponse {
    // ... get file ...

    let etag = format!("\"{}\"", hex::encode(content.metadata.sha256_hash()));

    // Check If-None-Match
    if let Some(if_none_match) = headers.get(header::IF_NONE_MATCH) {
        if if_none_match.to_str().ok() == Some(&etag) {
            return StatusCode::NOT_MODIFIED.into_response();
        }
    }

    // Return with ETag and Cache-Control headers
    (
        [
            (header::CONTENT_TYPE, mime),
            (header::ETAG, &etag),
            (header::CACHE_CONTROL, "public, max-age=0, must-revalidate"),
        ],
        content.data,
    ).into_response()
}
```

**Dependencies**: Add `hex` crate for encoding hash to string.

---

### 2. Compression

**Approach**: Use tower-http's `CompressionLayer` middleware.

**Changes to `Cargo.toml`**:
```toml
tower-http = { version = "0.6", features = ["trace", "compression-gzip", "compression-br"] }
```

**Changes to `src/server.rs`**:
```rust
use tower_http::compression::CompressionLayer;

let app = Router::new()
    .route("/ws", get(ws_handler))
    .with_state(state)
    .fallback(static_handler)
    .layer(CompressionLayer::new())  // Add compression
    .layer(trace);
```

This automatically compresses responses based on `Accept-Encoding` header. Works for HTML, CSS, JS.

---

### 3. URL Digests (Content-Hashed URLs)

**Approach**: Generate hashed asset URLs at startup and serve index.html with placeholders replaced.

**Option A: Build-time hashing** (Recommended)
- At build time or app startup, compute short hashes for each asset
- Store a mapping: `asciinema-player.css` → `asciinema-player.a1b2c3d4.css`
- Modify index.html content in memory, replacing asset references
- Route hashed URLs back to original files, stripping the hash
- Serve hashed URLs with `Cache-Control: public, max-age=31536000, immutable`

**Implementation**:

1. Create asset manifest at startup:
```rust
use std::collections::HashMap;
use std::sync::LazyLock;

struct AssetManifest {
    // original_path -> (hashed_path, hash)
    mapping: HashMap<String, (String, String)>,
    // hashed_path -> original_path (reverse lookup)
    reverse: HashMap<String, String>,
    // Pre-processed index.html with hashed URLs
    index_html: Vec<u8>,
    index_etag: String,
}

static MANIFEST: LazyLock<AssetManifest> = LazyLock::new(|| {
    let mut manifest = AssetManifest::default();
    for path in Assets::iter() {
        if let Some(file) = Assets::get(&path) {
            let hash = &hex::encode(file.metadata.sha256_hash())[..8];
            let hashed_path = insert_hash_in_filename(&path, hash);
            manifest.mapping.insert(path.to_string(), (hashed_path.clone(), hash.to_string()));
            manifest.reverse.insert(hashed_path, path.to_string());
        }
    }
    manifest
});

fn insert_hash_in_filename(path: &str, hash: &str) -> String {
    // "asciinema-player.css" -> "asciinema-player.a1b2c3d4.css"
    if let Some(dot_pos) = path.rfind('.') {
        format!("{}.{}{}", &path[..dot_pos], hash, &path[dot_pos..])
    } else {
        format!("{}.{}", path, hash)
    }
}
```

2. Modify index.html serving to replace asset URLs:
```rust
fn get_index_html() -> Vec<u8> {
    let mut html = String::from_utf8_lossy(&Assets::get("index.html").unwrap().data).to_string();
    for (original, (hashed, _)) in &MANIFEST.mapping {
        if original != "index.html" {
            html = html.replace(original, hashed);
        }
    }
    html.into_bytes()
}
```

3. Update static_handler to resolve hashed paths:
```rust
async fn static_handler(headers: HeaderMap, uri: Uri) -> impl IntoResponse {
    let mut path = uri.path().trim_start_matches('/');

    if path.is_empty() {
        path = "index.html";
    }

    // Check if this is a hashed URL and resolve to original
    let (resolved_path, is_hashed) = if let Some(original) = MANIFEST.reverse.get(path) {
        (original.as_str(), true)
    } else {
        (path, false)
    };

    // Special handling for index.html (with replaced URLs)
    if resolved_path == "index.html" {
        let html = get_processed_index_html();
        let etag = /* compute or cache */;
        return (
            [
                (header::CONTENT_TYPE, "text/html"),
                (header::ETAG, etag),
                (header::CACHE_CONTROL, "public, max-age=0, must-revalidate"),
            ],
            html,
        ).into_response();
    }

    match Assets::get(resolved_path) {
        Some(content) => {
            let mime = mime_from_path(resolved_path);
            let etag = format!("\"{}\"", &hex::encode(content.metadata.sha256_hash())[..16]);

            // Hashed URLs get immutable caching
            let cache_control = if is_hashed {
                "public, max-age=31536000, immutable"
            } else {
                "public, max-age=0, must-revalidate"
            };

            // ETag check...

            (
                [
                    (header::CONTENT_TYPE, mime),
                    (header::ETAG, &etag),
                    (header::CACHE_CONTROL, cache_control),
                ],
                content.data,
            ).into_response()
        }
        None => (StatusCode::NOT_FOUND, "404").into_response(),
    }
}
```

---

## Files to Modify

1. **`Cargo.toml`**
   - Add `hex` dependency (for encoding SHA256 hash to string)
   - Add compression features to tower-http: `compression-gzip`, `compression-br`

2. **`src/server.rs`**
   - Add `AssetManifest` struct with `LazyLock` initialization
   - Update `static_handler` to:
     - Accept `HeaderMap` for `If-None-Match` checking
     - Resolve hashed URLs via manifest reverse lookup
     - Return `304 Not Modified` on ETag match
     - Add `ETag`, `Cache-Control` headers to responses
     - Serve processed index.html with hashed asset URLs
   - Add `CompressionLayer` to router middleware stack

---

## Verification

1. **ETag**:
   - `curl -I http://localhost:port/asciinema-player.css` shows `ETag` header
   - `curl -H "If-None-Match: \"<etag>\"" -I ...` returns 304

2. **Compression**:
   - `curl -H "Accept-Encoding: gzip" --compressed ...` shows smaller response
   - Check `Content-Encoding: gzip` header

3. **URL digests**:
   - View source of index.html shows hashed URLs like `asciinema-player.a1b2c3d4.css`
   - Request hashed URL returns file with `Cache-Control: immutable`
   - Request non-hashed URL still works (for backwards compatibility)
