# E-Bikes LWC — Environment Setup Guide

A step-by-step guide to setting up and maintaining the **Prod** and **Dev** Salesforce environments for the E-Bikes Lightning Web Components app.

---

## Table of Contents

- [Salesforce Background — What We Set Up and Why](#salesforce-background--what-we-set-up-and-why)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [One-Time Setup](#one-time-setup)
- [Prod Environment Setup](#prod-environment-setup-main-branch)
- [Dev Environment Setup](#dev-environment-setup-new_feature-branch)
- [GitHub Actions — Auto Deploy](#github-actions--auto-deploy)
- [Day-to-Day Workflow](#day-to-day-workflow)
- [Accessing the Environments](#accessing-the-environments)
- [Renewal — When Scratch Orgs Expire](#renewal--when-scratch-orgs-expire)
- [Troubleshooting](#troubleshooting)

---

## Salesforce Background — What We Set Up and Why

This section explains the Salesforce concepts behind our setup, what alternatives exist, and why we made the choices we did.

---

### What Type of App Is This?

The E-Bikes app is built on two Salesforce technologies:

| Technology | What It Is | What We Use It For |
|---|---|---|
| **Lightning Web Components (LWC)** | Salesforce's modern UI framework (similar to React) | Product cards, filters, order builder UI |
| **Experience Cloud** (formerly Communities) | Salesforce feature for building public-facing websites/portals | The E-Bikes storefront site that customers see |

The app is **source-driven** — all code lives in Git and gets deployed to Salesforce via CLI. Nothing is built by clicking in the Salesforce UI.

---

### Salesforce Org Types — What They Are and What We Use

A **Salesforce org** is an instance of Salesforce — like a database + server + UI all in one. There are several types:

| Org Type | What It Is | Licensing | Lifespan | Our Usage |
|---|---|---|---|---|
| **Production Org** | Live customer-facing org | Paid (varies by edition) | Permanent | Not used |
| **Developer Edition** | Free full-featured org for developers | Free (limited capacity) | Permanent | Used as **Dev Hub** |
| **Sandbox** | Clone of a production org for testing | Included with Enterprise/Unlimited paid plans | Permanent | Not available to us |
| **Scratch Org** | Lightweight, disposable org created from config | Free (requires Dev Hub) | Max 30 days | Used for **Prod** and **Dev** environments |

#### What We Have

```
Developer Edition Org  ←  This is our Dev Hub (control center)
│   URL: orgfarm-03e5c5577c-dev-ed.develop.my.salesforce.com
│   Alias: ebikes-org
│   Lifespan: Permanent (free forever)
│
├── Scratch Org: ebikes        ←  Prod environment
│     Alias: ebikes
│     Lifespan: 30 days (recreate when expired)
│
└── Scratch Org: ebikes-dev   ←  Dev environment
      Alias: ebikes-dev
      Lifespan: 30 days (recreate when expired)
```

#### Why Scratch Orgs Instead of Sandboxes?

| | Scratch Org | Sandbox |
|---|---|---|
| **Cost** | Free | Requires Enterprise/Unlimited license (~$150+/user/month) |
| **Created from** | A JSON config file in Git | A snapshot of your production org |
| **CI/CD friendly** | Yes — scripted creation | Partial support |
| **Lifespan** | Max 30 days | Permanent |
| **Good for** | Dev/test environments, demos | UAT, staging, production-like testing |

We chose scratch orgs because they are **free**, **fully scriptable**, and sufficient for our use case (development and demos). If the team ever moves to a paid Salesforce plan, sandboxes would be the preferred long-term option.

---

### What Is a Dev Hub?

A **Dev Hub** is a Salesforce org that acts as the "parent" for all scratch orgs. You need one to create scratch orgs. In our setup:

- **Dev Hub** = our Developer Edition org (`ebikes-org`)
- All scratch orgs are created under this Dev Hub
- The Dev Hub itself is **not** where the app runs — it's just the control center
- Developer Edition orgs can create up to **6 scratch orgs at a time** (free limit)

---

### What Is Experience Cloud?

**Experience Cloud** (formerly called Communities, Portals, or Sites) is a Salesforce product that lets you build externally-accessible websites powered by Salesforce data.

Our E-Bikes app uses Experience Cloud to create a **public storefront** where anyone can browse bikes without logging in.

| Feature | Details |
|---|---|
| **Site type** | ChatterNetworkPicasso (Aura-based Experience Cloud) |
| **URL path** | `/ebikes` |
| **Guest access** | Enabled — public can browse products without login |
| **Framework** | Aura + LWC components |
| **Licensing** | Included in Developer Edition and most paid Salesforce editions |

#### Do You Need a License for Experience Cloud?

| Scenario | License Required |
|---|---|
| Developer Edition org (our setup) | No — included free |
| Enterprise Edition org | Yes — Experience Cloud add-on (~$2/login or $35/user/month) |
| Unlimited Edition org | Included in base license |

Since we're on a Developer Edition org, Experience Cloud is **free**.

---

### What Is the Salesforce CLI (`sf`)?

The **Salesforce CLI** (`sf`) is the command-line tool used to interact with Salesforce orgs programmatically. It replaces the older `sfdx` tool (both names still work).

We use it to:
- Authenticate to orgs
- Create scratch orgs
- Deploy metadata (code, components, configs)
- Import sample data
- Publish Experience Cloud sites

Install: `npm install -g @salesforce/cli`

---

### Alternative Deployment Options We Considered

| Option | How It Works | Why We Didn't Use It |
|---|---|---|
| **Salesforce UI (point-and-click)** | Manually build components in Setup | Not version-controlled, not repeatable |
| **Change Sets** | Package changes and deploy between orgs via UI | Manual, no Git integration |
| **Ant Migration Tool** | Legacy XML-based deployment | Deprecated, replaced by SF CLI |
| **Gearset / Copado** | Third-party DevOps tools for Salesforce | Paid tools ($300-$500+/user/month) |
| **GitHub Actions + SF CLI (our choice)** | Push to Git → auto deploy via CLI | Free, automated, Git-native |

---

### Licensing Summary

| Component | License Needed | Cost |
|---|---|---|
| Developer Edition org | None | Free |
| Dev Hub capability | None (enabled in Dev Edition) | Free |
| Scratch orgs (up to 6) | Dev Hub required | Free |
| Experience Cloud site | None on Dev Edition | Free |
| Salesforce CLI | None | Free |
| GitHub Actions (2000 min/month) | GitHub free account | Free |
| **Total for our setup** | | **$0** |

> If the team scales to a paid Salesforce org (Enterprise/Unlimited), sandboxes become available and scratch orgs are still free. The GitHub Actions workflows work identically — just update the auth secrets.

---

---

## Architecture Overview

```
GitHub Repo: anil8x/ebikes-lwc
│
├── main branch
│     └── Auto-deploys → Prod Scratch Org (ebikes)
│           └── https://speed-flow-2049-dev-ed.scratch.my.site.com/ebikes
│
└── new_feature branch
      └── Auto-deploys → Dev Scratch Org (ebikes-dev)
            └── https://power-dream-2385-dev-ed.scratch.my.site.com/ebikes
```

| | Prod | Dev |
|---|---|---|
| **Git Branch** | `main` | `new_feature` |
| **Org Alias** | `ebikes` | `ebikes-dev` |
| **Dev Hub** | `ebikes-org` | `ebikes-org` |
| **Scratch Org Lifespan** | 30 days max | 30 days max |
| **Auto-Deploy** | On every push to `main` | On every push to `new_feature` |

> **Note:** Scratch orgs expire after 30 days. See [Renewal](#renewal--when-scratch-orgs-expire) for how to recreate them.

---

## Prerequisites

### Tools Required

| Tool | Install Command | Verify |
|---|---|---|
| Salesforce CLI | `npm install -g @salesforce/cli` | `sf --version` |
| Node.js | [nodejs.org](https://nodejs.org) | `node --version` |
| Git | [git-scm.com](https://git-scm.com) | `git --version` |

### Accounts Required

- Salesforce Developer Edition org (free at [developer.salesforce.com](https://developer.salesforce.com))
- GitHub account with access to `anil8x/ebikes-lwc`

---

## One-Time Setup

These steps only need to be done **once** per machine.

### Step 1 — Clone the Repository

```bash
git clone git@github.com:anil8x/ebikes-lwc.git
cd ebikes-lwc
```

### Step 2 — Authenticate CLI to Dev Hub

```bash
sf org login web --alias ebikes-org --set-default-dev-hub
```

> A browser window opens. Log in with your Salesforce credentials. Once authenticated, the CLI is connected.

**Verify connection:**

```bash
sf org list
```

Expected output:
```
┌──┬────────┬────────────┬─────────────────────────────────────────┬──────────────────┬───────────┐
│  │ Type   │ Alias      │ Username                                │ Org Id           │ Status    │
├──┼────────┼────────────┼─────────────────────────────────────────┼──────────────────┼───────────┤
│  │ DevHub │ ebikes-org │ anilld.675.8543caf23bb9@agentforce.com  │ 00DdL00000qq8Mv  │ Connected │
└──┴────────┴────────────┴─────────────────────────────────────────┴──────────────────┴───────────┘
```

---

## Prod Environment Setup (`main` branch)

### Step 1 — Switch to Main Branch

```bash
git checkout main
```

### Step 2 — Create the Prod Scratch Org

```bash
sf org create scratch \
  --definition-file config/project-scratch-def.json \
  --alias ebikes \
  --duration-days 30 \
  --target-dev-hub ebikes-org
```

**Expected output:**
```
 ────────────── Creating Scratch Org ──────────────

 ✔ Prepare Request 15ms
 ✔ Send Request 26.03s
 ✔ Wait For Org 9.20s
 ✔ Available 2ms
 ✔ Authenticate 887ms
 ✔ Deploy Settings 6.48s
 ✔ Done 0ms

 Alias: ebikes
 Elapsed Time: 42.64s

Your scratch org is ready.
```

### Step 3 — Deploy App Metadata

```bash
sf project deploy start --target-org ebikes
```

**Expected output:**
```
 ✔ Deploying Metadata 31.31s
   ▸ Components: 101/101 (100%)
 Status: Succeeded
 Elapsed Time: 32.64s
```

### Step 4 — Assign Permission Sets

```bash
sf org assign permset -n ebikes --target-org ebikes
sf org assign permset -n Walkthroughs --target-org ebikes
```

**Expected output:**
```
Permsets Assigned
┌───────────────────────────┬───────────────────────────┐
│ Username                  │ Permission Set Assignment │
├───────────────────────────┼───────────────────────────┤
│ test-xxxxx@example.com    │ ebikes                    │
└───────────────────────────┴───────────────────────────┘
```

### Step 5 — Import Sample Data

```bash
sf data tree import -p ./data/sample-data-plan.json --target-org ebikes
```

**Expected output:**
```
Import Results
┌─────────────────┬───────────────────┬────────────────────┐
│ Reference ID    │ Type              │ ID                 │
├─────────────────┼───────────────────┼────────────────────┤
│ AccountRef1     │ Account           │ 001C100000Q2VULIA3 │
│ DynamoRef       │ Product_Family__c │ a02C1000007i78DIAQ │
│ Product__cRef1  │ Product__c        │ a03C100000NAVCDIA5 │
│ ...             │ ...               │ ...                │
└─────────────────┴───────────────────┴────────────────────┘
```

### Step 6 — Publish the Experience Cloud Site

```bash
sf community publish -n E-Bikes --target-org ebikes
```

**Expected output:**
```
Publish Site Result
┌────────────────────┬─────────┬────────┬──────────────────────────────────────────┐
│ Id                 │ Name    │ Status │ Url                                      │
├────────────────────┼─────────┼────────┼──────────────────────────────────────────┤
│ 0DBC10000000yQnOAI │ E-Bikes │ Live   │ https://speed-flow-2049-dev-ed...        │
└────────────────────┴─────────┴────────┴──────────────────────────────────────────┘
```

### Step 7 — Deploy Guest Profile Metadata

```bash
sf project deploy start --metadata-dir=guest-profile-metadata -w 10 --target-org ebikes
```

**Expected output:**
```
 ✔ Deploying Metadata 2.26s
   ▸ Components: 7/7 (100%)
 Status: Succeeded
```

### Step 8 — Verify the Site is Live

```bash
sf org open --target-org ebikes --path /lightning/setup/SetupNetworks/home
```

In the browser, click the URL next to **E-Bikes** in the Digital Experiences list.

**Prod site URL:**
```
https://speed-flow-2049-dev-ed.scratch.my.site.com/ebikes
```

---

## Dev Environment Setup (`new_feature` branch)

### Step 1 — Switch to Dev Branch

```bash
git checkout new_feature
```

### Step 2 — Create the Dev Scratch Org

```bash
sf org create scratch \
  --definition-file config/project-scratch-def.json \
  --alias ebikes-dev \
  --duration-days 30 \
  --target-dev-hub ebikes-org
```

### Step 3 — Deploy App Metadata

```bash
sf project deploy start --target-org ebikes-dev
```

### Step 4 — Assign Permission Sets

```bash
sf org assign permset -n ebikes --target-org ebikes-dev
sf org assign permset -n Walkthroughs --target-org ebikes-dev
```

### Step 5 — Import Sample Data

```bash
sf data tree import -p ./data/sample-data-plan.json --target-org ebikes-dev
```

### Step 6 — Publish the Experience Cloud Site

```bash
sf community publish -n E-Bikes --target-org ebikes-dev
```

### Step 7 — Deploy Guest Profile Metadata

```bash
sf project deploy start --metadata-dir=guest-profile-metadata -w 10 --target-org ebikes-dev
```

### Step 8 — Verify the Site is Live

```bash
sf org open --target-org ebikes-dev --path /lightning/setup/SetupNetworks/home
```

**Dev site URL:**
```
https://power-dream-2385-dev-ed.scratch.my.site.com/ebikes
```

---

## GitHub Actions — Auto Deploy

Every push to either branch automatically triggers a deployment. No manual steps needed.

### Workflow Files

| File | Branch | Trigger |
|---|---|---|
| `.github/workflows/deploy-prod.yml` | `main` | Push to `main` |
| `.github/workflows/deploy-dev.yml` | `new_feature` | Push to `new_feature` |

---

#### `.github/workflows/deploy-prod.yml` — Prod Deployment

```yaml
name: Deploy to Prod (main)

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Salesforce CLI
        run: npm install -g @salesforce/cli

      - name: Authenticate to Prod Scratch Org
        run: |
          echo "${{ secrets.SF_PROD_SFDX_URL }}" > /tmp/prod_auth_url.txt
          sf org login sfdx-url --sfdx-url-file /tmp/prod_auth_url.txt --alias ebikes
          rm /tmp/prod_auth_url.txt

      - name: Deploy Metadata
        run: sf project deploy start --target-org ebikes --ignore-conflicts

      - name: Deploy Guest Profile Metadata
        run: sf project deploy start --metadata-dir=guest-profile-metadata -w 10 --target-org ebikes --ignore-conflicts

      - name: Publish Experience Cloud Site
        run: sf community publish -n E-Bikes --target-org ebikes
```

**What each step does:**

| Step | Purpose |
|---|---|
| Checkout code | Pulls the latest `main` branch code onto the runner |
| Install Salesforce CLI | Installs `sf` CLI on the GitHub Actions runner |
| Authenticate | Uses the `SF_PROD_SFDX_URL` secret to log into the prod scratch org |
| Deploy Metadata | Pushes all 101 app components to the org |
| Deploy Guest Profile | Deploys sharing rules and guest user profile settings |
| Publish Site | Makes the Experience Cloud site publicly accessible |

---

#### `.github/workflows/deploy-dev.yml` — Dev Deployment

```yaml
name: Deploy to Dev (new_feature)

on:
  push:
    branches:
      - new_feature

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Salesforce CLI
        run: npm install -g @salesforce/cli

      - name: Authenticate to Dev Scratch Org
        run: |
          echo "${{ secrets.SF_DEV_SFDX_URL }}" > /tmp/dev_auth_url.txt
          sf org login sfdx-url --sfdx-url-file /tmp/dev_auth_url.txt --alias ebikes-dev
          rm /tmp/dev_auth_url.txt

      - name: Deploy Metadata
        run: sf project deploy start --target-org ebikes-dev --ignore-conflicts

      - name: Deploy Guest Profile Metadata
        run: sf project deploy start --metadata-dir=guest-profile-metadata -w 10 --target-org ebikes-dev --ignore-conflicts

      - name: Publish Experience Cloud Site
        run: sf community publish -n E-Bikes --target-org ebikes-dev
```

**What each step does:**

| Step | Purpose |
|---|---|
| Checkout code | Pulls the latest `new_feature` branch code onto the runner |
| Install Salesforce CLI | Installs `sf` CLI on the GitHub Actions runner |
| Authenticate | Uses the `SF_DEV_SFDX_URL` secret to log into the dev scratch org |
| Deploy Metadata | Pushes all 101 app components to the org |
| Deploy Guest Profile | Deploys sharing rules and guest user profile settings |
| Publish Site | Makes the Experience Cloud site publicly accessible |

> **Note:** The two workflows are identical in structure — the only difference is the branch trigger (`main` vs `new_feature`), the secret used (`SF_PROD_SFDX_URL` vs `SF_DEV_SFDX_URL`), and the org alias (`ebikes` vs `ebikes-dev`).

### Required GitHub Secrets

Go to: **GitHub repo → Settings → Secrets and variables → Actions → New repository secret**

| Secret Name | Used By | Description |
|---|---|---|
| `SF_PROD_SFDX_URL` | `deploy-prod.yml` | SFDX auth URL for the prod scratch org (`ebikes`) |
| `SF_DEV_SFDX_URL` | `deploy-dev.yml` | SFDX auth URL for the dev scratch org (`ebikes-dev`) |

---

#### How to Get the Secret Value for Prod (`SF_PROD_SFDX_URL`)

**Step 1** — Make sure the prod scratch org exists and is authenticated:
```bash
sf org list
# You should see "ebikes" listed as Active
```

**Step 2** — Run this command to get the auth URL:
```bash
sf org display --target-org ebikes --verbose --json
```

**Step 3** — In the JSON output, find and copy the `sfdxAuthUrl` field:
```json
{
  "result": {
    "alias": "ebikes",
    "instanceUrl": "https://speed-flow-2049-dev-ed.scratch.my.salesforce.com",
    "sfdxAuthUrl": "force://PlatformCLI::5Aep861rIi_xxxxx@speed-flow-2049-dev-ed.scratch.my.salesforce.com",
    ...
  }
}
```

**Step 4** — Go to GitHub → repo **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
- Name: `SF_PROD_SFDX_URL`
- Value: paste the full `sfdxAuthUrl` value (starts with `force://`)

---

#### How to Get the Secret Value for Dev (`SF_DEV_SFDX_URL`)

**Step 1** — Make sure the dev scratch org exists:
```bash
sf org list
# You should see "ebikes-dev" listed as Active
```

**Step 2** — Run this command:
```bash
sf org display --target-org ebikes-dev --verbose --json
```

**Step 3** — Copy the `sfdxAuthUrl` value from the output.

**Step 4** — Add it as a GitHub secret:
- Name: `SF_DEV_SFDX_URL`
- Value: paste the full `sfdxAuthUrl` value (starts with `force://`)

---

> **Important:** These auth URLs are sensitive — treat them like passwords. Never commit them to the repo or share them in plain text. They expire when the scratch org expires, so update the secrets whenever you recreate the orgs.

### Monitoring Deployments

Go to: `https://github.com/anil8x/ebikes-lwc/actions`

- Green check = deployment succeeded
- Red cross = deployment failed (click to view logs)

### Manually Trigger a Deployment

```bash
# Trigger prod deployment
git checkout main
git commit --allow-empty -m "Trigger prod deployment"
git push origin main

# Trigger dev deployment
git checkout new_feature
git commit --allow-empty -m "Trigger dev deployment"
git push origin new_feature
```

---

## Day-to-Day Workflow

### Making Changes to Prod

```bash
git checkout main
# make your code changes
git add .
git commit -m "Your commit message"
git push origin main
# GitHub Actions auto-deploys to prod
```

### Making Changes to Dev

```bash
git checkout new_feature
# make your code changes
git add .
git commit -m "Your commit message"
git push origin new_feature
# GitHub Actions auto-deploys to dev
```

### Deploying Manually (without GitHub Actions)

```bash
# Prod
git checkout main
sf project deploy start --target-org ebikes --ignore-conflicts

# Dev
git checkout new_feature
sf project deploy start --target-org ebikes-dev --ignore-conflicts
```

### Opening Orgs in Browser

```bash
sf org open --target-org ebikes        # prod admin backend
sf org open --target-org ebikes-dev    # dev admin backend
```

---

## Accessing the Environments

### Site URLs (share these for demos)

| Environment | URL |
|---|---|
| **Prod** | `https://speed-flow-2049-dev-ed.scratch.my.site.com/ebikes` |
| **Dev** | `https://power-dream-2385-dev-ed.scratch.my.site.com/ebikes` |

These are **public URLs** — no Salesforce login required to view them.

### Admin Backend (Salesforce Setup)

```bash
sf org open --target-org ebikes        # opens prod org
sf org open --target-org ebikes-dev    # opens dev org
```

### Dev Hub Login

- **URL:** `https://orgfarm-03e5c5577c-dev-ed.develop.my.salesforce.com`
- **Username:** `anilld.675.8543caf23bb9@agentforce.com`
- **Password:** *(your Salesforce password)*

---

## Renewal — When Scratch Orgs Expire

Scratch orgs expire after **30 days**. When they expire, recreate them by running the full setup steps again.

### Check Expiry Dates

```bash
sf org list --all
```

```
┌──┬─────────┬────────────┬──────────────────────────┬──────────┬───────────┬────────────┐
│  │ Type    │ Alias      │ Username                  │ Org Id   │ Status    │ Expires    │
├──┼─────────┼────────────┼──────────────────────────┼──────────┼───────────┼────────────┤
│  │ Scratch │ ebikes     │ test-xxxxx@example.com    │ 00DC1... │ Active    │ 2026-05-10 │
│  │ Scratch │ ebikes-dev │ test-yyyyy@example.com    │ 00DBl... │ Active    │ 2026-05-10 │
└──┴─────────┴────────────┴──────────────────────────┴──────────┴───────────┴────────────┘
```

### Recreate an Expired Org

**Prod:**
```bash
sf org delete scratch --target-org ebikes --no-prompt  # if it still exists
git checkout main
sf org create scratch \
  --definition-file config/project-scratch-def.json \
  --alias ebikes \
  --duration-days 30 \
  --target-dev-hub ebikes-org
sf project deploy start --target-org ebikes
sf org assign permset -n ebikes --target-org ebikes
sf org assign permset -n Walkthroughs --target-org ebikes
sf data tree import -p ./data/sample-data-plan.json --target-org ebikes
sf community publish -n E-Bikes --target-org ebikes
sf project deploy start --metadata-dir=guest-profile-metadata -w 10 --target-org ebikes
```

**After recreation — update GitHub Secret with new SFDX URL:**
```bash
sf org display --target-org ebikes --verbose --json
# Copy sfdxAuthUrl → update SF_PROD_SFDX_URL secret in GitHub
```

> Repeat the same steps for `ebikes-dev` / `SF_DEV_SFDX_URL` when the dev org expires.

---

## Troubleshooting

### Error: `No authorization information found for ebikes`

The scratch org auth was lost (usually after CLI update or machine restart).

```bash
sf org login web --alias ebikes-org --set-default-dev-hub
sf org list
```

If the scratch org is gone, follow the [Renewal](#renewal--when-scratch-orgs-expire) steps.

---

### Error: `Conflict — changes in org conflict with local changes`

Add `--ignore-conflicts` flag:

```bash
sf project deploy start --target-org ebikes --ignore-conflicts
```

---

### Error: `URL No Longer Exists` on the site URL

The correct URL format is:
```
https://[org-domain].scratch.my.site.com/ebikes
```

Find the correct URL via:
```bash
sf org open --target-org ebikes --path /lightning/setup/SetupNetworks/home
```

Then click the URL next to **E-Bikes** in the Digital Experiences list.

---

### GitHub Actions failing

1. Go to `github.com/anil8x/ebikes-lwc/actions`
2. Click the failed run → view logs
3. Common causes:
   - **Scratch org expired** → recreate org, update GitHub secret
   - **Auth URL invalid** → regenerate with `sf org display --verbose --json`, update secret
   - **Conflict error** → ensure `--ignore-conflicts` is in the workflow deploy command

---

*Last updated: April 2026*
