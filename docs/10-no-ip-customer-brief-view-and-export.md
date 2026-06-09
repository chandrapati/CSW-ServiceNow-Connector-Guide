# 10 — Customer Brief: "No IP Address Key" — Build a View and Export Attributes to CSW

> **Who this is for.** A customer or platform/CMDB team whose Cisco Secure Workload
> (CSW) ServiceNow connector **failed or imported nothing** because the chosen
> CMDB table **does not have an IP Address field to use as the key**. This page is
> a self-contained, hand-to-the-customer brief. For the full platform-level
> walkthrough see [`08-no-ip-create-a-view.md`](./08-no-ip-create-a-view.md); for
> the server-side-logic path see [`09-scripted-rest-api-example.md`](./09-scripted-rest-api-example.md).

---

## 1. Why the integration failed (root cause)

The CSW ServiceNow connector is an **Inventory Enrichment** connector: it does not
create workloads — it **annotates IP addresses CSW already knows about** with
attributes pulled from your ServiceNow CMDB, turning them into **labels** you can
use in scopes and segmentation policy.

Because CSW matches CMDB rows to inventory **by IP address**, Cisco requires that:

> **Each source table (or view) must contain an IP Address field, and during
> connector setup you must select the `ip_address` attribute as the _key_.**
> *"ServiceNow Connector can only support integrating with tables having IP Address field."*
> — CSW 4.0 User Guide (verified; see [`00-official-references.md`](./00-official-references.md))

If you pointed the connector at a table that has no IP field (a Business
Application table, a custom `u_*` app inventory, a CI class that doesn't carry an
IP, etc.), there is **no key to map on** — so the integration cannot be configured
correctly and no labels land on inventory. **That is the failure you hit.**

---

## 2. The fix in one sentence — and a critical clarification

**Build a ServiceNow _Database View_ that JOINs your attribute-rich table (no IP)
to an IP-bearing table, then point the connector at the view and select the
view's IP column as the key.**

> ⚠️ **"List view" vs "Database View" — this trips everyone up.**
> In ServiceNow, a **List view** and a **Form view** are just *UI layout
> configurations* (which columns/fields show on screen). **The connector cannot
> consume a UI List view** — it reads data through the **Table API**. What you
> need is a **Database View** (`sys_db_view`): a real, queryable, table-like JOIN
> object that the Table API can read by name and that **exposes an IP column
> without changing your CMDB schema**. So your instinct ("use a view, not a form")
> is right — the precise construct is a **Database View**, not a UI list layout.

A Database View is **read-only**, which is exactly what the read-only connector
(`cmdb_read`) needs.

---

## 3. Yes — you can export IP + hostname + appid + owner + tags (and use them in policy)

This is the most common question, so here it is explicitly. **Every attribute
below can be exported as a CSW label** as long as it is a stored (non-calculated)
field reachable through the view's joined tables. CSW lets you select roughly
**10 attributes per table/view** as labels (Cisco's docs say 10 in one place and
15 in another — confirm in your tenant; see [`07-validation-notes.md`](./07-validation-notes.md)).

| You want in CSW | Example ServiceNow source field | View column (Pattern A) | Role in CSW |
|---|---|---|---|
| **IP address** | `ip_address` on `cmdb_ci_network_adapter` | `na_ip_address` | **The key** (required) — maps the row to inventory |
| **Hostname / FQDN** | `name` / `fqdn` on the CI | `ci_name`, `ci_fqdn` | Label — readable identity in inventory & flows |
| **App ID** | `number`, `u_app_id`, or `u_application_id` on the app/CI table | `ci_number` / `ci_u_app_id` | Label — group/segment by application |
| **Owner** | `owned_by` / `managed_by` (reference) | `ci_owned_by` | Label — ownership-based scopes & reviews |
| **Environment** | `u_environment` / `environment` | `ci_u_environment` | Label — Prod vs Non-Prod separation |
| **Support group** | `support_group` (reference) | `ci_support_group` | Label — operational ownership |
| **Business service / unit** | `business_service` / `business_unit` | `ci_business_service` | Label — scope-tree structure |
| **Location / cost center** | `location`, `u_cost_center` | `ci_location`, `ci_u_cost_center` | Label — geo / chargeback scoping |

Once imported, these labels appear on the **Inventory** view for each IP and in
the **filter/scope/column** pickers. You then write policy that reads like the
business — for example:

> *Allow `application = Payments` · `environment = Production` workloads to reach
> `application = Payments-DB` · `environment = Production` on the documented ports;
> deny lateral movement from `environment = NonProd`.*

See [`04-using-the-labels.md`](./04-using-the-labels.md) for how labels feed
scopes, ADM, and the Monitor → Simulate → Enforce lifecycle.

> **Reference fields must import as text, not `sys_id`.** Owner, support group,
> location, and business service are ServiceNow **reference** fields. Add this to
> the connector's **Additional REST API URL params** so they import as readable
> values: `sysparm_exclude_reference_link=true&sysparm_display_value=true`.

---

## 4. Step-by-step (condensed)

> Full detail, including the two-join pattern for the IP Address CI class, is in
> [`08-no-ip-create-a-view.md`](./08-no-ip-create-a-view.md).

**A. Decide where the IP lives (the join).**

| Pattern | IP source table | Joins | Use when |
|---|---|---|---|
| **A (simplest)** | `cmdb_ci_network_adapter` (has `cmdb_ci` ref **and** `ip_address`) | 1 | NIC table is populated by Discovery |
| **B (IP Address class)** | `cmdb_ci_ip_address` → `nic` → `cmdb_ci_network_adapter` → CI | 2 | Org uses the IP Address class (multiple IPs/NICs per CI) |

**B. Create the Database View.** `System Definition → Database Views → New`.
Name it like a table (lowercase, no spaces), e.g. `u_csw_cmdb_ip`. Submit.

**C. Add the View Tables (define the JOIN).** In the **View Tables** related list,
add one row per table; the **Variable prefix** becomes the prefix on every output
column (`prefix_field`); **Order** sets join order (lowest = base table).

*Pattern A example:*

| Order | Table | Prefix | Where clause |
|---|---|---|---|
| 100 (base) | your no-IP table, e.g. `cmdb_ci_appl` | `ci` | *(empty)* |
| 200 (IP) | `cmdb_ci_network_adapter` | `na` | `na_cmdb_ci=ci_sys_id` |

→ IP key column: **`na_ip_address`**. Attribute columns: `ci_*`.
Keep it an **inner join** (Left join unchecked) so only CIs that actually have an
IP appear.

**D. Pin the fields you want (View Fields).** ⚠️ **All-or-nothing per table:** add
**no** View Fields and the table returns all columns; add **even one** and it
returns **only** the ones you list. List every field you may ever want as a label
now — the IP column plus `ci_name`, `ci_fqdn`, your appid field, `ci_owned_by`,
`ci_u_environment`, `ci_support_group`, etc.

**E. Test the export in ServiceNow (same surface the connector uses):**

```bash
curl -u "csw_integration_user:PASSWORD" \
  "https://YOURINSTANCE.service-now.com/api/now/table/u_csw_cmdb_ip?sysparm_limit=5&sysparm_display_value=true&sysparm_exclude_reference_link=true"
```

You should get JSON rows containing a valid IP in `na_ip_address` plus your `ci_*`
attributes as readable text.

**F. Grant read access.** The CSW service account needs **`cmdb_read`** and read
access to every joined base table. If your instance restricts view access, add a
read ACL on the view name.

**G. Point the connector at the view.** In `Manage → Connectors → ServiceNow`
(detail in [`03-configuration.md`](./03-configuration.md)): select your **view
name** instead of the table → choose the **prefixed IP column** (`na_ip_address`)
as the **`ip_address` key** → select the `ci_*` attributes you want as labels →
add the reference-display URL params from §3 → set a sensible **Data fetch
frequency** (joined views are heavier; raise it for large estates) → apply.

**H. Verify.** After the first sync, open **Inventory** for a known IP and confirm
the labels (hostname, appid, owner, tags) appear. Read the **actual label keys**
CSW created and use those exact keys in filters/scopes/policy.

---

## 5. When a Database View isn't enough → Scripted REST API

A Database View (§4) only returns **stored, directly-joinable columns**. If the
attributes you want as labels need server-side logic, use a **Scripted REST API**
instead — same IP-key rule applies. Reach for it when you need any of:

- **Calculated/scripted fields** (`type: calculated` fields) — Database Views
  silently drop these.
- **Multi-hop reference resolution** — e.g. walking `owned_by → sys_user → email`
  to label workloads with the owner's email, which a view's WHERE clause can't do.
- **Derived/business-logic labels** — risk tier, compliance scope, asset
  criticality, or any value computed from several CMDB fields.
- **Joins across more than two tables** where the reference chain makes a view
  WHERE clause impractical.

**How CSW consumes it.** The connector reads JSON in the same envelope as the
Table API — **one object per IP**, each containing the IP-key field:

```json
{ "result": [
  { "ip_address": "10.10.1.25", "name": "payments-api-01", "app_id": "APP0001234",
    "environment": "production", "owned_by": "Jane Smith",
    "owner_email": "jane.smith@example.com", "risk_tier": "high",
    "compliance_scope": "pci-dss", "asset_criticality": "critical" }
] }
```

**Roles.** The connector account uses **`web_service_admin`** for Scripted REST
APIs (plus `cmdb_read` for the underlying tables) — per the CSW User Guide
([`00-official-references.md`](./00-official-references.md)).

**Point the connector at it.** Set the **table name** field to the resource path
relative to the instance root — `api/<namespace>/<api_id>/v1/records`
(e.g. `api/x_cisco_csw/csw_cmdb_ip_export/v1/records`) — then select
**`ip_address`** as the key and pick the derived fields (`risk_tier`,
`compliance_scope`, `owner_email`, …) as labels. The script honors
`sysparm_limit` / `sysparm_offset`, so the connector's normal pagination works.

> Those derived labels are powerful in policy — e.g. *"quarantine any workload
> where `risk_tier = critical`"* or *"apply the PCI rule set where
> `compliance_scope contains pci-dss`"* — none of which a plain CMDB column gives you.

**A complete, copy-paste working example** — full GlideRecord script (pagination,
the four calculated fields above, the owner-email reference walk), endpoint test,
role grants, scope notes, and connector config — is in
[`09-scripted-rest-api-example.md`](./09-scripted-rest-api-example.md).

> **Which path should you choose?** Use a **Database View** (§4) when everything
> you need is a stored column reachable by a one- or two-table join — no code,
> easiest to maintain. Use a **Scripted REST API** the moment you need a
> calculated field, a multi-hop reference, or a derived label.

---

## 6. Common gotchas

- **A UI List/Form layout is not a Database View.** Only a `sys_db_view` is
  Table-API-queryable by the connector.
- **Multiple IPs per CI is fine** — each NIC/IP becomes its own view row, so every
  IP gets annotated.
- **Without `sysparm_display_value=true`**, reference labels import as `sys_id`s
  (useless). Always add the display-value URL params.
- **Calculated fields are silently dropped** from Database Views → use the
  Scripted REST API path.
- **Label staleness:** the connector ages out labels after consecutive missed
  syncs, so a long ServiceNow outage can flush labels (Cisco's docs are
  inconsistent on the exact count — see
  [`05-operations-and-troubleshooting.md`](./05-operations-and-troubleshooting.md)).
- **Empty CMDB fields → empty labels.** Fix data quality at the source; the next
  sync re-imports the corrected value.

---

## 7. Official documentation links

**Cisco Secure Workload (validate against your release):**

- Configure and Manage Connectors — **On-Premises 4.0**:
  https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-and-manage-connectors-for-secure-workload.html
- Configure and Manage Connectors — **SaaS 4.0**:
  https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-connectors.html
- Configure and Manage Connectors — **On-Premises 3.9** (corroborates the IP-key requirement):
  https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/3_9/cisco-secure-workload-user-guide-on-prem-v39/configure-and-manage-connectors-for-secure-workload.html

**ServiceNow (Database Views):**

- Database Views (product docs): https://docs.servicenow.com/bundle/xanadu-platform-administration/page/use/using-lists/concept/c_DatabaseViews.html
- Create a Database View: https://docs.servicenow.com/bundle/xanadu-platform-administration/page/use/using-lists/task/t_CreateADatabaseView.html
- Table API (the surface the connector uses): https://docs.servicenow.com/bundle/xanadu-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html

> **Disclaimer.** This is companion enablement material, not official Cisco
> documentation. ServiceNow menu labels, table/field names, and CSW limit values
> vary by release and by your CMDB schema — always confirm against your tenant's
> in-product documentation.

---

**Related pages:** [08 — Build a View (full)](./08-no-ip-create-a-view.md) ·
[09 — Scripted REST API Example](./09-scripted-rest-api-example.md) ·
[03 — Configuration](./03-configuration.md) ·
[04 — Using the Labels](./04-using-the-labels.md) ·
[06 — Limitations & FAQ](./06-limitations-and-faq.md)
