# AWS Real User Monitoring (RUM) Implementation Analysis

## Domain: `partner.coderdemo.io`
## Distribution: `E2MVGWVLS4TJ0J` (CloudFront)
## Current Microsites: `/aws+coder`, `/flywheeldata`, `/wwt-fsi-bmo`

---

## What AWS RUM Provides

CloudWatch RUM captures real browser telemetry from visitors to your site:

| Category | Metrics |
|----------|---------|
| **Web Vitals** | LCP, FID, CLS, INP (Core Web Vitals) |
| **Navigation** | Full page load time, resource durations, Apdex scoring |
| **Page Views** | Count, pages/session, bounce rate |
| **Sessions** | Total sessions, duration, geography |
| **Errors** | JavaScript errors, HTTP 4xx/5xx, error rate per page |
| **HTTP** | XHR/Fetch call latency, status codes, failed requests |

All metrics publish to the `AWS/RUM` CloudWatch namespace with `application_name` dimension.

---

## Implementation Requirements

### 1. Amazon Cognito Identity Pool (Required)

RUM uses an **unauthenticated** Cognito Identity Pool to grant temporary AWS credentials to browsers so they can call `rum:PutRumEvents`.

```bash
# Create identity pool (allow unauthenticated access)
aws cognito-identity create-identity-pool \
  --identity-pool-name "partner-coderdemo-rum" \
  --allow-unauthenticated-identities \
  --region us-east-1
```

Then attach an IAM role to the unauthenticated identity with this policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "rum:PutRumEvents",
      "Resource": "arn:aws:rum:us-east-1:227462955655:appmonitor/partner-coderdemo-microsites"
    }
  ]
}
```

### 2. Create RUM App Monitor

```bash
aws rum create-app-monitor \
  --name "partner-coderdemo-microsites" \
  --domain "partner.coderdemo.io" \
  --app-monitor-configuration '{
    "IdentityPoolId": "us-east-1:<IDENTITY_POOL_ID>",
    "SessionSampleRate": 1.0,
    "Telemetries": ["errors", "performance", "http"],
    "AllowCookies": false,
    "EnableXRay": false
  }' \
  --cw-log-enabled \
  --region us-east-1
```

**Configuration choices:**
| Setting | Value | Rationale |
|---------|-------|-----------|
| `SessionSampleRate` | `1.0` | 100% — low traffic site, want complete visibility |
| `Telemetries` | `["errors", "performance", "http"]` | All three for full picture |
| `AllowCookies` | `false` | Avoids cookie consent requirement for partner-facing site |
| `EnableXRay` | `false` | Not needed for static content (no server-side traces to correlate) |
| `--cw-log-enabled` | yes | Enables Logs Insights queries on raw RUM events |

### 3. JavaScript Snippet (Add to `index.html`)

Place in `<head>` before any other scripts:

```html
<script>
(function(n,i,v,r,s,c,u,x,z){x=window.AwsRumClient={q:[],n:n,i:i,v:v,r:r,c:c,u:u};
window[n]=function(c,p){x.q.push({c:c,p:p});};z=document.createElement('script');
z.async=true;z.src=s;document.head.insertBefore(z,document.getElementsByTagName('script')[0]);
})(
  'cwr',
  '<APP_MONITOR_ID>',
  '1.0.0',
  'us-east-1',
  '/aws+coder/cwr.js',
  {
    sessionSampleRate: 1,
    identityPoolId: 'us-east-1:<IDENTITY_POOL_ID>',
    endpoint: 'https://dataplane.rum.us-east-1.amazonaws.com',
    telemetries: ['performance', 'errors', 'http'],
    allowCookies: false,
    enableXRay: false
  }
);
</script>
```

**Note:** The snippet references `/aws+coder/cwr.js` (self-hosted) rather than the AWS CDN URL. This avoids ad-blocker interference.

### 4. Self-Host the RUM Client Library

```bash
# Download the RUM client JS
curl -o cwr.js "https://client.rum.us-east-1.amazonaws.com/1.18.0/cwr.js"

# Upload to S3
aws s3 cp cwr.js s3://aws-coder-microsites-coderdemo/aws+coder/cwr.js \
  --content-type "application/javascript" \
  --cache-control "public, max-age=31536000, immutable"
```

Self-hosting is recommended because:
- Ad blockers frequently block `*.amazonaws.com` script sources
- Reduces external dependencies
- Loads from same CloudFront distribution (faster, no additional DNS lookup)

### 5. CloudFront Response Headers Policy (CSP)

Create a response headers policy to allow the RUM dataplane connection:

```bash
aws cloudfront create-response-headers-policy \
  --response-headers-policy-config '{
    "Name": "partner-coderdemo-rum-headers",
    "Comment": "CSP headers for AWS RUM integration",
    "SecurityHeadersConfig": {
      "ContentSecurityPolicy": {
        "ContentSecurityPolicy": "default-src '\''self'\''; script-src '\''self'\'' '\''unsafe-inline'\''; connect-src '\''self'\'' https://dataplane.rum.us-east-1.amazonaws.com https://cognito-identity.us-east-1.amazonaws.com; img-src '\''self'\'' data: https://fonts.gstatic.com; font-src '\''self'\'' https://fonts.googleapis.com https://fonts.gstatic.com; style-src '\''self'\'' '\''unsafe-inline'\''",
        "Override": true
      },
      "StrictTransportSecurity": {
        "AccessControlMaxAgeSec": 31536000,
        "IncludeSubdomains": true,
        "Preload": true,
        "Override": true
      }
    }
  }' \
  --region us-east-1
```

Then attach this policy to the `/aws+coder` and `/aws+coder/*` cache behaviors.

### 6. IAM Permissions Needed

| Principal | Permissions Required |
|-----------|---------------------|
| Admin (setup) | `rum:CreateAppMonitor`, `rum:GetAppMonitor`, `cognito-identity:CreateIdentityPool`, `iam:CreateRole`, `iam:AttachRolePolicy` |
| Cognito unauth role (browser) | `rum:PutRumEvents` (scoped to app monitor ARN) |
| Dashboard viewer | `rum:GetAppMonitorData`, `rum:ListAppMonitors`, `cloudwatch:GetMetricData` |

---

## What You'll See in CloudWatch

Once implemented, the RUM console at `CloudWatch → Application Monitoring → RUM` shows:

### Overview Dashboard (automatic)
- **Page load performance** — histogram of navigation duration
- **Web Vitals** — LCP, FID, CLS with Good/Needs Improvement/Poor breakdown
- **Errors** — JS error count, top errors by message
- **Sessions** — Active users, session duration, geography map

### Per-Page Breakdown
- `/aws+coder` — load time, error rate, web vitals
- `/aws+coder/styles.css` — resource load time
- `/aws+coder/aws-coder-refarch.png` — image load time

### Custom Queries (via Logs Insights)
```sql
-- Top pages by load time
fields @timestamp, event_type, event_details.navigationTiming.duration
| filter event_type = "com.amazon.rum.performance_navigation_event"
| stats avg(event_details.navigationTiming.duration) as avg_load by metadata.pageId
| sort avg_load desc
```

---

## Pricing Estimate

| Traffic Level | Events/Month | Monthly Cost |
|---------------|-------------|-------------|
| 500 visits/month | ~10,000 | Free (within 1M free tier) |
| 5,000 visits/month | ~100,000 | Free |
| 50,000 visits/month | ~1,000,000 | ~$10 |

For `partner.coderdemo.io` (internal AWS account team + customer distribution), expect <5K visits/month → **$0/month** (free tier).

---

## Implementation Checklist

| Step | Action | Dependency |
|------|--------|-----------|
| 1 | Request IAM permissions for `rum:*` and `cognito-identity:*` | Account admin |
| 2 | Create Cognito Identity Pool (unauthenticated) | Step 1 |
| 3 | Create IAM role for unauth identity with `rum:PutRumEvents` | Step 2 |
| 4 | Create RUM App Monitor for `partner.coderdemo.io` | Steps 2-3 |
| 5 | Download + self-host `cwr.js` in S3 bucket | Step 4 (need app monitor ID) |
| 6 | Add RUM snippet to `index.html` `<head>` | Step 5 |
| 7 | Create CloudFront Response Headers Policy (CSP) | Independent |
| 8 | Attach response headers policy to `/aws+coder*` behaviors | Step 7 |
| 9 | Deploy to S3, invalidate CloudFront | Steps 5-8 |
| 10 | Verify events appear in RUM console (~60s) | Step 9 |

---

## Gotchas & Considerations

| Issue | Impact | Mitigation |
|-------|--------|-----------|
| Ad blockers block AWS CDN scripts | ~15-25% of users may not report | Self-host `cwr.js` in same S3 bucket |
| `AllowCookies: true` requires consent banner | GDPR/ePrivacy compliance | Set `false` — loses cross-session user tracking but avoids legal requirement |
| CloudFront HTML caching | Snippet updates won't propagate until cache expires | Use short TTL or invalidate after changes |
| Cognito identity pool region | Must match app monitor region | Both in `us-east-1` |
| Multiple microsites on same domain | Single app monitor covers all paths | Use `pageId` dimension to filter by `/aws+coder` vs `/flywheeldata` |
| Current IAM role lacks RUM permissions | Cannot create resources from this workspace | Requires admin to grant `rum:*` + `cognito-identity:*` |

---

## Cross-Microsite Coverage

A single RUM App Monitor with domain `partner.coderdemo.io` will automatically capture traffic across **all** microsites on this domain:

- `/aws+coder` — AWS + Coder Better Together
- `/flywheeldata` — Flywheel Data microsite
- `/wwt-fsi-bmo` — WWT FSI BMO microsite

The `pageId` dimension in CloudWatch metrics distinguishes traffic by path, so you get per-microsite analytics without creating separate monitors.
