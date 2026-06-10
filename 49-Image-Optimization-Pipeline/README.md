# On-the-Fly Image Optimization With S3, Lambda Sharp, and CloudFront: AVIF/WebP 60-80% Lighter Without a SaaS Bill

One prompt generates a production-ready transform pipeline — originals in S3, sharp on Lambda, format negotiation at the edge — so every image is transformed exactly once, served millions of times, and your image-CDN SaaS invoice goes to zero.

## The Problem

Images are 40-60% of the bytes on a typical product page, and most startups serve full-size JPEGs straight out of S3: a 2.8 MB hero photo where a 960 px, 280 KB AVIF was needed. The usual fixes are bad trades:

- **Image-CDN SaaS**: solves it, but pricing scales with transformations and bandwidth — typically $300-500/month by 10M image views, for math your own account does for under $10.
- **Pre-generating every variant at upload**: 1M originals x 6 widths x 3 formats = 18M objects — roughly $144 of arm64 Lambda compute plus $90 of S3 PUTs ($0.005 per 1,000), spent mostly on variants nobody requests; long-tail catalogs see under 20% of images viewed in a month.
- **Naive DIY attempts**: forwarding the `Accept` header into the cache key fragments the cache into dozens of entries per image, or sharp ships compiled for macOS and throws `Could not load the "sharp" module using the linux-arm64 runtime` at 2 AM.

The right architecture transforms on first request, writes the variant back to S3, and lets CloudFront absorb everything after. This prompt encodes the four details that make or break it: the resize-on-request vs pre-generate decision, cache-safe Accept-header negotiation, origin-group failover wiring, and Lambda memory sizing for sharp.

## Who This Is For

Startups and solo developers serving catalog or user-uploaded images (e-commerce, marketplaces, UGC, media) who want Core Web Vitals wins and an image bill that rounds to zero — without adopting another vendor. Comfortable with Node.js; CloudFront experience helpful but not required.

## How to Use

1. Open your AI coding assistant (Kiro CLI, Claude Code, or Amazon Q Developer CLI) in an empty project directory.
2. Copy the System Prompt below into the chat (or save it as `image-pipeline.md` and reference it).
3. Replace the bracketed placeholders: `[ORIGINALS_BUCKET]`, `[AWS_REGION]`, `[MONTHLY_IMAGE_VIEWS]`.
4. Let the assistant run its reality checks first (credentials, bucket existence, Node/CDK versions); adjust the width allow-list if your frontend uses non-default `srcset` breakpoints.
5. Deploy with `npm install && npx cdk deploy`, then run `scripts/verify.sh` and keep its output — that is your acceptance evidence.

**Prerequisites**

- **Required Access:** AWS account with permissions to create CloudFront distributions, Lambda functions, S3 buckets, IAM roles, and CloudWatch alarms; CDK bootstrapped (`npx cdk bootstrap aws://ACCOUNT_ID/us-east-1`).
- **Recommended Background:** Basic Node.js; responsive images (`srcset`/`sizes`); CloudFront caching concepts.
- **Tools Required:** AWS CLI v2, Node.js 20+, AWS CDK v2, an agentic assistant with the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) so it verifies sharp packaging and OAC support against live docs, not memory.

**Key Parameters:** Width allow-list (320/640/960/1280/1920/2560 px), AVIF quality 60 effort 3, WebP quality 80, JPEG quality 80, Lambda 1536 MB arm64 / 30 s timeout, transformed-object `Cache-Control: public, max-age=31536000, immutable`, transformed-bucket lifecycle expiry 90 days, origin-group failover on 403/404/500/502/503/504.

**Troubleshooting:** If the first deploy fails at runtime with `Could not load the "sharp" module using the linux-arm64 runtime`, it's expected when sharp was installed on your laptop for your laptop — the function package must be built with `npm install --os=linux --cpu=arm64 sharp` so the Linux binary ships in the bundle.

## System Prompt

```
# On-the-Fly Image Optimization Pipeline — Architecture & Implementation Request

## Project Overview
Build a production-ready, on-demand image optimization pipeline on AWS with zero third-party SaaS. Architecture: originals in S3 -> CloudFront -> CloudFront Function (format negotiation + URL normalization) -> origin group (transformed S3 bucket primary, Lambda function URL failover) -> sharp transform -> write-back to S3 + inline response. Each variant is computed exactly once, then served from cache for its lifetime. Region: [AWS_REGION]. Originals bucket: [ORIGINALS_BUCKET]. Expected traffic: [MONTHLY_IMAGE_VIEWS] image views/month.

## Reality Checks — Run These BEFORE Generating Anything
1. Verify credentials: `aws sts get-caller-identity`. Stop if unauthenticated.
2. Verify `[ORIGINALS_BUCKET]` exists via `aws s3api head-bucket`; if absent, plan its creation and say so.
3. Verify Node.js >= 20, AWS CDK v2 installed, and CDK bootstrap in [AWS_REGION].
4. Use aws_read_documentation to confirm current guidance for packaging sharp for Lambda and that CloudFront Origin Access Control supports Lambda function URLs. Never invent flags or APIs; if unverifiable, omit.
5. If any check fails, report Cause -> Resolution and halt. No partial infrastructure.

## Detailed Requirements

### 1. Transform Function
- Lambda: Node.js 22, arm64 (Graviton), 1536 MB memory, 30 s timeout, function URL with AWS_IAM auth behind CloudFront OAC.
- Package sharp with `npm install --os=linux --cpu=arm64 sharp` so the linux-arm64 binary ships in the bundle.
- Flow: parse normalized path (`/<key>/w=<width>,f=<format>`), GET original from [ORIGINALS_BUCKET], resize + encode, PUT to the transformed bucket with correct Content-Type and `Cache-Control: public, max-age=31536000, immutable`, return the image inline (base64).
- Width allow-list: 320, 640, 960, 1280, 1920, 2560. Snap requests up to the nearest value; return 400 for anything else (prevents cache-busting abuse).
- Encoding: AVIF quality 60 effort 3, WebP quality 80, JPEG quality 80 with mozjpeg. Auto-rotate from EXIF orientation, then strip all metadata (EXIF/GPS). Keep sharp's default limitInputPixels guard (~268 megapixels) and reject non-image content types.
- Respect the 6 MB buffered-response limit: keep encoded output under 4.5 MB binary; log a warning if exceeded.

### 2. Format Negotiation via Accept Header
- CloudFront Function (runtime cloudfront-js-2.0, viewer-request, under the 10 KB size limit): inspect `Accept` — `image/avif` -> avif, else `image/webp` -> webp, else jpeg. Check avif BEFORE webp; browsers send both.
- Rewrite `/cats/1.jpg?w=900` -> `/cats/1.jpg/w=960,f=avif` and strip the query string.
- NEVER put the Accept header in the cache key. Normalize format into the path so each variant is exactly one cache entry. Cache policy: no headers, no cookies, no query strings; min TTL 86400, default and max TTL 31536000.

### 3. Caching and Origin Failover
- Origin group: primary = transformed S3 bucket via OAC; failover to the Lambda function URL on status 403, 404, 500, 502, 503, 504 (S3 returns 403 for missing keys without ListBucket — include it).
- Lifecycle rule: expire transformed objects after 90 days; cold variants regenerate on demand.

### 4. Cost and Performance Targets
- Lambda compute < $10/month at 1M unique transforms: arm64 at $0.0000133334/GB-s means a 400 ms transform at 1536 MB costs ~$0.000008, ≈ $8.20 per million including request charges.
- Steady-state cache hit ratio >= 95%; p95 cache-hit latency < 100 ms; p95 first-transform < 1.5 s.
- README must include: (a) the resize-on-request vs pre-generate decision table — pre-generate only when the catalog is small (<5,000 originals) with a fixed variant matrix; on-request wins for long-tail catalogs (18 variants x 1M originals ≈ $234 up front vs ~$22 on demand for the ~15% actually viewed); (b) a Lambda memory-sizing table for sharp (12 MP JPEG -> 960 px): ~1.2 s @ 512 MB, ~650 ms @ 1024 MB, ~450 ms @ 1536 MB, ~400 ms @ 1769 MB (= 1 full vCPU; little gain above). Note that cost-per-transform stays nearly flat as memory doubles and duration halves — extra memory buys latency, not cost. Recommend AWS Lambda Power Tuning for the user's own corpus.

### 5. Security and Operations
- Both buckets private with BlockPublicAccess ALL; access only via CloudFront OAC.
- Lambda execution role: s3:GetObject on originals/* and s3:PutObject on transformed/* only. Grant lambda:InvokeFunctionUrl solely to cloudfront.amazonaws.com conditioned on this distribution's AWS:SourceArn.
- CloudWatch alarms: Lambda Errors > 5 per 5 min, p95 Duration > 3000 ms, CloudFront 5xxErrorRate > 1%, CacheHitRate < 80%.

## Deliverables Requested
1. AWS CDK (TypeScript) app — one stack: both buckets, Lambda + function URL + OAC, CloudFront Function, cache policy, origin group, distribution, lifecycle rules, alarms.
2. `functions/url-rewrite/index.js` — the CloudFront Function.
3. `functions/transform/index.mjs` + `package.json` — the sharp handler with packaging script.
4. `scripts/verify.sh` — uploads a test JPEG, requests it with three Accept headers, asserts Content-Type per format, asserts `x-cache: Hit from cloudfront` on the second request, prints byte savings.
5. `README.md` — decision table, memory-sizing table, monthly cost model at [MONTHLY_IMAGE_VIEWS], rollback notes (`cdk destroy`; buckets retained).

## Acceptance Evidence
After deploy, run scripts/verify.sh and paste its full output. Pass = AVIF response >= 50% smaller than the original, second request served from cache, all alarms in OK state. Output files and code only, without any preamble. The stack must be deployable in under 15 minutes.
```

## What You Get

1. **CDK TypeScript stack** — both private S3 buckets, Lambda (Node.js 22, arm64, 1536 MB) with OAC-secured function URL, CloudFront distribution with origin-group failover, custom cache policy, 90-day lifecycle rule, four CloudWatch alarms.
2. **`functions/url-rewrite/index.js`** — CloudFront Function (cloudfront-js-2.0) for Accept-header negotiation and width snapping.
3. **`functions/transform/index.mjs`** — sharp handler with S3 write-back, metadata stripping, and the linux-arm64 packaging script.
4. **`scripts/verify.sh`** — end-to-end acceptance test producing the evidence output.
5. **`README.md`** — resize-on-request vs pre-generate decision table, sharp memory-sizing table, cost model, rollback notes.

## Example Output

```
$ ./scripts/verify.sh
Uploading test/photo.jpg (2,841 KB, 4032x3024) to s3://acme-originals/test/photo.jpg
GET https://d1a2b3c4d5e6f7.cloudfront.net/test/photo.jpg?w=900
  Accept: image/avif,image/webp  -> 200 image/avif   312 KB  (89.0% smaller)  x-cache: Miss from cloudfront  1,184 ms
  Accept: image/webp             -> 200 image/webp   421 KB  (85.2% smaller)  x-cache: Miss from cloudfront    488 ms
  Accept: image/jpeg             -> 200 image/jpeg   694 KB  (75.6% smaller)  x-cache: Miss from cloudfront    402 ms
Repeat AVIF request               -> 200 image/avif   312 KB  x-cache: Hit from cloudfront   23 ms
Transformed objects written: 3. Alarms: 4/4 OK. PASS
```

## AWS Services Used

Amazon CloudFront, CloudFront Functions, AWS Lambda, Amazon S3, Amazon CloudWatch, AWS IAM, AWS CDK

## Well-Architected Alignment

- **Performance Efficiency:** Right-sized 1536 MB arm64 Lambda (data-driven via the sizing table and Lambda Power Tuning); edge caching turns a 400 ms transform into a 23 ms hit.
- **Cost Optimization:** Transform-once economics; Graviton pricing; 90-day lifecycle eviction of cold variants; the decision table prevents paying for 18M never-viewed pre-generated objects; the CloudFront always-free tier (1 TB egress, 10M requests, 2M function invocations) absorbs early traffic.
- **Reliability:** Origin-group failover means a Lambda outage degrades to cache-only serving, not a hard failure; the transformed bucket is fully reconstructable state.
- **Security:** Both origins locked behind OAC; least-privilege Lambda role; `lambda:InvokeFunctionUrl` restricted to the distribution ARN; EXIF/GPS stripped from every served image.
- **Operational Excellence:** Four named alarms, a scripted acceptance test, and rollback notes generated alongside the code.
- **Sustainability:** 60-80% fewer bytes transferred per view, on arm64 silicon.

## Cost Notes

Worked example — 10M image views/month, 50,000 unique variants, 70 KB average response:

- Lambda (first month, all 50k transforms): 50,000 x 1.5 GB x 0.4 s x $0.0000133334 ≈ **$0.40**; thereafter only new uploads transform.
- S3: variant storage 50,000 x 70 KB ≈ 3.5 GB x $0.023 = **$0.08/month**; 50k PUTs = **$0.25** once.
- CloudFront: 700 GB egress — **$0** inside the 1 TB always-free tier; ≈ **$59.50** at $0.085/GB beyond it. 10M HTTPS requests ≈ **$10** past the free 10M.
- CloudFront Functions: 10M invocations — first 2M free, then $0.10/M ≈ **$0.80**.
- **Total: under $1/month within free tier; ~$71/month at full list price — vs $300-500/month for an equivalent managed image CDN.** At 1M unique transforms/month, Lambda compute is still only ~$8.20.

## Troubleshooting

1. **`Could not load the "sharp" module using the linux-arm64 runtime`** — Cause: sharp installed for your dev machine's OS/CPU. Fix: rebuild the package with `npm install --os=linux --cpu=arm64 sharp` (npm 10.4+).
2. **Every request shows `x-cache: Miss from cloudfront`** — Cause: query strings still in the cache key, so `?w=900` fragments the cache. Fix: confirm the viewer-request function strips them and the cache policy forwards no query strings, headers, or cookies.
3. **Browser sends `Accept: image/avif,image/webp` but receives WebP** — Cause: the rewrite function tests `image/webp` before `image/avif`; both substrings are present. Fix: check AVIF first.
4. **CloudFront returns 403 and never invokes Lambda** — Cause: OAC missing on an origin, the bucket policy lacks the `cloudfront.amazonaws.com` principal with the distribution `AWS:SourceArn`, or the function URL auth is NONE while CloudFront signs with AWS_IAM. Fix: align OAC on both origins; set the function URL to AWS_IAM.
5. **500 errors on very large panoramas** — Cause: encoded output exceeded the ~4.5 MB binary ceiling under Lambda's 6 MB buffered-response limit. Fix: cap the width allow-list at 2560 px and keep default quality; the S3 write-back still succeeds, so the retry serves from the primary origin.
