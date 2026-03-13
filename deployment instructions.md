# Hearts Distro — Coming Soon Site
## Deployment Guide for GitHub Pages + heartsdistro.com

> Last updated: March 2026.

---

## What's Included

| File | Purpose |
|---|---|
| `index.html` | Coming soon page with producer/retailer email signup, ToS modal, Privacy Policy modal |
| `.github/workflows/deploy.yml` | Not required for GitHub Pages — you can delete this file |

---

## Step 1: Enable GitHub Pages

1. Push your repository to GitHub if you haven't already
2. Go to your repo → **Settings → Pages**
3. Under **Source**, select **Deploy from a branch**
4. Set branch to `main` and folder to `/ (root)`, click **Save**
5. GitHub will publish your site at `https://YOUR_USERNAME.github.io/YOUR_REPO_NAME` within 1–2 minutes

Every subsequent push to `main` automatically redeploys — no workflow file or configuration needed.

---

## Step 2: Connect Your Custom Domain (heartsdistro.com)

### 2a. Add the domain in GitHub

1. In your repo → **Settings → Pages**, enter `heartsdistro.com` under **Custom domain**
2. Click **Save** — GitHub will commit a `CNAME` file to your repo automatically

### 2b. Add DNS records

In your DNS provider, add the following records:

```
# Root domain — four A records pointing to GitHub's servers
Type: A   Name: @   Value: 185.199.108.153
Type: A   Name: @   Value: 185.199.109.153
Type: A   Name: @   Value: 185.199.110.153
Type: A   Name: @   Value: 185.199.111.153

# www subdomain
Type: CNAME   Name: www   Value: YOUR_USERNAME.github.io
```

### 2c. Enable HTTPS

Once DNS propagates (5–30 minutes), return to **Settings → Pages** and check **Enforce HTTPS**. GitHub provisions a free Let's Encrypt certificate automatically — no configuration required.

To verify DNS is set up correctly before enabling HTTPS:
```bash
dig heartsdistro.com +noall +answer
# Should show the four GitHub IP addresses above
```

---

## Step 3: Set Up Firebase Firestore for Email Collection

### 3a. Create a Firebase Project linked to your GCP project

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Click **Add project**
3. Select your existing GCP project from the dropdown
4. Disable Google Analytics if not needed, click **Create project**

### 3b. Create a Firestore Database

1. Firebase console → **Build → Firestore Database**
2. Click **Create database**
3. Choose **Production mode**
4. Select a region — `us-central` is a safe default
5. Click **Enable**

### 3c. Set Firestore Security Rules

Firebase console → **Firestore → Rules**, paste:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /signups/{docId} {
      allow create: if request.resource.data.keys().hasOnly(['email', 'type', 'timestamp'])
                    && request.resource.data.email is string
                    && request.resource.data.email.size() < 200
                    && request.resource.data.type in ['producer', 'retailer'];
      allow read, update, delete: if false;
    }
  }
}
```

Click **Publish**.

### 3d. Register your web app and get the config

1. Firebase console → gear icon → **Project settings**
2. Scroll to **Your apps** → click **</>** (Web)
3. Name it `heartsdistro-web`, click **Register app**
4. Copy the `firebaseConfig` object and paste it into the placeholder block near the bottom of `index.html`

### 3e. Restrict your API key to your domain

1. Go to [console.cloud.google.com/apis/credentials](https://console.cloud.google.com/apis/credentials)
2. Click the **Browser key** auto-created by Firebase
3. Under **Application restrictions** → **HTTP referrers**, add:
   ```
   https://heartsdistro.com/*
   https://www.heartsdistro.com/*
   ```
4. Click **Save**

### 3f. View your signups

Firebase console → **Firestore Database → Data** → `signups` collection.

| Field | Value |
|---|---|
| `email` | Submitted email address |
| `type` | `producer` or `retailer` |
| `timestamp` | Server-side UTC timestamp |

To export signups via CLI:
```bash
npm install -g firebase-tools
firebase login
firebase firestore:export gs://YOUR_PROJECT_ID.appspot.com/exports/signups
```

---

## Step 4: Clean Up Your GCP Bucket

Since you're hosting on GitHub Pages, the GCP bucket and load balancer resources are no longer needed. GCP charges for load balancer forwarding rules even when idle (~$18/month), so tear them down promptly.

**Important:** GCP won't let you delete a resource that another resource still depends on. Delete in this exact order — outermost resources first.

### 4a. Delete forwarding rules

```bash
# HTTPS forwarding rule
gcloud compute forwarding-rules delete heartsdistro-https-rule --global --quiet

# HTTP forwarding rule (redirect)
gcloud compute forwarding-rules delete heartsdistro-http-rule --global --quiet
```

### 4b. Delete target proxies

```bash
# HTTPS proxy
gcloud compute target-https-proxies delete heartsdistro-https-proxy --global --quiet

# HTTP proxy
gcloud compute target-http-proxies delete heartsdistro-http-proxy --global --quiet
```

### 4c. Delete SSL certificate

```bash
gcloud compute ssl-certificates delete heartsdistro-cert --global --quiet
```

### 4d. Delete URL maps

```bash
gcloud compute url-maps delete heartsdistro-url-map --global --quiet
gcloud compute url-maps delete heartsdistro-http-redirect --global --quiet
```

### 4e. Delete the backend bucket

```bash
gcloud compute backend-buckets delete heartsdistro-backend --global --quiet
```

> Note: deleting the backend bucket does NOT delete the underlying Cloud Storage bucket — that's a separate step below.

### 4f. Release the static IP address

```bash
gcloud compute addresses delete heartsdistro-ip --global --quiet
```

### 4g. Delete the Cloud Storage bucket and its contents

```bash
# Delete all objects in the bucket first
gcloud storage rm --recursive gs://YOUR_BUCKET_NAME/**

# Then delete the bucket itself
gcloud storage buckets delete gs://YOUR_BUCKET_NAME
```

### 4h. Verify everything is removed

```bash
# Should return empty lists for all of these
gcloud compute forwarding-rules list --global
gcloud compute target-https-proxies list --global
gcloud compute target-http-proxies list --global
gcloud compute ssl-certificates list --global
gcloud compute url-maps list --global
gcloud compute backend-buckets list
gcloud compute addresses list --global
gcloud storage buckets list
```

If you created a service account for the GCP deployment workflow, clean that up too:

```bash
gcloud iam service-accounts delete \
  heartsdistro-deployer@YOUR_PROJECT_ID.iam.gserviceaccount.com --quiet
```

---

## Estimated Costs

| Service | Cost |
|---|---|
| GitHub Pages | Free |
| Custom domain HTTPS | Free (Let's Encrypt via GitHub) |
| Firebase Firestore (free tier) | $0/month at signup volumes |
| GCP (after teardown) | $0/month |

---

## Deploying Updates

Push any change to `main` and GitHub Pages will redeploy automatically within 1–2 minutes. No CLI commands or cache invalidation needed.

To check deploy status:
- Go to your repo → **Actions** tab → look for the **pages-build-deployment** workflow

---

## Legal Notes

- Update the **Effective Date** in both the ToS and Privacy Policy modals before going live
- If targeting EU users, add a cookie consent banner (GDPR)
- Replace `hello@heartsdistro.com` throughout the HTML with your actual contact address
- Have a lawyer review both documents before your full platform launch

---

## Need Help?

Email: hello@heartsdistro.com
