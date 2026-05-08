# Apigee API Management — Workshop Lab Guide

**EduRamp Learning Services**

---

| Detail | Info |
|---|---|
| **Level** | Beginner to Intermediate |
| **Duration** | ~4 Hours |
| **Platform** | Google Cloud — Apigee X |
| **Format** | Concept + Hands-on Lab |

---

## Workshop Agenda

| Module | Topic | Duration |
|---|---|---|
| 1 | Apigee Architecture & Concepts | 45 min |
| 2 | API Proxy Development | 60 min |
| 3 | Security & Monitoring | 45 min |
| 4 | Hands-on Lab: Migrate Sample APIs to Apigee | 60 min |

---

## Prerequisites

- GCP account with an active project
- Apigee X provisioned (or Apigee Eval org)
- Basic understanding of REST APIs
- A browser — all steps are done via the Apigee UI

### Enable Required APIs

In GCP Console search bar, enable these one by one:

| API | Search Term |
|---|---|
| Apigee API | `Apigee` |
| Cloud DNS API | `Cloud DNS` |
| Compute Engine API | `Compute Engine` |

---

# Module 1: Apigee Architecture & Concepts

## 1.1 What is Apigee?

Apigee is Google Cloud's **API Management Platform**. It sits between your API consumers (apps, clients) and your backend services, acting as a **gateway** that handles security, traffic management, analytics, and transformation.

```
  API Consumer          Apigee Gateway            Backend Service
  (Mobile App)  ──────▶  API Proxy     ──────▶   (Your REST API /
  (Web App)              - Security               Database /
  (Partner)              - Rate Limit             Microservice)
                         - Analytics
                         - Transform
```

---

## 1.2 API Proxy Fundamentals

An **API Proxy** is the core building block of Apigee. It is a lightweight wrapper around your backend API that adds policies and logic **without changing your backend code**.

### Key Concepts

| Concept | Description |
|---|---|
| **API Proxy** | The facade that sits in front of your backend |
| **Proxy Endpoint** | The URL that clients call (Apigee-side) |
| **Target Endpoint** | The backend URL that Apigee calls |
| **Flow** | The processing pipeline (Request → Target → Response) |
| **Policy** | A reusable unit of logic (auth, transform, rate limit) |

### How a Request Flows

```
Client Request
      │
      ▼
 Proxy Endpoint  ──▶  Request Flow (Pre-processing)
                              │
                              ▼
                       Target Endpoint  ──▶  Backend API
                              │
                              ▼
                       Response Flow (Post-processing)
                              │
                              ▼
                       Client Response
```

---

## 1.3 Target Servers and Virtual Hosts

### Target Servers

A **Target Server** is a named reference to a backend URL. Instead of hardcoding `https://api.mybackend.com` in every proxy, you register it once as a Target Server and reference it by name.

**Benefits:**
- Change backend URL in one place
- Enable load balancing across multiple backends
- Toggle backends per environment (dev / staging / prod)

**Creating a Target Server in the UI:**

1. Go to **Apigee Console → Admin → Environments**
2. Select your environment (e.g., `eval`)
3. Click **Target Servers → + Add**
4. Fill in:
   - **Name:** `my-backend`
   - **Host:** `api.mybackend.com`
   - **Port:** `443`
   - **SSL:** Enabled
5. Click **Add**

### Virtual Hosts

A **Virtual Host** defines how Apigee listens for incoming client traffic — the hostname and port that clients use to reach your API proxy.

| Component | Example |
|---|---|
| Hostname | `api.mycompany.com` |
| Port | `443` |
| Protocol | HTTPS |
| Base Path | `/v1` |

> In **Apigee X**, virtual hosts are managed through **Apigee Environment Groups** linked to a load balancer hostname.

---

## 1.4 Apigee Runtime and Management Components

### Architecture Overview

```
┌─────────────────────────────────────────────────┐
│               Management Plane                  │
│  (Apigee Console, API, Config, Analytics UI)    │
└───────────────────┬─────────────────────────────┘
                    │ Deploy / Config Sync
┌───────────────────▼─────────────────────────────┐
│                Runtime Plane                    │
│  ┌────────────────────────────────────────────┐ │
│  │  Message Processor  │  Router / MP Cluster │ │
│  └────────────────────────────────────────────┘ │
│              (Handles live traffic)             │
└───────────────────┬─────────────────────────────┘
                    │ Calls
┌───────────────────▼─────────────────────────────┐
│              Backend Services                   │
│         (Your APIs / Microservices)             │
└─────────────────────────────────────────────────┘
```

| Component | Role |
|---|---|
| **Management Plane** | Where you build, configure, and deploy proxies |
| **Runtime Plane** | Where live API traffic is processed |
| **Message Processor** | Executes policy logic on each request/response |
| **Router** | Routes incoming traffic to the correct proxy |
| **Analytics** | Collects and reports traffic data |

---

# Module 2: API Proxy Development

## 2.1 Creating Your First API Proxy (UI)

1. Open **Apigee Console** → [apigee.google.com](https://apigee.google.com)
2. Select your organization
3. Click **Develop → API Proxies → + Create**
4. Select **Reverse Proxy (most common)**
5. Fill in:

| Field | Value |
|---|---|
| Proxy Name | `hello-proxy` |
| Base Path | `/hello` |
| Target URL | `https://httpbin.org/get` |

6. Click **Next → Next → Create and Deploy**
7. Once deployed, click the proxy URL to test it

---

## 2.2 Request and Response Flows

Every API proxy has a **4-stage flow pipeline**:

```
┌──────────────────────────────────────────────────────────┐
│                    PROXY ENDPOINT                        │
│                                                          │
│  ┌──────────────┐              ┌───────────────────────┐ │
│  │ Request Flow │              │    Response Flow      │ │
│  │              │              │                       │ │
│  │ PreFlow      │              │ PostFlow              │ │
│  │ Conditional  │              │ Conditional           │ │
│  │ Flow         │              │ Flow                  │ │
│  │ PostFlow     │              │ PreFlow               │ │
│  └──────┬───────┘              └──────────┬────────────┘ │
└─────────┼────────────────────────────────┼──────────────┘
          │                                │
┌─────────▼────────────────────────────────▼──────────────┐
│                   TARGET ENDPOINT                        │
│  ┌──────────────┐              ┌───────────────────────┐ │
│  │ Request Flow │──▶ Backend ─▶│    Response Flow      │ │
│  └──────────────┘              └───────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

### Flow Types

| Flow | When It Runs |
|---|---|
| **PreFlow** | Always runs first — good for auth checks |
| **Conditional Flow** | Runs only when a condition is true (e.g., path = `/users`) |
| **PostFlow** | Always runs last — good for logging |
| **PostClientFlow** | Runs after response is sent to client (logging only) |

### Adding a Policy to a Flow (UI)

1. Open your proxy → Click **Develop** tab
2. In the flow diagram, click the **+ Step** button on the Request flow
3. Select a policy (e.g., **Assign Message**)
4. Give it a name and click **Add**
5. Edit the XML config in the editor panel
6. Click **Save → Deploy**

---

## 2.3 Conditional Logic and Flow Variables

### Conditions

Conditions control whether a flow or policy executes. They use **flow variables** — key-value pairs automatically populated by Apigee.

**Condition syntax:**

```xml
<Condition>request.verb = "GET" AND proxy.pathsuffix MatchesPath "/users/*"</Condition>
```

### Common Flow Variables

| Variable | Value Example | Description |
|---|---|---|
| `request.verb` | `GET`, `POST` | HTTP method |
| `request.path` | `/v1/users/123` | Full request path |
| `proxy.pathsuffix` | `/users/123` | Path after base path |
| `request.header.content-type` | `application/json` | Request header |
| `response.status.code` | `200`, `404` | Backend response code |
| `system.time` | `1714000000` | Current timestamp |

### Example: Conditional Flow

```xml
<!-- In ProxyEndpoint flows — only runs for GET /products -->
<Flow name="GetProducts">
  <Condition>
    request.verb = "GET" AND proxy.pathsuffix MatchesPath "/products"
  </Condition>
  <Request>
    <Step><Name>Verify-API-Key</Name></Step>
  </Request>
  <Response>
    <Step><Name>Add-CORS-Headers</Name></Step>
  </Response>
</Flow>
```

### Setting a Custom Flow Variable (Assign Message Policy)

```xml
<AssignMessage name="Set-Custom-Variable">
  <AssignVariable>
    <Name>my.custom.variable</Name>
    <Value>hello-world</Value>
  </AssignVariable>
  <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
</AssignMessage>
```

---

# Module 3: Security & Monitoring

## 3.1 API Security Best Practices

### Common Security Policies in Apigee

| Policy | Purpose | When to Use |
|---|---|---|
| **Verify API Key** | Validates client API keys | Public APIs with registered apps |
| **OAuth 2.0** | Token-based authorization | Enterprise / partner APIs |
| **JWT Validation** | Validates JSON Web Tokens | Microservices, SSO |
| **IP Allowlist** | Restricts by IP address | Internal APIs |
| **Quota** | Limits calls per time window | Prevent abuse |
| **Spike Arrest** | Limits burst traffic | Protect backend from spikes |
| **Threat Protection** | Blocks malicious payloads | Public-facing APIs |

### Adding API Key Verification (UI)

1. Open your proxy → **Develop** tab
2. Click **+ Step** on **PreFlow → Request**
3. Search for and select **Verify API Key**
4. Set the key location:

```xml
<VerifyAPIKey name="Verify-API-Key">
  <APIKey ref="request.queryparam.apikey"/>
</VerifyAPIKey>
```

5. Click **Save → Deploy**

Now all requests to this proxy must include `?apikey=YOUR_KEY`.

### Spike Arrest Policy (Rate Limiting)

```xml
<SpikeArrest name="Spike-Arrest">
  <Rate>10ps</Rate>  <!-- 10 requests per second max -->
</SpikeArrest>
```

| Rate Format | Meaning |
|---|---|
| `10ps` | 10 per second |
| `100pm` | 100 per minute |

---

## 3.2 Analytics and Monitoring Dashboards

### Accessing Analytics in the UI

1. Go to **Apigee Console → Analyze → API Metrics**
2. Select your **environment** and **proxy**
3. Set the time range (last 1 hour / 24 hours / 7 days)

### Key Dashboards

| Dashboard | What It Shows |
|---|---|
| **API Proxy Performance** | Traffic volume, latency, error rates per proxy |
| **Cache Performance** | Cache hit/miss ratio |
| **Error Code Analysis** | 4xx and 5xx breakdown |
| **Target Performance** | Backend response time and error rate |
| **Developer Engagement** | Which apps are calling which APIs |

### Metrics to Monitor

| Metric | Healthy Range | Alert If |
|---|---|---|
| Total Traffic | — | Sudden drop (possible outage) |
| Error Rate | < 1% | > 5% |
| p99 Latency | < 500ms | > 2000ms |
| Policy Error Rate | < 0.5% | > 2% |
| Target Availability | > 99.9% | < 99% |

---

## 3.3 Debugging and Troubleshooting

### Using the Apigee Debug Tool (Trace)

The **Trace** tool lets you watch a live request flow through your proxy step by step.

**How to use:**

1. Open your proxy → Click **Debug** tab
2. Select your **environment** and **revision**
3. Click **Start Trace Session**
4. Send a request to your proxy URL
5. Click the request in the trace panel
6. Step through each policy to inspect:
   - Flow variables at each stage
   - Policy inputs and outputs
   - Headers and body transformations
   - Errors and their root cause

### Reading a Trace

```
Request Received
      │
      ▼
  ✅ Verify-API-Key      ← Green = passed
      │
      ▼
  ✅ Spike-Arrest        ← Green = passed
      │
      ▼
  ➡  Send to Target      ← Arrow = forwarded to backend
      │
      ▼
  ✅ Assign-Response     ← Green = passed
      │
      ▼
Response Sent to Client
```

> A **red step** in the trace means a policy raised a fault — click it to see the error code and message.

### Common Errors and Fixes

| Error Code | Meaning | Fix |
|---|---|---|
| `401 - Invalid API Key` | Key missing or wrong | Check `?apikey=` param |
| `429 - Spike Arrest Violation` | Too many requests | Reduce request rate |
| `500 - Script error` | JS policy syntax error | Check JavaScript policy code |
| `503 - Backend unavailable` | Target server down | Check Target Server config |
| `policies.ratelimit.QuotaViolation` | Quota exceeded | Wait or increase quota limit |

---

# Module 4: Hands-on Lab — Migrating Sample APIs to Apigee

## Lab Overview

In this lab you will take a simple public REST API and migrate it behind an Apigee proxy with security, rate limiting, and response transformation.

**Backend API used:** `https://jsonplaceholder.typicode.com` (free public mock API)

**Time:** ~60 minutes

---

## Lab Step 1: Create the API Proxy

1. Go to **Apigee Console → Develop → API Proxies**
2. Click **+ Create**
3. Select **Reverse Proxy**
4. Enter:

| Field | Value |
|---|---|
| Proxy Name | `users-api` |
| Base Path | `/v1/users` |
| Target URL | `https://jsonplaceholder.typicode.com/users` |
| Description | Migrated users API |

5. Click **Next → Next → Create and Deploy**
6. Note the proxy URL shown — it looks like:
   `https://YOUR_ENV_GROUP_HOSTNAME/v1/users`

---

## Lab Step 2: Test the Proxy

Open a browser or use a REST client and call:

```
GET https://YOUR_ENV_GROUP_HOSTNAME/v1/users
```

You should see a JSON array of 10 users returned from the backend through your proxy.

---

## Lab Step 3: Add API Key Security

1. Open the `users-api` proxy → **Develop** tab
2. On the left panel, click **PreFlow** under **Proxy Endpoints**
3. Click **+ Step** on the **Request** side
4. Select **Verify API Key** → name it `verify-api-key` → click **Add**
5. Update the policy XML:

```xml
<VerifyAPIKey name="verify-api-key">
  <APIKey ref="request.header.x-api-key"/>
</VerifyAPIKey>
```

6. Click **Save → Deploy**

**Test it:**

```bash
# Without key — should return 401
curl https://YOUR_ENV_GROUP_HOSTNAME/v1/users

# With key — should return 200
curl -H "x-api-key: YOUR_API_KEY" \
     https://YOUR_ENV_GROUP_HOSTNAME/v1/users
```

---

## Lab Step 4: Add Rate Limiting

1. Click **+ Step** on **PreFlow → Request** (after the API Key step)
2. Select **Spike Arrest** → name it `spike-arrest` → click **Add**
3. Set the config:

```xml
<SpikeArrest name="spike-arrest">
  <Rate>5ps</Rate>
</SpikeArrest>
```

4. Click **Save → Deploy**

> Now if a client sends more than 5 requests per second, Apigee returns `429 Too Many Requests` automatically.

---

## Lab Step 5: Transform the Response

Add a custom response header so clients can identify the gateway.

1. Click **+ Step** on **PostFlow → Response**
2. Select **Assign Message** → name it `add-gateway-header` → click **Add**
3. Configure:

```xml
<AssignMessage name="add-gateway-header">
  <Set>
    <Headers>
      <Header name="x-powered-by">Apigee</Header>
      <Header name="x-request-id">{messageid}</Header>
    </Headers>
  </Set>
  <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
  <AssignTo createNew="false" type="response"/>
</AssignMessage>
```

4. Click **Save → Deploy**

**Test:** Check the response headers — you should see `x-powered-by: Apigee`.

---

## Lab Step 6: Add a Conditional Flow for Single User

1. In the **Develop** tab, click **+ Add Flow** under **Proxy Endpoints**
2. Name the flow `GetSingleUser`
3. Set the condition:

```xml
<Condition>
  request.verb = "GET" AND proxy.pathsuffix MatchesPath "/*"
</Condition>
```

4. Update the Target URL to pass the user ID:

```xml
<!-- In Target Endpoint, update the path -->
<HTTPTargetConnection>
  <URL>https://jsonplaceholder.typicode.com/users</URL>
</HTTPTargetConnection>
```

5. Add a **JS Policy** to extract the user ID from the path and append it to the target:

```javascript
// extract-user-id.js
var pathSuffix = context.getVariable("proxy.pathsuffix");
var userId = pathSuffix.replace("/", "");
context.setVariable("target.url",
  "https://jsonplaceholder.typicode.com/users/" + userId);
```

6. Click **Save → Deploy**

**Test:**

```bash
# Get single user by ID
curl -H "x-api-key: YOUR_KEY" \
     https://YOUR_ENV_GROUP_HOSTNAME/v1/users/1
```

---

## Lab Step 7: Debug with Trace

1. Open the proxy → Click **Debug** tab
2. Click **Start Trace Session**
3. Send a request to your proxy
4. In the trace panel, click the request
5. Walk through each step and verify:
   - `verify-api-key` shows the key was validated
   - `spike-arrest` shows the request was allowed
   - `add-gateway-header` shows headers were added
6. Try sending a request **without the API key** and observe the `401` fault in the trace

---

## Lab Step 8: View Analytics

1. Go to **Analyze → API Metrics**
2. Select `users-api` from the proxy dropdown
3. Set time range to **Last 1 Hour**
4. Observe:
   - Total traffic from your test calls
   - Any 4xx errors from the unauthenticated test
   - Average latency

---

## Lab Complete ✅

You have successfully:

- Created an API proxy in front of a real backend
- Added API Key security
- Applied rate limiting with Spike Arrest
- Transformed the response with custom headers
- Added a conditional flow for dynamic routing
- Debugged with the Trace tool
- Reviewed live analytics

---

## Quick Reference

```
┌──────────────────────────────────────────────────┐
│             APIGEE POLICY CHEATSHEET             │
├────────────────────┬─────────────────────────────┤
│ Verify API Key     │ Auth via key in header/param │
│ OAuth 2.0          │ Token-based auth             │
│ Spike Arrest       │ Burst rate limiting          │
│ Quota              │ Calls per day/month          │
│ Assign Message     │ Set headers, body, variables │
│ JavaScript         │ Custom logic                 │
│ Extract Variables  │ Parse JSON/XML/path          │
│ Service Callout    │ Call another API mid-flow    │
│ Response Cache     │ Cache backend responses      │
│ JSON Threat Prot.  │ Block malicious payloads     │
└────────────────────┴─────────────────────────────┘
```

---

*EduRamp Learning Services — Apigee API Management Workshop*
*Google Cloud | Apigee X | Beginner–Intermediate*
