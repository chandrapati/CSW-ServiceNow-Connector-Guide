# 00 — Official References & Claim Verification

This guide is grounded in the **Cisco Secure Workload 4.0 User Guide**. Read this page first: it lists the authoritative sources and shows, claim by claim, where each fact in this repo comes from.

## Primary Cisco sources

| # | Document | Link |
|---|---|---|
| 1 | Configure and Manage Connectors for Secure Workload — **On-Premises 4.0** | https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-and-manage-connectors-for-secure-workload.html |
| 2 | Configure and Manage Connectors for Secure Workload — **SaaS 4.0** | https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-connectors.html |
| 3 | External Orchestrators in Secure Workload — On-Prem 4.0 (context for metadata-feed behavior) | https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-external-orchestrators-in-secure-workload.html |
| 4 | Virtual Appliances for Connectors — On-Prem 4.0 | https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-and-manage-connectors-for-secure-workload.html |
| 5 | Cisco Secure Workload data sheet (positioning) | https://www.cisco.com/c/en/us/products/collateral/data-center-analytics/tetration-analytics/cisco-secure-workload-dsv3a.html |
| 6 | Configure and Manage Connectors for Secure Workload — **On-Premises 3.9** (corroborates the IP-Address-key requirement, verbatim) | https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/3_9/cisco-secure-workload-user-guide-on-prem-v39/configure-and-manage-connectors-for-secure-workload.html |

> Earlier releases (3.7 / 3.9 / 3.10) document the ServiceNow connector with the same architecture and workflow; specific limit values and UI labels can differ by release. **Always confirm against the User Guide that matches your deployed CSW version.**

## Claim verification table

Every row below is stated by Cisco in source **#1 (CSW 4.0 On-Prem Connectors chapter)** unless noted. "Verified" means the wording in this repo reflects the Cisco source; it does **not** mean Cisco's own text is internally consistent (see the two ⚠ rows and [`07-validation-notes.md`](./07-validation-notes.md)).

| Claim in this repo | Status | Source detail |
|---|---|---|
| ServiceNow is an **Inventory Enrichment** connector | ✅ Verified | Listed under "Connectors for Inventory Enrichment." |
| Connects to ServiceNow, pulls **CMDB labels/attributes**, annotates endpoints | ✅ Verified | "ServiceNow connector connects with ServiceNow Instance to get all the ServiceNow CMDB related labels for the endpoints…" |
| It **annotates IP addresses in inventory** (does not create inventory) | ✅ Verified | "Secure Workload annotates the ServiceNow labels to IP addresses in its inventory." (Consistent with the orchestrator metadata-feed model in #3.) |
| Runs on the **Secure Workload Edge** appliance | ✅ Verified | Connector table lists *Secure Workload Edge*; Edge appliance section lists ServiceNow connector among those deployable on Edge. |
| Required config: **username, password, instance URL** | ✅ Verified | "ServiceNow Instance Configuration" item list. |
| Optional: **Scripted REST APIs**, **additional URL params** | ✅ Verified | Instance configuration + Sync Interval Configuration. |
| **Only tables/views with an IP Address field** are supported; CSW keys off it — you **must select `ip_address` as the key** | ✅ Verified | "User has to chose the `ip_address` attribute from the table as the key…" and Note: "ServiceNow Connector can only support integrating with tables having IP Address field." (Same wording in source #1 and source #6 / 3.9.) |
| User selects **up to 10 unique attributes** per table | ⚠ Conflict | "user can chose upto 10 unique attributes from the table" **vs.** Limits table "Maximum number of attributes … 15." See `07`. |
| Scripted REST APIs: **no path parameters**; must support `sysparm_limit`, `sysparm_fields`, `sysparm_offset` | ✅ Verified | Scripted REST API note. |
| ServiceNow roles: **`cmdb_read`** (tables), **`web_service_admin`** (Scripted REST APIs) | ✅ Verified | "The ServiceNow user roles must include cmdb_read for tables and web_service_admin for scripted REST APIs…" |
| Schema discovery via `…/api/now/doc/table/schema` (Table API) | ✅ Verified | "Processing ServiceNow records." |
| **Default sync interval 60 minutes** (*Data fetch frequency*) | ✅ Verified | "Sync Interval Configuration" / Processing note. |
| **Delete-entry interval** removes records not seen for N consecutive syncs | ⚠ Conflict | "10 continuous sync intervals" (Processing) **vs.** "48 consecutive sync intervals" (Sync Interval Config). See `07`. |
| Example extra URL param: `sysparm_exclude_reference_link=true&sysparm_display_value=true` | ✅ Verified | Sync Interval Configuration. |
| **Ports:** Edge → ServiceNow 443; Edge → CSW cluster 443; Edge initiates | ✅ Verified | "Edge Appliance Communication Flow with Secure Workload and ServiceNow." |
| Kafka broker list at `/usr/local/tet-controller/cert/kafkaBrokerIps.txt` on the Edge VM | ✅ Verified | Same section. |
| **MFA-protected** ServiceNow instances are **not supported** | ✅ Verified | FAQ: "Currently we do not support integrating with ServiceNow instance with MFA." |
| Table without IP → **create a View with an IP field** (e.g., JOIN) and use it | ✅ Verified | FAQ. |
| **Maintenance Explorer** cleanup: POST, host `orchestrator.service.consul`, path `servicenow_cleanup_annotations?args=…`; needs Customer Support privileges | ✅ Verified | "Explore Command to Delete the Labels" / "Running the Commands." |
| Deleted labels **repopulate** if the record still exists in ServiceNow | ✅ Verified | Note under "Running the Commands." |
| **Limits:** 20 instances/connector · 15 attributes/instance · 1 connector/Edge · 1 connector/tenant · 150 connectors/CSW | ✅ Verified | "Limitations of ServiceNow Connectors" table. |
| Connector alerts: down / system usage >90% / config error | ✅ Verified | "Connector Alerts." |

✅ = wording reflects the Cisco source · ⚠ = Cisco's own documentation is internally inconsistent; this repo presents both values and recommends confirming in your tenant.
