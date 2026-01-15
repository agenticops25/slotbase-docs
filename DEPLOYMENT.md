# SlotBase Deployment Guide

This guide covers deploying SlotBase to production environments.

## Architecture Overview

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Mobile App    │     │    Web App      │     │   API Server    │
│   (Expo/EAS)    │────▶│   (Vercel)      │────▶│   (Render)      │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                         │
                        ┌────────────────────────────────┼────────────────────────────────┐
                        │                                │                                │
                        ▼                                ▼                                ▼
                ┌───────────────┐              ┌───────────────┐              ┌───────────────┐
                │  PostgreSQL   │              │    Redis      │              │    Sentry     │
                │   (Neon)      │              │  (Upstash)    │              │  (Monitoring) │
                └───────────────┘              └───────────────┘              └───────────────┘
```

## Prerequisites

- GitHub repository access
- Neon PostgreSQL account
- Upstash Redis account
- Clerk account
- Sentry account
- Vercel account
- Render account
- Expo/EAS account

---

## 1. Database Setup (Neon)

### Create Neon Project

1. Go to [neon.tech](https://neon.tech) and create a project
2. Create two branches:
   - `main` (production)
   - `development` (staging)
3. Copy the connection strings

### Environment Variables

```bash
DATABASE_URL="postgresql://user:pass@host/dbname?sslmode=require"
DIRECT_URL="postgresql://user:pass@host/dbname?sslmode=require"
```

### Run Migrations

```bash
cd api
npx prisma migrate deploy
npx prisma db seed
```

---

## 2. Redis Setup (Upstash)

### Create Upstash Database

1. Go to [upstash.com](https://upstash.com) and create a Redis database
2. Enable TLS
3. Copy the connection URL

### Environment Variable

```bash
REDIS_URL="rediss://default:password@host:6379"
```

---

## 3. Clerk Authentication

### Configure Clerk

1. Go to [clerk.com](https://clerk.com) and create an application
2. Enable Email + Google OAuth sign-in methods
3. Configure webhook endpoint: `https://api.yourdomain.com/api/v1/webhooks/clerk`
4. Copy the API keys

### Environment Variables

```bash
# API
CLERK_SECRET_KEY="sk_live_..."
CLERK_WEBHOOK_SECRET="whsec_..."

# Web
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="pk_live_..."
CLERK_SECRET_KEY="sk_live_..."

# Mobile
EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY="pk_live_..."
```

---

## 4. Sentry Error Tracking

### Create Sentry Projects

Create three projects in Sentry:
- `slotbase-api` (Node.js)
- `slotbase-web` (Next.js)
- `slotbase-mobile` (React Native)

### Environment Variables

```bash
# API
SENTRY_DSN="https://...@sentry.io/..."

# Web
NEXT_PUBLIC_SENTRY_DSN="https://...@sentry.io/..."
SENTRY_ORG="your-org"
SENTRY_PROJECT="slotbase-web"

# Mobile
EXPO_PUBLIC_SENTRY_DSN="https://...@sentry.io/..."
```

---

## 5. API Deployment (Render)

### Deploy to Render

1. Go to [render.com](https://render.com) and create a new Web Service
2. Connect your GitHub repository (`slotbase-api`)
3. Render will detect the `Dockerfile` or use Node.js buildpack

### Configure Environment Variables

In Render dashboard, add these variables:

```bash
NODE_ENV=production
PORT=3001
DATABASE_URL=<from Neon>
DIRECT_URL=<from Neon>
REDIS_URL=<from Upstash>
CLERK_SECRET_KEY=<from Clerk>
CLERK_WEBHOOK_SECRET=<from Clerk>
SENTRY_DSN=<from Sentry>
RESEND_API_KEY=<from Resend>
```

### Custom Domain

1. Add a custom domain in Render (e.g., `api.slotbase.com`)
2. Configure DNS CNAME record pointing to Render

### Post-Deployment

```bash
# Run migrations (via Render shell or deploy script)
npx prisma migrate deploy

# Seed database (first time only)
npx prisma db seed
```

---

## 6. Web Deployment (Vercel)

### Deploy to Vercel

1. Go to [vercel.com](https://vercel.com) and import the `slotbase-web` repository
2. Vercel will auto-detect Next.js

### Configure Environment Variables

```bash
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=<from Clerk>
CLERK_SECRET_KEY=<from Clerk>
NEXT_PUBLIC_API_URL=https://api.slotbase.com
NEXT_PUBLIC_SENTRY_DSN=<from Sentry>
SENTRY_ORG=<your-org>
SENTRY_PROJECT=slotbase-web
```

### Custom Domain

1. Add custom domain in Vercel (e.g., `app.slotbase.com`)
2. Configure DNS records as instructed by Vercel

---

## 7. Mobile Deployment (EAS)

### Install EAS CLI

```bash
npm install -g eas-cli
eas login
```

### Configure EAS

```bash
cd mobile
eas build:configure
```

### Create `eas.json`

```json
{
  "cli": {
    "version": ">= 7.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {}
  },
  "submit": {
    "production": {}
  }
}
```

### Environment Variables in EAS

```bash
eas secret:create --name EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY --value "pk_live_..."
eas secret:create --name EXPO_PUBLIC_API_URL --value "https://api.slotbase.com"
eas secret:create --name EXPO_PUBLIC_SENTRY_DSN --value "https://...@sentry.io/..."
```

### Build Commands

```bash
# Development build (for internal testing)
eas build --profile development --platform all

# Preview build (for TestFlight/Internal testing)
eas build --profile preview --platform all

# Production build
eas build --profile production --platform all
```

### Submit to App Stores

```bash
# iOS App Store
eas submit --platform ios

# Google Play Store
eas submit --platform android
```

---

## 8. DNS Configuration

### Recommended Domain Structure

| Domain | Service | Purpose |
|--------|---------|---------|
| slotbase.com | Vercel | Marketing site |
| app.slotbase.com | Vercel | Web application |
| api.slotbase.com | Render | API server |

### DNS Records

```
# A/CNAME records for Vercel
app.slotbase.com    CNAME   cname.vercel-dns.com

# CNAME for Render
api.slotbase.com    CNAME   <render-provided-domain>
```

---

## 9. Monitoring & Alerts

### Health Checks

- API: `https://api.slotbase.com/api/v1` (returns health status)
- Web: Vercel provides automatic health checks

### Sentry Alerts

Configure alerts in Sentry for:
- New errors
- Error rate spikes
- Performance degradation

### Uptime Monitoring

Consider using:
- [Better Uptime](https://betteruptime.com)
- [UptimeRobot](https://uptimerobot.com)

---

## 10. Rollback Procedures

### API (Render)

1. Go to Render dashboard → Your Service → Deploys
2. Click on a previous successful deploy
3. Click "Rollback to this deploy"

### Web (Vercel)

1. Go to Vercel dashboard → Deployments
2. Click on previous deployment
3. Click "Promote to Production"

### Mobile

- Submit a new build with the fix
- For critical issues, use EAS Update for OTA updates:
  ```bash
  eas update --branch production
  ```

---

## Environment Variables Summary

### API (.env.production)

```bash
NODE_ENV=production
PORT=3001
DATABASE_URL=postgresql://...
DIRECT_URL=postgresql://...
REDIS_URL=rediss://...
CLERK_SECRET_KEY=sk_live_...
CLERK_WEBHOOK_SECRET=whsec_...
SENTRY_DSN=https://...@sentry.io/...
RESEND_API_KEY=re_...
```

### Web (.env.production)

```bash
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_...
CLERK_SECRET_KEY=sk_live_...
NEXT_PUBLIC_API_URL=https://api.slotbase.com
NEXT_PUBLIC_SENTRY_DSN=https://...@sentry.io/...
SENTRY_ORG=your-org
SENTRY_PROJECT=slotbase-web
```

### Mobile (EAS Secrets)

```bash
EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_...
EXPO_PUBLIC_API_URL=https://api.slotbase.com
EXPO_PUBLIC_SENTRY_DSN=https://...@sentry.io/...
```

---

## Security Checklist

- [ ] All secrets stored in environment variables, not in code
- [ ] CORS configured for production domains only
- [ ] Rate limiting enabled on API
- [ ] HTTPS enforced everywhere
- [ ] Database connection uses SSL
- [ ] Clerk webhook endpoint validates signatures
- [ ] Sentry filters sensitive data (auth headers, cookies)
- [ ] Regular security updates for dependencies

---

## Pilot Launch Checklist

- [ ] Database migrations complete
- [ ] Seed data loaded (pilot facility)
- [ ] All environment variables configured
- [ ] Custom domains configured
- [ ] SSL certificates active
- [ ] Sentry receiving errors
- [ ] Health checks passing
- [ ] Mobile builds submitted to stores
- [ ] Team accounts created
- [ ] Support email configured
