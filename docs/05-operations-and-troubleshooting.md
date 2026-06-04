# 05 — Operations & Troubleshooting

## Day-2 operations

### Sync behavior
- The connector queries ServiceNow on the **Data fetch frequency** (default **60 minutes**) and updates labels each cycle.
- Records that stop appearing are removed after the **Delete entry interval** (⚠ see the documented-value conflict in [`07-validation-notes.md`](./07-validation-notes.md)). **A long ServiceNow outage can therefore age out labels** — plan the interval with that risk in mind.

### Connector alerts (built in)
CSW raises connector alerts you should monitor/route (e.g., to Syslog/Email via the alert notifier connectors):

| Alert | Meaning | Severity |
|---|---|---|
| **Appliance/Connector down** | Missing heartbeats from the appliance/connector. | High |
| **Appliance/Connector system usage** | CPU/memory/disk > 90% (normal during heavy processing). | High |
| **Connector configuration error** | CSW could not connect to the configured **ServiceNow server** ("Cannot connect to server, check config"). | High / Low |

## Manually deleting labels (Maintenance Explorer)

Use this only when you need to remove labels for a specific IP **immediately**, without waiting for the delete interval.

> Requires **Customer Support** privileges. If the **Troubleshoot → Maintenance Explorer** menu isn't visible, your account lacks the needed permission.

1. **Find the tenant's VRF ID:** **Platform → Tenants**. The **ID** field of the Tenants table is the VRF ID.
2. **Open the Explorer:** **Troubleshoot → Maintenance Explorer**, then the **explore** tab.
3. **Run the command:**
   - Action: **POST**
   - Snapshot host: **`orchestrator.service.consul`**
   - Snapshot path (delete labels for a given IP on a ServiceNow instance):
     ```
     servicenow_cleanup_annotations?args=<arguments>
     ```
   - Click **Send**.

> **Important:** if the record still exists in the ServiceNow instance, the label **will be repopulated on the next sync.** To remove it permanently, remove/exclude it at the ServiceNow source.

## Troubleshooting guide

| Symptom | Likely cause | What to check / do |
|---|---|---|
| **No labels appear after a sync** | IP key not matching inventory | Confirm the table has an **IP Address field** and you selected **`ip_address`** as the key; confirm CSW already knows that IP (ServiceNow does **not** create inventory). |
| **Connector won't connect** ("check config") | Credentials, URL, port, or **MFA** | Verify username/password and **Instance URL**; confirm **443** egress Edge→ServiceNow; confirm the account is **not MFA-protected** (unsupported). |
| **Some attributes missing as labels** | Attribute not selected / empty in CMDB | Re-check attribute selection (**up to 10/table**); confirm the field is actually populated in ServiceNow. |
| **Scripted REST API fails** | API shape or role | Ensure **no path parameters**; ensure it supports `sysparm_limit`, `sysparm_fields`, `sysparm_offset`; ensure the account has **`web_service_admin`**. |
| **Labels disappeared unexpectedly** | Delete-entry interval aged them out during an outage | Check ServiceNow reachability; review the **Delete entry interval** (⚠ conflicting documented values — see `07`); restore connectivity and let a sync repopulate. |
| **Sync seems to never finish / overlaps** | Table too large for interval | Increase **Data fetch frequency**; narrow the table/view; reduce attributes. |
| **Can't reach Maintenance Explorer** | Insufficient privileges | Needs **Customer Support** role. |
| **A CMDB table has no IP field** | Table not usable directly | Create a **View** that includes an IP field (e.g., JOIN) and configure that view. |
| **Connector down alert** | Edge appliance/connector heartbeat lost | Check Edge VM health, resources, and connectivity to the CSW cluster (443, Kafka brokers in `kafkaBrokerIps.txt`). |

## Verifying connectivity from the Edge VM

- Confirm the Edge VM can reach the ServiceNow instance on **443** and the **CSW cluster** Kafka brokers on **443**.
- Kafka broker addresses are in `/usr/local/tet-controller/cert/kafkaBrokerIps.txt` on the Edge VM. The **Edge VM initiates** these connections.

Next: limits and FAQ in [`06-limitations-and-faq.md`](./06-limitations-and-faq.md).
