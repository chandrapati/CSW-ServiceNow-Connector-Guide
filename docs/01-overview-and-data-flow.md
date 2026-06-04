# 01 — Overview & Data Flow

## What the ServiceNow connector is

The ServiceNow connector is one of Cisco Secure Workload's **Inventory Enrichment** connectors. Its job is narrow and useful: read attributes about your hosts from a **ServiceNow CMDB** (tables or Scripted REST APIs) and attach those attributes as **labels** to the matching IP addresses in CSW inventory.

Per the CSW 4.0 User Guide, the connector performs two high-level functions:

1. **Update ServiceNow metadata** in Secure Workload's inventory for the endpoints it can match.
2. **Periodically take a snapshot** and update the labels on those endpoints.

The result: the business and operational context your organization already maintains in ServiceNow (application name, owner, environment, location, support group, lifecycle status, compliance scope, etc.) becomes available inside CSW as labels you can use to **build scopes, inventory filters, and segmentation policy.**

## What it is *not*

This is the most common source of confusion in a POV, so it's worth being explicit:

- **It does not ingest flow telemetry.** That comes from software agents or flow connectors — not ServiceNow.
- **It does not create inventory.** It is a *metadata feed*. It annotates IP addresses CSW **already knows about** (learned from agents or other orchestrators). ServiceNow records for IPs CSW has never seen do not create new workloads.
- **It is not a real-time stream.** Updates arrive on a **sync interval** (default 60 minutes), not instantly.
- **It does not push anything back to ServiceNow.** The flow is read-only *from* ServiceNow *into* CSW. (If after a manual label cleanup the record still exists in ServiceNow, the next sync re-imports it.)
- **It is not the same as a ServiceNow "external orchestrator."** CSW historically supported several metadata sources as *external orchestrators*; the ServiceNow integration in current releases is delivered as a **connector** that runs on the **Edge** appliance. The annotation behavior (decorate existing IPs, don't create them) mirrors the orchestrator metadata-feed model.

## Data flow

```
                         (read-only, every sync interval — default 60 min)
                          ┌───────────────────────────────────────────┐
                          │                                            ▼
┌───────────────────┐   TCP 443                          ┌─────────────────────────┐
│  ServiceNow        │ ◄───────────  Secure Workload      │  ServiceNow CMDB         │
│  instance (CMDB)   │               Edge appliance ──────►  tables / views /        │
│                    │               (ServiceNow          │  Scripted REST APIs      │
│  Table API:        │                connector)          └─────────────────────────┘
│  /api/now/doc/     │                   │
│  table/schema      │                   │ TCP 443 (Edge initiates)
└───────────────────┘                   ▼
                              ┌───────────────────────────┐
                              │  Secure Workload cluster   │
                              │  (Kafka brokers, 443)      │
                              │                            │
                              │  Inventory IPs annotated   │
                              │  with ServiceNow labels    │
                              └───────────────────────────┘
```

Step by step (as documented in CSW 4.0):

1. The **Edge appliance** connects to the ServiceNow instance using the configured **instance URL** and credentials.
2. It calls the ServiceNow **Table API schema endpoint** (`https://{Instance URL}/api/now/doc/table/schema`) to learn table structure.
3. For each configured table/view (or Scripted REST API), it **queries the records** and reads the selected attributes.
4. It **matches each record to an inventory IP** using the table's IP Address field (the `ip_address` key).
5. It **annotates the matching CSW inventory IPs** with the selected attributes as labels.
6. It **repeats on the sync interval**, adding/updating labels and (after the delete-entry interval) removing labels for records that have disappeared.

The Edge VM talks to ServiceNow on **443** and to the CSW cluster on **443**; the Kafka broker addresses it uses are listed in `/usr/local/tet-controller/cert/kafkaBrokerIps.txt` on the Edge VM. **The Edge VM always initiates** these connections — the cluster never calls back into the Edge.

## Why customers use it

- **System-of-record-driven labels.** Instead of hand-maintaining labels in CSW, you inherit them from the CMDB your org already governs.
- **Consistency.** App owner / environment / business-unit labels match what the rest of IT sees.
- **Policy that survives change.** When the CMDB is updated, the next sync updates CSW labels, so **label-based scopes and policy stay accurate** as ownership and lifecycle change.

Next: [`02-prerequisites.md`](./02-prerequisites.md).
