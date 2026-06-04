# 04 — Using the Imported Labels

Importing ServiceNow attributes is only step one. The value comes from **using those labels** to organize workloads and drive segmentation policy.

## Where the labels show up

After a successful sync, the selected ServiceNow attributes are attached to the matching **inventory IP addresses** as labels. You'll see them on:

- the **Inventory** detail view for an individual IP/workload,
- the **label/annotation** selectors when you build **inventory filters** and **scopes**, and
- the **column pickers** in inventory and flow views.

> **Label key naming — confirm in your tenant.** The exact key/namespace under which ServiceNow attributes appear (and any prefix CSW adds) can vary by release and configuration. Rather than assume a prefix, open the Inventory view for a known IP after the first sync and read the **actual label keys** CSW created. Use those exact keys everywhere below. (This repo intentionally does **not** assert a fixed label prefix because the CSW 4.0 connector documentation does not pin one down for ServiceNow attributes; the DNS *external orchestrator*, by contrast, documents `orchestrator_system/dns_name`.)

## Turn labels into structure

1. **Inventory filters** — define dynamic groups using the ServiceNow labels, e.g.
   - `environment = Production`
   - `application = Payments` **and** `lifecycle_status = Active`
2. **Scopes** — build (or refine) your scope tree from the CMDB's source-of-truth fields (business unit → application → environment). Scopes are how CSW delegates visibility and policy ownership.
3. **Policy** — author segmentation policy in terms of these labels so intent reads like the business:
   - "Allow `application=Payments` `environment=Production` workloads to reach `application=Payments-DB` `environment=Production` on the documented ports; deny lateral movement from `environment=NonProd`."

## Why CMDB-driven labels are powerful

- **Intent that matches the org.** Policy written against `application` / `environment` / `business_unit` is readable and reviewable by app and audit teams.
- **Self-healing context.** When ownership, environment, or lifecycle changes in ServiceNow, the next connector sync updates the CSW labels — and any **label-based** scope/policy follows automatically (no manual re-tagging).
- **Cleaner ADM and faster reviews.** Application Dependency Mapping and policy reviews are easier when workloads already carry meaningful business labels.

## Good practices

- **Start with a few high-signal attributes** (environment, application, owner group) rather than importing everything.
- **Prefer label-based (dynamic) scopes and filters** over static IP lists so they track CMDB changes.
- **Treat the CMDB as the source of truth.** If a label is wrong in CSW, fix it in ServiceNow — the connector will re-import the correct value. Manual cleanup in CSW (see `05`) is repopulated on the next sync if the record still exists.
- **Watch data quality.** Empty or inconsistent CMDB fields become empty/inconsistent labels. Coordinate with the CMDB owner to improve population of the fields you segment on.

## How this feeds the policy lifecycle

ServiceNow labels are an input to the standard CSW journey:

```
ServiceNow CMDB  →  connector sync  →  labels on inventory
        │
        ▼
  scopes + inventory filters  →  ADM / policy discovery  →  Monitor → Simulate → Enforce
```

For the discovery-through-enforcement half, see the **CSW-Policy-Lifecycle** repo linked from the [README](../README.md).
