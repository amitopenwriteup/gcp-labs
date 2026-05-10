# Google Cloud Bigtable — CBT CLI Lab

> **Goal:** Configure credentials, create a table with column families, write rows, and look up data using the `cbt` command-line tool.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Google Cloud project | e.g. `oval-turbine-495504-h2` |
| Bigtable instance | e.g. `rideshare-instance` |
| `cbt` CLI installed | Part of Google Cloud SDK |
| `gcloud` CLI installed | [Install guide](https://cloud.google.com/sdk/docs/install) |

---

## Step 1 — Authenticate & Set Credentials

### 1a. Login with your Google account

```bash
gcloud auth login
```

This opens a browser window. Sign in with the account that has access to your Bigtable instance.

### 1b. Set Application Default Credentials (ADC)

```bash
gcloud auth application-default login
```

> ADC is what `cbt` uses under the hood to authenticate API calls.

### 1c. Set your active project

```bash
gcloud config set project oval-turbine-495504-h2
```

### 1d. Verify credentials are working

```bash
gcloud auth list
```

Expected output:

```
ACTIVE  ACCOUNT
*       you@example.com
```

---

## Step 2 — Configure CBT

Configure `cbt` so you don't need to pass `-project` and `-instance` flags on every command.

### 2a. Create the `.cbtrc` config file

```bash
echo project = oval-turbine-495504-h2 > ~/.cbtrc
echo instance = rideshare-instance    >> ~/.cbtrc
```

### 2b. Verify the config

```bash
cat ~/.cbtrc
```

Expected output:

```
project = oval-turbine-495504-h2
instance = rideshare-instance
```

### 2c. Test the connection

```bash
cbt listinstances
```

Expected output:

```
rideshare-instance   PRODUCTION   ...
```

---

## Step 3 — Create Table & Column Families

### 3a. Create the table

```bash
cbt createtable users
```

### 3b. Verify the table exists

```bash
cbt ls
```

Expected output:

```
users
```

### 3c. Create a column family

Column families group related columns together. Here we create a `profile` family.

```bash
cbt createfamily users profile
```

### 3d. (Optional) Create additional column families

```bash
cbt createfamily users trips
cbt createfamily users payments
```

### 3e. Verify column families

```bash
cbt ls users
```

Expected output:

```
Family Name  GC Policy
-----------  ---------
payments     <default>
profile      <default>
trips        <default>
```

---

## Step 4 — Define Row ID Scheme

Bigtable has no schema — the **row key** is your primary index. Plan it carefully.

### Row Key Convention Used in This Lab

```
<entity_type>#<entity_id>
```

| Row Key | Meaning |
|---|---|
| `user#user-001` | User with ID 001 |
| `user#user-042` | User with ID 042 |
| `trip#trip-101` | Trip with ID 101 |

> The `#` separator allows efficient prefix scans across all users with `cbt read users prefix=user#`.

---

## Step 5 — Write Data (Set Entries)

### 5a. Write a single user row

```bash
cbt set users user#user-042 \
  profile:name="Priya Sharma" \
  profile:email="priya@example.com" \
  profile:rating="4.8"
```

Each argument is in the format `family:column=value`.

### 5b. Write another user

```bash
cbt set users user#user-001 \
  profile:name="Arjun Mehta" \
  profile:email="arjun@example.com" \
  profile:rating="4.6"
```

### 5c. Write to multiple column families in one command

```bash
cbt set users user#user-042 \
  profile:city="Jodhpur" \
  trips:total="38" \
  payments:method="UPI"
```

---

## Step 6 — Read / Lookup Data

### 6a. Read a single row by exact row key

```bash
cbt read users prefix=user#user-042
```

Expected output:

```
----------------------------------------
user#user-042
  payments:method                          @ 2026/...
    "UPI"
  profile:city                             @ 2026/...
    "Jodhpur"
  profile:email                            @ 2026/...
    "priya@example.com"
  profile:name                             @ 2026/...
    "Priya Sharma"
  profile:rating                           @ 2026/...
    "4.8"
  trips:total                              @ 2026/...
    "38"
```

### 6b. Lookup a specific row with `cbt lookup`

```bash
cbt lookup users user#user-042
```

> `lookup` fetches exactly one row. `read` with `prefix=` scans one or more rows.

### 6c. Read all rows with a prefix (scan)

```bash
cbt read users prefix=user#
```

This returns **all user rows** in sorted order.

### 6d. Read only specific columns

```bash
cbt read users prefix=user#user-042 columns=profile:name,profile:rating
```

### 6e. Read the entire table

```bash
cbt read users
```

---

## Step 7 — Verify & Inspect

### 7a. Count rows in the table

```bash
cbt count users
```

### 7b. Read with row limit

```bash
cbt read users count=5
```

### 7c. Read latest N versions of a cell

```bash
cbt read users prefix=user#user-042 cells-per-column=3
```

> Bigtable stores multiple timestamped versions per cell. Default reads return the latest.

---

## Step 8 — Cleanup (Optional)

### 8a. Delete a single row

```bash
cbt deleterow users user#user-001
```

### 8b. Delete the entire table

```bash
cbt deletetable users
```

---

## Quick Reference Cheat Sheet

```bash
# Config
echo project = PROJECT_ID   > ~/.cbtrc
echo instance = INSTANCE_ID >> ~/.cbtrc

# Table & families
cbt createtable TABLE
cbt createfamily TABLE FAMILY
cbt ls                        # list tables
cbt ls TABLE                  # list families

# Write
cbt set TABLE ROW_KEY FAMILY:COL="value"

# Read
cbt lookup TABLE ROW_KEY                   # exact row
cbt read TABLE prefix=ROW_PREFIX           # prefix scan
cbt read TABLE columns=FAMILY:COL          # specific columns
cbt read TABLE count=N                     # first N rows

# Delete
cbt deleterow TABLE ROW_KEY
cbt deletetable TABLE
```
