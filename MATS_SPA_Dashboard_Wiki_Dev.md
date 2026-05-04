# MATS SPA Dashboard — Technical Reference & Developer Guide

> **Audience:** Developers, Data Engineers, On-Call Engineers, Technical PMs
>
> **Purpose:** Complete technical reference for the MATS SPA Dashboard — metric definitions, Kusto queries, ADF pipeline operations, MSAL.js authentication internals, and on-call runbook.
>
> **Maintained by:** SISU / MATS Optics Team (IDC) | **Version:** 2.0 | **Last updated:** May 2026

---

## Quick Access Links

| Resource | Link / Value |
|---|---|
| 📊 MATS SPA Dashboard | [ADX Dashboard](https://dataexplorer.azure.com/dashboards/f87c5205-85da-43ee-9ef1-d882690098ac?p-timespan=v-7d&p-app=v-14638111-3389-403d-b206-a6a71d9f8f16&p-clientErrorCode=all&p-iframeErrorCode=all#9aff6e3f-4ffa-4ce1-bff9-6e017117f8fc) |
| 🔷 Kusto Cluster | `https://idsharedeus2.eastus2.kusto.windows.net/` |
| 🗄️ Metrics Database | `MATS_Reporting` (on `idsharedeus2`) |
| 📡 Raw Telemetry Database | `MSALJS Tenant` (on `idsharedeus2`) |
| 🏭 ADF Factory | [MATS-ADF](https://portal.azure.com/#/subscriptions/10acc9a9-ecbe-4d31-a4f5-4b86c1687040/resourceGroups/MatsData/providers/Microsoft.DataFactory/factories/MATS-ADF) |
| 🔐 Access (MATS) | [aka.ms/idweb](https://aka.ms/idweb) → join `odxArMATSRprtCtb` |
| 🔐 Access (MSALjs) | [IDWeb](https://idweb.microsoft.com/IdentityManagement/aspx/groups/MyGroups.aspx?popupFromClipboard=%2Fidentitymanagement%2Faspx%2FGroups%2FEditGroup.aspx%3Fid%3D8ebd87c2-123f-4bda-9c6d-1ac588498eb5) → join `MSALJSTelemReadOnly` (ID: `8ebd87c2-123f-4bda-9c6d-1ac588498eb5`) |
| 📄 Metric Spec | `SSO_Metrics_CMC.docx` (SharePoint) |
| 🍪 Cookie Chaining | [MSALjs Dashboard](https://dataexplorer.azure.com/dashboards/e79a3b2e-4f31-4773-aa30-61588d098124?p-_flowType=v-silent&p-_startTime=7days&p-_endTime=now&p-_isVisible=all&p-_browser=all&p-_clientId=v-14638111-3389-403d-b206-a6a71d9f8f16&p-_embeddedClientId=all&p-_tenant=all&p-_libVer=all&p-_accountType=v-&p-_isNative=all&p-_isBroker=all&p-_isPlatformBrokerRequest=all&p-_isPlatformAuthorizeRequest=all&p-_clientName=all&p-_clientVersion=all#f1a252ea-64fc-4f4e-b3db-c040957fd91c) |

---

## Table of Contents

1. [Architecture & Data Flow](#1-architecture--data-flow)
2. [Apps Covered & App ID Reference](#2-apps-covered--app-id-reference)
3. [Data Sources & Access](#3-data-sources--access)
4. [Dashboard Parameters](#4-dashboard-parameters)
5. [Metric Definitions (Formal)](#5-metric-definitions-formal)
6. [Dashboard Pages — Technical Detail](#6-dashboard-pages--technical-detail)
7. [Kusto Queries (All Widgets)](#7-kusto-queries-all-widgets)
8. [ADF Pipelines](#8-adf-pipelines)
9. [MSAL.js Authentication Flow](#9-msaljs-authentication-flow)
10. [On-Call Guide](#10-on-call-guide)
11. [Infrastructure Improvements (Roadmap)](#11-infrastructure-improvements-roadmap)
12. [Glossary (Full)](#12-glossary-full)
13. [Revision History](#13-revision-history)

---

## 1. Architecture & Data Flow

```
User Browser (CMC SPA / other MSAL.js app)
        │
        │ MSAL.js API calls: ssoSilent, acquireTokenSilent,
        │ acquireTokenRedirect, acquireTokenPopup
        ▼
MSALjs Tenant (Kusto)                   ADF Pipelines (MATS-ADF)
  └─ Raw allevents table         ◄──────── CMC_SSO_Success
  └─ getMaterializedData()                  SSO_Reauth_Error
  └─ allEventsView (materialized)           SSO_Reliability_Top_Error
        │                                        │
        │ Direct query (KPI cards,               │ Write to MATS_Reporting
        │ reliability scores)                    │ (TableSetOrAppend)
        ▼                                        ▼
                    MATS_Reporting (Kusto)
                      └─ MSALJS_SSORate
                      └─ SSO Reliability Top Error tables
                      └─ Reauth Error tables
                              │
                              ▼
                    ADX Dashboard
                      (Kusto Web Explorer / ADX Portal)
                      └─ Page 1: SSO Success Rate
                      └─ Page 2: SSO Reliability
                      └─ Page 3: SignIn Reliability
                      └─ Page 4: Reauth Rate
```

**Key distinction:**
- **KPI cards** (Daily SSO Score, Weekly Avg, 1D Score, WoW) — query `MSALjs Tenant` directly
- **Stacked bar charts** — read from pre-aggregated `MATS_Reporting` tables computed by ADF pipelines

---

## 2. Apps Covered & App ID Reference

The dashboard supports any MSAL.js-instrumented SPA app via the `App` filter parameter. The following apps are monitored:

| App | Description | App ID |
|---|---|---|
| **CMC** | Microsoft 365 Copilot (browser SPA) — primary focus | `14638111-3389-403d-b206-a6a71d9f8f16` |
| Other MSAL.js SPAs | Additional apps can be onboarded by registering their App ID in the dashboard filter | Contact SISU/MATS team |

> To add a new app: provide the App ID (Azure AD application registration GUID) to the SISU/MATS team. The dashboard App parameter dropdown is updated by the dashboard admin.

---

## 3. Data Sources & Access

| Data Source | Cluster | Database | Description |
|---|---|---|---|
| **MATS_Reporting** | `idsharedeus2.kusto.windows.net` | `MATS_Reporting` | Pre-aggregated SSO metrics. Computed by ADF pipelines. Feeds stacked bar chart widgets. |
| **MSALjs Tenant** | `idsharedeus2.kusto.windows.net` | `MSALJS Tenant` | Raw MSAL.js client telemetry. `allevents` table. App-level events from SPAs. Feeds KPI and line chart widgets. |
| **MATS Admin DB** | `idsharedeus2.kusto.windows.net` | `MATS` | Pipeline ingestion audit and commands. Access restricted to CoreIdentity security group. |

### Requesting Access

**MATS_Reporting:**
- Go to [aka.ms/idweb](https://aka.ms/idweb)
- Join security group: **`odxArMATSRprtCtb`**

**MSALjs Tenant:**
- Go to [IDWeb Groups](https://idweb.microsoft.com/IdentityManagement/aspx/groups/MyGroups.aspx?popupFromClipboard=%2Fidentitymanagement%2Faspx%2FGroups%2FEditGroup.aspx%3Fid%3D8ebd87c2-123f-4bda-9c6d-1ac588498eb5)
- Join group: **MSALJSTelemReadOnly** (Group ID: `8ebd87c2-123f-4bda-9c6d-1ac588498eb5`)

---

## 4. Dashboard Parameters

Global parameters applied across all pages:

| Filter | Default | What It Does |
|---|---|---|
| **App** | CMC (Copilot) — `14638111-3389-403d-b206-a6a71d9f8f16` | Switch between monitored SPA apps |
| **Timespan** | 7 days | Change the data window (1d / 7d / 30d / custom) |
| **BrowserType** | (empty = All) | Filter by browser type (Chrome, Edge, Firefox, Safari) — available on Pages 2, 3, 4 |
| **SSO type** | SSO | Filter SSO reliability by sso or silent path — available on Page 2 only |
| **Error Code** | All | Filter charts by specific MSAL.js client error code — available on Pages 1, 2, 3 |
| **Iframe Error Code** | All | Filter iframe-related breakdown charts by specific error code — available on Page 1 only |

---

## 5. Metric Definitions (Formal)

All definitions sourced from `SSO_Metrics_CMC.docx`. All metrics are **profile-based** unless noted.

### 5.1 SSO Rate

```
SSO Rate = (Profiles that SSO successfully AND never see an interactive prompt)
           ─────────────────────────────────────────────────────────────────────
                  Total profiles that called any MSAL.js API
```

A profile **succeeds** if it achieves silent token acquisition via either:
1. `ssoSilent()` — successful call
2. `acquireTokenSilent()` — with `silentRefreshReason` set OR `silentIframeClientAcquireTokenCallCount > 0`

A profile **fails** if it made **any** interactive call (`acquireTokenRedirect` / `acquireTokenPopup`) during the window — even if it also had silent successes.

> ⚠️ Cookie chaining (the third SSO path) is **not** included due to limitations in determining RT usability from client telemetry. View on the [MSALjs Dashboard](https://dataexplorer.azure.com/dashboards/e79a3b2e-4f31-4773-aa30-61588d098124?p-_flowType=v-silent&p-_startTime=7days&p-_endTime=now&p-_isVisible=all&p-_browser=all&p-_clientId=v-14638111-3389-403d-b206-a6a71d9f8f16&p-_embeddedClientId=all&p-_tenant=all&p-_libVer=all&p-_accountType=v-&p-_isNative=all&p-_isBroker=all&p-_isPlatformBrokerRequest=all&p-_isPlatformAuthorizeRequest=all&p-_clientName=all&p-_clientVersion=all#f1a252ea-64fc-4f4e-b3db-c040957fd91c).

### 5.2 IframeRenewalFailureRate

```
IframeRenewalFailureRate = Failed iframe token renewal attempts
                           ─────────────────────────────────────
                           Total iframe token renewal attempts
```

- Isolates only the `acquireTokenSilent()` iframe path
- An attempt is counted when `isRTIframeRenewal == true`

### 5.3 SSO Reliability

```
SSO Reliability = (Profiles that called ssoSilent successfully) / (Total profiles that called ssoSilent)
```

> Note: This is a simplified definition from the live dashboard. The previous definition that included both ssoSilent and acquireTokenSilent (iframe) paths was the old definition.

Key sub-view: `monitor_window_timeout` is tracked separately — it is the dominant error code for MSA accounts.

### 5.4 SignIn Reliability

```
SignIn Reliability = Profiles that completed interactive sign-in successfully
                     (includes abandoned redirect correction)
                    ───────────────────────────────────────────────────────────
                    Total profiles that initiated an interactive sign-in
```

Interactive APIs tracked: `acquireTokenRedirect()`, `acquireTokenPopup()`

A profile that initiates redirect/popup but never completes (network failure, backend error, user abandonment) is counted as **failure**. Abandoned redirects are computed as `PreRedirectProfiles - PostRedirectProfiles` when positive.

### 5.5 Reauth Rate

Measures how often users with active sessions are **forced into re-authentication**. Triggered by error codes:

```
InteractionRequiredConditions = [
  "interaction_required", "consent_required", "login_required",
  "native_account_unavailable"
]

Reauth Rate = Profiles with any InteractionRequired error in silent flow
              ────────────────────────────────────────────────────────────
              Total profiles in silent flow
```

### 5.6 Score Cards

| Widget | Definition |
|---|---|
| **Daily SSO Score** | SSO Rate % for most recent completed day. Uses `ago(2d)` lag for data completeness. |
| **Weekly SSO Score** | Rolling 7-day average SSO Rate |
| **1D Score** | Most recent day's value for SSO Reliability / SignIn Reliability / Reauth Rate |
| **Weekly Avg** | Trailing 7-day average |
| **WoW Change** | `CurrentWeekAvg - LastWeekAvg` (positive = improvement) |

---

## 6. Dashboard Pages — Technical Detail

### Page 1: SSO Success Rate

**Filters:** Timespan, App

| Widget | Type | Data Source | Notes |
|---|---|---|---|
| Getting access | Info tile | Static | Access instructions for MATS and MSALjs groups |
| SSO Rate Definition | Info tile | Static | Formal definition of SSO Rate and IframeRenewalFailureRate |
| Daily SSO Score | KPI | MSALjs Tenant | SSO Rate % for most recent completed day. Uses `ago(2d)` lag. |
| Weekly SSO Score | KPI | MSALjs Tenant | Rolling 7-day average SSO Rate |
| SSO Rate | Line chart | MATS_Reporting | Daily SSO Rate trend (`MSALJS_SSORate` table) |
| IframeRenewalFailureRate | Line chart | MSALjs Tenant | Daily iframe renewal failure rate |
| SSO Error Breakdown | Stacked bar (profile-based) | MSALjs Tenant | Error codes by SSOImpactRate; filterable by Error Code parameter |
| IframeTokenRenewalFailure Error Breakdown | Stacked bar | MSALjs Tenant / MATS_Reporting | Top error codes for iframe token renewal failures; filterable by Iframe Error Code |
| SSO Detail Error Breakdown | Stacked bar (detailed) | MSALjs Tenant | Detailed error breakdown including brokerErrorCode and serverErrorCode dimensions |
| IframeTokenRenewalFailure Detail Error Breakdown | Stacked bar (detailed) | MSALjs Tenant | Detailed iframe failure breakdown with broker and server error dimensions |
| SSO Server Error Breakdown | Stacked bar | MATS_Reporting | Server-level error breakdown for SSO failures |
| Iframe Server Error Breakdown | Stacked bar | MATS_Reporting | Server-level error breakdown for iframe renewal failures |

### Page 2: SSO Reliability

**Filters:** Timespan, App, BrowserType, SSO type, Error Code

| Widget | Type | Data Source | Notes |
|---|---|---|---|
| SSO Reliability Definition | Info tile | Static | "SSO Reliability is defined as the number of profiles that called sso silent successfully over the total number of profiles that called sso silent." |
| 1D Score | KPI | MSALjs Tenant | SSO Reliability % for most recent day |
| Weekly Avg | KPI | MSALjs Tenant | Trailing 7-day average SSO Reliability |
| WoW Change | KPI | MSALjs Tenant | Week-over-week delta |
| SSO Reliability Error Breakdown | Stacked bar | MATS_Reporting | 22 error codes by SSOImpactRate daily |
| SSO Reliability | Line chart | MSALjs Tenant | Daily SSO Reliability trend |
| Breakdown of SSO Reliability | Stacked bar | MSALjs Tenant | SSO Reliability by outcome path |
| Server Error Breakdown | Stacked bar | MATS_Reporting | Server-level error breakdown for SSO Reliability |
| monitor_window_timeout MSA breakdown | Info tile | Static | Links to MSALjs dashboard MSA timeout analysis; context for the following timeout charts |
| MSA timeout error codes, overall | Bar chart | MSALjs Tenant | `CopilotAggregatedTimeouts` table — MSA timeout sub-types |
| MSA timeout error codes, timeline | Line chart | MSALjs Tenant | `CopilotAggregatedTimeouts` table — timeline of MSA timeout errors |

> **Note:** The `monitor_window_timeout MSA breakdown` widget is an info tile (not a chart) that provides context and links to the [MSALjs Dashboard](https://dataexplorer.azure.com/dashboards/e79a3b2e-4f31-4773-aa30-61588d098124?p-_startTime=7days&p-_endTime=now#dbd204ea-8e88-4dd6-a324-81bc156a1557) for detailed MSA timeout analysis. The actual chart tiles immediately following it (`MSA timeout error codes, overall` and `MSA timeout error codes, timeline`) contain the data visualizations.

**Top Error Codes (SSO Reliability):**

| Error Code | Root Cause |
|---|---|
| `monitor_window_timeout` | Iframe monitor window times out — most frequent for MSA; dedicated breakdown section |
| `login_required` | Server returns login_required — forces interactive prompt; common for expired AAD sessions |
| `interaction_required` | Conditional Access / MFA step-up / token revocation requires interactive flow |
| `invalid_grant` | Refresh token invalid or expired (password change, revoked session, token rotation) |
| `invalid_request` | Malformed request to STS — may indicate client config issue or scope mismatch |
| `hash_empty_error` | Redirect flow returned empty hash — response lost |
| `failed_to_parse_response` | AAD response could not be parsed — network or server-side error |
| `post_request_failed` | Network-level POST failure to token endpoint |
| `server_error` | Generic AAD server error response |
| `state_mismatch` | CSRF state mismatch in redirect flow |
| `consent_required` | Missing OAuth consent for requested scopes |
| `block_iframe_reload` | Iframe reload blocked by CSP or browser policy — critical for iframe SSO |
| `no_network_connectivity` | Client-side network unavailable |
| `endpoints_resolution_error` | Could not resolve OIDC metadata endpoint |
| `authority_mismatch` | Token authority in response does not match request |
| `temporarily_unavailable` | Server returns 503 |

### Page 3: SignIn Reliability

**Filters:** Timespan, App, BrowserType, Error Code

| Widget | Type | Data Source | Notes |
|---|---|---|---|
| SignIn Reliability Definition | Info tile | Static | Formal definition including abandoned redirect handling |
| Weekly Avg | KPI | MSALjs Tenant | 7-day rolling SignIn Reliability |
| 1D Score | KPI | MSALjs Tenant | Yesterday's SignIn Reliability |
| WoW Change | KPI | MSALjs Tenant | Week-over-week delta |
| SignIn Reliability Breakdown (weekly) | Stacked bar (weekly) | MATS_Reporting | 24+ error series over the timespan window |
| SignIn Reliability | Line chart | MSALjs Tenant | Daily SignIn Reliability trend |
| SignIn Reliability Breakdown (daily) | Stacked bar (daily) | MSALjs Tenant | Daily "As of now" breakdown with broker and server error dimensions |
| Server Error Breakdown | Stacked bar | MATS_Reporting | Server-level error breakdown for SignIn failures |

### Page 4: Reauth Rate

**Filters:** Timespan, App, BrowserType

| Widget | Type | Data Source | Notes |
|---|---|---|---|
| Reauth Rate Definition | Info tile | Static | "Reauth Rate is calculated as the number of profiles that hit any of: interaction_required, consent_required, login_required, native_account_unavailable — over the total number of profiles that called an MSALjs API." |
| Weekly Avg | KPI | MSALjs Tenant | 7-day rolling Reauth Rate |
| 1D Score | KPI | MSALjs Tenant | Yesterday's Reauth Rate |
| WoW Change | KPI | MSALjs Tenant | Week-over-week delta |
| Reauth Rate Error Breakdown | Stacked bar | MATS_Reporting | Top error codes driving reauth |
| Reauth Rate | Line chart | MSALjs Tenant | Daily Reauth Rate trend |
| Reauth Breakdown | Stacked bar (daily) | MSALjs Tenant | Reauth by error, broker, and server error code ("As of now") |
| Reauth Server Error Breakdown | Stacked bar | MATS_Reporting | Server-level error breakdown for Reauth ("As of now") |
| Reauth Breakdown (ssoCapable mock data) | Stacked bar | MSALjs Tenant | Experimental: breakdown using ssoCapable flag ("As of now") |

---

## 7. Kusto Queries (All Widgets)

**Cluster:** `https://idsharedeus2.eastus2.kusto.windows.net/`

All queries using `['app']` expect the CMC App ID: `14638111-3389-403d-b206-a6a71d9f8f16` (or whichever app is selected in the dashboard filter).

---

### Page 1 — SSO Success Rate

#### Daily SSO Score
**Source:** `MSALjs Tenant` | Database: `d634483c08244c1ca09af2b2d952c92e`

```kusto
let AppId = ['app'];
let Day = ago(2d);
let StartofDay = startofday(Day);
let EndofDay   = endofday(Day);
database("MSALJS Tenant").allevents
| where name in ("acquireTokenSilent", "ssoSilent", "acquireTokenRedirect", "acquireTokenPopup")
| where clientId == AppId
| where EventInfo_Time between (StartofDay .. EndofDay)
| extend ProfileId = profileTelemetryId
// ---- Per-profile daily classification ----
| summarize hint.shufflekey=ProfileId
    SilentSuccess =
        countif(
            (
                name == "ssoSilent"
                or (
                    name == "acquireTokenSilent"
                    and (isnotempty(silentRefreshReason) or parse_json(ext)["silentIframeClientAcquireTokenCallCount"] > 0)
                )
            )
            and success == true
        ),
    InteractiveRequests =
        countif(name in ("acquireTokenPopup", "acquireTokenRedirect"))
    by ProfileId
// ---- Daily SSR computation ----
| summarize
    SSOProfiles =
        countif(SilentSuccess > 0 and InteractiveRequests == 0),
    TotalProfiles = count()
| extend SSORate = 100.0 * todouble(SSOProfiles) / TotalProfiles
| project
    UploadTime = StartofDay,
    SSORate = round(SSORate, 2)
```

> **Note:** The 2-day lag (`ago(2d)`) is intentional — ensures MSALjs data has fully landed before being queried.

#### Weekly SSO Score
**Source:** `MSALjs Tenant`

```kusto
let AppId = ['app'];

// ---- Rolling past 7 days (including today-1) ----
let EndTime   = now();
let StartTime = ago(7d);

database("MSALJS Tenant").allevents
| where name in ("acquireTokenSilent", "ssoSilent", "acquireTokenRedirect", "acquireTokenPopup")
| where clientId == AppId
| where EventInfo_Time between (StartTime .. EndTime)
| extend ProfileId = profileTelemetryId

// ---- Per-profile 7-day classification ----
| summarize hint.shufflekey=ProfileId
    SilentSuccess =
        countif(
            (
                name == "ssoSilent"
                or (
                    name == "acquireTokenSilent"
                    and (isnotempty(silentRefreshReason) or parse_json(ext)["silentIframeClientAcquireTokenCallCount"] > 0)
                )
            )
            and success == true
        ),
    InteractiveRequests =
        countif(name in ("acquireTokenPopup", "acquireTokenRedirect"))
    by ProfileId

// ---- 7-day SSR computation ----
| summarize
    SSOProfiles =
        countif(SilentSuccess > 0 and InteractiveRequests == 0),
    TotalProfiles = count()
| extend SSORate =
    iff(
        TotalProfiles == 0,
        0.0,
        100.0 * todouble(SSOProfiles) / TotalProfiles
    )
| project
    UploadTime = startofday(EndTime),
    SSORate = round(SSORate, 2)
```

#### SSO Rate (Line Chart)
**Source:** `MATS_Reporting`

```kusto
let app = '14638111-3389-403d-b206-a6a71d9f8f16';
let ['timespan'] = '7d';
MSALJS_SSORate
| where EventInfo_Time > ago(totimespan(['timespan']))
| where clientId == ['app']
| project SSORate, EventInfo_Time
```

#### IframeRenewalFailureRate
**Source:** `MSALjs Tenant`

```kusto
let app = '14638111-3389-403d-b206-a6a71d9f8f16';
let ['timespan'] = '7d';
let LookbackTime = startofday(ago(totimespan(['timespan'])));
database("MSALJS Tenant").getMaterializedData(_flowType="silent", _startTime=LookbackTime,
_endTime=endofday(ago(1d)), _clientId=['app'])
| as T
| where success==false and isRTIframeRenewal == true
| summarize SilentIframeFailures=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize TotalIframeRenewals=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
) on EventInfo_Time
| extend IframeRenewalFailureRate = round((todouble(SilentIframeFailures)/TotalIframeRenewals) * 100, 5)
| project EventInfo_Time, IframeRenewalFailureRate
```

#### SSO Rate Error Breakdown
**Source:** `MSALjs Tenant`

```kusto
let app = '14638111-3389-403d-b206-a6a71d9f8f16';
let ['timespan'] = '7d';
let LookbackTime = startofday(ago(totimespan(['timespan'])));
let T =
    database("MSALJS Tenant").getMaterializedData(_flowType="sso", _startTime=LookbackTime,
    _endTime=endofday(ago(1d)), _clientId=['app']);
let totalProfiles =
    T
    | summarize TotalSSOProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d);
T
| where success == false
| summarize ImpactedSSOProfiles=dcount_hll(hll_merge(profilesHll)) by errorCode, EventInfo_Time = bin(EventInfo_Time, 1d)
| lookup totalProfiles on EventInfo_Time
| extend SSOImpactRate = round((todouble(ImpactedSSOProfiles) / TotalSSOProfiles) * 100, 5)
| project EventInfo_Time, errorCode, SSOImpactRate
| sort by SSOImpactRate desc
```

#### IframeTokenRenewal Failure Error Breakdown
**Source:** `MSALjs Tenant` / `MATS_Reporting`

```kusto
let app = '14638111-3389-403d-b206-a6a71d9f8f16';
let ['timespan'] = '7d';
let LookbackTime = startofday(ago(totimespan(['timespan'])));
let T =
    database("MSALJS Tenant").getMaterializedData(_flowType="silent", _startTime=LookbackTime,
    _endTime=endofday(ago(1d)), _clientId=['app']);
let totalProfiles =
    T
    | summarize TotalSSOProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d);
T
| where success == false and isRTIframeRenewal == true
| summarize ImpactedSSOProfiles=dcount_hll(hll_merge(profilesHll)) by errorCode, EventInfo_Time = bin(EventInfo_Time, 1d)
| lookup totalProfiles on EventInfo_Time
| extend SSOImpactRate = round((todouble(ImpactedSSOProfiles) / TotalSSOProfiles) * 100, 5)
| project EventInfo_Time, errorCode, SSOImpactRate
| sort by SSOImpactRate desc
```

#### SSO Reliability Breakdown
**Source:** `MSALjs Tenant`

```kusto
let app = '14638111-3389-403d-b206-a6a71d9f8f16';
materialized_view('allEventsView')
| where name == "ssoSilent"
| where EventInfo_Time between (startofday(ago(1d)) .. endofday(ago(1d)))
| where clientId == ['app']
| as T
| where success==false
| summarize ImpactedSSOProfiles=dcount_hll(hll_merge(profilesHll)) by errorCode, brokerErrorCode,
serverErrorCode=tostring(split(serverErrorNo, ".")[0]), bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize TotalSSOProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
) on EventInfo_Time
| extend SSOImpactRate = round((todouble(ImpactedSSOProfiles)/TotalSSOProfiles) * 100, 5)
| sort by SSOImpactRate desc
```

#### IframeTokenRenewal Breakdown
**Source:** `MSALjs Tenant`

```kusto
let app = '14638111-3389-403d-b206-a6a71d9f8f16';
materialized_view('allEventsView')
| where name == "acquireTokenSilent"
| where EventInfo_Time between (startofday(ago(1d)) .. endofday(ago(1d)))
| where clientId == ['app']
| as T
| where success==false and isRTIframeRenewal == true
| summarize ImpactedSSOProfiles=dcount_hll(hll_merge(profilesHll)) by errorCode, brokerErrorCode,
serverErrorCode=tostring(split(serverErrorNo, ".")[0]), bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize TotalSSOProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
) on EventInfo_Time
| extend SSOImpactRate = round((todouble(ImpactedSSOProfiles)/TotalSSOProfiles) * 100, 5)
| sort by SSOImpactRate desc
```

---

### Page 2 — SSO Reliability

#### 1D Score
**Source:** `MSALjs Tenant`

```kusto
let 1dscore=
union
(
materialized_view('allEventsView')
| where name == "ssoSilent"
| where EventInfo_Time between (startofday(ago(1d)) .. endofday(ago(1d)))
| where clientId == ['app']
| where isempty(['ssoType']) or ['ssoType'] == "sso"
| where browser == ['browser_type'] or isempty(['browser_type'])
),
(
materialized_view('allEventsView')
| where name == "acquireTokenSilent"
| where EventInfo_Time between (startofday(ago(1d)) .. endofday(ago(1d)))
| where clientId == ['app']
| where isempty(['ssoType']) or ['ssoType'] == "silent"
| where browser == ['browser_type'] or isempty(['browser_type'])
| where isRTIframeRenewal == true
)
| as T
| where success==false
| summarize SSOFailedProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize SSOProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d) ) on EventInfo_Time
| extend SSOReliability = round((1 - todouble(SSOFailedProfiles)/SSOProfiles) * 100, 2)
| project SSOReliability;
print(strcat(toscalar(1dscore), "%"))
```

#### Weekly Avg
**Source:** `MSALjs Tenant`

```kusto
let weeklyscore =
union
(
getMaterializedData(_flowType="sso", _startTime=startofday(ago(7d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| where isempty(['ssoType']) or ['ssoType'] == "sso"
),
(
getMaterializedData(_flowType="sso", _startTime=startofday(ago(7d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| where isempty(['ssoType']) or ['ssoType'] == "sso"
| where isRTIframeRenewal == true
)
| as T
| where success==false
| summarize SSOFailedProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize SSOProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d) ) on EventInfo_Time
| extend SSOReliability = round((1 - todouble(SSOFailedProfiles)/SSOProfiles) * 100, 2)
| summarize round(avg(SSOReliability), 2);
print(strcat(toscalar(weeklyscore), "%"))
```

#### WoW Change
**Source:** `MSALjs Tenant`

```kusto
let ThisWeekAvg =
union
(
getMaterializedData(_flowType="sso", _startTime=startofday(ago(7d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| where isempty(['ssoType']) or ['ssoType'] == "sso"
),
(
getMaterializedData(_flowType="silent", _startTime=startofday(ago(7d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| where isempty(['ssoType']) or ['ssoType'] == "silent"
| where isRTIframeRenewal == true
)
| as T
| where success==false
| summarize SSOFailedProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize SSOProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d) ) on EventInfo_Time
| extend SSOReliability = (1 - todouble(SSOFailedProfiles)/SSOProfiles) * 100
| summarize round(avg(SSOReliability), 2);
let LastWeekAvg =
union
(
getMaterializedData(_flowType="sso", _startTime=startofday(ago(14d)), _endTime=endofday(ago(8d)), _clientId=['app'], _browser=['browser_type'])
| where isempty(['ssoType']) or ['ssoType'] == "sso"
),
(
getMaterializedData(_flowType="sso", _startTime=startofday(ago(14d)), _endTime=endofday(ago(8d)), _clientId=['app'], _browser=['browser_type'])
| where isempty(['ssoType']) or ['ssoType'] == "silent"
| where isRTIframeRenewal == true
)
| as T
| where success==false
| summarize SSOFailedProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize SSOProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d) ) on EventInfo_Time
| extend SSOReliability = (1 - todouble(SSOFailedProfiles)/SSOProfiles) * 100
| summarize round(avg(SSOReliability), 2);
let WoWChange = round(toscalar(ThisWeekAvg) - toscalar(LastWeekAvg), 2);
print(strcat(iff(WoWChange > 0, "+", ""), WoWChange, "%"))
```

#### SSO Reliability Error Breakdown
**Source:** `MATS_Reporting`

```kusto
let LookbackTime = startofday(ago(totimespan(['timespan'])));
let BaseData = union
(
    database("MSALJS Tenant").getMaterializedData(_flowType="sso", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | where isempty(['ssoType']) or ['ssoType'] == "sso"
),
(
    database("MSALJS Tenant").getMaterializedData(_flowType="silent", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | where isempty(['ssoType']) or ['ssoType'] == "silent"
    | where isRTIframeRenewal == true
);
let TotalPerDay = BaseData
| summarize TotalSSOProfiles = dcount_hll(hll_merge(profilesHll)) by Day = bin(EventInfo_Time, 1d);
BaseData
| where success == false
| summarize ImpactedSSOProfiles = dcount_hll(hll_merge(profilesHll)) by errorCode, brokerErrorCode,
  serverErrorCode = tostring(split(serverErrorNo, ".")[0]), Day = bin(EventInfo_Time, 1d)
| join kind=inner TotalPerDay on Day
| extend SSOImpactRate = round(todouble(ImpactedSSOProfiles) / TotalSSOProfiles * 100, 5)
| project Day, errorCode, SSOImpactRate, ImpactedSSOProfiles, TotalSSOProfiles
| order by SSOImpactRate desc
```

#### SSO Reliability (Line Chart)
**Source:** `MSALjs Tenant`

```kusto
let LookbackTime = startofday(ago(totimespan(['timespan'])));
union
(
materialized_view('allEventsView')
| where name == "ssoSilent"
| where EventInfo_Time between (LookbackTime .. endofday(ago(1d)))
| where clientId == ['app']
| where browser == ['browser_type'] or isempty(['browser_type'])
| where isempty(['ssoType']) or ['ssoType'] == "sso"
),
(
materialized_view('allEventsView')
| where name == "acquireTokenSilent"
| where EventInfo_Time between (LookbackTime .. endofday(ago(1d)))
| where clientId == ['app']
| where browser == ['browser_type'] or isempty(['browser_type'])
| where isempty(['ssoType']) or ['ssoType'] == "silent"
| where isRTIframeRenewal == true
)
| as T
| where success==false
| summarize SSOFailedProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize SSOProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
) on EventInfo_Time
| extend SSOReliability = round((1 - todouble(SSOFailedProfiles)/SSOProfiles) * 100, 2)
| project EventInfo_Time, SSOReliability
```

#### MSA timeout error codes, overall
**Source:** `MSALjs Tenant` — `CopilotAggregatedTimeouts` table

```kusto
CopilotAggregatedTimeouts
| where Env_time between (startofday(ago(1d)) .. endofday(ago(1d)))
| summarize Total = sum(Total) by ErrorCode = GlobalErrorCode_FN
```

#### MSA timeout error codes, timeline
**Source:** `MSALjs Tenant` — `CopilotAggregatedTimeouts` table

```kusto
let LookbackTime = startofday(ago(totimespan(['timespan'])));
CopilotAggregatedTimeouts
| where Env_time between (LookbackTime .. endofday(ago(1d)))
| project Env_time, ErrorCode = GlobalErrorCode_FN, Total
```

#### Breakdown of SSO Reliability
**Source:** `MSALjs Tenant`

```kusto
union
(
materialized_view('allEventsView')
| where name == "ssoSilent"
| where EventInfo_Time between (startofday(ago(1d)) .. endofday(ago(1d)))
| where clientId == ['app']
| where isempty(['ssoType']) or ['ssoType'] == "sso"
| where browser == ['browser_type'] or isempty(['browser_type'])
),
(
materialized_view('allEventsView')
| where name == "acquireTokenSilent"
| where EventInfo_Time between (startofday(ago(1d)) .. endofday(ago(1d)))
| where clientId == ['app']
| where isempty(['ssoType']) or ['ssoType'] == "silent"
| where browser == ['browser_type'] or isempty(['browser_type'])
| where isRTIframeRenewal == true
)
| as T
| where success==false
| summarize ImpactedSSOProfiles=dcount_hll(hll_merge(profilesHll)) by errorCode, brokerErrorCode,
serverErrorCode=tostring(split(serverErrorNo, ".")[0]), bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize TotalSSOProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
) on EventInfo_Time
| extend SSOImpactRate = round((todouble(ImpactedSSOProfiles)/TotalSSOProfiles) * 100, 5)
| sort by SSOImpactRate desc
```

---

### Page 3 — SignIn Reliability

#### Weekly Avg
**Source:** `MSALjs Tenant`

```kusto
let redirectProfiles = materialize(
    getMaterializedData(_flowType="redirect", _startTime=startofday(ago(7d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | summarize PostRedirectProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d)
    | join kind=leftouter (
        getMaterializedData(_flowType="preredirect", _startTime=startofday(ago(7d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
        | where success == true
        | summarize PreRedirectProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d)
    ) on ($left.EventInfo_Time == $right.EventInfo_Time)
    | project
        PostRedirectProfiles,
        PreRedirectProfiles = iff(isempty(PreRedirectProfiles), PostRedirectProfiles, PreRedirectProfiles),
        EventInfo_Time
);
let weeklyscore =
getMaterializedData(_flowType="popup", _startTime=startofday(ago(7d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| union getMaterializedData(_flowType="redirect", _startTime=startofday(ago(7d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| as T
| where success==false
| summarize SignInFailedProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize SignInProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d) ) on EventInfo_Time
| lookup (
    redirectProfiles
    | project
        AbandonedRedirectProfiles = iff(PreRedirectProfiles > PostRedirectProfiles, PreRedirectProfiles - PostRedirectProfiles, 0),
        EventInfo_Time
) on EventInfo_Time
| extend SignInReliability = round((1 - todouble(AbandonedRedirectProfiles + SignInFailedProfiles)/SignInProfiles) * 100, 2)
| summarize round(avg(SignInReliability), 2);
print(strcat(toscalar(weeklyscore), "%"))
```

#### 1D Score
**Source:** `MSALjs Tenant`

```kusto
let redirectProfiles = materialize(
    getMaterializedData(_flowType="redirect", _startTime=startofday(ago(1d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | summarize PostRedirectProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d)
    | join kind=leftouter (
        getMaterializedData(_flowType="preredirect", _startTime=startofday(ago(1d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
        | where success == true
        | summarize PreRedirectProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d)
    ) on ($left.EventInfo_Time == $right.EventInfo_Time)
    | project
        PostRedirectProfiles,
        PreRedirectProfiles = iff(isempty(PreRedirectProfiles), PostRedirectProfiles, PreRedirectProfiles),
        EventInfo_Time
);
let 1dscore=
getMaterializedData(_flowType="popup", _startTime=startofday(ago(1d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| union getMaterializedData(_flowType="redirect", _startTime=startofday(ago(1d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| as T
| where success==false
| summarize SignInFailedProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize SignInProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d) ) on EventInfo_Time
| lookup (
    redirectProfiles
    | project
        AbandonedRedirectProfiles = iff(PreRedirectProfiles > PostRedirectProfiles, PreRedirectProfiles - PostRedirectProfiles, 0),
        EventInfo_Time
) on EventInfo_Time
| extend SignInReliability = round((1 - todouble(AbandonedRedirectProfiles + SignInFailedProfiles)/SignInProfiles) * 100, 2)
| project SignInReliability;
print(strcat(toscalar(1dscore), "%"))
```

#### SignIn Reliability Breakdown (Weekly)
**Source:** `MATS_Reporting`

```kusto
let LookbackTime = startofday(ago(totimespan(['timespan'])));
let redirectProfiles = materialize(
    database("MSALJS Tenant").getMaterializedData(_flowType="redirect", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | summarize PostRedirectProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d)
    | join kind=leftouter (
        database("MSALJS Tenant").getMaterializedData(_flowType="preredirect", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
        | where success == true
        | summarize PreRedirectProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d)
    ) on ($left.EventInfo_Time == $right.EventInfo_Time)
    | project
        PostRedirectProfiles,
        PreRedirectProfiles = iff(isempty(PreRedirectProfiles), PostRedirectProfiles, PreRedirectProfiles),
        EventInfo_Time
);
let T =
    database("MSALJS Tenant").getMaterializedData(_flowType="redirect", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | union database("MSALJS Tenant").getMaterializedData(_flowType="preredirect", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | union database("MSALJS Tenant").getMaterializedData(_flowType="popup", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type']);
let totalProfiles =
    T
    | summarize TotalSignInProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d);
union
(
    T
    | where success == false
    | summarize ImpactedSignInProfiles=dcount_hll(hll_merge(profilesHll)) by errorCode, EventInfo_Time = bin(EventInfo_Time, 1d)
    | lookup totalProfiles on EventInfo_Time
    | extend SignInImpactRate = round((todouble(ImpactedSignInProfiles) / TotalSignInProfiles) * 100, 5)
),
(
    totalProfiles
    | lookup (
        redirectProfiles
        | project
            ImpactedSignInProfiles = iff(PreRedirectProfiles > PostRedirectProfiles, PreRedirectProfiles - PostRedirectProfiles, 0),
            EventInfo_Time
    ) on EventInfo_Time
    | extend errorCode = "abandoned_redirect",
            SignInImpactRate = round((todouble(ImpactedSignInProfiles) / TotalSignInProfiles) * 100, 5)
)
| project EventInfo_Time, errorCode, SignInImpactRate
| sort by SignInImpactRate desc
```

#### SignIn Reliability (Line Chart)
**Source:** `MSALjs Tenant`

```kusto
let LookbackTime = startofday(ago(totimespan(['timespan'])));
let redirectProfiles = materialize(
    database("MSALJS Tenant").getMaterializedData(_flowType="redirect", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | summarize PostRedirectProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d)
    | join kind=leftouter (
        database("MSALJS Tenant").getMaterializedData(_flowType="preredirect", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'], _success=true)
        | summarize PreRedirectProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d)
    ) on ($left.EventInfo_Time == $right.EventInfo_Time)
    | project
        PostRedirectProfiles,
        PreRedirectProfiles = iff(isempty(PreRedirectProfiles), PostRedirectProfiles, PreRedirectProfiles),
        EventInfo_Time
);
database("MSALJS Tenant").getMaterializedData(_flowType="redirect", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| union database("MSALJS Tenant").getMaterializedData(_flowType="popup", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| as T
| where success==false
| summarize SignInFailedProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    T
    | summarize SignInProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
) on EventInfo_Time
| lookup (
    redirectProfiles
    | project
        AbandonedRedirectProfiles = iff(PreRedirectProfiles > PostRedirectProfiles, PreRedirectProfiles - PostRedirectProfiles, 0),
        EventInfo_Time
) on EventInfo_Time
| extend SignInReliability = round((1 - todouble(AbandonedRedirectProfiles + SignInFailedProfiles)/SignInProfiles) * 100, 2)
| where EventInfo_Time >= datetime(2026-03-28)
| project EventInfo_Time, SignInReliability
```

---

### Page 4 — Reauth Rate

#### Weekly Avg
**Source:** `MSALjs Tenant`

```kusto
let InteractionRequiredConditions = datatable(errorCode:string)[
    "interaction_required", "consent_required", "login_required",
    "bad_token", "no_tokens_found", "native_account_unavailable", "refresh_token_expired"];
let weeklyscore =
getMaterializedData(_flowType="silent", _startTime=startofday(ago(7d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| as T
| where errorCode in (InteractionRequiredConditions)
| summarize ReauthProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    getMaterializedData(_flowType="silent", _startTime=startofday(ago(7d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | summarize AllProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d) ) on EventInfo_Time
| extend ReauthRate = round((todouble(ReauthProfiles)/AllProfiles) * 100, 2)
| summarize round(avg(ReauthRate), 2);
print(strcat(toscalar(weeklyscore), "%"))
```

#### 1D Score
**Source:** `MSALjs Tenant`

```kusto
let InteractionRequiredConditions = datatable(errorCode:string)[
    "interaction_required", "consent_required", "login_required",
    "bad_token", "no_tokens_found", "native_account_unavailable", "refresh_token_expired"];
let 1dscore=
getMaterializedData(_flowType="silent", _startTime=startofday(ago(1d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| as T
| where errorCode in (InteractionRequiredConditions)
| summarize ReauthProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    getMaterializedData(_flowType="silent", _startTime=startofday(ago(1d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | summarize AllProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d) ) on EventInfo_Time
| extend ReauthRate = round((todouble(ReauthProfiles)/AllProfiles) * 100, 2)
| project ReauthRate;
print(strcat(toscalar(1dscore), "%"))
```

#### Reauth Rate Error Breakdown
**Source:** `MATS_Reporting`

```kusto
let LookbackTime = startofday(ago(totimespan(['timespan'])));
let InteractionRequiredConditions = datatable(errorCode:string)[
    "interaction_required", "consent_required", "login_required", "native_account_unavailable"];
let T =
   database("MSALJS Tenant").getMaterializedData(_flowType="silent", _startTime=LookbackTime,
   _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type']);
let totalProfiles =
    T
    | summarize AllProfiles=dcount_hll(hll_merge(profilesHll)) by EventInfo_Time = bin(EventInfo_Time, 1d);
T
| where errorCode in (InteractionRequiredConditions)
| summarize ReauthProfiles=dcount_hll(hll_merge(profilesHll))
    by errorCode,
       EventInfo_Time = bin(EventInfo_Time, 1d)
| lookup totalProfiles on EventInfo_Time
| extend ReauthRate = round((todouble(ReauthProfiles) / AllProfiles) * 100, 5)
| project EventInfo_Time, errorCode, ReauthRate
| sort by ReauthRate desc
```

#### Reauth Rate (Line Chart)
**Source:** `MSALjs Tenant`

```kusto
let InteractionRequiredConditions = datatable(errorCode:string)[
    "interaction_required", "consent_required", "login_required", "native_account_unavailable"];
let LookbackTime = startofday(ago(totimespan(['timespan'])));
getMaterializedData(_flowType="silent", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| as T
| where errorCode in (InteractionRequiredConditions)
| summarize ReauthProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
| lookup (
    getMaterializedData(_flowType="silent", _startTime=LookbackTime, _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | summarize AllProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
) on EventInfo_Time
| extend ReauthRate = round((todouble(ReauthProfiles)/AllProfiles) * 100, 2)
| where EventInfo_Time >= datetime(2026-03-28)
| project EventInfo_Time, ReauthRate
```

#### Reauth Breakdown
**Source:** `MSALjs Tenant`

```kusto
let InteractionRequiredConditions = datatable(errorCode:string)[
    "interaction_required", "consent_required", "login_required", "native_account_unavailable"];

getMaterializedData(_flowType="silent", _startTime=startofday(ago(1d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
| as T
| where errorCode in (InteractionRequiredConditions)
| summarize ReauthProfiles=dcount_hll(hll_merge(profilesHll)) by errorCode, brokerErrorCode,
serverErrorCode=tostring(split(serverErrorNo, ".")[0]), bin(EventInfo_Time, 1d)
| lookup (
    getMaterializedData(_flowType="silent", _startTime=startofday(ago(1d)), _endTime=endofday(ago(1d)), _clientId=['app'], _browser=['browser_type'])
    | summarize AllProfiles=dcount_hll(hll_merge(profilesHll)) by bin(EventInfo_Time, 1d)
) on EventInfo_Time
| extend ReauthRate = round((todouble(ReauthProfiles)/AllProfiles) * 100, 5)
| extend serverErrorCode=coalesce(serverErrorCode, errorCode)
| sort by ReauthRate desc
```

---

## 8. ADF Pipelines

**Factory:** `/subscriptions/10acc9a9-ecbe-4d31-a4f5-4b86c1687040/resourceGroups/MatsData/providers/Microsoft.DataFactory/factories/MATS-ADF`

| Pipeline | Purpose | Feeds |
|---|---|---|
| `CMC_SSO_Success` | Computes SSO success rate and error breakdown per profile per day | SSO Rate Error Breakdown, IframeTokenRenewal Failure Error Breakdown, SSO Reliability Breakdown, IframeTokenRenewal Breakdown (Page 1) |
| `SSO_Reauth_Error` | Computes reauth error rate and top reauth error codes | Reauth Rate Error Breakdown, Reauth Rate chart, Reauth Breakdown (Pages 4) |
| `SSO_Reliability_Top_Error` | Computes top error codes for SSO Reliability and SignIn Reliability | SSO Reliability Error Breakdown (Page 2), SignIn Reliability Breakdown (Page 3) |

### 8.1 Operational Notes

- All 3 CMC pipelines use the **MATS-ADF service principal** for Kusto ingestion commands (`TableSetOrAppend`)
- Pipelines write to tables in `MATS_Reporting` database on `idsharedeus2.kusto.windows.net`
- **Triggers:** Use **v2-suffixed triggers** (created by Chase Hawthorne) — do NOT re-enable deprecated old triggers
- **Kusto concurrency:** Cluster allows up to **30 concurrent requests per service principal** (increased from 8 in April 2026). Max cluster concurrency: ~160 requests
- MATS workloads observed at ~23 requests/second at peak

### 8.2 Pipeline Failure Response

If CMC pipelines fail, stacked bar chart widgets will show stale or empty results. Follow these steps:

1. **Check ADF run history** — review pipeline run history in [Azure Data Factory portal](https://portal.azure.com/#/subscriptions/10acc9a9-ecbe-4d31-a4f5-4b86c1687040/resourceGroups/MatsData/providers/Microsoft.DataFactory/factories/MATS-ADF)
2. **Verify Kusto cluster health** — confirm no concurrency throttling or ingestion backlog
3. **Confirm trigger** — ensure v2-suffixed trigger is enabled; do NOT re-enable old triggers
4. **SISU pipeline concurrency** — if re-enabling SISU pipelines alongside CMC, confirm total concurrent pipeline count ≤ 6 to avoid cluster overload
5. **Escalate** — contact Chase Hawthorne for cluster-level issues or principal configuration changes

### 8.3 Kusto Ingestion Monitoring

To check pipeline ingestion health:

```kusto
// Run on: idsharedeus2/MATS (admin access required)
// Filter by service principal ID for MATS-ADF
.show commands
| where StartedOn > ago(1d)
| where Principal contains "a9f8e906"
| summarize count() by CommandType, bin(StartedOn, 1h)
| order by StartedOn desc
```

---

## 9. MSAL.js Authentication Flow

### 9.1 What is MSAL.js?

MSAL.js (Microsoft Authentication Library for JavaScript) is Microsoft's OAuth 2.0 / OpenID Connect client library for browser-based SPAs. CMC and other monitored apps use MSAL.js to acquire access tokens for Microsoft 365 services (Graph, Substrate, etc.) on behalf of signed-in users.

### 9.2 MSAL.js APIs Monitored

The MATS SPA Dashboard monitors four MSAL.js APIs:

| API | Flow Type | SSO Impact | Description |
|---|---|---|---|
| `ssoSilent()` | `sso` | Yes — success | Silent SSO using existing AAD session cookie. No network token request. Primary SSO path. |
| `acquireTokenSilent()` | `silent` | Yes — success (if iframe) | Checks cache, then attempts hidden iframe token renewal. Falls back to interactive if both fail. |
| `acquireTokenRedirect()` | `redirect` / `preredirect` | Yes — failure | Full-page redirect to AAD login. Counts as SSO failure. |
| `acquireTokenPopup()` | `popup` | Yes — failure | Popup window to AAD login. Non-disruptive to SPA state. Counts as SSO failure. |

### 9.3 Silent Authentication Paths

| Path | Tracked in SSO Rate? | Key Field | Description |
|---|---|---|---|
| `ssoSilent()` | ✅ YES | `name == "ssoSilent"` | Browser session cookie. Zero network calls if session exists. Fastest path. |
| `acquireTokenSilent` (iframe renewal) | ✅ YES | `silentRefreshReason != ""` OR `silentIframeClientAcquireTokenCallCount > 0` | Hidden iframe navigates to STS endpoint. Slower but robust fallback. |
| Cookie Chaining (RT-based) | ❌ NO | N/A | Refresh token passed via cookie to STS. Cannot reliably determine RT usability from client telemetry. View on [MSALjs Dashboard](https://dataexplorer.azure.com/dashboards/e79a3b2e-4f31-4773-aa30-61588d098124?p-_flowType=v-silent&p-_startTime=7days&p-_endTime=now&p-_isVisible=all&p-_browser=all&p-_clientId=v-14638111-3389-403d-b206-a6a71d9f8f16&p-_embeddedClientId=all&p-_tenant=all&p-_libVer=all&p-_accountType=v-&p-_isNative=all&p-_isBroker=all&p-_isPlatformBrokerRequest=all&p-_isPlatformAuthorizeRequest=all&p-_clientName=all&p-_clientVersion=all#f1a252ea-64fc-4f4e-b3db-c040957fd91c). |

### 9.4 Why `monitor_window_timeout` is Critical

The `monitor_window_timeout` error occurs when the hidden iframe used for silent token renewal does not return a response within the allowed timeout window. It is the **#1 SSO Reliability error code by volume** for CMC.

- **Root cause:** MSA (Microsoft Account) STS endpoints sometimes respond slowly, causing the iframe monitor to time out before receiving the token response
- **Impact:** Affected profiles fall through to interactive sign-in, breaking the seamless Copilot experience
- **Dedicated analysis:** Dashboard Page 2 has three dedicated MSA timeout widgets: `monitor_window_timeout MSA breakdown`, `MSA timeout error codes, overall`, `MSA timeout error codes, timeline` — all querying `CopilotAggregatedTimeouts`
- **Mitigation being explored:** Increase iframe timeout threshold; optimize MSA STS response time

### 9.5 Key Telemetry Fields

| Field | Table | Description |
|---|---|---|
| `name` | `allevents` | API name: `ssoSilent`, `acquireTokenSilent`, `acquireTokenRedirect`, `acquireTokenPopup` |
| `success` | `allevents` | Boolean — whether the API call succeeded |
| `clientId` | `allevents` | App ID of the calling SPA |
| `profileTelemetryId` | `allevents` | Anonymized per-user profile identifier |
| `silentRefreshReason` | `allevents` | Set when `acquireTokenSilent` used iframe renewal |
| `isRTIframeRenewal` | `getMaterializedData()` | Indicates iframe renewal path in `acquireTokenSilent` |
| `errorCode` | Both | MSAL.js error code on failure |
| `brokerErrorCode` | Both | Underlying broker/server error code |
| `serverErrorNo` | Both | AAD server error number |
| `browser` | `getMaterializedData()` | Browser type: Chrome, Edge, Firefox, Safari |
| `ext` | `allevents` | JSON extension field; contains `silentIframeClientAcquireTokenCallCount` |
| `GlobalErrorCode_FN` | `CopilotAggregatedTimeouts` | MSA timeout error category |

---

## 10. On-Call Guide

### 10.1 Daily Health Check Checklist

During the MATS/SISU on-call rotation, verify the following each day:

| Check Item | What to Look For | Action If Wrong |
|---|---|---|
| **Daily SSO Score (Page 1)** | Should be > 10%. Drop below 8% needs investigation. | Check ADF pipeline runs; review MSALjs event trends |
| **IframeRenewalFailureRate (Page 1)** | Should be < 4%. Spikes indicate iframe path issues. | Investigate iframe renewal errors in Page 1 charts |
| **ADF Pipeline Runs** | All 3 CMC pipelines must show successful runs | See Section 8.2 — Pipeline Failure Response |
| **SSO Rate Error Breakdown (Page 1)** | Review top error codes. Flag new errors or known errors spiking. | Cross-reference with known incidents; escalate if new pattern |
| **SignIn Reliability (Page 3)** | 1D Score should be > 60%. Watch WoW trend direction. | If declining, review SignIn Reliability Breakdown for dominant errors |
| **monitor_window_timeout (Page 2)** | #1 error by volume — any spike needs investigation. | Review MSA timeout breakdown; check MSA STS health |

### 10.2 Escalation Contacts

| Issue Type | Contact |
|---|---|
| Dashboard query errors / data freshness | Gaurav Usadadiya, Shivani Singh, Abhishek Bag, Sridhar Dantuluri |
| ADF pipeline failures / triggers | IDC team (Gaurav, Shivani, Abhishek) |
| Kusto cluster throttling / capacity | **Chase Hawthorne** (concurrency limit: 30 per principal, 160 overall) |
| CMC product requirements / metric questions | **Sameera Gajjarapu** (Sameera.Gajjarapu@microsoft.com) |
| MSALjs telemetry data issues | Join MSALJSTelemReadOnly; check MSALjs Dashboard |
| Access provisioning (MATS security group) | [aka.ms/idweb](https://aka.ms/idweb) → join `odxArMATSRprtCtb` |

### 10.3 Known Limitations

| Limitation | Detail |
|---|---|
| **Cookie chaining not tracked** | Cookie chaining SSO path is not in the SSO Rate metric. Use the separate [MSALjs Dashboard](https://dataexplorer.azure.com/dashboards/e79a3b2e-4f31-4773-aa30-61588d098124?p-_flowType=v-silent&p-_startTime=7days&p-_endTime=now&p-_isVisible=all&p-_browser=all&p-_clientId=v-14638111-3389-403d-b206-a6a71d9f8f16&p-_embeddedClientId=all&p-_tenant=all&p-_libVer=all&p-_accountType=v-&p-_isNative=all&p-_isBroker=all&p-_isPlatformBrokerRequest=all&p-_isPlatformAuthorizeRequest=all&p-_clientName=all&p-_clientVersion=all#f1a252ea-64fc-4f4e-b3db-c040957fd91c). |
| **2-day data lag** | All KPI cards use `ago(2d)` as the reference point to ensure full event data landing. |
| **Sparse line chart data points** | SSO Rate line chart may show only a few data points if certain days have insufficient profile volume. |
| **BrowserType filter not universal** | Only available on Page 2 (SSO Reliability). |
| **Weekly SSO Score "No results"** | May appear if trailing window has insufficient data or pipeline delay. Check ADF. |

---

## 11. Infrastructure Improvements (Roadmap)

The following infrastructure improvements are in progress by **Chase Hawthorne (MATS Data Infra)** to optimize pipeline performance and reduce Kusto cluster load:

| Item | Description | Status |
|---|---|---|
| Replace `GetOneAuthActions` | Move from expensive multi-table UNIONs to a Kusto **update policy** that formats and consolidates data into a single table at ingestion time | In Progress |
| Macro-expand approach | Replace `GetOneAuthActions` unions with a macro-expand approach to reduce query complexity | In Progress |
| Remove JOIN from `MAR_GetActions` | Use a separate dedicated job to append `CONCURRENT_UI_ACCESS` and other transaction-only failures to the Hlls table | Planned |

---

## 12. Glossary (Full)

| Term | Technical Definition |
|---|---|
| **ADX / KWE** | Azure Data Explorer / Kusto Web Explorer — the platform hosting the dashboard and its databases |
| **ADF** | Azure Data Factory — pipeline orchestration service that computes and ingests metrics into MATS_Reporting |
| **CMC** | Microsoft 365 Copilot App (browser SPA). App ID: `14638111-3389-403d-b206-a6a71d9f8f16` |
| **MATS** | Microsoft Authentication Telemetry Service — the team and data infrastructure for auth telemetry |
| **MSAL.js** | Microsoft Authentication Library for JavaScript — OAuth 2.0 / OIDC client library for SPAs |
| **SPA** | Single Page Application — web app where CMC and other monitored apps run |
| **SSO** | Single Sign-On — authenticating silently using an existing browser session |
| **Profile / profileTelemetryId** | Anonymized per-user identifier used in MSALjs telemetry |
| **Profile-based metric** | Metric computed at the per-profile granularity (1 user = 1 profile per day) |
| **Request-based metric** | Metric computed per individual API call (not yet implemented on all pages) |
| **SSO Rate** | % of profiles with silent SSO success and no interactive calls |
| **IframeRenewalFailureRate** | % of iframe-based `acquireTokenSilent` attempts that fail |
| **SSO Reliability** | Profile-level reliability of all silent auth paths combined |
| **SSOImpactRate** | % of total SSO profiles impacted by a given error code on a given day |
| **SignIn Reliability** | % of interactive sign-in initiations that complete successfully (accounting for abandoned redirects) |
| **Reauth Rate** | % of silent flow profiles that hit InteractionRequired-class errors |
| **ssoSilent()** | MSAL.js API: attempts silent SSO using existing browser session cookie (no network call) |
| **acquireTokenSilent()** | MSAL.js API: cache check + iframe renewal fallback for silent token acquisition |
| **acquireTokenRedirect()** | MSAL.js API: full-page redirect to AAD login page |
| **acquireTokenPopup()** | MSAL.js API: popup window to AAD login page |
| **silentRefreshReason** | MSALjs event field indicating iframe renewal was used in `acquireTokenSilent` |
| **isRTIframeRenewal** | Materialized data field: `true` if `acquireTokenSilent` used the iframe renewal path |
| **monitor_window_timeout** | MSAL.js error: iframe monitor window timed out waiting for token response — #1 error for MSA |
| **interaction_required** | Error: Conditional Access or MFA step-up required; forces interactive prompt |
| **login_required** | Error: Session expired; user must re-authenticate interactively |
| **invalid_grant** | Error: Refresh token is invalid or expired (password change, session revocation, token rotation) |
| **abandoned_redirect** | Synthetic error code: user started redirect flow but never completed it (`PreRedirect > PostRedirect`) |
| **getMaterializedData()** | Kusto function in MSALjs Tenant that returns pre-materialized aggregated data |
| **allEventsView** | Kusto materialized view in MSALjs Tenant over the `allevents` table |
| **CopilotAggregatedTimeouts** | Kusto table in MSALjs Tenant with MSA timeout error aggregations |
| **MATS_Reporting** | Kusto database: `idsharedeus2/MATS_Reporting` — pre-aggregated metrics from ADF |
| **MSALjs Tenant** | Kusto database: `idsharedeus2/MSALJS Tenant` — raw MSALjs client event telemetry |
| **TableSetOrAppend** | Kusto ingestion command used by ADF pipelines to write metrics |
| **hll_merge / dcount_hll** | Kusto HyperLogLog functions for approximate distinct count over aggregated profile HLL sketches |
| **WoW** | Week-over-Week — current 7-day average minus previous 7-day average |
| **SSO** | Single Sign-On |
| **MSA** | Microsoft Account (personal accounts, e.g., Outlook.com) |
| **AAD / Entra ID** | Azure Active Directory — enterprise identity platform for work/school accounts |
| **IDC** | India Development Center — where the SISU/MATS Optics team is based |
| **SISU** | Service Insights & SLI Unit — team managing service health and metrics dashboards |
| **OCE** | On-Call Engineer — person responsible for monitoring dashboards and responding to incidents during their rotation |
| **v2 trigger** | ADF trigger naming convention used by Chase Hawthorne; use these — do NOT enable deprecated old triggers |

---

## 13. Revision History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | April 22, 2026 | Abhishek Bag | Initial wiki creation. Covers all 6 dashboard pages, metric definitions, ADF pipelines, MSAL.js background, and ops guide. |
| 2.0 | May 2026 | Abhishek Bag | Split into PM and Dev wikis. Addressed open comments: expanded app coverage description, removed current metric snapshots, removed Native SignIn and Comparison WIP sections. Full Kusto query library added. |

---

*For business overview, stakeholder summary, and plain-English metric descriptions, see the [PM & Business Guide](./MATS_SPA_Dashboard_Wiki_PM.md).*
