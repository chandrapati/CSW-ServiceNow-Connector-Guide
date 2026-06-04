# 03 — Configuration Walkthrough

This is the documented configuration flow from the CSW 4.0 User Guide, expanded with practical notes. Screens are referenced by the User Guide's figure names so you can cross-check against Cisco's screenshots.

> **Before you start:** complete [`02-prerequisites.md`](./02-prerequisites.md). You need a healthy **Edge** appliance, a ServiceNow service account with `cmdb_read` (and `web_service_admin` for Scripted REST APIs), no MFA on that account, and **443** open from the Edge VM to both ServiceNow and the CSW cluster.

## Step 1 — Enable the ServiceNow connector on the Edge appliance

1. In CSW, go to **Manage → Connectors**.
2. Select **ServiceNow** (under *Inventory Enrichment*).
3. Enable/deploy it on your **Secure Workload Edge** appliance for the target tenant.

The connector supports these configuration areas:

| Config area | Purpose |
|---|---|
| **ServiceNow Tables** | Define the ServiceNow instance (credentials + URL) and which tables to read. |
| **Scripted REST API** | Same idea as tables, but reads Scripted REST API endpoints. |
| **Sync Interval** | How often CSW queries ServiceNow (*Data fetch frequency*) and when stale records are deleted (*Delete entry interval*). |
| **Log** | Connector log configuration (see *Log Configuration* in the User Guide). |

## Step 2 — Add a ServiceNow instance

Provide (Figure 19, *ServiceNow Instance Configuration*):

- **ServiceNow username**
- **ServiceNow password**
- **ServiceNow Instance URL** (e.g., `https://yourcompany.service-now.com`)
- **Scripted APIs** — enable the *Include Scripted APIs* checkbox **only** if you intend to read Scripted REST APIs.
- **(Optional) Additional URL parameters** — per-table extra query params (see Step 5).

> You can configure **up to 20 ServiceNow instances on one connector** (see limits).

## Step 3 — Discover and select a table (or view)

After the instance is saved, CSW **discovers all tables** from the instance (and Scripted REST APIs if the checkbox is enabled) and presents them for selection (Figures 20–24):

1. **Select a table** from the discovered list.
   - The table (or view) **must have an IP Address field.** If it doesn't, create a ServiceNow **View** that includes an IP field and select that instead (see FAQ).
2. CSW **fetches the table's attributes**; choose the **`ip_address` attribute as the key.** This is how records map to CSW inventory IPs.
3. **Select the attributes** you want to import as labels — **up to 10 unique attributes** per table.
   - ⚠ See [`07-validation-notes.md`](./07-validation-notes.md): Cisco documents "up to 10" here and a per-instance maximum of 15. Plan around **10 per table** and verify in your tenant.
4. **Review and apply** the configuration (Figure 24).

### Practical attribute-selection tips

- Pick attributes that are **populated and stable** — empty/volatile fields make noisy labels.
- Favor attributes you'll actually **segment on**: environment, application, business unit, owner/support group, lifecycle status, data classification.
- Remember labels are **namespaced** under the orchestrator/connector; see [`04-using-the-labels.md`](./04-using-the-labels.md) for how they appear and how to reference them.

## Step 4 — (Optional) Scripted REST APIs

If you enabled **Include Scripted APIs**, the workflow mirrors tables. Requirements (enforced by CSW):

- The Scripted REST API **cannot use path parameters.**
- It **must support** `sysparm_limit`, `sysparm_fields`, and `sysparm_offset` as query parameters (CSW uses these to page and field-filter).
- The service account needs the **`web_service_admin`** role.

## Step 5 — Sync interval & deletion behavior

In the **Sync Interval** configuration (Figure 25):

| Setting | Meaning | Default |
|---|---|---|
| **Data fetch frequency** | How often CSW queries ServiceNow for updates. | **60 minutes** |
| **Delete entry interval** | After how many consecutive missed syncs a record's labels are removed. | See note ⚠ |
| **Additional REST API url params** | Optional extra query params appended to table calls. | (none) |

- For **large tables**, increase *Data fetch frequency* so each sync finishes comfortably within the window.
- ⚠ **Delete entry interval:** Cisco's docs are inconsistent — the *Processing* section says labels are removed after **10 continuous sync intervals**, while the *Sync Interval Configuration* section says **48 consecutive sync intervals**. Either way, **a prolonged ServiceNow outage can cause labels to be cleaned up.** Set this value deliberately and confirm the effective behavior in your tenant. See [`07-validation-notes.md`](./07-validation-notes.md).
- **Example additional URL param** (reference lookups with display values):

  ```
  sysparm_exclude_reference_link=true&sysparm_display_value=true
  ```

## Step 6 — Verify

- Wait for at least one sync (or shorten the interval temporarily for testing).
- In CSW, open **inventory** for an IP you know exists in both CSW and ServiceNow and confirm the **ServiceNow attributes appear as labels.**
- Then use them in **scopes / inventory filters / policy** — see [`04-using-the-labels.md`](./04-using-the-labels.md).

If labels don't appear, jump to [`05-operations-and-troubleshooting.md`](./05-operations-and-troubleshooting.md).
