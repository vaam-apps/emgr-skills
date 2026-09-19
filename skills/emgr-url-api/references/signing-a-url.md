# Building a signed URL, end to end

The single most common thing to get wrong, with a vector you can check
yourself.

## The recipe

```python
import hashlib, hmac, base64

def sign(key_hex: str, salt_hex: str, path: str) -> str:
    key  = bytes.fromhex(key_hex)
    salt = bytes.fromhex(salt_hex)
    mac  = hmac.new(key, salt + path.encode(), hashlib.sha256).digest()
    return base64.urlsafe_b64encode(mac).rstrip(b"=").decode()

def encode_source(url: str) -> str:
    return base64.urlsafe_b64encode(url.encode()).rstrip(b"=").decode()
```

Four details, each of which silently produces a `403` if you get it wrong:

1. **`SIGNING_KEY` and `SIGNING_SALT` are hex**, and must be hex-decoded
   before use. They are not the raw strings.
2. **The salt and the path are concatenated with no separator**, salt first.
   This matches imgproxy.
3. **The signed path includes the leading `/` and excludes the signature
   segment.** It is the request path as the server received it, still
   percent-encoded, minus the first segment.
4. **base64url, no padding**, for both the signature and the encoded source.
   Standard base64 will not verify.

## A vector you can check

Using the placeholder key and salt from `.env.example` — hex for
`my-signing-key` and `my-salt`, **never use them for anything real**:

```text
SIGNING_KEY   6d792d7369676e696e672d6b6579
SIGNING_SALT  6d792d73616c74
source        https://images.example.com/photo.jpg
              → aHR0cHM6Ly9pbWFnZXMuZXhhbXBsZS5jb20vcGhvdG8uanBn
signed_path   /rs:fill:300:300/q:80/aHR0cHM6Ly9pbWFnZXMuZXhhbXBsZS5jb20vcGhvdG8uanBn.jpg
signature     de7BKgwO8wFeNZWRWgp3UB9jKwOkVoYM_eMKau2ECgw
```

Giving:

```text
GET /de7BKgwO8wFeNZWRWgp3UB9jKwOkVoYM_eMKau2ECgw/rs:fill:300:300/q:80/aHR0cHM6Ly9pbWFnZXMuZXhhbXBsZS5jb20vcGhvdG8uanBn.jpg
```

**This exact signature is pinned by a test in the source repository** —
`documented_example::docs_worked_example_signature_is_correct` in
`src/modules/signing/verify.rs`, whose own doc comment says it exists so the
README and the API reference "can never silently drift from what this module
actually computes". The Python above was run against it and reproduces it
byte-for-byte (checked 2026-09-19).

So if your implementation disagrees with this vector, your implementation is
wrong — not the vector.

## Following it

```bash
curl -LI 'http://localhost:13001/de7BKgwO8wFeNZWRWgp3UB9jKwOkVoYM_eMKau2ECgw/rs:fill:300:300/q:80/aHR0cHM6Ly9pbWFnZXMuZXhhbXBsZS5jb20vcGhvdG8uanBn.jpg'
```

Expect `301` with `Location: /api/images/files/{key}`; following it returns
`200` and the bytes. **A plain `curl` with no `-L` saves the empty redirect
body, not an image** — which looks like a broken service and is not.

## Presets change the path you sign

The signature covers the path **as sent**, so `pr:thumb` is what you sign —
not its expansion. But remember a preset named `default` is prepended by the
server to every request, invisibly, after verification. It does not affect
what you sign; it does affect what you get.

## The `unsigned` escape

Substituting the literal segment `unsigned` for the signature works **only**
when the deployment sets `ALLOW_UNSIGNED_REQUESTS=true`. Otherwise it is a
`403` whose message names that variable. A deployment with no key configured
at all does not start — see [`emgr-security`](../../emgr-security/).
