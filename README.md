# AWS S3 Static Website Hosting — Level Up Bank

## Business Use Case
Level Up Bank is a fictitious fintech startup that needed to migrate 
their website off an expensive on-premises server. As the cloud engineer 
on this project, I migrated the website to AWS S3 static website hosting 
to achieve:

- **Cost savings** — Pay only for storage used, no server maintenance costs
- **Scalability** — S3 automatically handles traffic spikes with no downtime
- **High availability** — Built-in redundancy across AWS availability zones
- **Reduced overhead** — Fully managed service, no infrastructure to maintain

## Architecture

## Tools & Services Used
- AWS S3 (Simple Storage Service)
- AWS Console (Console method)
- AWS CLI (CLI method — coming soon)

## Prerequisites
- AWS Account with appropriate IAM permissions
- AWS CLI installed and configured (for CLI method)
- Basic understanding of HTML and cloud storage

---

## Implementation — Console Method

### Step 1: Create the S3 Bucket
- Created bucket: `levelupbank-website-gold-2026`
- Region: `us-east-1`
- Disabled "Block all public access" to allow public website hosting

> **Security Note:** Block Public Access acts as a master override above 
> all bucket policies. Disabling it does not make files public by itself — 
> a bucket policy is still required to grant access.

### Step 2: Upload Website Files
- Uploaded `index.html` to the root of the bucket
- Filename must be exactly `index.html` — S3 uses this as the default 
  index document when serving the website

### Step 3: Enable Static Website Hosting
- Navigated to: Bucket → Properties → Static website hosting → Edit
- Set hosting to **Enabled**
- Set index document to `index.html`
- This generated the bucket website endpoint:
  `http://levelupbank-website-gold-2026.s3-website-us-east-1.amazonaws.com`

### Step 4: Apply Bucket Policy
Applied a least-privilege bucket policy granting public read-only access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::levelupbank-website-gold-2026/*"
    }
  ]
}
```

> **Security Decision:** `s3:GetObject` was deliberately chosen over `s3:*` 
> to grant read-only access. This means the public can view files but 
> cannot upload, modify, or delete anything in the bucket.

### Step 5: Verify
- Tested using the bucket website endpoint in an incognito browser
- Confirmed the website loaded correctly without being logged into AWS

---

## Security Findings & Next Steps

| Finding | Risk | Remediation |
|---|---|---|
| Website running on HTTP | High — data is unencrypted in transit | Add CloudFront with HTTPS enforcement |
| S3 bucket directly public | Medium — no edge caching or DDoS protection | Add CloudFront as intermediary |
| No access logging enabled | Low — no audit trail of who accessed the site | Enable S3 server access logging |

---

## What I Learned
- The difference between Block Public Access (master switch) and bucket 
  policies (granular rules) and how they interact
- Why the bucket website endpoint must be used to test static hosting 
  rather than the direct object URL
- Why `index.html` naming matters for S3 to serve the correct default page
- HTTP vs HTTPS and why HTTPS is non-negotiable for financial services 
  (PCI-DSS compliance)

---

## Next Iterations
- [ ] Repeat full implementation using AWS CLI
- [ ] Add CloudFront distribution for HTTPS and edge caching
- [ ] Restrict S3 bucket so only CloudFront can access it directly
  (Origin Access Control)

---

## Implementation — CLI Method

### Prerequisites
- AWS CLI installed and configured (`aws configure list` to verify)
- `index.html` saved locally
- Windows users: use JSON config files instead of inline JSON to avoid 
  PowerShell quoting issues

### Commands Used

**Step 1: Create the bucket**
```bash
aws s3 mb s3://levelupbank-website-cli-2026
```

**Step 2: Disable Block Public Access**
```bash
aws s3api put-public-access-block \
  --bucket levelupbank-website-cli-2026 \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,
  BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

**Step 3: Upload the website file**
```bash
aws s3 cp "C:\Users\Darien\Downloads\index.html" s3://levelupbank-website-cli-2026/
```

**Step 4: Enable static website hosting**

Created `website-config.json`:
```json
{
    "IndexDocument": {
        "Suffix": "index.html"
    },
    "ErrorDocument": {
        "Key": "error.html"
    }
}
```
```bash
aws s3api put-bucket-website \
  --bucket levelupbank-website-cli-2026 \
  --website-configuration file://website-config.json
```

**Step 5: Apply bucket policy**

Created `bucket-policy.json`:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::levelupbank-website-cli-2026/*"
        }
    ]
}
```
```bash
aws s3api put-bucket-policy \
  --bucket levelupbank-website-cli-2026 \
  --policy file://bucket-policy.json
```

### Key CLI Lessons Learned
- CLI commands are case sensitive — `BlockPublicPolicy` ≠ `BlockPublicpolicy`
- Silent output means success — errors are always explicit
- On Windows PowerShell, use `file://` with JSON config files to avoid 
  quoting issues
- Always cross-verify CLI changes in the console

- ---

## Advanced Tier — CloudFront + HTTPS

### What This Adds
- HTTPS enforcement via CloudFront — eliminates the "Not secure" warning
- Global edge caching — website is served from AWS edge locations 
  closest to the user
- HTTP to HTTPS redirect — any unencrypted request is automatically 
  upgraded

### Architecture

### CloudFront Distribution Settings

| Setting | Value | Reason |
|---|---|---|
| Origin | S3 website endpoint | Required for index.html routing |
| Viewer protocol policy | Redirect HTTP to HTTPS | Forces encrypted connections |
| Allowed HTTP methods | GET, HEAD | Read-only — static website |
| Cache policy | CachingOptimized | Recommended for S3 content |
| WAF | Disabled | Learning project — not required |
| Price class | Pay-as-you-go | Minimal cost for low traffic |

### CloudFront Domain
`https://d1vtfi1auj4t0r.cloudfront.net`

### Cost Considerations
| Service | Learning Project | Production Recommendation |
|---|---|---|
| CloudFront | ~$0/month (minimal traffic) | Pay-as-you-go based on usage |
| WAF | Disabled | ~$14/month — recommended for public-facing apps |
| WAF (Enterprise) | N/A | $200+/month for advanced DDoS, bot protection |

> **Executive Note:** WAF should be considered a non-negotiable cost 
> for any production financial services website. The $14/month base 
> cost is negligible compared to the cost of a successful web attack 
> or compliance violation.

### Security Limitation of This Tier
The S3 bucket remains publicly accessible. A user who knows the bucket 
endpoint can bypass CloudFront entirely and access the site over HTTP 
with no caching or security controls. This is addressed in the 
Complex tier below.

---

## Complex Tier — OAC (Production-Grade Security)

### What This Adds
- S3 bucket is completely private — no public access whatsoever
- Only the specific CloudFront distribution can access S3
- Eliminates the ability to bypass CloudFront entirely

### Architecture

### What is OAC?
Origin Access Control (OAC) gives CloudFront a verified identity that 
S3 recognizes. The S3 bucket policy grants read access to one thing 
only — the specific CloudFront distribution ARN. This is similar to 
an IAM role: instead of managing credentials, AWS handles identity 
verification internally.

> **Real-world analogy:** OAC is a backstage pass. CloudFront has it. 
> Everyone else — including users who know the S3 URL — gets stopped 
> at the door.

### Key Configuration Changes from Advanced Tier

1. **Origin changed** from S3 website endpoint to S3 REST API endpoint
   - Website endpoint does not support OAC
   - REST endpoint: `levelupbank-website-gold-2026.s3.us-east-1.amazonaws.com`

2. **Default root object** set to `index.html` in CloudFront
   - REST endpoint does not handle index documents automatically
   - CloudFront must be explicitly told what to serve at the root

3. **Bucket policy replaced** with CloudFront-generated OAC policy:

```json
{
    "Version": "2008-10-17",
    "Id": "PolicyForCloudFrontPrivateContent",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::levelupbank-website-gold-2026/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::287870619366:distribution/E29W2FNFTFKCZ6"
                }
            }
        }
    ]
}
```

4. **Block Public Access re-enabled** — final lockdown

### Verification Results

| Test | URL | Expected Result | Actual Result |
|---|---|---|---|
| CloudFront HTTPS | `https://d1vtfi1auj4t0r.cloudfront.net` | Website loads | ✅ Pass |
| Direct S3 access | `http://levelupbank-website-gold-2026.s3-website-us-east-1.amazonaws.com` | 403 Forbidden | ✅ Pass |

### Why OAC is the Production Standard
- **Least privilege** — S3 trusts exactly one identity, nothing else
- **No public attack surface** — bucket name exposure means nothing
- **Compliance ready** — satisfies requirements that storage never be public-facing (PCI-DSS, HIPAA, FedRAMP)
- **Full auditability** — all traffic flows through one controlled point with consistent logging

---

## Teardown (Cost Management)
When this project is not actively in use, disable or delete these 
resources to avoid unnecessary charges:

```bash
# Disable CloudFront distribution first (must be disabled before deletion)
aws cloudfront update-distribution --id E29W2FNFTFKCZ6 --no-enabled

# Delete S3 objects
aws s3 rm s3://levelupbank-website-gold-2026 --recursive
aws s3 rm s3://levelupbank-website-cli-2026 --recursive

# Delete S3 buckets
aws s3 rb s3://levelupbank-website-gold-2026
aws s3 rb s3://levelupbank-website-cli-2026
```

> **Engineer's Note:** Always clean up cloud resources after learning 
> projects. Leaving idle resources running is a common source of 
> unexpected AWS bills and is poor cloud hygiene.
