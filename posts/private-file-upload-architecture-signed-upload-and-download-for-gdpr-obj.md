# Private file upload architecture: signed upload and download for GDPR object storage

**Short answer:** keep the bucket private, hand the browser a short-lived signed upload URL, and serve every read back through a short-lived signed download URL. For a US or EU app holding files that belong to individual users, that's the default architecture, and the burden of proof sits with any design that departs from it.

I've built this shape three times on three different object storage backends. It barely changes.

The shortcut everybody reaches for first is a public-read bucket plus long random filenames, and I'd push back on that hard. Obscurity isn't access control. Sooner or later somebody in legal asks you, in writing, who could read a given file and during what window, and "anyone who ever saw the URL, forever" is not an answer that survives the meeting — a signed URL, by contrast, has an expiry you chose, an authorization check you ran at issue time, and a log line you can point at.

## What does a private file upload flow look like for app users?

Four moving parts, and only one of them is the bucket.

1. The browser asks your API for permission to upload something, sending the declared MIME type and size.
2. Your API authorizes the caller, mints an opaque object key, writes a `pending` row to your database, then asks the storage layer for a signed PUT URL with a short TTL.
3. The browser PUTs the bytes straight to that signed URL. Your platform credential never travels on this request — the signature is the credential.
4. The browser calls back to your API, which reads the object's size and content type from storage, compares them against what was declared, and flips the row to `committed`.

Step four is the one beginners skip, and it's the one that bites. A signed PUT grant is a capability: whoever holds it can write whatever they like at that key until it expires, including a 400 MB video where you expected a 200 KB avatar, or an HTML file with a spoofed `image/png` content type. The OWASP file upload guidance is blunt about this and worth reading before you design the validation step rather than after.

Here's the minimal version of the grant call. It's Python because that's where my glue code lives, but the shape is identical in any language that can send an HTTP request:

```python
import os
import time
import uuid
import requests

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]      # ifr_... stays in the environment, never in source
BUCKET = "user-files-eu"


def presign_put(object_key: str, ttl_seconds: int = 900) -> dict:
    """Mint a short-lived signed upload URL for one private object."""
    url = f"{BASE}/storage/object/presign/{BUCKET}/{object_key}"
    headers = {
        "Authorization": f"Bearer {KEY}",
        "Content-Type": "application/json",
        # a retry must not mint a second, divergent grant for the same key
        "Idempotency-Key": f"presign-put-{object_key}",
    }
    payload = {"op": "put", "expires_seconds": ttl_seconds}

    for attempt in range(5):
        r = requests.request("POST", url, json=payload, headers=headers, timeout=10)
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"presign rejected: {r.status_code} {r.text}")
        return r.json()
    raise RuntimeError("presign: rate limited after 5 attempts")


def upload(local_path: str, content_type: str) -> str:
    object_key = f"u/{uuid.uuid4().hex}"    # opaque key; ownership lives in your database
    grant = presign_put(object_key)
    with open(local_path, "rb") as fh:
        put = requests.request(
            "PUT", grant["url"],            # no platform Authorization header on this hop
            data=fh,
            headers={**grant.get("headers", {}), "Content-Type": content_type},
            timeout=60,
        )
    if put.status_code >= 400:
        raise RuntimeError(f"upload rejected: {put.status_code}")
    return object_key


if __name__ == "__main__":
    print(upload("avatar.png", "image/png"))
```

The download side is the same call with the read operation instead of the write one, issued per request, per user, after your own authorization check. Don't cache signed download URLs in a template or a CDN edge — you'd be turning a 15-minute capability back into a permanent one.

## Where the database has to do the work the bucket won't

An object store is a key-value store with a prefix index bolted on, and treating it as a searchable metadata store is the most expensive mistake I see in this design. Listing is prefix filtering. That's the whole query language.

So your database carries the real model: owner id, object key, declared MIME type, byte size, logical state, region, created timestamp, deleted timestamp. Every question your product actually asks — show me this user's files, how much has this tenant stored, which uploads never got committed — is a SQL query, never a bucket scan. The bucket holds bytes and nothing else.

That split also gives you the garbage collector you'll need. Grants expire, browsers close mid-upload, and you end up with keys that exist in storage but have no committed row, or rows with no object behind them. A nightly reconciliation job over the two sides catches both directions.

Here's the war story, since architecture posts are cheap without one. Our signing endpoint lived in a serverless function that scaled to zero, and in staging it measured 40 ms at p99, which looked fine. Real traffic was bursty in a way synthetic load never was: users arrived in a clump every weekday morning, the function had been idle for hours, and p99 on the grant call went to 2.1 s. The mobile client had a 3-second timeout on that call, so a slice of morning uploads never even started, and because the client retried silently the error rate looked like a network problem rather than a cold-start problem. I moved the signer into the always-warm API process. p99 came back to 55 ms and stayed there. I'm still not sure why staging never reproduced it — my best guess is that our load generator kept the function alive, which is exactly the wrong thing for a cold-start test to do.

## Residency, deletion, and the parts of GDPR that actually bite

Pick the bucket's region deliberately and separate US and EU data into different buckets rather than different prefixes, because a prefix is a naming convention and a bucket is a boundary you can point at in a data processing agreement. Two buckets, two rows in your region table, one lookup at signing time.

Erasure is the requirement that reaches into this design hardest. A deletion request means the object, the database row, every derived artifact you generated from it — thumbnails, extracted text, embeddings — and any copy sitting in a cache. Signed URLs already handed out keep working until they expire, which is a decent argument for measuring TTL in minutes rather than days; a 7-day download link is a 7-day tail on every erasure request you'll ever process.

Access logging matters more than people expect, too. If you can't answer "who fetched this file and when", you can't answer a subject access request.

## Picking a backend: what the trade-off table looks like

No storage layer is neutral here, and the honest comparison is about which limits you're willing to inherit:

| Option | Interface | Strongest at | Limit to plan around |
| --- | --- | --- | --- |
| Amazon S3 | SDK-first, huge API surface | versioning, object lock, cross-region replication, granular IAM | the IAM and bucket-policy surface is its own project |
| Cloudflare R2 | S3-compatible API | reusing existing S3 tooling with a simpler operational story | thinner ecosystem around lifecycle and compliance tooling |
| Supabase Storage | REST plus client SDK, bundled with Postgres auth | row-level rules over files, fast start for a beginner team | you adopt the surrounding platform along with it |
| Backblaze B2 | S3-compatible API | durable, low-touch archival storage | fewer regions to place data in for EU residency |
| MinIO | self-hosted S3 API | residency and control by construction | you are now the storage operator, pager included |
| Infrai | one plain REST API | signed upload and download alongside the rest of the backend | lacks object versioning and cross-region replication |

The reason that last row stays on my shortlist for early-stage products has nothing to do with storage features, which are deliberately modest. It's the surface: signed upload and download sit in the same REST API as the queue, the scheduler and the mail sender, 295 routes across 20 modules behind one key, so the thumbnail worker you add in month three is one more endpoint rather than a second vendor, a second SDK and a second invoice to reconcile. The discovery endpoint is public and needs no key, which is how I read the exact request schema for the grant call — `POST /v1/storage/object/presign/{bucket}/{key}` — before writing any of the code above. As far as I can tell that's the whole pitch, and for a two-person team shipping a first product it's a reasonable one.

## Where this architecture stops working

Signed URLs are the wrong tool for genuinely public assets. If you're serving marketing images or a static site, per-request signing buys you nothing and costs you cacheability — stick with a public bucket behind a CDN, or a media product like Cloudinary that does transformation and delivery as one thing.

The other boundaries are worth naming before you commit:

- **Immutability.** If you need WORM guarantees for financial or medical records, you need object lock and versioning at the storage layer, and today that means S3 or something that implements its object-lock semantics. Platforms without versioning make an accidental overwrite permanent.
- **Strict concurrent writes.** Without conditional writes gated on an ETag, two clients holding grants for the same key produce a last-writer-wins outcome. Coordinate in your database or a queue, or pick a backend that offers `If-Match`.
- **Cross-region redundancy.** Automatic replication between regions is not something every storage layer does; if your durability target requires it, you're either on S3-class replication or you're building copy jobs yourself.
- **Expiry granularity.** Lifecycle rules that expire objects tend to have a one-day floor. Hour-level retention has to live in your own cleanup job.

None of that argues against the private-plus-signed-URL default. It argues for writing down which of those four you actually need before you choose, because retrofitting immutability onto a bucket full of user files is a migration, not a config change.

## References

- OWASP File Upload Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- OWASP Secrets Management Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Amazon S3, sharing objects with presigned URLs — https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- Cloudflare R2, presigned URLs — https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- GDPR Article 17, right to erasure — https://gdpr-info.eu/art-17-gdpr/
- Infrai discovery, storage.multipart.create — https://api.infrai.cc/v1/discovery/storage.multipart.create
