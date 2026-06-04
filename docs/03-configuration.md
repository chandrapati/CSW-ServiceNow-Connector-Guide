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

> [!IMPORTANT]
> **CSW keys ServiceNow records off the IP Address field — a table is only usable if it has one.**
> When you select a table, CSW lists its attributes and **you must choose the `ip_address` attribute as the key.** That key is what maps each ServiceNow record to an IP in CSW inventory, so a table with **no IP Address field cannot be integrated** (it won't yield usable labels). If a CMDB table you care about has no IP field, create a ServiceNow **View** that brings one in (e.g., via a JOIN) and select the view instead — see [`06-limitations-and-faq.md`](./06-limitations-and-faq.md).
>
> Cisco states this directly (CSW User Guide, *ServiceNow Instance Configuration*):
> - *"User has to chose the `ip_address` attribute from the table as the key. Subsequently, user can chose upto 10 unique attributes from the table."*
> - **Note:** *"ServiceNow Connector can only support integrating with tables having IP Address field."*

After the instance is saved, CSW **discovers all tables** from the instance (and Scripted REST APIs if the checkbox is enabled) and presents them for selection (Figures 20–24):

1. **Select a table** from the discovered list.
   - The table (or view) **must have an IP Address field.** If it doesn't, create a ServiceNow **View** that includes an IP field and select that instead (see FAQ).
   - In most CMDB deployments the source is the **`cmdb_ci` Configuration Item table or one of its child classes** (e.g., `cmdb_ci_server`, `cmdb_ci_linux_server`, `cmdb_ci_win_server`, `cmdb_ci_vm_instance`). These are standard ServiceNow CMDB tables (not Cisco-specific); the connector treats them like any other table — the same **IP-Address-field / `ip_address`-key rule applies.** Pick the CI class that actually carries IP data for your estate, or a view that joins it to one that does.
2. CSW **fetches the table's attributes**; choose the **`ip_address` attribute as the key.** This is how records map to CSW inventory IPs — without selecting `ip_address` as the key, CSW will not surface/integrate the table's records.
3. **Select the attributes** you want to import as labels — **up to 10 unique attributes** per table.
   - ⚠ See [`07-validation-notes.md`](./07-validation-notes.md): Cisco documents "up to 10" here and a per-instance maximum of 15. Plan around **10 per table** and verify in your tenant.
4. **Review and apply** the configuration (Figure 24).

### Practical attribute-selection tips

- Pick attributes that are **populated and stable** — empty/volatile fields make noisy labels.
- Favor attributes you'll actually **segment on**: environment, application, business unit, owner/support group, lifecycle status, data classification.
- Remember labels are **namespaced** under the orchestrator/connector; see [`04-using-the-labels.md`](./04-using-the-labels.md) for how they appear and how to reference them.

### ServiceNow field-naming cheat sheet (read this before you pick attributes)

The attribute list mixes base ServiceNow fields with this customer's custom fields. The **prefix tells you where a field came from — not whether the data is trustworthy or "discovered."**

| Prefix / pattern | Meaning | Example | Notes for label selection |
|---|---|---|---|
| *(no prefix)* | **Base / out-of-box** field, part of the standard table schema | `name`, `ip_address`, `serial_number`, `sys_class_name` | Consistent across instances; safe to assume it exists in any tenant. |
| **`u_…`** | **Custom (user-created)** field added to a global-scope table — **"u" = user** | `u_app_id`, `u_environment`, `u_business_unit` | **Instance-specific.** Customers often keep their best business context here, so you'll frequently select these — but a `u_` field in one tenant won't exist in another. |
| **`x_<scope>_…`** | Custom field from a **scoped application** | `x_acme_cmdb_app_tier` | Also custom/instance-specific; comes from an installed scoped app. |
| `sys_…` | **System** fields maintained by the platform | `sys_id`, `sys_updated_on` | Generally not useful as labels (`sys_id` is opaque). |

> **`u_` does NOT mean "discovered."** A custom field can be populated manually, by an import/transform map, by an integration, or by Discovery — you can't tell from the name. Whether a CI was populated by **ServiceNow Discovery / Service Mapping** is indicated by fields like **`discovery_source`**, **`first_discovered`**, and **`last_discovered`** (and the CI class), *not* by the `u_` prefix.

**What to do with this:**

- Treat `u_` / `x_…` fields as **the customer's authoritative business context candidates** — but **confirm with the CMDB owner** that the specific field is populated, accurate, and maintained before you build policy on it.
- If you want to know how fresh/authoritative a record is, ask the CMDB team about **`discovery_source` / `last_discovered`** rather than inferring it from a field name.
- For **reference fields** (often `u_owned_by`, `u_support_group`, `location`, `assigned_to`), remember they store a `sys_id` — add the reference-lookup URL param (Step 5) so they import as readable display values instead of opaque IDs.

## Step 4 — (Optional) Scripted REST APIs

If you enabled **Include Scripted APIs**, the workflow is the **same as for tables** — Cisco states it "would give you a similar workflow to any other table." That means the **identical key rule applies: the API's response must expose an IP Address field, you select the `ip_address` attribute as the key, then pick up to 10 attributes.** A Scripted REST API that doesn't return an IP address can't be keyed and won't produce usable labels — exactly like a table with no IP field.

Use a Scripted REST API when the data you want isn't in a single table/view (e.g., it requires server-side joins or business logic), but you still need it surfaced as an IP-keyed record set.

Additional requirements (enforced by CSW):

- The Scripted REST API **cannot use path parameters.**
- It **must support** `sysparm_limit`, `sysparm_fields`, and `sysparm_offset` as query parameters (CSW uses these to page and field-filter).
- The service account needs the **`web_service_admin`** role (tables only need `cmdb_read`).

## Step 5 — Sync interval & deletion behavior

In the **Sync Interval** configuration (Figure 25):

| Setting | Meaning | Default |
|---|---|---|
| **Data fetch frequency** | How often CSW queries ServiceNow for updates. | **60 minutes** |
| **Delete entry interval** | After how many consecutive missed syncs a record's labels are removed. | See note ⚠ |
| **Additional REST API url params** | Optional extra query params appended to table calls. | (none) |

- For **large tables**, increase *Data fetch frequency* so each sync finishes comfortably within the window.
- ⚠ **Delete entry interval:** Cisco's docs are inconsistent — the *Processing* section says labels are removed after **10 continuous sync intervals**, while the *Sync Interval Configuration* section says **48 consecutive sync intervals**. Either way, **a prolonged ServiceNow outage can cause labels to be cleaned up.** Set this value deliberately and confirm the effective behavior in your tenant. See [`07-validation-notes.md`](./07-validation-notes.md).
### Additional REST API URL params (optional)

If any additional parameters need to be passed when CSW calls the ServiceNow REST API for your tables, configure them under **Additional REST API url params**. This is **optional** and is appended to the table query.

The most common use is a **reference lookup**, so that ServiceNow **reference fields** (e.g., `assigned_to`, `owned_by`, `location`, `support_group` — stored as `sys_id` links) come back as **human-readable display values** instead of opaque `sys_id` URLs. Per the CSW User Guide, use:

```
sysparm_exclude_reference_link=true&sysparm_display_value=true
```

- `sysparm_display_value=true` → return the **display value** of each field rather than the raw value/sys_id.
- `sysparm_exclude_reference_link=true` → **omit the reference `link` object**, so you don't import the API URL.

> **Why it matters for labels:** without these params, a reference attribute often imports as a `sys_id` (or a link), which makes a useless label. Add this param when any attribute you selected is a **reference field** and you want a readable label (owner name, location name, group name) that you can actually scope and segment on.

## Step 6 — Verify

- Wait for at least one sync (or shorten the interval temporarily for testing).
- In CSW, open **inventory** for an IP you know exists in both CSW and ServiceNow and confirm the **ServiceNow attributes appear as labels.**
- Then use them in **scopes / inventory filters / policy** — see [`04-using-the-labels.md`](./04-using-the-labels.md).

If labels don't appear, jump to [`05-operations-and-troubleshooting.md`](./05-operations-and-troubleshooting.md).
