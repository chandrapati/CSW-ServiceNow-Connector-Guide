# 02 — Prerequisites

Work through this checklist **before** you open the connector configuration. Most failed POV attempts are a missing role, a blocked port, or an MFA-protected service account — all avoidable here.

## 1. Secure Workload side

- [ ] **A Secure Workload *Edge* virtual appliance** is deployed and healthy for the target **tenant (rootscope)**. The ServiceNow connector runs on the **Edge** appliance (not the Ingest appliance).
  - Sizing, OVA/QCOW deployment, and appliance networking are covered in the *Virtual Appliances for Connectors* section of the CSW User Guide.
- [ ] You have a CSW account with rights to **Manage → Connectors** for that scope.
- [ ] For the optional **Maintenance Explorer** label-cleanup workflow, you'll need **Customer Support** privileges and the tenant's **VRF ID** (see [`05-operations-and-troubleshooting.md`](./05-operations-and-troubleshooting.md)). Not required for normal operation.

## 2. ServiceNow side

- [ ] A **ServiceNow service account** (username + password) dedicated to this integration.
- [ ] The account holds the required **roles**:

  | Integration method | Required ServiceNow role |
  |---|---|
  | CMDB **tables** | `cmdb_read` |
  | **Scripted REST APIs** | `web_service_admin` |

- [ ] The instance does **not require MFA** for this account. **MFA-protected ServiceNow instances are not supported** by the connector. Use a service account exempt from MFA (per your ServiceNow security policy).
- [ ] Your target **table(s) contain an IP Address field.** If a CMDB table you care about has no IP field, plan to **create a View** that brings in an IP field (e.g., via a JOIN to another table) and use the view instead. See the FAQ in [`06-limitations-and-faq.md`](./06-limitations-and-faq.md).
- [ ] If you'll use **Scripted REST APIs**, confirm they:
  - have **no path parameters**, and
  - support the `sysparm_limit`, `sysparm_fields`, and `sysparm_offset` query parameters.

## 3. Network / firewall

The **Edge VM initiates** all connections. Open egress accordingly:

| Source | Destination | Port/Proto | Purpose |
|---|---|---|---|
| Secure Workload **Edge VM** | **ServiceNow instance** | **TCP 443** | Query CMDB tables / Scripted REST APIs |
| Secure Workload **Edge VM** | **CSW cluster** (Kafka brokers) | **TCP 443** | Deliver annotations to inventory |

- The Kafka broker IPs/FQDNs the Edge uses are in `/usr/local/tet-controller/cert/kafkaBrokerIps.txt` on the Edge VM — use them to scope the cluster-side firewall rule precisely.
- If ServiceNow is reached through an **egress proxy / fixed source IPs**, ensure the IP addresses used match what ServiceNow expects/allow-lists. (Cisco notes: "the IP addresses used for the configuration should be the same as configured on the ServiceNow instance.")

## 4. Data-design decisions (decide before configuring)

- **Which table(s) or view(s)** are the authoritative source for the labels you want? Prefer a small number of clean sources over many overlapping ones.
- **Which attributes** become labels? You can pick **up to 10 attributes per table** in the selection workflow. Choose attributes that are (a) populated, (b) stable, and (c) useful for segmentation — e.g., `environment`, `application`, `business_unit`, `owner_group`, `lifecycle_status`.
  - ⚠ Cisco's docs state "up to 10 attributes" in the workflow but list a per-instance limit of **15**. Treat **10 per table** as the safe planning number and confirm in your tenant. See [`07-validation-notes.md`](./07-validation-notes.md).
- **Sync cadence vs. table size.** Large tables → set a **longer sync interval** so a sync completes well within the window. Default is 60 minutes.

Once every box is checked, continue to [`03-configuration.md`](./03-configuration.md).
