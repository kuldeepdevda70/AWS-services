# CloudFront

## **What is Amazon CloudFront? What problem does it solve, and why do we use it?**

Amazon CloudFront is a fully managed **Content Delivery Network (CDN)** provided by AWS.

CloudFront delivers static and dynamic content to users through a global network of **Edge Locations**, reducing latency and improving application performance.

Instead of every user directly accessing the origin server, such as S3, EC2, or an Application Load Balancer, users can access CloudFront.

CloudFront can cache frequently requested content at Edge Locations and serve it directly to users.

We use CloudFront because it improves performance, reduces latency, reduces load on the origin server, provides HTTPS, and adds security features such as AWS WAF and Origin Access Control (OAC).

---

## **What problem does CloudFront solve?**

Suppose an application is hosted in an AWS region in the USA and a user is accessing the application from India.

Without CloudFront:

```text
User in India
      |
      | Request
      v
AWS Origin in USA
      |
      v
Response
      |
      v
User in India
```

The request has to travel a long distance between the user and the origin.

With CloudFront:

```text
User in India
      |
      v
CloudFront Edge Location
      |
      v
Cached Content
      |
      v
User
```

If the content is already cached at the Edge Location, CloudFront can return it without requesting the origin again.

---

# CloudFront Architecture

```text
                         Users
                           |
                           | HTTPS
                           v
                  +-------------------+
                  |    CloudFront     |
                  |       CDN         |
                  +---------+---------+
                            |
                            |
                     Cache Hit / Miss
                            |
                            v
                         Origin
                            |
            +---------------+---------------+
            |               |               |
            v               v               v
           S3              ALB         API Gateway
                           |
                           v
                          EC2
```

---

# How CloudFront Works

When a user requests an object:

```text
User
 |
 v
CloudFront Edge Location
 |
 v
Check Cache
 |
 +-------------------+
 |                   |
 v                   v
Cache Hit          Cache Miss
 |                   |
 v                   v
Return Object       Origin
                     |
                     v
                  Response
                     |
                     v
                CloudFront
                     |
                     v
                   User
```

---

## **Cache Hit**

A cache hit occurs when the requested object is already available in the CloudFront cache.

```text
User
 |
 v
CloudFront
 |
 | Cache Hit
 v
Cached Object
 |
 v
User
```

### Benefits:

- Faster response
- Lower latency
- Reduced origin requests
- Reduced origin load

---

## **Cache Miss**

A cache miss occurs when the requested object is not available in the CloudFront cache.

```text
User
 |
 v
CloudFront
 |
 | Cache Miss
 v
Origin
 |
 v
Object
 |
 v
CloudFront Cache
 |
 v
User
```

CloudFront can cache the object so that future requests can potentially be served from the Edge Location.

---

# CloudFront Components

## **1. CloudFront Distribution**

A CloudFront Distribution is the main configuration used to deliver content through CloudFront.

It contains configuration for:

- Origins
- Cache behaviors
- Cache policies
- HTTPS
- Security
- Domain names
- Edge caching

Example CloudFront domain:

```text
https://d123456abcdef.cloudfront.net
```

---

## **2. Origin**

An origin is the source from which CloudFront gets the original content.

CloudFront can use:

- Amazon S3
- EC2
- Application Load Balancer
- API Gateway
- Custom HTTP Server

Example:

```text
CloudFront
    |
    v
    S3
```

Another example:

```text
CloudFront
    |
    v
Application Load Balancer
    |
    v
    EC2
```

---

## **3. Edge Location**

An Edge Location is an AWS location where CloudFront can cache and deliver content.

Example:

```text
User
 |
 v
Nearest CloudFront Edge Location
 |
 v
Cached Content
```

CloudFront automatically routes users to an appropriate Edge Location.

---

## **4. Cache Behavior**

Cache behavior controls how CloudFront handles requests for different URL paths.

Example:

```text
/images/*
    |
    +---- Long Cache

/css/*
    |
    +---- Long Cache

/js/*
    |
    +---- Long Cache

/api/*
    |
    +---- Dynamic / Short Cache
```

---

# CloudFront + S3

One of the most common use cases is hosting a static website in Amazon S3 and delivering it through CloudFront.

```text
                         USER
                           |
                           | HTTPS
                           v
                  +----------------+
                  |   CloudFront   |
                  |      CDN       |
                  +-------+--------+
                          |
                          | OAC
                          v
                  +----------------+
                  |       S3       |
                  |     Bucket     |
                  +-------+--------+
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
         index.html      CSS         JS
```

---

## **Why use CloudFront with S3?**

Without CloudFront:

```text
User
 |
 v
S3
```

With CloudFront:

```text
User
 |
 v
CloudFront
 |
 v
S3
```

### Benefits:

- Global content delivery
- Lower latency
- Edge caching
- HTTPS
- Reduced S3 requests
- Reduced origin load
- Private S3 bucket
- Integration with AWS WAF
- Origin protection

---

# Origin Access Control (OAC)

## **What is OAC?**

OAC stands for **Origin Access Control**.

OAC is the recommended method for allowing CloudFront to securely access a private Amazon S3 bucket.

CloudFront signs requests to S3 using **AWS Signature Version 4 (SigV4)**.

```text
User
 |
 | HTTPS
 v
CloudFront
 |
 | OAC
 | SigV4 Signed Request
 v
Private S3 Bucket
```

The S3 bucket does not need to be publicly accessible.

---

# OAC Security Flow

```text
                         USER
                           |
                           | HTTPS
                           v
                     CloudFront
                           |
                           | OAC
                           | SigV4
                           v
                          S3
                           |
                           v
                    Bucket Policy
                           |
                 +---------+---------+
                 |                   |
               Allow                Deny
                 |                   |
                 v                   v
          Return Object         AccessDenied
```

---

# Origin Access Identity (OAI)

OAI stands for **Origin Access Identity**.

OAI is the older mechanism used to allow CloudFront to access private S3 content.

OAI was commonly used before OAC.

For new CloudFront + S3 deployments, AWS recommends using **OAC instead of OAI**.

```text
OAI = Legacy
OAC = Recommended
```

---

# OAC vs OAI

| Feature | OAC | OAI |
|---|---|---|
| Full Name | Origin Access Control | Origin Access Identity |
| Status | Recommended | Legacy |
| S3 Private Bucket | Yes | Yes |
| Request Signing | SigV4 | Legacy mechanism |
| SSE-KMS | Supported | Not supported |
| New Projects | Recommended | Not recommended |
| Modern AWS Architecture | Yes | Older architecture |

### Recommendation

For new projects:

```text
CloudFront + S3 + OAC
```

For older projects:

```text
CloudFront + S3 + OAI
```

---

# OAC Advantages

OAC provides:

- Secure S3 access
- Private S3 bucket
- SigV4 request signing
- Better integration with S3
- SSE-KMS support
- Origin protection
- Modern AWS security architecture

---

# S3 Bucket Security With OAC

Recommended configuration:

```text
S3 Bucket
   |
   +---- Block Public Access = ON
   |
   +---- Objects = Private
   |
   +---- Bucket Policy
            |
            +---- CloudFront Service Principal
            |
            +---- CloudFront Distribution ARN
```

The user accesses the application through CloudFront rather than directly through S3.

```text
User
 |
 v
CloudFront
 |
 | OAC
 v
Private S3
```

---

# S3 Bucket Policy for OAC

Example bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipalReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_DISTRIBUTION_ID"
        }
      }
    }
  ]
}
```

Replace:

```text
YOUR_BUCKET_NAME
YOUR_ACCOUNT_ID
YOUR_DISTRIBUTION_ID
```

with the actual values.

---

# S3 Origin vs S3 Website Endpoint

This is an important concept when configuring CloudFront with OAC.

## S3 Bucket Origin

Example:

```text
bucket.s3.us-east-1.amazonaws.com
```

This is the recommended origin for:

```text
CloudFront + OAC
```

---

## S3 Website Endpoint

Example:

```text
bucket.s3-website-us-east-1.amazonaws.com
```

This is used for S3 Static Website Hosting.

For a private S3 bucket using OAC, use the **S3 bucket origin**, not the S3 website endpoint.

---

# CloudFront HTTPS

CloudFront supports HTTPS for secure content delivery.

Recommended configuration:

```text
Viewer Protocol Policy:

Redirect HTTP to HTTPS
```

Flow:

```text
HTTP Request
      |
      v
CloudFront
      |
      v
HTTPS Redirect
      |
      v
Secure Website
```

For a custom domain, AWS Certificate Manager (ACM) can be used to provide the TLS certificate.

---

# CloudFront Advantages

## **1. Low Latency**

CloudFront serves cached content from Edge Locations closer to users.

---

## **2. Global Content Delivery**

CloudFront provides a global network for delivering content.

---

## **3. Caching**

Frequently requested content can be cached at Edge Locations.

---

## **4. Better Performance**

Users can receive cached content faster.

---

## **5. Reduced Origin Load**

If content is served from the CloudFront cache, the request does not need to reach the origin.

---

## **6. HTTPS Support**

CloudFront supports secure HTTPS connections.

---

## **7. Security**

CloudFront can integrate with:

- AWS WAF
- AWS Shield
- Origin Access Control
- HTTPS
- Signed URLs
- Signed Cookies
- Geo Restrictions

---

## **8. Private S3 Access**

Using OAC, the S3 bucket can remain private.

```text
CloudFront
    |
    | OAC
    v
Private S3
```

---

## **9. Scalability**

CloudFront can handle large amounts of traffic using AWS global infrastructure.

---

# CloudFront Limitations

## **1. Additional Cost**

CloudFront can generate charges for:

- Data transfer
- HTTP/HTTPS requests
- Origin requests
- Additional CloudFront features

---

## **2. Stale Cache**

CloudFront may serve an older cached version of an object until the cache expires or is invalidated.

---

## **3. Cache Invalidation**

Sometimes cache invalidation is required after deploying new files.

Example:

```bash
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

---

## **4. Configuration Complexity**

CloudFront has many configuration options:

- Cache policies
- Cache behaviors
- Origin request policies
- Response headers
- OAC
- HTTPS
- Cookies
- Query strings

Incorrect configuration can result in:

```text
403 AccessDenied
404 Not Found
502 Bad Gateway
CORS Errors
Stale Content
```

---

## **5. Configuration Propagation**

CloudFront configuration changes may take some time to propagate globally.

---

## **6. Dynamic Data Requires Care**

Do not blindly cache sensitive data such as:

```text
User Profiles
Shopping Carts
Payment Information
Authentication Data
Private User Data
```

---

# CloudFront Use Cases

## **1. Static Websites**

Example:

```text
React
Angular
Vue
HTML
CSS
JavaScript
```

Architecture:

```text
User
 |
 v
CloudFront
 |
 v
S3
```

---

## **2. Web Applications**

```text
User
 |
 v
CloudFront
 |
 v
ALB
 |
 v
EC2
```

---

## **3. API Applications**

```text
User
 |
 v
CloudFront
 |
 v
API Gateway
 |
 v
Lambda
 |
 v
Database
```

---

## **4. Media Delivery**

CloudFront can deliver:

- Images
- Videos
- Audio
- Large files
- Downloads

---

# CloudFront + React/Vite

For a React/Vite application:

```text
React Application
       |
       | npm run build
       v
      dist/
       |
       v
       S3
       |
       | OAC
       v
   CloudFront
       |
       v
      Users
```

Build the application:

```bash
npm run build
```

The generated `dist` folder can be uploaded to S3.

---

# CloudFront + ALB + EC2

CloudFront can be placed in front of an Application Load Balancer.

```text
                         USER
                           |
                           | HTTPS
                           v
                     CloudFront
                           |
                           v
                 Application Load Balancer
                           |
                +----------+----------+
                |                     |
                v                     v
              EC2-1                 EC2-2
                |                     |
                +----------+----------+
                           |
                           v
                      Application
```

Benefits:

- CDN caching
- Global delivery
- HTTPS
- WAF integration
- Reduced load
- Better performance

---

# CloudFront + API Gateway

CloudFront can also be used with API Gateway.

```text
User
 |
 v
CloudFront
 |
 v
API Gateway
 |
 v
Lambda
 |
 v
Database
```

Caching should be configured carefully for APIs.

Sensitive user-specific responses should not be blindly cached.

---

# CloudFront Is Not a Database Cache

CloudFront is primarily an **HTTP content cache**.

It is not a replacement for:

- Redis
- ElastiCache
- Database caching
- DynamoDB

Example:

```text
User
 |
 v
CloudFront
 |
 v
Backend API
 |
 v
RDS
```

CloudFront can cache an HTTP response if configured, but CloudFront itself is not a database.

---

# RDS + CloudFront Example

Suppose a user submits an order:

```text
POST /orders
```

Typical architecture:

```text
User
 |
 v
CloudFront
 |
 v
ALB
 |
 v
Backend
 |
 v
RDS
```

The backend processes the order and stores the data in RDS.

CloudFront should not be treated as the database layer.

---

# CloudFront POC

## Objective

The objective of this POC is to:

1. Deploy a static website to S3.
2. Keep the S3 bucket private.
3. Create a CloudFront Distribution.
4. Configure S3 as the CloudFront origin.
5. Configure Origin Access Control.
6. Allow CloudFront to access S3.
7. Enable HTTPS.
8. Access the website using the CloudFront URL.
9. Verify that direct S3 access is blocked.

---

# POC Architecture

```text
                         INTERNET
                            |
                            | HTTPS
                            v
                   +----------------+
                   |   CloudFront   |
                   |      CDN       |
                   +-------+--------+
                           |
                           | OAC
                           | SigV4
                           v
                   +----------------+
                   |       S3       |
                   | Private Bucket |
                   +----------------+
                           |
              +------------+------------+
              |            |            |
              v            v            v
         index.html       CSS          JS
```

---

# POC Prerequisites

You need:

- AWS Account
- S3 Bucket
- Static website files
- CloudFront
- Origin Access Control
- IAM permissions

Example files:

```text
index.html
style.css
script.js
```

---

# Step 1: Create S3 Bucket

Create an S3 bucket.

Example:

```text
cloudfront-static-website-poc
```

Keep:

```text
Block Public Access = ON
```

Do not make the bucket publicly accessible.

---

# Step 2: Upload Website Files

Upload:

```text
index.html
style.css
script.js
```

Example:

```text
S3
 |
 +---- index.html
 |
 +---- style.css
 |
 +---- script.js
```

---

# Step 3: Create CloudFront Distribution

Go to:

```text
AWS Console
    |
    v
CloudFront
    |
    v
Create Distribution
```

---

# Step 4: Configure Origin

Select the S3 bucket.

Use:

```text
bucket.s3.region.amazonaws.com
```

Do not select the S3 website endpoint when using OAC.

---

# Step 5: Create OAC

Create an Origin Access Control.

Use:

```text
Origin Type:
S3
```

Configure:

```text
Signing Behavior:
Sign Requests
```

Use:

```text
Signing Protocol:
SigV4
```

---

# Step 6: Attach OAC

Attach the OAC to the CloudFront S3 origin.

Flow:

```text
CloudFront
     |
     | OAC
     v
S3
```

---

# Step 7: Configure S3 Bucket Policy

Allow CloudFront to read objects from S3.

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipalReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_DISTRIBUTION_ID"
        }
      }
    }
  ]
}
```

---

# Step 8: Configure Default Root Object

Set:

```text
index.html
```

as the Default Root Object.

This means:

```text
https://d123456.cloudfront.net/
```

can load:

```text
index.html
```

---

# Step 9: Configure HTTPS

Set:

```text
Viewer Protocol Policy:

Redirect HTTP to HTTPS
```

---

# Step 10: Create Distribution

Create the CloudFront Distribution.

Wait until the distribution status becomes:

```text
Enabled
```

and deployment is completed.

---

# Step 11: Get CloudFront URL

CloudFront provides a domain such as:

```text
https://d123456abcdef.cloudfront.net
```

Open this URL in the browser.

Expected result:

```text
Your static website should load.
```

---

# Complete POC Flow

```text
START
  |
  v
Create S3 Bucket
  |
  v
Enable Block Public Access
  |
  v
Upload Website Files
  |
  v
Create CloudFront Distribution
  |
  v
Select S3 Origin
  |
  v
Create OAC
  |
  v
Enable SigV4 Signing
  |
  v
Configure S3 Bucket Policy
  |
  v
Set Default Root Object
  |
  v
Configure HTTP -> HTTPS
  |
  v
Create Distribution
  |
  v
Open CloudFront URL
  |
  v
Website Loads
  |
  v
Test Direct S3 Access
  |
  v
AccessDenied
  |
  v
POC Completed
```

---

# Security Testing

## Test 1: CloudFront URL

Open:

```text
https://d123456abcdef.cloudfront.net
```

Expected:

```text
Website loads successfully
```

---

## Test 2: Direct S3 URL

Try accessing:

```text
https://YOUR_BUCKET.s3.YOUR_REGION.amazonaws.com/index.html
```

Expected:

```text
AccessDenied
```

This confirms that the S3 object is not publicly accessible.

---

## Test 3: HTTP Request

Open:

```text
http://d123456abcdef.cloudfront.net
```

Expected:

```text
Redirect to HTTPS
```

---

# CloudFront Security Model

Recommended architecture:

```text
                         USER
                           |
                           | HTTPS
                           v
                     CloudFront
                           |
                           | OAC + SigV4
                           v
                    Private S3 Bucket
                           |
                           v
                    Bucket Policy
```

The user should access the website through CloudFront.

The S3 bucket remains private.

---

# CloudFront + OAC Security Flow

```text
User
 |
 | HTTPS
 v
CloudFront
 |
 | Signed Request
 | SigV4
 v
S3
 |
 | Check Bucket Policy
 v
Allow
 |
 v
Return Object
```

If the request does not have permission:

```text
Request
   |
   v
S3
   |
   v
Bucket Policy
   |
   v
Deny
   |
   v
AccessDenied
```

---

# Cache Invalidation

When website files are updated, CloudFront may still have the old files in its cache.

You can invalidate the cache.

Example:

```bash
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

For production applications, versioned asset filenames are often preferred to reduce unnecessary invalidations.

---

# CloudFront + AWS WAF

CloudFront can integrate with AWS WAF.

```text
Internet
   |
   v
CloudFront
   |
   | AWS WAF
   v
Origin
```

AWS WAF can be used for:

- SQL Injection protection
- XSS protection
- IP filtering
- Rate limiting
- Bot protection
- Custom rules

---

# CloudFront + AWS Shield

CloudFront integrates with AWS Shield for DDoS protection.

```text
Internet
    |
    v
AWS Shield
    |
    v
CloudFront
    |
    v
Origin
```

---

# Custom Domain With CloudFront

CloudFront can use a custom domain.

Example:

```text
www.example.com
```

Architecture:

```text
User
 |
 v
Route 53
 |
 v
CloudFront
 |
 | OAC
 v
S3
```

Common AWS services:

```text
Route 53
ACM
CloudFront
S3
```

For a CloudFront distribution, the ACM certificate must be created in:

```text
us-east-1
```

---

# Troubleshooting CloudFront 403 AccessDenied

If CloudFront returns:

```text
403 AccessDenied
```

check the following.

### 1. Check OAC

Make sure OAC is attached to the correct S3 origin.

### 2. Check Signing

Make sure:

```text
Signing Behavior = Sign Requests
```

### 3. Check Origin

Use:

```text
bucket.s3.region.amazonaws.com
```

instead of the S3 website endpoint when using OAC.

### 4. Check Bucket Policy

Make sure the principal is:

```json
{
  "Service": "cloudfront.amazonaws.com"
}
```

### 5. Check Source ARN

Verify:

```text
AWS:SourceArn
```

matches the correct CloudFront Distribution ARN.

Example:

```text
arn:aws:cloudfront::123456789012:distribution/ABC123
```

### 6. Check Object

Make sure:

```text
index.html
```

exists in the S3 bucket.

### 7. Check Default Root Object

Make sure:

```text
Default Root Object = index.html
```

---

# Common CloudFront Errors

| Error | Possible Cause |
|---|---|
| 403 AccessDenied | OAC or bucket policy issue |
| 404 Not Found | Object does not exist |
| 502 Bad Gateway | Origin problem |
| CORS Error | Incorrect CORS configuration |
| Old Content | CloudFront cache |
| S3 AccessDenied | Incorrect bucket permissions |
| Website Not Loading | Incorrect origin or root object |

---

# Security Checklist

- [ ] S3 Block Public Access = ON
- [ ] S3 bucket is private
- [ ] S3 objects are private
- [ ] CloudFront Distribution created
- [ ] S3 configured as origin
- [ ] OAC created
- [ ] OAC attached to origin
- [ ] Signing Behavior = Sign Requests
- [ ] Signing Protocol = SigV4
- [ ] S3 bucket policy configured
- [ ] CloudFront Service Principal configured
- [ ] AWS:SourceArn configured
- [ ] Default Root Object = `index.html`
- [ ] HTTP redirects to HTTPS
- [ ] CloudFront URL works
- [ ] Direct S3 URL returns `AccessDenied`

---

# CloudFront vs Direct S3

| Feature | Direct S3 | CloudFront + S3 |
|---|---|---|
| CDN | No | Yes |
| Edge Caching | No | Yes |
| Global Delivery | No | Yes |
| HTTPS | Yes | Yes |
| OAC | No | Yes |
| Private S3 Origin | Yes, with AWS access controls | Yes |
| WAF Integration | No CDN layer | Yes |
| Origin Protection | Limited | Better |
| Global Performance | Lower | Better |
| Caching | S3 caching headers | CloudFront caching |

---

# OAC vs OAI Summary

| OAC | OAI |
|---|---|
| Origin Access Control | Origin Access Identity |
| Modern | Legacy |
| Recommended by AWS | Older mechanism |
| Uses SigV4 | Older signing mechanism |
| Supports SSE-KMS | Does not support SSE-KMS |
| Recommended for new projects | Mainly for existing/legacy setups |

### Recommended Architecture

```text
New Project
    |
    v
CloudFront
    |
    v
OAC
    |
    v
Private S3
```

---

# Advantages of CloudFront

- Low latency
- Global content delivery
- Edge caching
- Better website performance
- Reduced origin load
- HTTPS support
- Private S3 access with OAC
- AWS WAF integration
- AWS Shield integration
- High scalability
- Custom domain support
- Signed URLs and cookies
- Geo restrictions

---

# Limitations of CloudFront

- Additional cost
- Cache invalidation may be required
- Cached content can become stale
- Configuration can be complex
- Configuration changes take time to propagate
- Dynamic content requires careful caching
- Incorrect configuration can cause 403, 404, and 502 errors
- Not a replacement for database caching

---

# Final CloudFront Architecture

```text
                         INTERNET
                            |
                            | HTTPS
                            v
                 +----------------------+
                 |      CloudFront      |
                 |                      |
                 |    CDN / Cache       |
                 |    Edge Locations    |
                 |    HTTPS             |
                 +----------+-----------+
                            |
                            |
                         OAC + SigV4
                            |
                            v
                 +----------------------+
                 |      Amazon S3       |
                 |                      |
                 |    PRIVATE BUCKET    |
                 |                      |
                 |    index.html        |
                 |    CSS               |
                 |    JavaScript        |
                 |    Images            |
                 +----------------------+
                            |
                            v
                     Bucket Policy
                            |
                            v
                 CloudFront Distribution
```

---

# Final Request Flow

```text
                         USER
                           |
                           | HTTPS
                           v
                  CloudFront Edge
                           |
                           v
                     Check Cache
                           |
              +------------+------------+
              |                         |
          Cache Hit                 Cache Miss
              |                         |
              v                         v
       Return Cached                OAC + SigV4
          Object                         |
                                        v
                                       S3
                                        |
                                 Bucket Policy
                                        |
                                        v
                                   index.html
                                        |
                                        v
                                   CloudFront
                                        |
                                        v
                                       USER
```

---

# Key Points to Remember

1. CloudFront is an AWS CDN.
2. CloudFront delivers content through Edge Locations.
3. CloudFront can cache frequently requested content.
4. Cache Hit means the object is already available in the CloudFront cache.
5. Cache Miss means CloudFront must request the object from the origin.
6. S3 can be used as a CloudFront origin.
7. OAC is the recommended method for new CloudFront + S3 deployments.
8. OAI is the legacy access mechanism.
9. OAC uses AWS Signature Version 4.
10. S3 can remain private when using OAC.
11. CloudFront supports HTTPS.
12. CloudFront can integrate with AWS WAF.
13. CloudFront can integrate with AWS Shield.
14. CloudFront can reduce origin load.
15. CloudFront can improve global application performance.
16. CloudFront is not a database.
17. Redis/ElastiCache are more suitable for application/database caching.
18. Cache behavior should be carefully configured for dynamic content.
19. For OAC, use the S3 bucket origin rather than the S3 website endpoint.
20. For new S3 + CloudFront projects, prefer OAC over OAI.

---

# Conclusion

Amazon CloudFront is a globally distributed AWS CDN that improves application performance, reduces latency, provides caching, and adds security capabilities.

For a secure static website hosted in S3, the recommended architecture is:

```text
                    USER
                      |
                      | HTTPS
                      v
                +-------------+
                | CloudFront  |
                | CDN / Cache |
                +------+------+
                       |
                       | OAC
                       | SigV4
                       v
                +-------------+
                |     S3      |
                |   PRIVATE   |
                |    BUCKET   |
                +-------------+
```

The important security principle is:

> Keep the S3 bucket private and allow only the CloudFront Distribution to access the S3 objects using Origin Access Control (OAC).

For new AWS deployments:

```text
OAC = Recommended
OAI = Legacy
```

This architecture provides a secure, scalable, and high-performance way to deliver static website content globally.
