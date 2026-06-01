# AWS + Coder Agentic AI Microsite

Co-branded microsite highlighting how Coder's Agent-Ready Workspaces and AWS Bedrock/AgentCore combine to deliver enterprise-grade agentic AI development at scale.

**Live URL:** https://partner.coderdemo.io/aws+coder

## Overview

This microsite is designed for distribution to AWS Account Teams to highlight the AWS + Coder "better together" story. It covers:

- **Enterprise AI Agent Challenge** — Security, consistency, and scale gaps solved by Coder + AWS
- **Live Showcase** — [DC Investments Agent](https://dcagent.coderdemo.io) built with Coder Agents + Kiro in governed workspaces
- **Migrate, Modernize, Multiply** — Coder's framework for cloud-native dev environments on AWS
- **Reference Architecture** — Marketplace deployment, agent-ready workspaces, Bedrock/AgentCore integration
- **Revenue Impact** — $1.2M–$2.0M estimated AWS ARR for a 250-developer deployment (2 vCPU / 8GB / 30GB per workspace)
- **Kiro + Coder** — AWS's AI IDE integrated with governed Coder workspaces
- **Clash of Agents GameDay** — CTA to request a hands-on workshop/GameDay

## Tech Stack

- Static HTML/CSS (no build step, no dependencies)
- Hosted on S3 + CloudFront
- AWS system fonts (no external font loading)
- Branding: AWS Products visual language (light mode) + Coder brand accents (ember, violet)

## Deployment

The site is deployed to S3 with CloudFront serving it at the `/aws+coder` path on `partner.coderdemo.io`.

### Deploy manually

```bash
# Upload files
aws s3 sync . s3://partner-coderdemo-flywheeldata/aws+coder/ \
  --exclude ".git/*" --exclude "README.md" --exclude "package.json" \
  --content-type "text/html" --exclude "*.css" --exclude "*.png"

aws s3 cp styles.css s3://partner-coderdemo-flywheeldata/aws+coder/styles.css \
  --content-type "text/css"

aws s3 cp aws-competency.png s3://partner-coderdemo-flywheeldata/aws+coder/aws-competency.png \
  --content-type "image/png"

# Also copy index.html to bare path (for /aws+coder without trailing slash)
aws s3 cp index.html s3://partner-coderdemo-flywheeldata/aws+coder \
  --content-type "text/html"

# Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id E2MVGWVLS4TJ0J \
  --paths "/aws+coder*" "/aws%2Bcoder*"
```

### Infrastructure

| Component | Detail |
|-----------|--------|
| S3 Bucket | `partner-coderdemo-flywheeldata` (prefix: `aws+coder/`) |
| CloudFront | Distribution `E2MVGWVLS4TJ0J` |
| CloudFront Function | `aws-coder-url-rewrite` — handles `+` URL encoding and default document |
| Domain | `partner.coderdemo.io` (shared with Coder instance) |

## Files

| File | Purpose |
|------|---------|
| `index.html` | Single-page microsite |
| `styles.css` | Full CSS design system (AWS + Coder co-branded) |
| `aws-competency.png` | AWS Competency Partner Program badge (from coder.com/partners/aws) |

## Key References

- [DC Investments Agent (Live Demo)](https://dcagent.coderdemo.io)
- [Showcase Source Repo](https://github.com/greg-the-coder/aws-coder-agentic-ai-showcase)
- [Coder + AWS Partners Page](https://coder.com/partners/aws)
- [AWS Marketplace Install](https://coder.com/docs/install/cloud/aws-marketplace)
- [Clash of Agents Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/0e51c8ed-274c-4bf1-97b1-7758d17d5e1b/en-US)
- [Kiro IDE](https://kiro.dev)

## Revenue Estimate Assumptions (250 developers)

- Average workspace: 2 vCPU / 8GB RAM / 30GB gp3 EBS
- ~60% business-hours utilization + 20% off-hours agent activity
- 400 AI agent sessions/month per developer via Kiro and autonomous agents
- Mix of Claude Sonnet, Mistral Large, and Titan models via Bedrock
- US-East-1 pricing with 1-year savings plans where applicable
