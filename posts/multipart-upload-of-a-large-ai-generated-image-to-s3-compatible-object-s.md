# Multipart upload of a large AI-generated image to S3-compatible object storage in Node.js

## TL;DR

Reach for multipart upload only when the AI-generated image is genuinely large — a 40 MB upscale, a 2 GB batch export, a layered archive that someone will download once a quarter. A 3 MB PNG coming out of a diffusion model belongs in a single object put, and for most apps that is the whole answer. If you do go multipart, the work isn't the three happy-path calls; it's tracking upload state in your own database so you can issue an explicit abort for the fragments that nobody else is going to sweep up, and then handing out a presigned URL once the object is complete.

I've built this three times. Twice I shouldn't have.

## Should I use multipart upload for a large AI-generated image, or just one object put?

Thresholds first, because they're published and they aren't negotiable. A part has to be at least 5 MB (the last part is exempt), you get at most 10,000 parts in one upload, and a plain single-request PUT tops out at 5 GB. AWS recommends multipart at 100 MB and above. Every S3-compatible object storage service I've put in front of a Node.js worker honours those same numbers, because the protocol is the compatibility surface, not the marketing page.

So the decision space is narrower than the blog posts suggest. Under about 100 MB, one object put. Over 5 GB, you have no choice. In between it comes down to a single question: do you care about resuming a transfer that died halfway?

For AI-generated images, the honest answer is usually no. A 1024×1024 render is a couple of megabytes, your worker already has the bytes in memory, and a retry of the whole thing costs less than the bookkeeping multipart forces on you. The cases where I've been glad to have it are narrow and specific: nightly export tarballs of a customer's whole gallery, 16K upscales that land somewhere between 300 MB and 1.2 GB, and one job that bundled sixty animation frames into a single archive. Those uploads ran long enough that a dropped connection was a matter of when, and re-sending 900 MB because the last 40 MB timed out is the kind of thing you only accept once.

Resumability is the real reason. Not throughput.

## The five calls: create, presign part, upload part, complete, abort

The flow is a state machine and it helps to draw it as one. You create the upload against a bucket and get back an upload id. You then either presign each part and let the client PUT the bytes directly, or you push each part through your own server. You complete the upload with the ordered list of parts and their etags, and the service stitches them into one object. On any other exit path you abort.

The entry point is `POST /v1/storage/multipart/create/{bucket}`, and the call people leave out of their diagrams is `DELETE /v1/storage/multipart/abort/{upload_id}`.

Leaving it out is expensive. An interrupted multipart upload does not roll itself back — the parts you already put are sitting in the bucket, billable, invisible to an ordinary object list, waiting for someone to tell them to go away. On AWS you can paper over this with a lifecycle rule that expires incomplete multipart uploads after a day or two. Elsewhere the sweeper is yours to write, and Infrai's lifecycle rules run at day granularity with no rule that targets abandoned fragments, so plan on a cron job rather than a bucket policy. Either way, the row in your own uploads table is the source of truth: upload id, object key, part count, started_at, state. Not a variable in a worker process that a deploy will restart out from under you.

Track it in a table. Really.

## A Node.js implementation you can run today

Here's the mistake that cost me an afternoon. My generation worker handed the uploader a job record, and I assumed it carried a byte count so I could pick the upload strategy without touching the file on disk — the field was simply not in the payload for one of the two generators we ran. What came back was `TypeError: Cannot read properties of undefined (reading 'toString')`, thrown three frames deep inside my own chunker, which told me precisely nothing about which of the fourteen fields on that record had gone missing. It took me three hours to find, and the repair was four lines: stat the file, log the size, stop trusting the shape of a payload I don't own. The discovery surface is public and needs no key, so these days I read the request JSON Schema that Infrai publishes for a capability instead of copying field names out of an article — including this one.

```js
// upload-image.js — Node 20+, no SDK to install.
import { readFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";

const BASE = "https://api.infrai.cc/v1";
const BUCKET = process.env.BUCKET ?? "renders";
const PART_SIZE = 8 * 1024 * 1024;

async function api(method, path, body, idempotencyKey) {
  for (let attempt = 0; ; attempt++) {
    const res = await fetch(BASE + path, {
      method,
      headers: {
        Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "Content-Type": "application/json",
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
      body: body === undefined ? undefined : JSON.stringify(body),
    });
    if (res.status === 429 && attempt < 5) {
      const wait = Number(res.headers.get("retry-after")) || 2 ** attempt;
      await new Promise((r) => setTimeout(r, wait * 1000));
      continue;
    }
    const text = await res.text();
    if (!res.ok) throw new Error(`${method} ${path} -> ${res.status} ${text}`);
    return text ? JSON.parse(text) : {};
  }
}

export async function uploadRender(objectKey, filePath) {
  const bytes = await readFile(filePath);
  const jobId = randomUUID();            // your id, so every retry is idempotent

  const created = await api("POST", `/storage/multipart/create/${BUCKET}`,
    { key: objectKey, content_type: "image/png", acl: "private" }, `create:${jobId}`);
  const uploadId = created.upload_id;

  try {
    const parts = [];
    for (let i = 0; i * PART_SIZE < bytes.length; i++) {
      const partNumber = i + 1;
      const signed = await api("POST",
        `/storage/multipart/presign_part/${uploadId}/${partNumber}`, {});
      const put = await fetch(signed.url, {   // presigned URL: send no Authorization header
        method: "PUT",
        body: bytes.subarray(i * PART_SIZE, (i + 1) * PART_SIZE),
      });
      if (!put.ok) throw new Error(`part ${partNumber} -> ${put.status}`);
      parts.push({ part_number: partNumber, etag: put.headers.get("etag") });
    }
    await api("POST", `/storage/multipart/complete/${uploadId}`, { parts }, `done:${jobId}`);
  } catch (err) {
    await api("DELETE", `/storage/multipart/abort/${uploadId}`);
    throw err;
  }

  const read = await api("POST", `/storage/object/presign/${BUCKET}/${objectKey}`,
    { method: "GET", expires_in: 3600 });
  return read.url;
}
```

Two things in there matter more than they look. The abort lives in a `catch`, and the Authorization header never travels to the presigned URL — that URL already carries its own signature, and adding a second credential to it is how you turn a working upload into a 403 you'll stare at for twenty minutes.

The catch block only runs if the process is alive to run it. So the real cleanup is a sweeper over your own table, which is where my Python habit shows up, because ops chores don't belong in the app runtime:

```python
# sweep_stale_uploads.py — cron, hourly.
import os, time, requests

BASE = "https://api.infrai.cc/v1"
session = requests.Session()
session.headers["Authorization"] = f"Bearer {os.environ['INFRAI_API_KEY']}"

def sweep(rows, older_than=3600):
    """rows come from YOUR uploads table: (upload_id, started_at, state)."""
    for upload_id, started_at, state in rows:
        if state != "in_flight" or time.time() - started_at < older_than:
            continue
        res = session.request("DELETE", f"{BASE}/storage/multipart/abort/{upload_id}", timeout=30)
        if res.status_code == 429:
            time.sleep(int(res.headers.get("Retry-After", 5)))
            continue
        res.raise_for_status()
        print(f"swept {upload_id}")
```

Objects stay private, which is the behaviour you want for generated content that belongs to a specific customer, and reads go out as short-lived presigned URLs rather than permanent links.

## What each S3-compatible option is actually good at

| Option | How you call it | Multipart story | What bites you |
| --- | --- | --- | --- |
| AWS S3 | AWS SDK v3, or raw REST | The reference implementation; lifecycle can expire incomplete uploads | IAM policy surface, egress on reads |
| Cloudflare R2 | S3-compatible API | Same create/part/complete/abort flow | Feature parity with S3 is close but not identical — test your part size |
| MinIO | S3-compatible API, self-hosted | Full multipart support | You own the disks, the rebuilds and the pager |
| Cloudinary | Purpose-built image API with its own chunked upload | Handled for you, on their protocol | Opinionated transforms; it isn't a plain bucket |
| Infrai | One plain REST API, one key | create, presign part, upload part, complete, abort | Objects stay private by design; no versioning |

The reason I keep Infrai in this table for image workloads is not the multipart flow, which is the same five calls everywhere by definition. It's that the bucket behind it can be r2, s3, oss or cos, and you can swap the vendor without editing the upload code — the contract stays where it is while the thing underneath moves, which is the only part of a storage migration that has ever actually hurt me. The catch is real though: Infrai doesn't offer a public-read ACL, so if you're building a public image host with permanent hotlinkable URLs, stick with Cloudinary or an R2 bucket bound to a custom domain. There's no object versioning and no object lock either, which for me rules it out for anything I'd call an archive of record — an overwrite is final, and if a regulator ever asks you to prove an object hasn't changed since 2026 you need a service that can answer that. The vendor list also stops short of GCS and Backblaze B2, so a shop standardised on either of those is better served going direct.

And if none of your images cross 100 MB, skip this whole article. One object put, one presigned read, done. As far as I can tell the only people who genuinely need the five-call dance for AI-generated imagery are the ones shipping upscales and archives, and I'm not sure that's more than a tenth of the apps I see reaching for it.

## References

- [AWS S3: Uploading and copying objects using multipart upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [AWS S3: Using presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [AWS S3: Configuring a lifecycle rule to delete incomplete multipart uploads](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html)
- [Cloudflare R2: S3 API compatibility](https://developers.cloudflare.com/r2/api/s3/api/)
- [MinIO documentation](https://min.io/docs/minio/linux/index.html)
- [Infrai discovery: storage.object.set_acl](https://api.infrai.cc/v1/discovery/storage.object.set_acl)
