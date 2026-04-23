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
