# Apigee on GCP — Beginner's 2-Hour Workshop
### EduRamp Learning Services | Hands-On Lab Guide

---

> **Audience:** Absolute beginners with a GCP account
> **Format:** Instructor-led with hands-on lab
> **Pre-req:** A GCP project with billing enabled, browser open to [console.cloud.google.com](https://console.cloud.google.com)

---

## Workshop Agenda

| Segment | Topic |
|---------|-------|
| Intro | What is Apigee? Why do we need it? |
| Module 1 | Apigee Architecture & Core Concepts |
| Module 2 | API Proxy Development (hands-on) |
| Module 3 | Security & Monitoring |
| Lab (25 min) | Migrating a Sample API to Apigee |
| Wrap-up | Q&A and Next Steps |

---

## Learning Objectives

By the end of this workshop, you will be able to:

- Explain Apigee's role as an API management platform on GCP
- Navigate the Apigee UI in the Google Cloud Console
- Create a basic API proxy from scratch
- Apply conditional logic and use flow variables
- Implement basic API key security
- Read the Apigee analytics dashboard
- Debug a failing API using the Trace tool

---

## MODULE 1 — Apigee Architecture & Core Concepts

### 1.1 What Is Apigee?

Apigee is GCP's **API Management platform**. Think of it as the front door to your backend services — it sits between your API consumers (apps, partners, developers) and your backend (Cloud Run, GKE, VMs, databases, etc.).

**Why use Apigee?**

- Security: authenticate and authorize every API call
- Traffic management: rate limiting, quotas, spike arrest
- Observability: analytics, logging, monitoring
- Developer experience: developer portals, documentation
- Transformation: change request/response formats on the fly

```
Client App
    │
    ▼
[ Apigee API Proxy ]  ◄── Policies (security, transform, analytics)
    │
    ▼
[ Your Backend Service ]  (Cloud Run / GKE / on-prem / etc.)
```

---

### 1.2 Core Concepts You Must Know

#### API Proxy

An **API proxy** is the core Apigee building block. It is a facade that:

1. Accepts incoming requests from clients
2. Applies policies (security, transformation, logging)
3. Forwards the request to a **target backend**
4. Returns the (optionally modified) response to the client

A proxy decouples your public API from your backend — change the backend without impacting the consumer.

#### Target Server

A **Target Server** is a named reference to a backend endpoint. Instead of hardcoding `https://my-backend.run.app` in every proxy, you register it as a Target Server named `my-backend-prod`. This makes environment promotion (dev → staging → prod) seamless.

```
Target Server "payments-api":
  Host: payments.internal.mycompany.com
  Port: 443
  SSL: Enabled
```

#### Virtual Host / Environment Group

- An **Environment** (e.g., `dev`, `prod`) is an isolated runtime context.
- An **Environment Group** maps a hostname to one or more environments.
- Example: `api.mycompany.com` → routes to the `prod` environment.

#### Apigee Runtime vs Management

| Component | Role |
|-----------|------|
| **Runtime (Data Plane)** | Processes actual API traffic; executes policies |
| **Management (Control Plane)** | Where you design, deploy, and configure proxies via the UI or API |

In **Apigee X** (the GCP-native version), the runtime runs inside your VPC or is Google-managed. The management plane is the GCP Console.

#### Policies

Policies are reusable units of logic you attach to a proxy. They require **zero code**. Examples:

| Policy | What it does |
|--------|-------------|
| `VerifyAPIKey` | Validates an API key in the request |
| `OAuthV2` | OAuth 2.0 token validation |
| `SpikeArrest` | Limits sudden traffic spikes |
| `Quota` | Enforces usage limits per key/app |
| `JSONToXML` | Transforms request/response format |
| `AssignMessage` | Modifies headers, body, path |
| `ServiceCallout` | Calls an external service mid-flow |
| `RaiseFault` | Returns a custom error response |

---

### 1.3 Runtime Components — The Request Lifecycle

Every API call through Apigee follows this path:

```
Client
  │
  │  HTTP Request
  ▼
[ ProxyEndpoint ]
  │
  ├── PreFlow (always runs)
  ├── Conditional Flows (match on path/verb)
  └── PostFlow (always runs)
  │
  │  (request forwarded to backend)
  ▼
[ TargetEndpoint ]
  │
  ├── PreFlow
  ├── Conditional Flows
  └── PostFlow
  │
  │  HTTP Response
  ▼
Client
```

**Two sides of every flow:**

- **Request side** — from client to backend
- **Response side** — from backend back to client

**Two endpoints:**

- **ProxyEndpoint** — where client connects; you define the URL path here
- **TargetEndpoint** — where Apigee calls your backend

---

## MODULE 2 — API Proxy Development

### 2.1 Opening Apigee in the GCP Console

**Step-by-step:**

1. Open [console.cloud.google.com](https://console.cloud.google.com)
2. In the top search bar, type **Apigee** → click **Apigee API Management**
3. If prompted, enable the Apigee API
4. You will land on the **Apigee Overview** page

> **Tip:** Bookmark `console.cloud.google.com/apigee` for quick access.

**Key navigation areas in the left sidebar:**

```
Apigee
 ├── Overview
 ├── API Proxies          ← Where you build and deploy proxies
 ├── API Products         ← Package proxies for developers
 ├── Apps                 ← Developer apps & API keys
 ├── Environments         ← dev, staging, prod
 ├── Environment Groups   ← Hostname routing
 ├── Target Servers       ← Named backend references
 ├── Analytics
 │    ├── API Metrics
 │    └── Reports
 └── Debug (Trace)        ← Live traffic debugging
```

---

### 2.2 Creating Your First API Proxy

We will create a proxy for a free public API: `https://jsonplaceholder.typicode.com/todos`

**Step 1 — Navigate to API Proxies**

1. Click **API Proxies** in the left menu
2. Click **+ CREATE** (top right)

**Step 2 — Choose Proxy Type**

Select **Reverse proxy (most common)** → click **Next**

**Step 3 — Configure the Proxy**

Fill in the form:

| Field | Value to enter |
|-------|---------------|
| Proxy Name | `my-first-proxy` |
| Base Path | `/todos` |
| Target (Existing API) | `https://jsonplaceholder.typicode.com/todos` |
| Description | `My first Apigee proxy — todos API` |

Click **Next**

**Step 4 — Security (skip for now)**

Select **Pass through (none)** → click **Next**

**Step 5 — Deploy**

- Check the box next to your `eval` or `dev` environment
- Click **Create and Deploy**

> Wait ~30 seconds for deployment to complete. You'll see a green checkmark.

**Step 6 — Test It!**

1. Click on your proxy name `my-first-proxy`
2. Go to the **Overview** tab, copy the **Proxy URL**
3. Open a new browser tab, paste the URL and append `/todos`
4. You should see a JSON list of todos from the backend!

---

### 2.3 Request & Response Flows

Now let's explore the proxy internals.

**Step 1 — Open the Proxy Editor**

1. Inside your proxy, click the **Develop** tab
2. You'll see a visual flow editor with:
   - **ProxyEndpoint** on the left
   - **TargetEndpoint** on the right
   - **Request** (top half) and **Response** (bottom half)

**Flow structure explained:**

```
ProxyEndpoint — Request
  ├── PreFlow         (always first)
  ├── /todos GET      (conditional flow — matches GET /todos)
  └── PostFlow        (always last)

TargetEndpoint — Request
  └── PreFlow

[Backend called here]

TargetEndpoint — Response
  └── PostFlow

ProxyEndpoint — Response
  └── PostFlow        (always last before client)
```

**Step 2 — Add a Simple Policy**

Let's add a response header so clients know the request passed through Apigee.

1. In the **Response PostFlow** of the ProxyEndpoint, click **+ Step**
2. Select **Assign Message** → name it `AM-AddProxyHeader` → click **Add**
3. In the XML editor that appears, replace the content with:

```xml
<AssignMessage name="AM-AddProxyHeader">
  <Add>
    <Headers>
      <Header name="X-Powered-By">Apigee</Header>
    </Headers>
  </Add>
  <AssignTo createNew="false" transport="http" type="response"/>
</AssignMessage>
```

4. Click **Save** (top right) → then **Deploy**

---

### 2.4 Conditional Logic & Flow Variables

**Flow Variables** are key-value pairs that exist during a transaction. Apigee auto-populates many:

| Variable | Contains |
|----------|---------|
| `request.verb` | HTTP method: GET, POST, etc. |
| `request.path` | URL path: `/todos/1` |
| `request.header.content-type` | A specific request header |
| `response.status.code` | The backend's HTTP status code |
| `client.ip` | Caller's IP address |

**Using a Condition:**

Add logic to only apply a policy when a condition is true. Example — only allow GET requests:

1. In the proxy editor, click **+ Flow** under ProxyEndpoint
2. Name it `Block-Non-GET`
3. Set the condition:

```
request.verb != "GET"
```

4. Attach a **RaiseFault** policy inside this flow:

```xml
<RaiseFault name="RF-MethodNotAllowed">
  <FaultResponse>
    <Set>
      <StatusCode>405</StatusCode>
      <ReasonPhrase>Method Not Allowed</ReasonPhrase>
      <Payload contentType="application/json">
        {"error": "Only GET is supported on this proxy"}
      </Payload>
    </Set>
  </FaultResponse>
  <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
</RaiseFault>
```

5. Save and deploy. Try a POST request — you'll get a 405!

---

## MODULE 3 — Security & Monitoring

### 3.1 API Security Best Practices

#### API Key Validation (most common starting point)

**How it works:**

1. You create an **API Product** (bundles one or more proxies)
2. A developer registers an **App** and gets an API key
3. The client passes the key in the request: `?apikey=abc123`
4. Apigee's `VerifyAPIKey` policy validates it automatically

**Adding API Key security to your proxy:**

1. In the proxy editor, go to **ProxyEndpoint → Request PreFlow**
2. Click **+ Step** → select **Verify API Key**
3. Name it `VAK-CheckKey` → click **Add**
4. The default XML is sufficient:

```xml
<VerifyAPIKey name="VAK-CheckKey">
  <APIKey ref="request.queryparam.apikey"/>
</VerifyAPIKey>
```

5. Save and deploy

> Now, any request without a valid `?apikey=` will receive a **401 Unauthorized**.

#### Other Security Layers to Know

| Mechanism | Use Case |
|-----------|---------|
| **OAuth 2.0** | Token-based auth for user-facing APIs |
| **Spike Arrest** | Prevent traffic spikes: `<Rate>30ps</Rate>` (30 per second) |
| **Quota** | Business-level limits: 1000 calls/day per API key |
| **IP Allowlist/Blocklist** | `<AccessControl>` policy filters by IP range |
| **TLS/HTTPS** | Always enforce HTTPS on virtual hosts (default in Apigee X) |
| **Threat Protection** | `<JSONThreatProtection>` blocks oversized/malformed payloads |

**Security layering (best practice):**

```
Request → [TLS] → [IP Filter] → [API Key / OAuth] → [Quota] → [Threat Protection] → Backend
```

Apply policies from the outside in — block unauthorized traffic as early as possible.

---

### 3.2 Analytics & Monitoring Dashboards

#### Accessing Analytics

1. In the left menu, click **Analytics → API Metrics**
2. Select your **environment** (e.g., `eval`)
3. Set a time range (last 1 hour)

**Key metrics on the dashboard:**

| Metric | What it tells you |
|--------|------------------|
| **Traffic** | Total API calls over time |
| **Error Rate** | % of 4xx/5xx responses |
| **Latency (p50/p99)** | Median and tail response times |
| **Top Proxies** | Which APIs receive the most traffic |
| **Top Developers** | Which API keys are most active |
| **Cache Hit Rate** | Efficiency of response caching |

> **Tip:** Spike in error rate + spike in latency usually means a backend issue, not an Apigee issue.

#### Custom Reports

1. Click **Analytics → Reports → + Create Report**
2. Choose dimensions (proxy name, developer app) and metrics (traffic, latency)
3. Schedule reports to be emailed to stakeholders daily

#### Alerting with Cloud Monitoring

Apigee X integrates with **Google Cloud Monitoring**:

1. Go to **Cloud Monitoring → Alerting → + Create Policy**
2. Select metric: `apigee.googleapis.com/proxy/request_count`
3. Set threshold: alert if error rate > 5% for 5 minutes
4. Add notification channel (email, PagerDuty, Slack)

---

### 3.3 Debugging with the Trace Tool

The **Trace tool** is your best friend when something goes wrong.

**How to use it:**

1. Go to **API Proxies** → click your proxy
2. Click the **Debug** tab
3. Select your environment and revision
4. Click **Start Trace Session**
5. Send a request to your proxy URL in a new tab
6. Watch the trace appear in real time!

**Reading a trace:**

```
Client Request Received
      │
      ▼
[PreFlow Request]
  ├── VAK-CheckKey  (200ms)
  ├── AM-AddProxyHeader  (1ms)
  │
      ▼
[Target Request Sent] → jsonplaceholder.typicode.com
      │
      ▼
[Target Response Received] 200 OK (340ms)
      │
      ▼
[PostFlow Response]
  └── AM-AddProxyHeader
      │
      ▼
Response Sent to Client (total: 545ms)
```

**Debugging tips:**

- Click any policy step to inspect variables **before and after** it ran
- Red step = policy threw a fault
- Check `error.message` and `error.cause` flow variables
- Use **Filter** to only capture requests matching a condition (e.g., a specific API key)

---

## HANDS-ON LAB (25 min) — Migrating a Sample API to Apigee

### Lab Overview

In this lab, you will migrate a real sample API (`https://petstore3.swagger.io/api/v3`) to Apigee by creating a proxy, securing it, and testing it end to end.

**What you'll build:**

```
Your Client
    │ GET /pets/v3/pet/findByStatus?status=available&apikey=<key>
    ▼
[ Apigee Proxy: petstore-proxy ]
    │ Verifies API Key
    │ Strips apikey param before forwarding
    │ Adds X-Request-ID header
    ▼
[ Backend: petstore3.swagger.io ]
    │
    ▼
JSON response → back to client
```

---

### Lab Step 1 — Create the Proxy (5 min)

1. **API Proxies** → **+ CREATE** → **Reverse Proxy**
2. Fill in:

| Field | Value |
|-------|-------|
| Proxy Name | `petstore-proxy` |
| Base Path | `/pets/v3` |
| Target URL | `https://petstore3.swagger.io/api/v3` |

3. Security: **Pass through (none)** for now
4. Deploy to your environment → **Create and Deploy**

**Quick test:**

```
GET https://<your-apigee-host>/pets/v3/pet/findByStatus?status=available
```

You should get a JSON array of pets.

---

### Lab Step 2 — Add API Key Security (5 min)

1. Open the proxy → **Develop** tab
2. **ProxyEndpoint → Request PreFlow** → **+ Step**
3. Add **Verify API Key** named `VAK-PetstoreKey`

```xml
<VerifyAPIKey name="VAK-PetstoreKey">
  <APIKey ref="request.queryparam.apikey"/>
</VerifyAPIKey>
```

4. Now add a second step to **remove the apikey param** before it reaches the backend (security hygiene!):

Add **Assign Message** named `AM-RemoveApiKey`:

```xml
<AssignMessage name="AM-RemoveApiKey">
  <Remove>
    <QueryParams>
      <QueryParam name="apikey"/>
    </QueryParams>
  </Remove>
  <AssignTo createNew="false" transport="http" type="request"/>
</AssignMessage>
```

5. Save and Deploy

---

### Lab Step 3 — Add a Request ID Header (5 min)

Let's stamp every request with a unique ID for traceability.

1. In **ProxyEndpoint → Request PreFlow**, add **Assign Message** named `AM-AddRequestId`

```xml
<AssignMessage name="AM-AddRequestId">
  <Add>
    <Headers>
      <Header name="X-Request-ID">{messageid}</Header>
    </Headers>
  </Add>
  <AssignTo createNew="false" transport="http" type="request"/>
</AssignMessage>
```

> `{messageid}` is a built-in Apigee flow variable — a unique UUID per request.

2. Save and Deploy

---

### Lab Step 4 — Create an API Product and Get a Key (5 min)

1. Go to **API Products** → **+ CREATE**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `petstore-product` |
| Display Name | `Petstore API` |
| Environment | your `eval` or `dev` env |
| Access | Public |

3. Under **API Proxies**, click **+ ADD A PROXY** → select `petstore-proxy` → **Add**
4. Click **Save**

**Create an App:**

1. Go to **Apps** → **+ CREATE**
2. Fill in:

| Field | Value |
|-------|-------|
| App Name | `my-test-app` |
| Developer | (select or create one) |

3. Under **Credentials**, click **+ ADD PRODUCT** → select `petstore-product` → **Add**
4. Click **Create**
5. After creation, click on the app → reveal the **Consumer Key** — this is your API key!

---

### Lab Step 5 — End-to-End Test (5 min)

Test in your browser or with curl:

```bash
# Replace with your actual values
APIGEE_HOST="https://34.xxx.xxx.xxx.nip.io"  # Your Apigee environment group hostname
API_KEY="your-consumer-key-here"

curl "${APIGEE_HOST}/pets/v3/pet/findByStatus?status=available&apikey=${API_KEY}"
```

**Expected results:**

| Test | Expected |
|------|---------|
| Valid API key | 200 OK with pet JSON |
| No API key | 401 Unauthorized |
| Wrong API key | 401 Unauthorized |
| POST request (if you added the block) | 405 Method Not Allowed |

**Verify with Trace:**

1. Start a Trace session on `petstore-proxy`
2. Send a request with a valid key
3. Confirm in the trace:
   - `VAK-PetstoreKey` — passed
   - `AM-RemoveApiKey` — `apikey` param removed
   - `AM-AddRequestId` — `X-Request-ID` header added
   - Backend response: 200

**Check Analytics:**

1. Go to **Analytics → API Metrics**
2. You should see your test traffic appear within ~2 minutes
3. Spot your proxy `petstore-proxy` in the **Top Proxies** chart

---

## Wrap-Up & Next Steps

### What We Covered Today

- Apigee's role as an API gateway on GCP
- Core concepts: proxy, target server, environment, virtual host, policies
- The request/response flow lifecycle
- Creating and deploying an API proxy from the GCP UI
- Conditional logic with flow variables
- API key security and policy layering
- Analytics dashboards and alerting
- Debugging with the Trace tool
- Full lab: migrating a real API to Apigee end-to-end

---

### Common Mistakes to Avoid

| Mistake | Fix |
|---------|-----|
| Forgetting to deploy after saving | Always click **Deploy** after **Save** |
| Hardcoding backend URLs in proxies | Use **Target Servers** for portability |
| Leaving `apikey` in the forwarded request | Always strip with `AssignMessage > Remove` |
| Not testing 4xx scenarios | Always test invalid key, missing key, wrong method |
| Ignoring latency in Trace | Backend slowness does not equal an Apigee problem |

---

### Next Steps & Learning Path

1. **Apigee Policies to explore next:**
   - `OAuthV2` — full OAuth 2.0 flows
   - `Quota` — call limits per developer
   - `ResponseCache` — cache backend responses in Apigee
   - `JavaScript / Python` — custom logic in policies

2. **Apigee X features to explore:**
   - Apigee Hub for API catalog
   - Integrated Developer Portal
   - Advanced API Security (WAAP)
   - CI/CD with Apigee Maven Plugin or Apigee CLI (`apigeectl`)

3. **Official Resources:**
   - Docs: [cloud.google.com/apigee/docs](https://cloud.google.com/apigee/docs)
   - Codelabs: [codelabs.developers.google.com](https://codelabs.developers.google.com) — search "Apigee"
   - Sample proxies: [github.com/apigee/api-platform-samples](https://github.com/apigee/api-platform-samples)
   - Certification: **Google Professional Cloud Developer** covers Apigee

---

## Quick Reference Card

```
CREATE PROXY:      API Proxies → + CREATE → Reverse Proxy
EDIT PROXY:        API Proxies → [name] → Develop tab
DEPLOY PROXY:      Develop tab → Save → Deploy button
TRACE/DEBUG:       API Proxies → [name] → Debug tab
VIEW ANALYTICS:    Analytics → API Metrics
TARGET SERVERS:    Admin → Environments → Target Servers
API PRODUCTS:      Publish → API Products
APPS & KEYS:       Publish → Apps
```

---

*EduRamp Learning Services — Apigee on GCP Beginner Workshop v1.0*
*For internal training use only.*
