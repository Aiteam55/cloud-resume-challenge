# Cloud Resume Challenge – Step 1 & 2: Hosting and Automating Resume on AWS

## **Context**

This project covers the first two steps of the Cloud Resume Challenge:

1. Host a static HTML/CSS resume on AWS with a **custom domain** over **HTTPS**.
2. Automate deployment from GitHub so any changes to HTML/CSS are automatically uploaded to S3 and CloudFront is invalidated.

---

## **Architecture**

```
User Browser
     │
     ▼
Custom Domain: cloudresume.aiteam.cfd
     │
     ▼
CloudFront Distribution (CDN + HTTPS + Cache)
     │
     ▼
S3 Bucket (Static Website Hosting Enabled, Private)
     │
     ▼
GitHub Actions Workflow (Automatic Deployment)
```

**Services Used:**

- **S3** – stores `index.html` and `style.css` for static hosting.
- **CloudFront** – CDN that serves the content securely, enables HTTPS, and points to the S3 bucket.
- **Route 53 / Namecheap / Cloudflare** – DNS management for the custom domain.
- **GitHub Actions** – automates uploading new changes to S3 and invalidating CloudFront cache.

---

## **Step-by-Step Guide**

### **1️⃣ Create S3 Bucket and Upload Files**

1. Log in to AWS → **S3 → Create Bucket**.
2. Name the bucket (e.g., `cloudresume.aiteam.cfd`) and choose the closest region.
3. Keep **Block all public access** enabled. ✅
4. Upload your **`index.html`** and **`style.css`**.
5. Enable **Static Website Hosting** (for reference only — CloudFront will serve the content, not the website endpoint).

> **Important:** CloudFront will use the **bucket itself**, not the website endpoint, for secure access.

---

### **2️⃣ Configure CloudFront Distribution**

1. Go to **CloudFront → Create Distribution → Web**.
2. Set **Origin Domain Name** to your S3 bucket (not website endpoint).
3. Enable **Origin Access Control (OAC)** to allow CloudFront to read the private bucket.
4. Set **Default Root Object** = `index.html`.
5. Save and wait for deployment (~10–15 minutes).

---

### **3️⃣ Link CloudFront with Custom Domain**

1. Request a certificate in **ACM** for `*.aiteam.cfd` (DNS validation).
2. Add **Alternate Domain Name (CNAME)** in CloudFront: `cloudresume.aiteam.cfd`.
3. Attach the ACM certificate to the CloudFront distribution.
4. In **Namecheap / Cloudflare DNS**, add a **CNAME** record:

| Type  | Host        | Value                         |
| ----- | ----------- | ----------------------------- |
| CNAME | cloudresume | d27dp52xd9etm2.cloudfront.net |

5. Wait for DNS propagation (10–60 min).
6. Test in a **private browser tab** or use **Google DNS (8.8.8.8)**.

---

### **4️⃣ Automate Deployment with GitHub Actions**

**Goal:** Whenever you push to the `main` branch, your updated HTML/CSS files are uploaded to S3, and CloudFront cache is invalidated.

**Steps:**

1. Create GitHub secrets for your repository:

| Secret name                  | Value                        |
| ---------------------------- | ---------------------------- |
| `AWS_ACCESS_KEY_ID`          | From IAM CSV                 |
| `AWS_SECRET_ACCESS_KEY`      | From IAM CSV                 |
| `AWS_REGION`                 | Your S3 bucket region        |
| `S3_BUCKET`                  | `cloudresume.aiteam.cfd`     |
| `CLOUDFRONT_DISTRIBUTION_ID` | From CloudFront Distribution |

2. Create workflow file `.github/workflows/deploy.yml`:

```yaml
name: Deploy Resume to S3

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Upload to S3
        run: |
          aws s3 sync . s3://${{ secrets.S3_BUCKET }} \
            --exclude ".git/*" \
            --exclude ".github/*" \
            --delete

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

**Explanation of Steps:**

- **Checkout code:** Download repo to the runner.
- **Configure AWS credentials:** Injects GitHub secrets into environment so AWS CLI can access S3/CloudFront.
- **Upload to S3:** Syncs repo files to the S3 bucket, ignoring GitHub metadata, and deletes removed files.
- **Invalidate CloudFront cache:** Forces CloudFront to serve latest files globally.

---

### **5️⃣ Troubleshooting / Common Errors**

| Problem                                 | Cause                                        | Solution                                                           |
| --------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------ |
| Access Denied on CloudFront root URL    | No default root object set                   | Set `index.html` as default root object                            |
| CloudFront doesn’t load HTML            | Origin set to S3 website endpoint            | Change origin to bucket itself (private)                           |
| AWS CLI fails to sync in GitHub Actions | Secrets not set or incorrect IAM permissions | Verify GitHub secrets and IAM policy                               |
| CloudFront invalidation fails           | Missing S3/CloudFront IAM permissions        | Attach policy allowing `cloudfront:CreateInvalidation` to IAM user |
| DNS not resolving                       | Local/ISP DNS cache                          | Flush system DNS, use private tab, or Google DNS                   |
| Works only in private window            | Browser caching DNS/SSL                      | Clear browser cache, flush OS DNS cache                            |

> **Note:** CloudFront invalidation requires an IAM policy that allows `cloudfront:CreateInvalidation`. Make sure your IAM user has that permission, otherwise cache won’t update.

---

## **Lessons Learned**

1. **CloudFront origin** must point to the S3 bucket, not the website endpoint.
2. Always **set a default root object** (`index.html`) to prevent Access Denied errors.
3. DNS propagation can take time; testing in **private mode** or with **Google DNS** helps bypass local cache.
4. **GitHub Actions** allows automation of deployments with secrets for secure AWS access.
5. CloudFront invalidation requires proper IAM permissions (`cloudfront:CreateInvalidation`) to work.

---

## **Next Step: Adding Visitor Count**

1. **AWS Lambda** – serverless function to update visitor count.
2. **API Gateway** – HTTP API to call the Lambda function.
3. **DynamoDB** – store visitor count data.

**Flow:**

```
User visits resume → CloudFront serves HTML → HTML fetches visitor count via API Gateway → Lambda updates/reads DynamoDB → Return count to frontend
```
