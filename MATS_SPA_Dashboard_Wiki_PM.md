# MATS SPA Dashboard — Product & Business Guide

> **Audience:** Product Managers, Clients, Business Stakeholders, Program Managers
>
> **Purpose:** Understand what the MATS SPA Dashboard measures, why it matters, and how to interpret key signals — without needing to know Kusto queries or pipeline internals.
>
> **Maintained by:** SISU / MATS Optics Team (IDC) | **Version:** 2.0 | **Last updated:** May 2026

---

## Quick Access Links

| Resource | Link |
|---|---|
| 📊 MATS SPA Dashboard (ADX) | [Open Dashboard](https://dataexplorer.azure.com/dashboards/f87c5205-85da-43ee-9ef1-d882690098ac?p-timespan=v-7d&p-app=v-14638111-3389-403d-b206-a6a71d9f8f16&p-clientErrorCode=all&p-iframeErrorCode=all#9aff6e3f-4ffa-4ce1-bff9-6e017117f8fc) |
| 📄 Metric Definitions Reference | `SSO_Metrics_CMC.docx` (SharePoint) |
| 🍪 Cookie Chaining Metrics | [MSALjs Dashboard](https://dataexplorer.azure.com/dashboards/e79a3b2e-4f31-4773-aa30-61588d098124?p-_flowType=v-silent&p-_startTime=7days&p-_endTime=now&p-_isVisible=all&p-_browser=all&p-_clientId=v-14638111-3389-403d-b206-a6a71d9f8f16&p-_embeddedClientId=all&p-_tenant=all&p-_libVer=all&p-_accountType=v-&p-_isNative=all&p-_isBroker=all&p-_isPlatformBrokerRequest=all&p-_isPlatformAuthorizeRequest=all&p-_clientName=all&p-_clientVersion=all#f1a252ea-64fc-4f4e-b3db-c040957fd91c) |
| 🏭 ADF Pipeline Factory | [MATS-ADF Factory (Azure Portal)](https://portal.azure.com/#/subscriptions/10acc9a9-ecbe-4d31-a4f5-4b86c1687040/resourceGroups/MatsData/providers/Microsoft.DataFactory/factories/MATS-ADF) |
| 🔐 Request Access (MATS) | [aka.ms/idweb](https://aka.ms/idweb) |

---

## Table of Contents

1. [Overview](#1-overview)
2. [Apps Covered](#2-apps-covered)
3. [Key Stakeholders](#3-key-stakeholders)
4. [What the Dashboard Measures](#4-what-the-dashboard-measures)
5. [Dashboard Pages — Summary](#5-dashboard-pages--summary)
6. [How to Get Access](#6-how-to-get-access)
7. [Open Work Items & Roadmap](#7-open-work-items--roadmap)
8. [Known Limitations](#8-known-limitations)
9. [Glossary (Business Terms)](#9-glossary-business-terms)

---

## 1. Overview

The **MATS SPA Dashboard** is a monitoring and analytics dashboard built on **Azure Data Explorer (ADX/Kusto)**. It tracks how reliably Microsoft's browser-based Single Page Applications (SPAs) allow users to sign in **silently** — that is, without showing any pop-up or redirect prompt.

The dashboard was built and is maintained by the **SISU / MATS Optics team** (India Development Center), in collaboration with the CMC Product Management team led by **Sameera Gajjarapu**.

### Why does silent sign-in matter?

Every time a user opens a Microsoft SPA (like Copilot in the browser), the app needs to get a security token to prove the user is signed in. If this happens silently in the background, the user sees nothing. If it fails, the user is interrupted with a sign-in prompt — breaking the seamless experience.

The MATS SPA Dashboard measures exactly this: **how often silent sign-in succeeds, and what's going wrong when it doesn't.**

---

## 2. Apps Covered

The dashboard is designed to monitor any Microsoft SPA application that uses MSAL.js for authentication. The **App** filter parameter on the dashboard allows switching between different applications. The following are the key apps monitored:

| App Name | Description | App ID |
|---|---|---|
| **CMC (Microsoft 365 Copilot)** | Primary focus — the Copilot browser app | `14638111-3389-403d-b206-a6a71d9f8f16` |
| Other SPA Apps | Additional Microsoft SPA apps using MSAL.js | Configured via App ID filter |

> **Note for Contributors:** To view data for a different app, change the **App** parameter in the dashboard filter bar to the appropriate App ID. CMC is the default selection, but the same metrics and queries apply to any MSAL.js-instrumented SPA. Contact the SISU/MATS team to officially onboard a new app.

> **Cookie Chaining metrics** (a third type of silent sign-in) are tracked separately on the [MSALjs Dashboard](https://dataexplorer.azure.com/dashboards/e79a3b2e-4f31-4773-aa30-61588d098124?p-_flowType=v-silent&p-_startTime=7days&p-_endTime=now&p-_isVisible=all&p-_browser=all&p-_clientId=v-14638111-3389-403d-b206-a6a71d9f8f16&p-_embeddedClientId=all&p-_tenant=all&p-_libVer=all&p-_accountType=v-&p-_isNative=all&p-_isBroker=all&p-_isPlatformBrokerRequest=all&p-_isPlatformAuthorizeRequest=all&p-_clientName=all&p-_clientVersion=all#f1a252ea-64fc-4f4e-b3db-c040957fd91c) — not on this dashboard.

---

## 3. Key Stakeholders

| Role | Person / Contact |
|---|---|
| CMC Engineering Lead | Sameera Gajjarapu (Sameera.Gajjarapu@microsoft.com) |
| CMC Engineering | Sridhar Dantuluri (sridhard@microsoft.com) |
| Dashboard & Metrics (IDC) | Gaurav Usadadiya, Shivani Singh, Abhishek Bag |
| Data Infrastructure | Chase Hawthorne (MATS Data Infra) |
| ADO Tracking | Sridhar Dantuluri |

---

## 4. What the Dashboard Measures

All metrics are tracked **per user profile** (not per request). A "profile" is an anonymized unique user.

### 4.1 SSO Rate ✅ *The headline metric*

> **In plain English:** Out of every 100 users who opened the app, how many got signed in completely silently — no pop-up, no redirect, no prompt?

- **Good:** Higher % = better silent sign-in experience.
- **Shown as:** Daily score and 7-day rolling average.
- **What counts as "success":** User authenticated silently AND never triggered an interactive prompt during that day.

### 4.2 Iframe Renewal Failure Rate

> **In plain English:** When the app tries to renew your sign-in token quietly in the background (via a hidden browser frame), how often does that fail?

- **Good:** Lower % = fewer background renewal failures.
- **Why it matters:** Iframe renewal failures are a major driver of SSO failures.

### 4.3 SSO Reliability

> **SSO Reliability** is defined as the number of profiles that called `ssoSilent` successfully over the total number of profiles that called `ssoSilent`.

### 4.4 Sign-In Reliability ⚠️ *Watch closely*

> **In plain English:** When silent sign-in fails and the app has to show a sign-in window (redirect/popup), how often does that interactive sign-in succeed?

- **Good:** Higher % = users who get prompted can actually complete sign-in.
- **Why it matters:** If Sign-In Reliability drops, users can't authenticate at all.
- **Current trend (as of Apr 2026):** Declining — under active investigation.

### 4.5 Reauth Rate

> **In plain English:** How often are users forced to re-authenticate despite having an active session? High reauth = users getting unexpectedly prompted, breaking the Copilot experience.

**Conditions that trigger Reauth:**
- `interaction_required`
- `consent_required`
- `login_required`
- `native_account_unavailable`

- **Good:** Lower % = fewer unexpected interruptions.

### 4.6 Score Cards (Daily / Weekly / WoW)

Each metric page shows three KPI cards:

| Card | Meaning |
|---|---|
| **1D Score** | Yesterday's metric value (note: 2-day lag — see Known Limitations) |
| **Weekly Avg** | Rolling 7-day average |
| **WoW Change** | This week vs. last week (positive = improving) |

---

## 5. Dashboard Pages — Summary

The dashboard has **4 active pages** (2 removed: Native SignIn and Comparison WIP):

| Page | What It Shows | Key Widgets |
|---|---|---|
| **Page 1: SSO Success Rate** | Core SSO health — daily score, trend, error breakdowns by error code and server error | Daily SSO Score, Weekly SSO Score, SSO Rate, IframeRenewalFailureRate, SSO Error Breakdown, SSO Detail Error Breakdown, SSO Server Error Breakdown, IframeTokenRenewalFailure Error Breakdown, IframeTokenRenewalFailure Detail Error Breakdown, Iframe Server Error Breakdown |
| **Page 2: SSO Reliability** | Deep-dive into silent auth error codes, MSA timeout analysis, server error view | SSO Reliability Error Breakdown, SSO Reliability (line), Breakdown of SSO Reliability, Server Error Breakdown, monitor_window_timeout MSA analysis |
| **Page 3: SignIn Reliability** | Interactive sign-in health — weekly and daily breakdowns, server error view | SignIn Reliability Breakdown (weekly), SignIn Reliability (line), SignIn Reliability Breakdown (daily), Server Error Breakdown |
| **Page 4: Reauth Rate** | How often users are forced to re-authenticate — with server error and ssoCapable breakdown | Reauth Rate Error Breakdown, Reauth Rate (line), Reauth Breakdown, Reauth Server Error Breakdown, Reauth Breakdown (ssoCapable mock data) |

### Dashboard Filters

All pages share global filters. The most important ones:

| Filter | Default | What It Does |
|---|---|---|
| **App** | CMC (Copilot) — `14638111-3389-403d-b206-a6a71d9f8f16` | Switch between monitored SPA apps |
| **Timespan** | 7 days | Change the data window (1d / 7d / 30d / custom) |
| **BrowserType** | (empty = All) | Filter by browser type (Chrome, Edge, Firefox, Safari) — available on Pages 2, 3, 4 |
| **SSO type** | SSO | Filter SSO reliability by sso or silent path — available on Page 2 only |
| **Error Code** | All | Filter charts by specific MSAL.js client error code — available on Pages 1, 2, 3 |
| **Iframe Error Code** | All | Filter iframe-related breakdown charts by specific error code — available on Page 1 only |

---

## 6. How to Get Access

The dashboard uses two data sources. You need access to both:

### Step 1 — MATS (Pre-aggregated Metrics)
- Go to [aka.ms/idweb](https://aka.ms/idweb)
- Search for and join security group: **`odxArMATSRprtCtb`**
- This grants read access to the `MATS_Reporting` Kusto database

### Step 2 — MSALjs Telemetry (Raw Events)
- Go to [IDWeb Groups](https://idweb.microsoft.com/IdentityManagement/aspx/groups/MyGroups.aspx?popupFromClipboard=%2Fidentitymanagement%2Faspx%2FGroups%2FEditGroup.aspx%3Fid%3D8ebd87c2-123f-4bda-9c6d-1ac588498eb5)
- Join group: **MSALJSTelemReadOnly** (Group ID: `8ebd87c2-123f-4bda-9c6d-1ac588498eb5`)
- This grants read access to the raw MSALjs telemetry database

---

## 7. Open Work Items & Roadmap

| Work Item | Status | Owner |
|---|---|---|
| Stacked bar chart: SSO Success Rate page | ✅ Done | Gaurav Usadadiya |
| Stacked bar chart: SSO Reliability page | ✅ Done | Gaurav Usadadiya |
| Stacked bar chart: Reauth Rate page | 🔄 In Progress | Shivani Singh |
| Stacked bar chart: Sign-In Reliability page | 📋 Assigned | Abhishek Bag |
| Chart performance comparison (bar vs area) | ✅ Done — bar loads faster | Team |
| Request-based stacked area charts (all pages) | 📋 TODO | TBD (est. 3–4 days) |
| Line charts for all pages (Requests-based) | 📋 TODO | TBD |
| This Wiki | ✅ Done (v2) | Abhishek Bag |
| ADO work item tracking | 🔄 Ongoing | Sridhar Dantuluri |
| Deck for Rakesh meeting | 🔄 First draft done | Shivani Singh |

> **Note on area vs. bar charts:** Performance testing (Apr 17, 2026) showed that bar/column charts load significantly faster than area charts across all test combinations. The decision is to keep bar charts as primary; area charts may be added later as a secondary view for Requests-based data.

---

## 8. Known Limitations

| Limitation | Detail |
|---|---|
| **2-day data lag** | The dashboard shows data from 2 days ago, not yesterday. This is intentional — it ensures MSALjs telemetry has fully landed in the cluster before being queried. |
| **Cookie chaining not tracked here** | The third SSO method (cookie chaining via refresh token) is not included in the SSO Rate metric. View it on the separate [MSALjs Dashboard](https://dataexplorer.azure.com/dashboards/e79a3b2e-4f31-4773-aa30-61588d098124?p-_flowType=v-silent&p-_startTime=7days&p-_endTime=now&p-_isVisible=all&p-_browser=all&p-_clientId=v-14638111-3389-403d-b206-a6a71d9f8f16&p-_embeddedClientId=all&p-_tenant=all&p-_libVer=all&p-_accountType=v-&p-_isNative=all&p-_isBroker=all&p-_isPlatformBrokerRequest=all&p-_isPlatformAuthorizeRequest=all&p-_clientName=all&p-_clientVersion=all#f1a252ea-64fc-4f4e-b3db-c040957fd91c). |
| **BrowserType filter is page-specific** | Not all pages support BrowserType filtering — check each page's filter bar. |
| **Sparse data points** | On days with low user volume, some metrics may not appear in charts (insufficient data for statistical validity). |

---

## 9. Glossary (Business Terms)

| Term | Plain-English Definition |
|---|---|
| **SPA** | Single Page Application — a type of web app (like Copilot in the browser) that loads once and updates dynamically without full page reloads |
| **SSO** | Single Sign-On — when you're already signed into Microsoft and an app silently reuses your sign-in without asking you again |
| **SSO Rate** | The % of users who were signed in completely silently — the core health metric of this dashboard |
| **Profile** | An anonymized individual user identity (not tied to a name — for privacy) |
| **Profile-based metric** | A metric that counts unique users (profiles), not individual requests |
| **Silent authentication** | Signing in without any user prompt — happens transparently in the background |
| **Interactive sign-in** | When the app has to show a login pop-up or redirect to the sign-in page because silent auth failed |
| **Iframe renewal** | A background technique where a hidden browser frame silently requests a new token from Microsoft's sign-in server |
| **Reauth** | Re-authentication — when a user with an active session is still forced to sign in again (unexpected and disruptive) |
| **CMC** | Microsoft 365 Copilot app (browser SPA) — the primary app monitored by this dashboard |
| **MSAL.js** | Microsoft Authentication Library for JavaScript — the sign-in library that CMC and other SPAs use |
| **MSA** | Microsoft Account — a personal Microsoft account (e.g., Outlook.com). Distinct from AAD/Entra ID work accounts |
| **AAD / Entra ID** | Microsoft's enterprise identity platform. Work/school accounts sign in through Entra ID |
| **WoW** | Week-over-Week — the change in a metric compared to the same period last week |
| **ADF** | Azure Data Factory — the automated pipeline system that processes raw telemetry data into the metrics shown on this dashboard |
| **ADX / Kusto** | Azure Data Explorer — the analytics database platform where the dashboard and its data live |
| **MATS** | Microsoft Authentication Telemetry Service — the team and data infrastructure behind this dashboard |
| **SISU** | Service Insights & SLI Unit — the IDC team that builds and maintains this dashboard |
| **IDC** | India Development Center — where the SISU/MATS Optics team is based |
| **monitor_window_timeout** | The most common sign-in failure — happens when the background iframe takes too long to get a response from Microsoft's sign-in server (especially for MSA/personal accounts) |
| **interaction_required** | An error meaning the app must show an interactive sign-in prompt (e.g., due to MFA policy or token expiry) |

---

*For technical details, Kusto queries, ADF pipeline operations, and on-call runbooks, see the [Developer Wiki](./MATS_SPA_Dashboard_Wiki_Dev.md).*
