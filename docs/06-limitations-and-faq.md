# 06 — Limitations & FAQ

All values below are from the **CSW 4.0 On-Premises User Guide** ("Limitations of ServiceNow Connectors" and the ServiceNow FAQ). Confirm against the User Guide for **your** release, as limits can change between versions.

## Documented limits (CSW 4.0)

| Metric | Limit |
|---|---|
| Maximum number of ServiceNow **instances** configured on one ServiceNow connector | **20** |
| Maximum number of **attributes** that can be fetched from one ServiceNow instance | **15** |
| Maximum number of ServiceNow **connectors on one Secure Workload Edge appliance** | **1** |
| Maximum number of ServiceNow **connectors on one Tenant (rootscope)** | **1** |
| Maximum number of ServiceNow **connectors on Secure Workload** | **150** |

> ⚠ **Attribute count nuance:** the configuration workflow text says you may select **"up to 10 unique attributes" per table**, while this limits table says **15 attributes per instance**. These are not obviously the same scope (per-table selection vs. per-instance total). Plan for **10 per table** and confirm the effective maximum in your tenant. Detail in [`07-validation-notes.md`](./07-validation-notes.md).

## Practical implications of the limits

- **One connector per tenant and per Edge appliance.** You consolidate all ServiceNow integration for a tenant into a **single connector** that can hold **up to 20 instances**. Design around that — don't expect multiple ServiceNow connectors in one scope.
- **Attribute budget is small.** With ~10 attributes/table to work with, **choose the highest-value segmentation fields**; don't try to mirror the whole CMDB.
- **150 connectors across the platform** is the global ceiling, relevant only in very large multi-tenant deployments.

## Frequently asked questions (from Cisco docs)

**Q: What if a ServiceNow CMDB table does not have an IP address?**
A: Create a **View** on ServiceNow that includes the desired fields from the current table **plus an IP address** (potentially via a JOIN with another table). Use that view in place of the table name. (The connector can only integrate with tables/views that have an IP Address field.)

**Q: What if a ServiceNow instance requires MFA?**
A: **Not supported.** The connector cannot integrate with a ServiceNow instance that requires MFA. Use a service account that is exempt from MFA per your ServiceNow security policy.

**Q: If I delete labels with the Explore command, will they come back?**
A: **Yes — if the record still exists in ServiceNow**, it will be repopulated on the next sync. To remove permanently, remove/exclude the record at the ServiceNow source.

## Additional validated facts worth knowing

- **Required ServiceNow roles:** `cmdb_read` (tables), `web_service_admin` (Scripted REST APIs).
- **Scripted REST API constraints:** no path parameters; must support `sysparm_limit`, `sysparm_fields`, `sysparm_offset`.
- **Schema discovery endpoint:** `https://{Instance URL}/api/now/doc/table/schema` (ServiceNow Table API).
- **Default sync interval:** 60 minutes (*Data fetch frequency*).
- **Ports:** Edge → ServiceNow **443**; Edge → CSW cluster **443**; the Edge VM always initiates.
- **Runs on:** the Secure Workload **Edge** appliance.
- **Optional extra URL params** example: `sysparm_exclude_reference_link=true&sysparm_display_value=true`.

See the full source-by-source audit in [`07-validation-notes.md`](./07-validation-notes.md).
