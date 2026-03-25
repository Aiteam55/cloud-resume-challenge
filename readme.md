# Cloud Resume Challenge – Step 1: Hosting Resume on AWS

## **Context**

This is the first step of the Cloud Resume Challenge. The goal is to host a static HTML/CSS resume on AWS with a **custom domain**, accessible over **HTTPS**. This step ensures a solid foundation for adding dynamic features later, like visitor count.

**Objectives:**

- Upload HTML and CSS files to **S3**.
- Serve the resume via **CloudFront**.
- Link the distribution to a **custom domain** purchased from Namecheap.

---

## **Architecture**

```
User Browser
     │
     ▼
Custom Domain: cloudresume.aiteam.cfd
     │
     ▼
CloudFront Distribution (CDN + HTTPS)
     │
     ▼
S3 Bucket (Static Website Hosting Enabled, Private)
```

**Services Used:**

- **S3** – stores `index.html` and `style.css` for static hosting.
- **CloudFront** – CDN that serves the content securely, enables HTTPS, and points to the S3 bucket.
- **Route 53 / Namecheap** – domain DNS management (CNAME pointing to CloudFront).

---

## **Step-by-Step Guide**

### **1️⃣ Create S3 Bucket and Upload Files**

1. Log in to AWS → **S3 → Create Bucket**.
2. Name the bucket (e.g., `cloudresume-bucket`) and choose the closest region.
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
4. In **Namecheap Advanced DNS**, add a **CNAME** record:

| Type  | Host        | Value                         |
| ----- | ----------- | ----------------------------- |
| CNAME | cloudresume | d27dp52xd9etm2.cloudfront.net |

5. Wait for DNS propagation (10–60 min).
6. Test in a **private browser tab** or switch your DNS to **Google DNS (8.8.8.8)** to bypass cached NXDOMAIN.

---

## **Troubleshooting**

| Problem                              | Cause                             | Solution                                                   |
| ------------------------------------ | --------------------------------- | ---------------------------------------------------------- |
| Access Denied on CloudFront root URL | No default root object set        | Set `index.html` as default root object                    |
| CloudFront doesn’t load HTML         | Origin set to S3 website endpoint | Change origin to the bucket itself (private)               |
| DNS not resolving                    | Local/ISP DNS cache               | Flush system DNS, try private tab, or switch to Google DNS |
| Works only in private window         | Browser caching DNS/SSL           | Clear browser cache, flush OS DNS cache                    |

---

## **Lessons Learned**

1. **CloudFront origin** must point to the S3 bucket, not the website endpoint, for private hosting.
2. Always **set a default root object** (`index.html`) to prevent Access Denied errors.
3. DNS propagation can take time; testing in **private mode** or with **Google DNS** helps bypass local cache.
4. ACM certificates and CloudFront deployment changes can take 10–15 minutes; patience is key.

---

## **Next Step: Adding Visitor Count**

To add a visitor counter to your resume:

1. **AWS Lambda** – serverless function to update visitor count.
2. **API Gateway** – HTTP API to call the Lambda function.
3. **DynamoDB** – store visitor count data.

**Flow:**

```
User visits resume → CloudFront serves HTML → HTML fetches visitor count via API Gateway → Lambda updates/reads DynamoDB → Return count to frontend
```

---

This documentation serves as the official **Step 1 guide** for the Cloud Resume Challenge and can be added to your GitHub project as reference.
