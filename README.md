# Zapier Workflow — Lead → Clay to HubSpot Automation

## Overview

This Zapier automation reliably processes inbound leads enriched by Clay and syncs them into HubSpot. It handles domain resolution, duplicate detection, company creation, and contact upserts — all within a single automated pipeline.

---

## Workflow Architecture

```
[Clay Webhook] → [Clean Last Name] → [Resolve Domain] → [Get Owner]
     → [Find Company by Domain]
          ├── FOUND → [Upsert Contact] (Path A)
          └── NOT FOUND → [Find/Create Company by Name] → [Upsert Contact] (Path B)
```

---

## Step-by-Step Breakdown

### Step 1 — Trigger: Incoming Webhook from Clay

**Type:** Webhook Trigger  
**Purpose:** Receives enriched lead data from Clay in real-time.

**Incoming Fields:**

| Field | Description |
|---|---|
| `Firstname` | Contact first name |
| `Lastname` | Contact last name (raw) |
| `Company` | Company name |
| `Email` | Contact email address |
| `Phone` | Phone number |
| `Mobilephone` | Mobile phone number |
| `Website` | Company website / domain |
| `City` | City |
| `State` | State/Region |
| `Zip` | Postal code |
| `Country` | Country/Region |
| `Street` | Street address |
| `Employeecount` | Employee range |
| `Primary Industry` | Industry classification |
| `Title` | Job title |
| `Sales Profile` | LinkedIn URL |
| `Category Notes` | Campaign lead source notes |
| `Owner` | Source/owner label |
| `Division` | Division |

---

### Step 2 — Code: Clean Last Name

**Type:** Code by Zapier  
**Purpose:** Strips extra spaces, special characters, and course/suffix names — returning only the clean last name.

```javascript
let name = inputData.rawName || "";
// Splits at the first comma or hyphen and keeps the first part
let cleanName = name.split(/[,\-]/)[0];
// Trims trailing whitespace
output = [{ last_name: cleanName.trim() }];
```

---

### Step 3 — Code: Resolve Redirect Domain

**Type:** Code by Zapier  
**Purpose:** Follows HTTP 301 redirects to find the actual current domain, even if the lead provided an old or shortened URL.

```javascript
const url = inputData.url.includes('http') ? inputData.url : `https://${inputData.url}`;
try {
  const response = await fetch(url, { redirect: 'follow', method: 'HEAD' });
  const finalUrl = new URL(response.url).hostname.replace('www.', '');
  return { cleanDomain: finalUrl };
} catch (error) {
  return {
    cleanDomain: inputData.url
      .replace('https://', '')
      .replace('http://', '')
      .replace('www.', '')
      .split('/')[0]
  };
}
```

**Example:** `appliedbiocode.com` → resolves to `apbiocode.com`

---

### Step 4 — HubSpot: Find Contact Owner by Email

**Type:** HubSpot — Find Owner  
**Purpose:** Looks up the designated contact owner (Ryan) by email address and assigns all new leads from the **NTJ Auto Funnel Jan 2026** flow to this owner upon creation in HubSpot.

---

### Step 5 — HubSpot: Find Company by Domain

**Type:** HubSpot — Find Company  
**Purpose:** Checks whether a company with the resolved domain (from Step 3) already exists in HubSpot.  
**Output:** Returns `Zap search was found status` → `true` or `false`

---

### Step 6 — Paths: Does Company Exist?

**Type:** Paths / Conditional Split  
**Decision:** Branch based on the `Zap search was found status` from Step 5.

| Condition | Path |
|---|---|
| `true` — Company domain found in HubSpot | → **Path A** (Step 7) |
| `false` — Company domain not found | → **Path B** (Step 9) |

---

### Step 7–8 — Path A: Company Exists → Upsert Contact

**Type:** HubSpot — Create or Update Contact  
**Matching Rule:** Email address (unique key)  
**Purpose:** If the contact already exists, update it. If not, create it. Associates the contact with the existing company found in Step 5.

**Field Mapping:**

| Source | HubSpot Contact Property |
|---|---|
| Email (Step 1) | Contact Email |
| Street (Step 1) | Street Address |
| Record ID (Step 5) | Primary Associated Company ID |
| Category Notes (Step 1) | Campaign Lead Source |
| City (Step 1) | City |
| Country (Step 1) | Country/Region |
| Firstname (Step 1) | First Name |
| Sales Profile (Step 1) | LinkedIn URL |
| Owner ID (Step 4) | Contact Owner |
| Title (Step 1) | Job Title |
| Lastname — cleaned (Step 2) | Last Name |
| Mobilephone (Step 1) | Mobile Phone Number |
| Phone (Step 1) | Phone Number |
| Owner (Step 1) | Source |
| State (Step 1) | State/Region |
| Zip (Step 1) | Postal Code |

---

### Step 9–10 — Path B: Company Does Not Exist → Find or Create Company

**Type:** HubSpot — Find Company (with create if not found)  
**Purpose:** Searches for the company by name. If it doesn't exist, creates a new company record using the values from the webhook.

**Field Mapping:**

| Source | HubSpot Company Property |
|---|---|
| Country (Step 1) | Country |
| Website (Step 1) | Company Domain Name |
| Employeecount (Step 1) | Employee Range |
| Company (Step 1) | Account Name (Salesforce Info) |

---

### Step 11 — Path B: Upsert Contact

**Type:** HubSpot — Create or Update Contact  
**Matching Rule:** Email address (unique key)  
**Purpose:** Same upsert logic as Step 8 — creates or updates the contact and associates it with the newly created or found company from Step 10.

> Uses the same field mapping table as **Step 8** above.

---

## Key Design Decisions

- **Domain resolution (Step 3)** ensures that redirected or legacy domains are normalized before the HubSpot lookup, reducing duplicate company records.
- **Last name cleaning (Step 2)** prevents dirty data (e.g., `"Smith, MBA"` or `"Johnson - Course"`) from being written to HubSpot.
- **Upsert on email** means existing contacts are updated rather than duplicated, keeping CRM data clean.
- **Path A / Path B split** ensures every contact is correctly associated with a company — whether it already exists or needs to be created.
- **Owner assignment (Step 4)** centralizes lead ownership for the NTJ Auto Funnel Jan 2026 campaign under a single rep (Ryan).

---

## Dependencies & Requirements

- **Zapier** account with access to: Code by Zapier, Paths, HubSpot integration
- **HubSpot** account with permissions to create/update Contacts and Companies
- **Clay** configured to send webhook POST requests with the fields listed in Step 1
- The contact owner (Ryan) must already exist in HubSpot and be locatable by email

---

## Trigger Setup (Clay Side)

Configure Clay to send a `POST` request to your Zapier Webhook URL after table enrichment. Ensure the payload keys match exactly the field names listed in Step 1 (case-sensitive).

---

## Notes

- If a website URL fails to resolve during domain lookup (Step 3), the code falls back to basic string cleaning rather than failing the Zap.
- The `Division` field is received in the webhook but is not currently mapped to a HubSpot property. Map it as needed.
- `Primary Industry` is received but not mapped in the current contact upsert steps — add it if a corresponding HubSpot property exists.

