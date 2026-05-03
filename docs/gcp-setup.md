# GCP Static Website Hosting Setup Guide

This guide walks through deploying the Veronica Mumm's Music website to a Google Cloud Storage bucket with a custom domain.

---

## Prerequisites

- A Google account
- A registered domain name
- Access to your domain's DNS settings
- [Google Cloud SDK](https://cloud.google.com/sdk/docs/install) installed (for CLI commands), or use the Cloud Console web UI

---

## 1. Create a GCP Project

### Via Console (console.cloud.google.com)

1. Click the project dropdown (top bar) → **New Project**
2. Project name: `veronica-mumm-music` (or similar)
3. Note the **Project ID** — you'll need it for CLI commands
4. Click **Create**

### Via CLI

```bash
gcloud projects create veronica-mumm-music --name="Veronica Mumm Music"
gcloud config set project veronica-mumm-music
```

### Enable Billing

Even for near-free usage, GCP requires a billing account:

1. Go to **Billing** in the Console
2. Link a billing account to the project
3. GCS costs for a small static site: **< $1/month**

---

## 2. IAM Roles — Admin (Not Owner)

> **Why?** The project creator is automatically granted the **Owner** role. To prepare for future ownership transfer, you should understand the role separation.

### Current State After Project Creation

- You (the creator) are **Owner** by default
- This is fine for now — you need Owner to set up billing and initial resources

### Preparing for Transfer (do this later, when ready)

When the new owner is identified:

```bash
# 1. Add the new owner
gcloud projects add-iam-policy-binding veronica-mumm-music \
  --member="user:newowner@example.com" \
  --role="roles/owner"

# 2. Downgrade yourself to admin + storage admin
gcloud projects add-iam-policy-binding veronica-mumm-music \
  --member="user:you@example.com" \
  --role="roles/resourcemanager.projectIamAdmin"

gcloud projects add-iam-policy-binding veronica-mumm-music \
  --member="user:you@example.com" \
  --role="roles/storage.admin"

# 3. Remove yourself as owner (the new owner must do this, or you can)
gcloud projects remove-iam-policy-binding veronica-mumm-music \
  --member="user:you@example.com" \
  --role="roles/owner"
```

> **Important:** A project must always have at least one Owner. Add the new owner *before* removing yourself.

---

## 3. Create a GCS Bucket

The bucket name **must exactly match your domain** for CNAME-based hosting.

```bash
# For a www subdomain (recommended — CNAME works directly)
gsutil mb -p veronica-mumm-music -l us-east1 gs://www.yourdomain.com

# Verify
gsutil ls
```

**Region choice:** `us-east1` (South Carolina) is the closest GCP region to Richmond, VA.

### Via Console

1. Go to **Cloud Storage** → **Buckets** → **Create**
2. Bucket name: `www.yourdomain.com`
3. Location: `us-east1`
4. Storage class: **Standard**
5. Access control: **Uniform**
6. Click **Create**

---

## 4. Configure Static Website Hosting

```bash
gsutil web set -m index.html -e 404.html gs://www.yourdomain.com
```

This tells GCS to:
- Serve `index.html` as the default page
- Serve `404.html` for missing pages

### Verify Configuration

```bash
gsutil web get gs://www.yourdomain.com
```

Expected output:
```json
{"mainPageSuffix": "index.html", "notFoundPage": "404.html"}
```

---

## 5. Make the Bucket Public

```bash
gsutil iam ch allUsers:objectViewer gs://www.yourdomain.com
```

### Via Console

1. Go to your bucket → **Permissions** tab
2. Click **Grant Access**
3. Principal: `allUsers`
4. Role: **Storage Object Viewer**
5. Click **Save**

> You'll see a warning that this makes the bucket public. That's expected for a website.

---

## 6. Upload Site Files

From the project root directory:

```bash
gsutil -m cp index.html gs://www.yourdomain.com/
gsutil -m cp 404.html gs://www.yourdomain.com/
gsutil -m cp -r css/ gs://www.yourdomain.com/
```

### Set Cache Headers (optional, improves performance)

```bash
gsutil -m setmeta \
  -h "Cache-Control:public, max-age=3600" \
  gs://www.yourdomain.com/**
```

### Verify Upload

```bash
gsutil ls gs://www.yourdomain.com/
```

Expected:
```
gs://www.yourdomain.com/index.html
gs://www.yourdomain.com/404.html
gs://www.yourdomain.com/css/
```

---

## 7. Configure DNS

### Verify Domain Ownership

Before GCS will serve your bucket via a custom domain, you must verify you own the domain:

1. Go to [Google Search Console](https://search.google.com/search-console)
2. Add your domain
3. Follow the TXT record verification steps
4. Or use the `google-site-verification` CNAME method

### Add DNS Records

At your domain registrar / DNS provider, add:

```
Type: CNAME
Host: www
Value: c.storage.googleapis.com.
TTL: 3600 (or default)
```

### Apex Domain (yourdomain.com without www)

GCS **does not support** apex domains directly via CNAME. Options:

1. **Use `www.yourdomain.com` as your canonical URL** (simplest)
2. **Set up an HTTP redirect** from apex → www at your DNS provider (many registrars support this)
3. **Use a DNS provider that supports CNAME flattening** (e.g., Cloudflare) — set a CNAME at the apex that resolves to `c.storage.googleapis.com.`

---

## 8. Verify Deployment

After DNS propagates (can take minutes to 48 hours):

```bash
# Test the main page
curl -I http://www.yourdomain.com/

# Test a non-existent page (should return 404.html)
curl http://www.yourdomain.com/nonexistent

# Or just open in a browser
open http://www.yourdomain.com/
```

You can also test immediately via the direct GCS URL:
```
http://storage.googleapis.com/www.yourdomain.com/index.html
```

---

## 9. HTTPS Options

> **Important:** Direct GCS bucket hosting does **not** support HTTPS on custom domains.

### Option A: Cloudflare Free Tier (Recommended for Free)

1. Create a free Cloudflare account
2. Add your domain to Cloudflare
3. Update your domain's nameservers to Cloudflare's
4. Cloudflare provides:
   - Free SSL/TLS certificate
   - Proxies traffic to your GCS bucket
   - CDN caching
   - DDoS protection
5. In Cloudflare, set SSL mode to **Flexible** (since GCS serves HTTP)
6. Enable **Always Use HTTPS** to redirect HTTP → HTTPS

### Option B: Firebase Hosting (Free Tier)

Firebase Hosting provides free HTTPS and can be backed by GCS:

```bash
# Install Firebase CLI
npm install -g firebase-tools

# Initialize
firebase login
firebase init hosting

# Deploy
firebase deploy
```

Firebase free tier: 10 GB storage, 360 MB/day transfer.

---

## 10. Updating the Site

When you make changes to the site files:

```bash
# Upload updated files
gsutil -m cp index.html gs://www.yourdomain.com/
gsutil -m cp -r css/ gs://www.yourdomain.com/

# Optionally invalidate cache (if using Cloudflare, purge from their dashboard)
```

---

## 11. Future Ownership Transfer

When ready to transfer to the new owner:

### Option A: Transfer the Entire Project

```bash
# 1. Add new owner
gcloud projects add-iam-policy-binding veronica-mumm-music \
  --member="user:newowner@example.com" \
  --role="roles/owner"

# 2. New owner accepts and verifies access

# 3. Remove yourself as owner
gcloud projects remove-iam-policy-binding veronica-mumm-music \
  --member="user:you@example.com" \
  --role="roles/owner"

# 4. Transfer billing to new owner's billing account (new owner does this in Console)
```

### Option B: New Owner Creates Their Own Bucket

```bash
# 1. New owner creates a new project and bucket
# 2. Copy site files to new bucket:
gsutil -m cp -r gs://www.yourdomain.com/* gs://new-owners-bucket/

# 3. New owner configures their bucket (steps 4-5 above)

# 4. Update DNS CNAME to point at new bucket
#    (same CNAME value: c.storage.googleapis.com. — just need the bucket name to match the domain)

# 5. If using Cloudflare, no DNS change needed — just update the origin
```

> **DNS propagation:** Typically minutes to a few hours, but can take up to 48 hours.

---

## Cost Summary

| Component | Monthly Cost |
|-----------|-------------|
| Storage (< 1 GB) | ~$0.02 |
| Class A operations (uploads) | ~$0.01 |
| Class B operations (reads) | ~$0.01 |
| Network egress (light traffic) | Free (< 1 GB/month within NA) |
| **Total** | **< $1/month** |

**Free tier (new accounts):** GCS offers 5 GB free in `us-east1`, `us-west1`, or `us-central1`.

---

## Quick Reference

| Task | Command |
|------|---------|
| Upload files | `gsutil -m cp -r ./* gs://www.yourdomain.com/` |
| Check bucket config | `gsutil web get gs://www.yourdomain.com` |
| List bucket contents | `gsutil ls gs://www.yourdomain.com/` |
| Remove a file | `gsutil rm gs://www.yourdomain.com/old-file.html` |
| Check permissions | `gsutil iam get gs://www.yourdomain.com` |
| View bucket in browser | `http://storage.googleapis.com/www.yourdomain.com/index.html` |
