# 08 — No IP on the table? Build a ServiceNow Database View

> **Why this page exists.** The CSW ServiceNow connector *"can only support
> integrating with tables having IP Address field"* — during configuration you
> must select an **`ip_address` attribute as the key** (see
> [`03-configuration.md`](./03-configuration.md) Step 3). When the CMDB table
> that holds the attributes you want (a Business Application table, a custom
> `u_*` table, a CI class that doesn't carry IP, etc.) has **no IP field**, the
> Cisco FAQ ([`06-limitations-and-faq.md`](./06-limitations-and-faq.md)) says to
> *"create a View on ServiceNow which will have the desired fields from the
> current table along with IP address (potentially coming from a JOIN with
> another table)"* and use that view in place of the table. This page is the
> **step-by-step** for actually building that view.

> **Source discipline.** The **CSW-side** requirement (IP-keyed, `cmdb_read`,
> Table API over 443) is validated against the CSW 4.0 User Guide — see
> [`00-official-references.md`](./00-official-references.md). The **ServiceNow-side**
> steps below describe standard ServiceNow **Database View** (`sys_db_view`)
> platform behavior; confirm exact menu labels and table/field names against your
> ServiceNow release and your CMDB's actual schema.

---

## The model in one line

You build a **Database View** that JOINs your **attribute-rich table (no IP)** to
an **IP-bearing table**, so each output row is `IP address + the attributes you
want as CSW labels`. A Database View is **read-only** and queryable through the
**Table API** by its view name — exactly what the read-only connector needs — and
it surfaces an IP column **without modifying your CMDB schema**.

---

## Step 1 — Decide where the IP lives (choose the join)

Most CMDBs store host IPs in one of two places. Pick the pattern that matches how
your CMDB is populated:

| Pattern | IP source table | Joins | Use when |
|---|---|---|---|
| **A (simplest)** | `cmdb_ci_network_adapter` (has a `cmdb_ci` reference **and** an `ip_address` field) | 1 | The network-adapter table is populated by Discovery |
| **B (IP Address class)** | `cmdb_ci_ip_address` (references `nic` → `cmdb_ci_network_adapter` → CI) | 2 | Your org uses the IP Address class (multiple IPs/NICs per CI) |

> **No dot-walking in view Where clauses.** A Database View Where clause cannot
> traverse a reference field. To cross two references you must add the
> intermediate table as its own view table — that is why Pattern B needs three
> view tables, not two.

---

## Step 2 — Prerequisites

- A ServiceNow account that can build views (**admin**, or a role such as
  `database_view_admin`).
- Your CSW integration service account with **`cmdb_read`** (see
  [`02-prerequisites.md`](./02-prerequisites.md)). It will read the view at runtime.
- The technical names of your no-IP table (e.g., `cmdb_ci_appl` or a custom
  `u_app_inventory`) and the IP-bearing table you chose in Step 1.

---

## Step 3 — Create the Database View

1. In ServiceNow, go to **System Definition → Database Views → New**.
2. **Name** it like a table — lowercase, no spaces, e.g. `u_csw_cmdb_ip`. The view
   name is what you will query (and what you'll select inside the CSW connector).
3. Set the **Label / Plural** for list display, then **Submit** (you must save the
   view before you can add tables to it).

---

## Step 4 — Add the View Tables (define the JOIN)

In the new view's **View Tables** related list, add a row per table. The
**Variable prefix** you assign each table becomes the prefix on every field name
in the view output (`prefix_fieldname`), and is how you reference fields in the
**Where clause** (`prefix_field=prefix_field`). **Order** controls join order
(lowest first = base/driving table).

### Pattern A — one join (CI table ➝ network adapter)

| View Table | Table | Variable prefix | Order | Left join | Where clause |
|---|---|---|---|---|---|
| 1 (base) | *your no-IP table*, e.g. `cmdb_ci_appl` | `ci` | `100` | unchecked | *(empty)* |
| 2 (IP) | `cmdb_ci_network_adapter` | `na` | `200` | unchecked\* | `na_cmdb_ci=ci_sys_id` |

\* Leave **Left join unchecked** for an inner join, so only CIs that actually have
an IP appear (the connector needs the IP). Check it only if you want every base
row regardless of IP.

→ IP key column in the view: **`na_ip_address`**. Attribute columns: `ci_*`.

### Pattern B — two joins (CI ➝ NIC ➝ IP Address class)

| View Table | Table | Variable prefix | Order | Where clause |
|---|---|---|---|---|
| 1 (base) | `cmdb_ci_appl` (your no-IP table) | `ci` | `100` | *(empty)* |
| 2 (NIC) | `cmdb_ci_network_adapter` | `na` | `200` | `na_cmdb_ci=ci_sys_id` |
| 3 (IP) | `cmdb_ci_ip_address` | `ip` | `300` | `ip_nic=na_sys_id` |

→ IP key column in the view: **`ip_ip_address`**. Attribute columns: `ci_*`.

> **Where-clause syntax.** Always `prefix_field=prefix_field` — the field name is
> appended to that table's Variable prefix with an underscore (so `na_cmdb_ci` =
> the `cmdb_ci` field on the network-adapter view table). Supported operators:
> `= != < <= > >= && ||`.

### Additional fields available from the joined tables

Beyond the IP key, the joined tables expose other attributes that make useful CSW labels. These are available as view columns if you add them to View Fields in Step 5 — they are silently excluded if you don't.

**From `cmdb_ci_network_adapter` (prefix `na_`):**

| View field name | Source field | Useful for |
|---|---|---|
| `na_ip_address` | `ip_address` | **Required** — the IP key |
| `na_mac_address` | `mac_address` | Hardware/asset correlation; detect NIC changes |
| `na_operational_status` | `operational_status` | Filter inactive or decommissioned NICs |
| `na_netmask` | `netmask` | Subnet-aware scoping in CSW |
| `na_name` | `name` | Interface name (e.g., `eth0`, `ens3`) |

**From `cmdb_ci_ip_address` (prefix `ip_`, Pattern B only):**

| View field name | Source field | Useful for |
|---|---|---|
| `ip_ip_address` | `ip_address` | **Required** — the IP key in Pattern B |
| `ip_subnet_mask` | `subnet_mask` | Subnet annotation on the IP record |
| `ip_nic` | `nic` | Reference back to the NIC (usually excluded) |

> **Add these in Step 5.** If you do not explicitly add a field to View Fields, it will not appear in the view output — even if the underlying table has it. See the all-or-nothing warning in Step 5.

---

## Step 5 — (Optional) Restrict the returned fields

> ⚠️ **All-or-nothing behavior — decide upfront.**
> ServiceNow View Fields work on an **all-or-nothing basis per view table**:
> - If you add **no** View Fields to a view table, that table returns **all** of its columns (noisy, but nothing is lost).
> - The moment you add **even one** View Field to a view table, ServiceNow switches to returning **only the fields you explicitly listed** for that table. Any attribute you omit — even one you add later to the underlying CMDB table — will be **silently absent** from the view output and therefore invisible to the CSW connector.
>
> **Recommendation:** treat View Fields as your final attribute contract. Before saving, list every field you might ever want as a CSW label — including the additional `na_*` / `ip_*` fields from Step 4. It is much easier to add all candidates now than to edit the view and re-sync the connector later.

Add View Fields via the **View Fields** related list on each view table:

- the IP column (`na_ip_address` or `ip_ip_address`) — **required**
- `ci_name`, `ci_sys_id`
- any `na_*` fields you want (e.g., `na_mac_address`, `na_operational_status`) — see Step 4 table
- the business attributes from the CI table you want as labels (e.g., `ci_u_environment`, `ci_support_group`, `ci_business_service`, `ci_owned_by`, `ci_location`, `ci_u_cost_center`)

If you are unsure which attributes you need, **leave View Fields empty** for the first sync so CSW shows you the full column list — then come back and pin the ones you actually use.

---

## Step 6 — Test the view in ServiceNow

- **In the UI:** type `u_csw_cmdb_ip.LIST` in the filter navigator and press Enter.
  You should see one row per IP with your attributes alongside.
- **Via the Table API** (the same surface the connector uses):

```bash
curl -u "csw_integration_user:PASSWORD" \
  "https://YOURINSTANCE.service-now.com/api/now/table/u_csw_cmdb_ip?sysparm_limit=5&sysparm_display_value=true&sysparm_exclude_reference_link=true"
```

You should get JSON rows containing the prefixed fields, including a valid IPv4/IPv6
value in `na_ip_address` / `ip_ip_address`.

---

## Step 7 — Grant the integration user access to the view

A Database View inherits readability from its underlying tables, but verify:

1. The CSW service account has **`cmdb_read`**.
2. It can **read every joined base table** (`cmdb_ci_appl`,
   `cmdb_ci_network_adapter`, and `cmdb_ci_ip_address` if Pattern B).
3. If your instance restricts view access, create a **read ACL**
   (`System Security → Access Control (ACL)`) on the view's table name
   (`u_csw_cmdb_ip`, operation **read**) granting the integration role.

---

## Step 8 — Point the CSW connector at the view

In the connector configuration (see [`03-configuration.md`](./03-configuration.md)):

1. When CSW fetches the table list from the instance schema, **select your view
   name** `u_csw_cmdb_ip` *in place of the original table*.
2. On the attribute-selection step, **choose the prefixed IP field**
   (`na_ip_address` / `ip_ip_address`) **as the `ip_address` key** — the same
   IP-key rule applies; in a view the key is just the prefixed column.
3. Select the prefixed attribute fields (`ci_*`) you want as labels.
4. **Strongly recommended:** if any selected attribute is a ServiceNow **reference
   field** (owner, support group, location, business service), add the reference
   lookup to **Additional REST API url params** so labels import as readable
   values, not `sys_id`s (see [`03-configuration.md`](./03-configuration.md)
   Step 5):

```
sysparm_exclude_reference_link=true&sysparm_display_value=true
```

5. Set a sensible **Data fetch frequency** — joined views are heavier than flat
   tables, so raise the interval for large estates.
6. **Review & apply**, then verify labels on a known IP per
   [`03-configuration.md`](./03-configuration.md) Step 6.

> **Label keys.** As elsewhere in this guide, don't assume a fixed label prefix —
> after the first sync, open the **Inventory** view for a known IP and read the
> actual label keys CSW created (see [`04-using-the-labels.md`](./04-using-the-labels.md)).

---

## Caveats & gotchas

- **Inner vs left join.** Keep it an inner join so rows without an IP are excluded;
  CSW ignores IP-less rows anyway, and the inner join keeps the dataset smaller.
- **Multiple IPs per CI is expected and fine.** A CI with several NICs/IPs produces
  several view rows, so each IP gets annotated — that is the desired behavior.
- **Field prefixes are part of your mapping.** View columns are `prefix_field`. If
  you later rename a Variable prefix, the connector's attribute/key mapping must be
  updated.
- **Display values.** Without `sysparm_display_value=true`, reference attributes
  import as `sys_id`s — useless as labels.
- **Performance / staleness.** Large joined views can be slow; raise the sync
  interval. Remember the connector ages out labels after consecutive missed syncs
  (Cisco's docs are inconsistent on the exact count — see
  [`05-operations-and-troubleshooting.md`](./05-operations-and-troubleshooting.md)
  and [`07-validation-notes.md`](./07-validation-notes.md)), so a long ServiceNow
  outage can flush labels.
- **Table rotation.** You cannot build a Database View on table-rotation tables
  (not a concern for CMDB CI tables).
- **Calculated/scripted fields are silently excluded.** ServiceNow **Calculated** fields (type: Calculated, backed by a Groovy or JavaScript script on the source table) cannot be exposed through a Database View — they are absent from the view output with no error. If the CMDB table you are joining has a calculated field you want as a CSW label (e.g., a derived risk score, a compliance tier, a dynamically resolved owner), use the **Scripted REST API** path instead (see below).
- **Scripted REST API alternative.** If the data needs server-side business logic, calculated fields, or joins a Database View cannot express, a **Scripted REST API** that returns an IP field is the other supported path — same IP-key rule applies (see [`03-configuration.md`](./03-configuration.md) Step 4). This is the right choice any time the attributes you need cannot be represented as stored, non-calculated columns across directly joinable tables.

---

**Related pages:** [03 — Configuration](./03-configuration.md) ·
[06 — Limitations & FAQ](./06-limitations-and-faq.md) ·
[02 — Prerequisites](./02-prerequisites.md) ·
[05 — Operations & Troubleshooting](./05-operations-and-troubleshooting.md)
