# Hearts Distro — Coming Soon Site
## Deployment Guide for GCP + heartsdistro.com

---

## What's Included

| File | Purpose |
|---|---|
| `index.html` | Main coming soon page with email signup, ToS modal, Privacy Policy modal |

---

## Step 1: Set Up GCP Cloud Storage for Static Hosting

### 1a. Create a Bucket

```bash
# Install gcloud CLI if not already: https://cloud.google.com/sdk/docs/install
gcloud auth login

# Create bucket (must match your domain exactly)
gsutil mb -l us-central1 gs://heartsdistro.com

# Also create www redirect bucket
gsutil mb -l us-central1 gs://www.heartsdistro.com
```

### 1b. Make the Bucket Publicly Readable

```bash
gsutil iam ch allUsers:objectViewer gs://heartsdistro.com
```

### 1c. Configure as a Website

```bash
gsutil web set -m index.html -e 404.html gs://heartsdistro.com
```

### 1d. Upload Your Files

```bash
gsutil cp index.html gs://heartsdistro.com/
```

### 1e. Set www to redirect to root

```bash
gsutil web set -r heartsdistro.com gs://www.heartsdistro.com
```

---

## Step 2: Point Your Domain to GCP

GCP Cloud Storage static sites are served via a CNAME. In your DNS provider (wherever heartsdistro.com is registered):

```
Type: CNAME
Name: www
Value: c.storage.googleapis.com
TTL: 300

Type: A
Name: @  (root domain)
Value: (Use Load Balancer IP — see Step 3 for HTTPS)
```

> ⚠️ **Note:** Direct CNAME on the root domain isn't possible without a Load Balancer for HTTPS. See Step 3.

---

## Step 3: Enable HTTPS (Strongly Recommended)

For HTTPS on the root domain (heartsdistro.com), use a **GCP Load Balancer + Cloud CDN**:

```bash
# 1. Reserve a static IP
gcloud compute addresses create heartsdistro-ip --global

# Get the IP
gcloud compute addresses describe heartsdistro-ip --global

# 2. Create a backend bucket
gcloud compute backend-buckets create heartsdistro-backend \
  --gcs-bucket-name=heartsdistro.com \
  --enable-cdn

# 3. Create URL map
gcloud compute url-maps create heartsdistro-lb \
  --default-backend-bucket=heartsdistro-backend

# 4. Create SSL cert (Google-managed, automatic)
gcloud compute ssl-certificates create heartsdistro-cert \
  --domains=heartsdistro.com,www.heartsdistro.com \
  --global

# 5. Create HTTPS proxy
gcloud compute target-https-proxies create heartsdistro-https-proxy \
  --url-map=heartsdistro-lb \
  --ssl-certificates=heartsdistro-cert

# 6. Create forwarding rule
gcloud compute forwarding-rules create heartsdistro-https \
  --address=heartsdistro-ip \
  --global \
  --target-https-proxy=heartsdistro-https-proxy \
  --ports=443

# 7. Add HTTP -> HTTPS redirect
gcloud compute url-maps import heartsdistro-http-redirect \
  --global \
  --source /dev/stdin << 'EOF'
name: heartsdistro-http-redirect
defaultUrlRedirect:
  redirectResponseCode: MOVED_PERMANENTLY_DEFAULT
  httpsRedirect: true
EOF
```

Then update DNS A record to point to the static IP from step 1.

**SSL cert takes ~15–30 min to provision after DNS propagates.**

---

## Step 4: Set Up Firebase Firestore for Email Collection

### 4a. Create a Firebase Project (linked to your GCP project)

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Click **Add project**
3. Select your existing GCP project from the dropdown (this links Firebase to your GCP billing account)
4. Disable Google Analytics if you don't need it, then click **Create project**

### 4b. Create a Firestore Database

1. In the Firebase console, go to **Build → Firestore Database**
2. Click **Create database**
3. Choose **Production mode** (you'll set security rules next)
4. Select a region close to your users — `us-central` is a safe default
5. Click **Enable**

### 4c. Set Firestore Security Rules

In the Firebase console go to **Firestore → Rules** and paste:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /signups/{docId} {
      // Allow anyone to create a signup, nobody to read/update/delete
      allow create: if request.resource.data.keys().hasOnly(['email', 'type', 'timestamp'])
                    && request.resource.data.email is string
                    && request.resource.data.email.size() < 200
                    && request.resource.data.type in ['producer', 'retailer'];
      allow read, update, delete: if false;
    }
  }
}
```

Click **Publish**. This ensures the public can only write new signups — they cannot read, modify, or delete any data.

### 4d. Register Your Web App and Get Config

1. In the Firebase console, click the gear icon → **Project settings**
2. Scroll to **Your apps** → click the **</>** (Web) icon
3. Give it a nickname (e.g. `heartsdistro-web`), click **Register app**
4. Copy the `firebaseConfig` object — it looks like:

```javascript
const firebaseConfig = {
  apiKey:            "AIzaSy...",
  authDomain:        "your-project.firebaseapp.com",
  projectId:         "your-project-id",
  storageBucket:     "your-project.appspot.com",
  messagingSenderId: "123456789",
  appId:             "1:123456789:web:abc123"
};
```

5. Open `index.html` and replace the placeholder values in the `firebaseConfig` block near the bottom of the file

### 4e. Restrict Your API Key (Important)

The API key in the config is visible in your HTML source. Restrict it so it can only be used from your domain:

1. Go to [console.cloud.google.com/apis/credentials](https://console.cloud.google.com/apis/credentials)
2. Click on your **Browser key** (auto-created by Firebase)
3. Under **Application restrictions** → select **HTTP referrers**
4. Add:
   ```
   https://heartsdistro.com/*
   https://www.heartsdistro.com/*
   ```
5. Click **Save**

### 4f. View Your Signups

In the Firebase console go to **Firestore Database → Data**. You'll see a `signups` collection. Each document contains:

| Field | Value |
|---|---|
| `email` | The submitted email address |
| `type` | `producer` or `retailer` |
| `timestamp` | Server-side UTC timestamp |

To export to CSV, use the Firebase CLI:

```bash
npm install -g firebase-tools
firebase login
firebase firestore:export gs://your-project.appspot.com/exports/signups
```

Or view and export directly from the GCP console under **Firestore → Import/Export**.

---

## Step 5: Quick Update Workflow

After editing `index.html` locally, re-deploy with:

```bash
gsutil cp index.html gs://heartsdistro.com/
gsutil setmeta -h "Cache-Control:no-cache" gs://heartsdistro.com/index.html
```

---

## Estimated Costs

| Service | Cost |
|---|---|
| Cloud Storage (< 1GB) | ~$0.02/month |
| Cloud CDN + Load Balancer | ~$15–20/month |
| SSL Certificate | Free (Google-managed) |
| Formspree Free Tier | 50 submissions/month free |

> For a temporary coming soon page, total cost is very low — typically **under $25/month** depending on traffic.

---

## Legal Notes

- Update the **Effective Date** in both ToS and Privacy Policy modals once you go live
- If you're targeting EU users, consider adding a cookie consent banner
- Update the contact email (`hello@heartsdistro.com`) throughout the HTML
- Consult a lawyer before your full product launch for comprehensive legal docs

---

## Need Help?

Email: hello@heartsdistro.com
