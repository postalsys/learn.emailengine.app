---
title: Render.com Deployment
description: Deploy EmailEngine to Render.com with one-click setup and managed Redis
sidebar_position: 4
---

# Deploy EmailEngine on Render.com

Deploy EmailEngine on Render.com with zero DevOps overhead. Render provides managed hosting with automatic SSL, built-in monitoring, and Redis add-ons.

:::tip Quick Deploy
[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/postalsys/emailengine)

Use the "Deploy to Render" button for automated setup with EmailEngine + Redis configured automatically.
:::

## Overview

[Render.com](https://render.com/) is a modern cloud platform that makes deploying applications simple. You can set up EmailEngine entirely from the web UI without SSH access.

**Benefits of Render deployment:**
- Fully managed infrastructure
- Automatic SSL certificates
- Built-in Redis support
- Auto-deploy from Git
- Simple environment configuration
- Automatic health checks
- Built-in monitoring and logs

**Costs:** two paid services, one web service and one Redis instance. Render revises its plans, so read the current figures on [Render's pricing page](https://render.com/pricing) rather than budgeting from a number quoted here.

:::warning Resource Requirements
Do not choose the smallest instance sizes. EmailEngine requires at least:
- **1GB RAM** for web service
- **256MB+** for Redis (with `noeviction` policy)
:::

## Step-by-Step Setup

### Step 1: Create Redis Instance

1. **Log in to Render Dashboard:** [https://dashboard.render.com](https://dashboard.render.com)

2. **Click "New +"** button (top right)

3. **Select "Redis"**

   ![Create Redis](/img/external/Screenshot-2022-05-06-at-11.16.42.png)

4. **Configure Redis:**

   | Setting | Value | Notes |
   |---------|-------|-------|
   | **Name** | `emailengine-redis` | Any name you prefer |
   | **Region** | Closest to you | For best performance |
   | **Plan** | Starter (256MB+) | Minimum recommended |
   | **Maxmemory Policy** | `noeviction` | **Required** |
   | **Eviction** | Disabled | Prevents data loss |

   ![Redis Configuration](/img/external/Screenshot-2022-05-06-at-11.23.37.png)

5. **Click "Create Redis"**

6. **Copy Redis URL:**
   - Wait for Redis to start (usually < 1 minute)
   - On the Redis info page, copy the **Internal Connection String**
   - Format: `redis://red-xxxxx:6379`

   ![Redis URL](/img/external/Screenshot-2022-05-06-at-11.20.57.png)

:::tip Internal vs External URL
Use the **Internal Connection String** for better performance and no data transfer costs. External URLs are only needed if connecting from outside Render.
:::

### Step 2: Deploy EmailEngine

1. **Click "New +"** button again

2. **Select "Web Service"**

   ![Create Web Service](/img/external/Screenshot-2022-05-06-at-11.26.28.png)

3. **Connect Repository:**
   - Select "Deploy an existing image from a registry" OR
   - Use public repository: `https://github.com/postalsys/emailengine`

   ![Connect Repository](/img/external/Screenshot-2022-05-06-at-10.53.33.png)

4. **Click "Connect"** next to `postalsys/emailengine`

5. **Configure Web Service:**

   | Setting | Value | Notes |
   |---------|-------|-------|
   | **Name** | `emailengine` | Any name you prefer |
   | **Region** | Same as Redis | **Important** for latency |
   | **Branch** | `master` | Or specific version tag |
   | **Environment** | `Node` | Required |
   | **Build Command** | `npm install --omit=dev` | Installs dependencies |
   | **Start Command** | `npm start` | Starts EmailEngine |
   | **Plan** | Starter (1GB RAM+) | Minimum recommended |

6. **Add Environment Variables:**

   Click **"Advanced"** section and add these variables:

   | Key | Value | Required |
   |-----|-------|----------|
   | `EENGINE_REDIS` | `redis://red-xxxxx:6379/8` | Yes |
   | `EENGINE_SECRET` | Generate random string (32+ chars) | Yes |
   | `EENGINE_WORKERS` | `2` | Optional |
   | `NODE_ENV` | `production` | Recommended |

   ![Environment Variables](/img/external/render-app.png)

   :::warning Generate Strong Secrets
   Use a password generator to create strong random strings:
   ```bash
   # Linux/Mac
   openssl rand -hex 32

   # Or use online generator
   # https://www.random.org/strings/
   ```
   :::

7. **Click "Create Web Service"**

### Step 3: Wait for Deployment

1. **Monitor deployment:**
   - Render will clone repository
   - Install dependencies
   - Build application
   - Start service
   - Usually takes 3-5 minutes

2. **Check logs:**
   - Click "Logs" tab to see real-time progress
   - Look for: `Started API server thread`

3. **Get application URL:**
   - Once deployed, URL appears at top: `https://emailengine-xxxx.onrender.com`

   ![Deployed Application](/img/external/Screenshot-2022-05-06-at-11.34.54.png)

4. **Access EmailEngine:**
   - Open the URL in your browser
   - You should see EmailEngine login/setup page

:::success Deployment Complete!
EmailEngine is now running on Render with automatic SSL and monitoring.
:::

## Post-Deployment Configuration

### Custom Domain

**Add your own domain:**

1. Go to **Settings** → **Custom Domains**
2. Click **"Add Custom Domain"**
3. Enter your domain (e.g., `emailengine.yourdomain.com`)
4. Add DNS record:
   ```
   CNAME emailengine.yourdomain.com → emailengine-xxxx.onrender.com
   ```
5. Wait for DNS propagation
6. SSL certificate is automatically provisioned

### Environment Variables

**Update environment variables:**

1. Go to **Environment** tab
2. Click variable to edit
3. Update value
4. Service automatically redeploys

**Common variables to add:**

```bash
# License key (use Render environment editor for multiline values)
EENGINE_PREPARED_LICENSE="<your-encoded-license>"

# Performance
EENGINE_WORKERS=4

# Pre-configured settings as JSON (webhooks, OAuth2, etc.)
EENGINE_SETTINGS='{"webhooks":"https://your-app.com/webhooks","webhookEvents":["*"],"webhooksEnabled":true}'
```

:::info OAuth2 and Webhook Configuration
OAuth2 credentials and webhook URLs are not configured via individual environment variables. Instead, use the EmailEngine web UI or the `EENGINE_SETTINGS` JSON environment variable to configure these settings after deployment.
:::

### Health Checks

**Render automatically monitors:**
- HTTP health checks every 30 seconds
- Automatic restart on failure
- View in **Events** tab

**Custom health check:**

Add to `render.yaml` in your repository:

```yaml
services:
  - type: web
    name: emailengine
    runtime: node
    healthCheckPath: /health
    buildCommand: npm install --omit=dev
    startCommand: npm start
    envVars:
      - key: EENGINE_REDIS
        sync: false
```

## Scaling and Performance

### Vertical Scaling

**Upgrade instance size:**

1. Go to **Settings** → **Instance Type**
2. Select larger plan:
   - **Starter:** 1 GB RAM, 0.5 CPU
   - **Standard:** 2 GB RAM, 1 CPU
   - **Pro:** 4 GB RAM, 2 CPU
3. Service automatically redeploys

**When to upgrade:**
- More than 100 active accounts
- High webhook volume
- Large mailboxes (10,000+ messages)
- Performance monitoring shows high resource usage

### Horizontal Scaling Not Supported

:::warning No Horizontal Scaling
EmailEngine does NOT support horizontal scaling. Running multiple EmailEngine instances connected to the same Redis will cause each instance to independently sync all accounts, leading to conflicts and wasted resources.
:::

**Instead:**
1. **Upgrade to higher tier** with more resources (vertical scaling)
2. **For very large deployments**: Contact EmailEngine support for guidance on manual sharding strategies (requires separate Redis instances and complex routing)

### Redis Scaling

**Upgrade Redis:**

1. Go to Redis dashboard
2. **Settings** → **Plan**
3. Select larger size:
   - **Starter:** 256 MB
   - **Standard:** 1 GB
   - **Pro:** 4 GB

**When to upgrade:**
- More than 100 accounts
- Memory usage > 80%
- Check metrics in Redis dashboard

## Monitoring and Logs

### View Logs

**Real-time logs:**

1. Go to **Logs** tab
2. Select time range
3. Filter by search terms
4. Download logs if needed

**Log examples:**

```log
{"level":30,"msg":"Started API server thread","port":3000,"host":"0.0.0.0"}
```

### Metrics

**Built-in metrics:**

1. Go to **Metrics** tab
2. View:
   - CPU usage
   - Memory usage
   - Network traffic
   - Request count
   - Response times

**Prometheus metrics:**

Create a token with metrics scope:

```bash
emailengine tokens issue -d "Prometheus" -s "metrics"
```

Access at: `https://your-app.onrender.com/metrics`

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" https://your-app.onrender.com/metrics
```

### Alerts

**Set up alerts:**

1. Go to **Settings** → **Notifications**
2. Add email or Slack webhook
3. Configure alert conditions:
   - Service down
   - High CPU usage
   - High memory usage
   - Deploy failures

## Backup and Disaster Recovery

### Redis Backup

**Automatic backups:**
- Render Pro Redis plans include automatic backups
- Daily snapshots retained for 7 days

**Manual backup:**

```bash
# Connect to Redis via external URL
redis-cli -u redis://default:password@red-xxxxx.render.com:6379

# Trigger backup
SAVE

# Or background save
BGSAVE
```

### Export Data

**Backup configuration:**

1. **Environment variables:**
   - Copy from Render dashboard
   - Store securely (encrypted)

2. **Account data:**
   - Use EmailEngine API to export account list
   - Store credentials separately

```bash
# Export accounts
curl https://your-app.onrender.com/v1/accounts \
  -H "Authorization: Bearer YOUR_TOKEN" > accounts.json
```

### Disaster Recovery

**Restore from backup:**

1. Create new Render services (Redis + Web)
2. Restore Redis data
3. Configure environment variables
4. Update DNS to point to new service

**Estimated recovery time:** 30-60 minutes

## Advanced Configuration

### Auto-Deploy from GitHub

**Enable automatic deployments:**

1. Connect GitHub repository
2. **Settings** → **Build & Deploy**
3. Enable **Auto-Deploy**
4. Choose branch (e.g., `main`)
5. Every push triggers deployment

**Deploy on PR merge:**
- Auto-deploy only on main branch
- Test in preview environments first

### Preview Environments

**Create staging environment:**

1. Create separate web service: `emailengine-staging`
2. Use separate Redis: `emailengine-redis-staging`
3. Configure different environment variables
4. Deploy from `develop` branch

**Benefits:**
- Test changes before production
- Separate data and credentials
- Different configuration

### Infrastructure as Code

**Use `render.yaml`:**

Create in repository root:

```yaml
services:
  - type: web
    name: emailengine
    runtime: node
    region: oregon
    plan: starter
    buildCommand: npm install --omit=dev
    startCommand: npm start
    # healthCheckPath: /health  # Optional - uncomment to enable
    envVars:
      - key: NODE_ENV
        value: production
      - key: EENGINE_REDIS
        fromService:
          type: redis
          name: emailengine-redis
          property: connectionString
      - key: EENGINE_SECRET
        generateValue: true
      - key: EENGINE_WORKERS
        value: "2"
      - key: EENGINE_HOST
        value: "0.0.0.0"
      - key: EENGINE_API_PROXY
        value: "1"

  - type: redis
    name: emailengine-redis
    plan: starter
    region: oregon
    maxmemoryPolicy: noeviction
```

**Deploy with render.yaml:**
```bash
# Commit to repository
git add render.yaml
git commit -m "Add Render configuration"
git push

# Render automatically detects and applies configuration
```

### Multiple Regions

**Deploy to multiple regions:**

1. Create separate services in each region
2. Use region-specific Redis
3. Configure global load balancer (external)
4. Or use DNS-based routing

**Regions available:**
- Oregon (US West)
- Ohio (US East)
- Frankfurt (Europe)
- Singapore (Asia)

## Migration from Render

### Export Configuration

**1. Document all settings:**
```bash
# List environment variables
render services list
render services env list emailengine

# Export to file
render services env list emailengine > env-vars.txt
```

**2. Backup Redis data:**
```bash
# Connect and backup
redis-cli -u <external-redis-url> --rdb ./dump.rdb
```

### Migrate to Other Platforms

**To Docker/VPS:**

1. Export environment variables
2. Create docker-compose.yml with same configuration
3. Restore Redis data
4. Update DNS

**To Kubernetes:**

1. Convert render.yaml to k8s manifests
2. Set up Redis StatefulSet
3. Deploy EmailEngine Deployment
4. Configure Ingress

## Cost Optimization Tips

1. **Start small:**
   - Begin on the Starter plans and scale up when the metrics say to

2. **Monitor usage:**
   - Check metrics weekly
   - Downgrade if over-provisioned

3. **Optimize configuration:**
   - Reduce workers on small instances
   - Limit connections per account
   - Disable unused features

4. **Consider alternatives at scale:**
   - A plain VPS is usually cheaper per account once the account count runs into the hundreds, at the cost of managing the host yourself
   - Render suits deployments in the low hundreds of accounts

## See Also

- [Deployment overview](/docs/deployment) - The other hosting options
- [Configuration](/docs/configuration) - Setting environment variables on the service
- [Redis](/docs/configuration/redis) - What the managed Redis needs to provide
- [Security](/docs/deployment/security) - Hardening a hosted deployment
