# Secure Static Web Hosting with Amazon CloudFront & Origin Access Control (OAC) 🌐

A hands-on implementation demonstrating how to securely deliver static web assets using Amazon CloudFront and a private Amazon S3 origin, enforcing least-privilege edge security. Instead of exposing the bucket publicly to the internet, this architecture restricts origin access exclusively to the CloudFront distribution using modern **Origin Access Control (OAC)** with Signature Version 4 (SigV4) request signing.

![Architecture Diagram](images/Screenshot%202026-09-13%20132438.png)

> **Credits & Acknowledgments:**  
> Based on the CloudFront and S3 architectural demo lab by **Adrian Cantrill** ([learn.cantrill.io](https://learn.cantrill.io)). Implemented, debugged, and documented as a hands-on portfolio build focusing on production-grade origin security and troubleshooting.

---

## 🏗️ Architecture Overview

The application architecture isolates origin storage while providing low-latency, globally distributed caching across several tiers:

1. **Private Storage Origin**: **Amazon S3** stores the static web assets (`index.html` and media files) with **Block Public Access** fully enabled.
2. **Global Edge Delivery**: **Amazon CloudFront** caches content globally across edge locations, terminating TLS/HTTPS and reducing origin load.
3. **Edge Authorization**: **Origin Access Control (OAC)** signs outgoing requests from CloudFront to S3 using AWS SigV4, verifying distribution identity.
4. **Scoped Access Policy**: A custom **S3 Bucket Policy** restricts read operations strictly to the CloudFront service principal and scopes down evaluation via the `AWS:SourceArn` condition key.

---

## 🛠️ AWS Services Used

* **Amazon CloudFront**: Global content delivery network (CDN), SSL/TLS termination, and edge caching.
* **Amazon CloudFront Origin Access Control (OAC)**: SigV4 request signing mechanism to secure S3 origins.
* **Amazon S3**: Object storage for web assets with strict private access boundaries.
* **AWS Identity and Access Management (IAM)**: Resource-based bucket policy defining least-privilege access rules.

---

## 🚀 Step-by-Step Implementation

### Stage 1: Configure S3 Bucket & Origin Isolation
Created a private S3 bucket, uploaded the website assets, and kept "Block all public access" fully enabled. Verified that direct HTTP/S access attempts to the S3 bucket URL fail with a `403 Forbidden` error, ensuring the origin cannot be reached directly from the internet.

![Direct S3 Access Blocked](Screenshot%202026-09-13%20123026.png)

---

### Stage 2: Create CloudFront Distribution & Attach OAC
Deployed an Amazon CloudFront distribution pointing to the S3 bucket's REST endpoint. Configured an Origin Access Control (OAC) profile (`E35BYL1VHRGIK2`) with **Always sign requests** using **Signature Version 4 (SigV4)** to authenticate edge requests to the bucket.

| Origin Settings | OAC Details |
| :---: | :---: |
| ![Origin Setup](Screenshot%202026-09-13%20130302.png) | ![OAC Details](Screenshot%202026-09-13%20125901.png) |

---

### Stage 3: Enforce Least-Privilege S3 Bucket Policy
Attached a resource-based bucket policy to the S3 bucket granting `s3:GetObject` solely to `cloudfront.amazonaws.com`. Restricted the statement using the `AWS:SourceArn` condition key pointing explicitly to the distribution ARN (`arn:aws:cloudfront::<ACCOUNT_ID>:distribution/ENPQQ6IL0E210`) to eliminate cross-account confused deputy vulnerabilities.

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
      "Resource": "arn:aws:s3:::cfands3-top10cats-z6nayfbavkxn/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/ENPQQ6IL0E210"
        }
      }
    }
  ]
}
```

![Bucket Policy Verification](Screenshot%202026-09-13%20122926.png)

---

### Stage 4: Configure Distribution Settings & Default Root Object
Navigated to Distribution Settings under the General tab and set the **Default root object** to `index.html`. This ensures that incoming apex requests to `https://dmcfwrd07pxw9.cloudfront.net/` automatically append the root document rather than failing on an S3 REST bucket lookup.

![Default Root Object Setting](Screenshot%202026-09-13%20124856.png)

---

### Stage 5: Create CloudFront Cache Invalidation
Created a cache invalidation for path `/*` to purge existing edge cache copies globally. Monitored the invalidation status until it moved to **Completed**, ensuring edge locations fetch the latest permissions and assets directly from the S3 origin.

![Cache Invalidation Completed](Screenshot%202026-09-13%20124958.png)

---

### Stage 6: Verify Edge Delivery via HTTPS
Tested the site by browsing directly to the CloudFront distribution domain (`https://dmcfwrd07pxw9.cloudfront.net`). Confirmed that the edge distribution successfully signs the origin request via OAC, retrieves the private objects from S3, and serves the website securely over HTTPS.

![CloudFront Live Delivery](Screenshot%202026-09-13%20123049.png)

---

### Stage 7: Account Cleanup
To ensure no unnecessary AWS billing occurs after testing:
* Disabled the CloudFront distribution and waited for status to change to "Disabled", then deleted it.
* Deleted the custom Origin Access Control (OAC) configuration.
* Emptied the S3 bucket objects and deleted the bucket.

---

## 💡 Key Learnings from this Lab

* **Securing Origins with Modern OAC**: Implemented Origin Access Control (OAC) with SigV4 signing, understanding why it supersedes legacy OAI by supporting KMS encryption, dynamic HTTP methods, and newer AWS regions.
* **Preventing Confused Deputy Attacks**: Applied the `AWS:SourceArn` condition key inside the S3 bucket policy to strictly bind bucket access to a single distribution ARN, preventing cross-account origin spoofing.
* **Handling Default Root Objects**: Learned why S3 REST endpoints fail to resolve root directory index files (`/`) by default, and how configuring `Default root object: index.html` on CloudFront resolves apex URL requests.
* **Managing Edge Cache Lifecycles**: Executed CloudFront cache invalidations (`/*`) to purge stale cached objects from global edge caches when updating origin permissions or configurations.
* **Debugging S3 REST vs. Website Endpoints**: Diagnosed edge permissions and learned why CloudFront OAC must point to the standard S3 REST endpoint instead of the static website hosting endpoint to support SigV4 authentication.
