# EKS + NGINX Ingress + Cert-Manager + Let's Encrypt SSL

This document explains the complete setup used to configure:

- Amazon EKS
- NGINX Ingress Controller
- Cert-Manager
- Let's Encrypt production SSL certificate
- Domain: `kuldeep.website`
- HTTPS
- HTTP → HTTPS redirection
- Automatic SSL certificate renewal

---

# 1. Architecture

The final traffic flow is:

Internet
   |
   | HTTP / HTTPS
   v
Domain: kuldeep.website
   |
   v
AWS Load Balancer
   |
   v
NGINX Ingress Controller
   |
   +----------------------+
   |                      |
   | /                    | /api
   v                      v
Frontend Service       Backend Service
   |                      |
   v                      v
Frontend Pod           Backend Pod
                          |
                          v
                      PostgreSQL
                          |
                          v
                     EBS Volume


SSL flow:

Browser
   |
   | HTTPS :443
   v
AWS Load Balancer
   |
   v
NGINX Ingress
   |
   | TLS certificate from Kubernetes Secret
   v
Application

Certificate management:

Cert-Manager
   |
   v
Let's Encrypt
   |
   v
Kubernetes TLS Secret
   |
   v
NGINX Ingress
```

---

# 2. Important Decision

We are NOT using:

- AWS ACM for application SSL
- NGINX installed manually on an EC2 server
- Certbot cron jobs

We are using:

- NGINX Ingress Controller
- Cert-Manager
- Let's Encrypt
- Kubernetes TLS Secret

Cert-manager handles certificate issuance and renewal automatically.

---

# 3. Install Helm Repositories

Add the Jetstack repository for cert-manager:

```bash
helm repo add jetstack https://charts.jetstack.io
```

Add the NGINX Ingress repository:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

Update Helm repositories:

```bash
helm repo update
```

---

# 4. Install Cert-Manager

Install cert-manager:

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Check cert-manager pods:

```bash
kubectl get pods -n cert-manager
```

Expected:

```text
cert-manager-xxxxx                  Running
cert-manager-cainjector-xxxxx      Running
cert-manager-webhook-xxxxx         Running
```

Check installation:

```bash
kubectl get pods -n cert-manager
```

All three components should eventually become:

```text
1/1 Running
```

---

# 5. Install NGINX Ingress Controller

Install NGINX Ingress Controller:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.ingressClassResource.name=nginx \
  --set controller.ingressClassResource.controllerValue=k8s.io/ingress-nginx
```

Check NGINX pod:

```bash
kubectl get pods -n ingress-nginx
```

Expected:

```text
ingress-nginx-controller-xxxxx   1/1   Running
```

Check the LoadBalancer:

```bash
kubectl get svc -n ingress-nginx
```

Example:

```text
NAME                       TYPE           EXTERNAL-IP
ingress-nginx-controller   LoadBalancer   k8s-ingressn-xxxxx.elb.amazonaws.com
```

The AWS Load Balancer is automatically created because the NGINX controller Service is of type:

```text
LoadBalancer
```

---

# 6. Verify IngressClass

Check:

```bash
kubectl get ingressclass
```

Expected:

```text
NAME    CONTROLLER
alb     ingress.k8s.aws/alb
nginx   k8s.io/ingress-nginx
```

For our application we use:

```yaml
ingressClassName: nginx
```

Do NOT use:

```yaml
ingressClassName: alb
```

because we are using NGINX Ingress for SSL termination and cert-manager.

---

# 7. Create Let's Encrypt ClusterIssuer

Create:

```text
clusterissuer.yaml
```

Content:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: kuldeepdevda134@gmail.com

    server: https://acme-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-prod

    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

Apply:

```bash
kubectl apply -f clusterissuer.yaml
```

Check:

```bash
kubectl get clusterissuer
```

Expected:

```text
NAME               READY
letsencrypt-prod   True
```

---

# 8. How ClusterIssuer Works

The ClusterIssuer tells cert-manager:

1. Use Let's Encrypt.
2. Use the production Let's Encrypt server.
3. Use email for ACME registration.
4. Use HTTP-01 challenge.
5. Use NGINX Ingress for the challenge.
6. Store the certificate in a Kubernetes Secret.

The important part is:

```yaml
server: https://acme-v02.api.letsencrypt.org/directory
```

This is the Let's Encrypt production environment.

---

# 9. Create Certificate

Create:

```text
certificate.yaml
```

Content:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kuldeep-website-tls
  namespace: 3-tier-app-eks
spec:
  secretName: kuldeep-website-tls

  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

  dnsNames:
    - kuldeep.website
```

Apply:

```bash
kubectl apply -f certificate.yaml
```

Check:

```bash
kubectl get certificate -n 3-tier-app-eks
```

Initially it may show:

```text
NAME                  READY   SECRET                AGE
kuldeep-website-tls   False   kuldeep-website-tls   10s
```

After successful issuance:

```text
NAME                  READY   SECRET                AGE
kuldeep-website-tls   True    kuldeep-website-tls   1m
```

---

# 10. Important Let's Encrypt Rate Limit

During testing, we received:

```text
429 urn:ietf:params:acme:error:rateLimited
```

The message indicated:

```text
too many certificates (5) already issued for this exact set of identifiers
```

This means Let's Encrypt has a rate limit.

The limit is not caused by Kubernetes or cert-manager.

Do NOT repeatedly delete and recreate certificates while testing production issuance.

For testing certificate issuance, use the Let's Encrypt staging server.

Production:

```text
https://acme-v02.api.letsencrypt.org/directory
```

Staging:

```text
https://acme-staging-v02.api.letsencrypt.org/directory
```

Use staging while troubleshooting to avoid consuming production certificate limits.

---

# 11. NGINX Ingress Configuration

The final `ingress.yml` should be:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: three-tier-ingress
  namespace: 3-tier-app-eks

  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod

    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

spec:
  ingressClassName: nginx

  tls:
    - hosts:
        - kuldeep.website
      secretName: kuldeep-website-tls

  rules:
    - host: kuldeep.website

      http:
        paths:

          # Backend API
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 8000

          # Frontend
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f ingress.yml
```

---

# 12. What the TLS Section Does

This part:

```yaml
tls:
  - hosts:
      - kuldeep.website
    secretName: kuldeep-website-tls
```

tells NGINX:

For:

```text
kuldeep.website
```

use this Kubernetes TLS Secret:

```text
kuldeep-website-tls
```

The Secret contains:

```text
tls.crt
tls.key
```

The Secret is created and managed by cert-manager.

---

# 13. What Cert-Manager Does

The complete process is:

```text
Certificate resource
        |
        v
Cert-manager
        |
        v
Let's Encrypt
        |
        v
HTTP-01 Challenge
        |
        v
NGINX Ingress
        |
        v
Let's Encrypt verifies domain
        |
        v
Certificate issued
        |
        v
Kubernetes TLS Secret
        |
        v
NGINX Ingress uses Secret
        |
        v
HTTPS works
```

---

# 14. Domain DNS Configuration

The domain:

```text
kuldeep.website
```

must point to the NGINX AWS Load Balancer.

First get the LoadBalancer address:

```bash
kubectl get svc -n ingress-nginx
```

Example:

```text
k8s-ingressn-ingressn-15824d9c84-79da8c209709a4ff.elb.ap-south-1.amazonaws.com
```

In the DNS provider, create:

```text
Type: CNAME
Name: @
Value: <NGINX-LOAD-BALANCER-DNS>
```

If the DNS provider supports ALIAS/ANAME at the root domain, that can also be used.

The important requirement is:

```text
kuldeep.website
        |
        v
NGINX AWS Load Balancer
```

---

# 15. Why DNS Is Required

Let's Encrypt must verify that we control:

```text
kuldeep.website
```

With HTTP-01, Let's Encrypt accesses:

```text
http://kuldeep.website/.well-known/acme-challenge/<token>
```

The request reaches:

```text
Internet
   |
   v
kuldeep.website
   |
   v
AWS Load Balancer
   |
   v
NGINX Ingress
   |
   v
Cert-manager challenge
```

If DNS is incorrect, certificate issuance will fail.

---

# 16. Check Certificate Status

Run:

```bash
kubectl get certificate -n 3-tier-app-eks
```

Expected after successful issuance:

```text
NAME                  READY   SECRET                AGE
kuldeep-website-tls   True    kuldeep-website-tls   5m
```

Check Secret:

```bash
kubectl get secret kuldeep-website-tls -n 3-tier-app-eks
```

Expected:

```text
NAME                  TYPE                DATA
kuldeep-website-tls   kubernetes.io/tls    2
```

---

# 17. Check Certificate Details

Run:

```bash
kubectl describe certificate kuldeep-website-tls \
  -n 3-tier-app-eks
```

Look for:

```text
Type: Ready
Status: True
```

Also check:

```bash
kubectl get certificaterequest -n 3-tier-app-eks
```

And:

```bash
kubectl get order -n 3-tier-app-eks
```

And:

```bash
kubectl get challenge -n 3-tier-app-eks
```

These resources help troubleshoot Let's Encrypt issuance.

---

# 18. Check Ingress

Run:

```bash
kubectl get ingress -n 3-tier-app-eks
```

Expected:

```text
NAME                 CLASS   HOSTS             ADDRESS
three-tier-ingress   nginx   kuldeep.website   <AWS-LOAD-BALANCER>
```

Check details:

```bash
kubectl describe ingress three-tier-ingress \
  -n 3-tier-app-eks
```

---

# 19. HTTP → HTTPS

We configured:

```yaml
nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

and:

```yaml
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

Therefore:

```text
http://kuldeep.website
```

automatically redirects to:

```text
https://kuldeep.website
```

The flow is:

```text
Browser
   |
   | HTTP :80
   v
AWS Load Balancer
   |
   v
NGINX Ingress
   |
   | 301/308 Redirect
   v
https://kuldeep.website
   |
   | HTTPS :443
   v
NGINX Ingress
   |
   v
Application
```

---

# 20. Test HTTP Redirect

Run:

```bash
curl -I http://kuldeep.website
```

You should see a redirect such as:

```text
HTTP/1.1 308 Permanent Redirect
Location: https://kuldeep.website
```

The exact redirect status may vary depending on NGINX configuration.

---

# 21. Test HTTPS

Run:

```bash
curl -I https://kuldeep.website
```

Expected:

```text
HTTP/2 200
```

or another valid application response such as:

```text
HTTP/2 404
```

depending on the application's route.

The important point is that TLS negotiation succeeds.

---

# 22. Test SSL Certificate

Run:

```bash
echo | openssl s_client \
  -connect kuldeep.website:443 \
  -servername kuldeep.website 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates
```

Expected:

```text
subject=CN=kuldeep.website
issuer=C=US, O=Let's Encrypt, CN=...
notBefore=...
notAfter=...
```

The issuer should be:

```text
Let's Encrypt
```

---

# 23. Automatic Renewal

We do NOT create a cron job.

We do NOT manually run Certbot.

Cert-manager continuously watches Certificate resources.

The flow is:

```text
Cert-manager
     |
     | watches Certificate
     v
Certificate approaching expiration
     |
     v
Cert-manager starts renewal
     |
     v
Let's Encrypt
     |
     v
New certificate
     |
     v
Kubernetes TLS Secret updated
     |
     v
NGINX reloads/uses updated certificate
     |
     v
HTTPS continues working
```

This is automatic.

---

# 24. How Renewal Works

For example, if the certificate is valid for approximately 90 days:

```text
Day 0
Certificate issued
       |
       v
Day 1
       |
       v
...
       |
       v
Certificate enters renewal window
       |
       v
Cert-manager requests a new certificate
       |
       v
Let's Encrypt validates domain
       |
       v
New certificate issued
       |
       v
Kubernetes Secret updated
       |
       v
NGINX uses new certificate
```

No cron job is required.

---

# 25. Check Renewal Configuration

Run:

```bash
kubectl describe certificate kuldeep-website-tls \
  -n 3-tier-app-eks
```

Check:

```text
Not Before
Not After
Renewal Time
```

Depending on cert-manager version, renewal information may be visible in the certificate status.

You can also use:

```bash
kubectl get certificate kuldeep-website-tls \
  -n 3-tier-app-eks \
  -o yaml
```

---

# 26. Important: Do Not Delete the TLS Secret

Do not manually delete:

```text
kuldeep-website-tls
```

under normal operation.

Cert-manager owns this Secret because:

```yaml
spec:
  secretName: kuldeep-website-tls
```

When renewal occurs, cert-manager updates the Secret.

---

# 27. Why We Don't Need Certbot

Traditional VM setup:

```text
EC2
 |
 +-- NGINX
 |
 +-- Certbot
 |
 +-- Cron Job
```

We are NOT using this architecture.

Our Kubernetes architecture is:

```text
EKS
 |
 +-- NGINX Ingress Controller
 |
 +-- Cert-manager
 |
 +-- Let's Encrypt
 |
 +-- Kubernetes TLS Secret
```

Therefore:

```text
Certbot cron job = NOT REQUIRED
```

---

# 28. Why We Don't Need ACM

Earlier, the ALB Ingress configuration used:

```yaml
alb.ingress.kubernetes.io/certificate-arn: ...
```

This uses AWS ACM.

We removed that because we decided to use cert-manager.

Do NOT add:

```yaml
alb.ingress.kubernetes.io/certificate-arn
```

to the NGINX Ingress.

The SSL certificate is now stored in Kubernetes:

```text
Secret:
kuldeep-website-tls
```

---

# 29. Old ALB Ingress Configuration

The old configuration looked like:

```yaml
spec:
  ingressClassName: alb
```

and:

```yaml
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
```

That configuration uses:

```text
AWS Load Balancer Controller
        |
        v
AWS ALB
        |
        v
ACM certificate
```

We are no longer using this for the application Ingress.

---

# 30. New NGINX Configuration

The new configuration is:

```yaml
spec:
  ingressClassName: nginx
```

with:

```yaml
tls:
  - hosts:
      - kuldeep.website
    secretName: kuldeep-website-tls
```

and:

```yaml
annotations:
  cert-manager.io/cluster-issuer: letsencrypt-prod
```

This uses:

```text
NGINX Ingress
       |
       v
cert-manager
       |
       v
Let's Encrypt
       |
       v
Kubernetes TLS Secret
```

---

# 31. Verify All Components

Check cert-manager:

```bash
kubectl get pods -n cert-manager
```

Check NGINX:

```bash
kubectl get pods -n ingress-nginx
```

Check ClusterIssuer:

```bash
kubectl get clusterissuer
```

Check Certificate:

```bash
kubectl get certificate -n 3-tier-app-eks
```

Check TLS Secret:

```bash
kubectl get secret kuldeep-website-tls \
  -n 3-tier-app-eks
```

Check Ingress:

```bash
kubectl get ingress -n 3-tier-app-eks
```

Check application:

```bash
kubectl get pods -n 3-tier-app-eks
```

---

# 32. Complete File Structure

The project can contain:

```text
eks-terraform-application/
│
├── README.md
│
├── backend.yaml
├── frontend.yaml
│
├── postgres.yml
├── postgres-pvc.yaml
│
├── clusterissuer.yaml
├── certificate.yaml
│
└── ingress.yml
```

---

# 33. Deployment Order

Recommended deployment order:

### Step 1 - Namespace

```bash
kubectl create namespace 3-tier-app-eks
```

If it already exists, do not recreate it.

---

### Step 2 - PostgreSQL

```bash
kubectl apply -f postgres-pvc.yaml
kubectl apply -f postgres.yml
```

Check:

```bash
kubectl get pvc -n 3-tier-app-eks
kubectl get pods -n 3-tier-app-eks
```

---

### Step 3 - Backend

```bash
kubectl apply -f backend.yaml
```

Check:

```bash
kubectl get pods -n 3-tier-app-eks
```

---

### Step 4 - Frontend

```bash
kubectl apply -f frontend.yaml
```

---

### Step 5 - Cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Install:

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

---

### Step 6 - NGINX Ingress

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Install:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.ingressClassResource.name=nginx \
  --set controller.ingressClassResource.controllerValue=k8s.io/ingress-nginx
```

---

### Step 7 - ClusterIssuer

```bash
kubectl apply -f clusterissuer.yaml
```

Check:

```bash
kubectl get clusterissuer
```

---

### Step 8 - Certificate

```bash
kubectl apply -f certificate.yaml
```

Check:

```bash
kubectl get certificate -n 3-tier-app-eks
```

---

### Step 9 - Ingress

```bash
kubectl apply -f ingress.yml
```

---

### Step 10 - DNS

Point:

```text
kuldeep.website
```

to the NGINX LoadBalancer DNS name.

Get it using:

```bash
kubectl get svc -n ingress-nginx
```

---

# 34. Final Verification

Run:

```bash
kubectl get pods -n cert-manager
```

```bash
kubectl get pods -n ingress-nginx
```

```bash
kubectl get clusterissuer
```

```bash
kubectl get certificate -n 3-tier-app-eks
```

```bash
kubectl get secret kuldeep-website-tls -n 3-tier-app-eks
```

```bash
kubectl get ingress -n 3-tier-app-eks
```

Then:

```bash
curl -I http://kuldeep.website
```

and:

```bash
curl -I https://kuldeep.website
```

Finally:

```bash
echo | openssl s_client \
  -connect kuldeep.website:443 \
  -servername kuldeep.website 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates
```

---

# 35. Final Production Architecture

The final production setup is:

```text
                         INTERNET
                            |
                            |
                     kuldeep.website
                            |
                     DNS / Route 53
                            |
                            v
                AWS Load Balancer
                (NGINX Service)
                            |
                            v
                NGINX Ingress Controller
                            |
             +--------------+--------------+
             |                             |
             | /                           | /api
             v                             v
      Frontend Service              Backend Service
             |                             |
             v                             v
      Frontend Pods                  Backend Pods
                                           |
                                           v
                                      PostgreSQL
                                           |
                                           v
                                      EBS Storage


                 SSL MANAGEMENT
                       |
                       v
                  Cert-manager
                       |
                       v
                 Let's Encrypt
                       |
                       v
              TLS Kubernetes Secret
                       |
                       v
              kuldeep-website-tls
                       |
                       v
                 NGINX Ingress


                 RENEWAL
                       |
                       v
              Cert-manager watches
                 Certificate
                       |
                       v
             Renewal automatically
                       |
                       v
                Let's Encrypt
                       |
                       v
             New certificate issued
                       |
                       v
             TLS Secret updated
                       |
                       v
                NGINX uses it
```

---

# 36. Key Points

## SSL Certificate

Managed by:

```text
Cert-manager + Let's Encrypt
```

## SSL Storage

Stored in:

```text
Kubernetes Secret
```

Secret name:

```text
kuldeep-website-tls
```

## Ingress

Managed by:

```text
NGINX Ingress Controller
```

Ingress class:

```text
nginx
```

## Certificate Issuer

```text
letsencrypt-prod
```

## Domain

```text
kuldeep.website
```

## HTTPS

```text
HTTPS :443
```

## HTTP Redirect

```text
HTTP :80
     |
     v
HTTPS :443
```

## Automatic Renewal

Handled by:

```text
Cert-manager
```

No:

```text
cron job
```

No:

```text
Certbot
```

No:

```text
ACM certificate
```

is required for this setup.

---

# 37. Troubleshooting Commands

### Certificate not ready

```bash
kubectl describe certificate kuldeep-website-tls \
  -n 3-tier-app-eks
```

### CertificateRequest

```bash
kubectl get certificaterequest \
  -n 3-tier-app-eks
```

### ACME Order

```bash
kubectl get order \
  -n 3-tier-app-eks
```

### ACME Challenge

```bash
kubectl get challenge \
  -n 3-tier-app-eks
```

### Cert-manager logs

```bash
kubectl logs \
  -n cert-manager \
  deployment/cert-manager
```

### NGINX logs

```bash
kubectl logs \
  -n ingress-nginx \
  deployment/ingress-nginx-controller
```

### Ingress

```bash
kubectl describe ingress \
  three-tier-ingress \
  -n 3-tier-app-eks
```

### DNS

```bash
nslookup kuldeep.website
```

### HTTPS

```bash
curl -I https://kuldeep.website
```

### Certificate from server

```bash
echo | openssl s_client \
  -connect kuldeep.website:443 \
  -servername kuldeep.website 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates
```

---

# 38. Important Production Note

During certificate testing, avoid repeatedly deleting and recreating the production Certificate.

Let's Encrypt has rate limits.

If you receive:

```text
429 urn:ietf:params:acme:error:rateLimited
```

wait until the indicated retry time.

For future testing, use a Let's Encrypt staging ClusterIssuer so production rate limits are not consumed.

Once the complete configuration is confirmed, use:

```text
letsencrypt-prod
```

for the real production certificate.

---

# 39. Summary

We configured a Kubernetes-native SSL solution:

```text
EKS
 |
 +-- NGINX Ingress Controller
 |
 +-- Cert-manager
 |      |
 |      +-- Let's Encrypt
 |
 +-- Kubernetes TLS Secret
 |
 +-- Frontend
 |
 +-- Backend
 |
 +-- PostgreSQL
```

Certificate lifecycle:

```text
Cert-manager
      |
      v
Let's Encrypt
      |
      v
Certificate
      |
      v
Kubernetes Secret
      |
      v
NGINX
      |
      v
HTTPS
```

Renewal lifecycle:

```text
Cert-manager watches certificate
             |
             v
Certificate enters renewal period
             |
             v
Cert-manager requests renewal
             |
             v
Let's Encrypt validates domain
             |
             v
New certificate
             |
             v
TLS Secret updated
             |
             v
NGINX continues serving HTTPS
```

Therefore, **no manual renewal and no cron job are required**.
