# 07 — Validation Notes

Because this guide is shared with customers, every technical claim was checked against the **CSW 4.0 On-Premises User Guide** ("Configure and Manage Connectors for Secure Workload"). This page records two things:

1. **Where Cisco's own documentation is internally inconsistent** — so you don't get caught out, and
2. **What this repo deliberately does *not* assert** — to avoid presenting a guess as fact.

The full claim-by-claim source map is in [`00-official-references.md`](./00-official-references.md).

## Flagged inconsistencies in Cisco's documentation

### A. Attribute count: "up to 10 per table" vs "15 per instance"
- The **configuration workflow** text states: *"user can chose upto 10 unique attributes from the table."*
- The **"Limitations of ServiceNow Connectors"** table states: *"Maximum number of attributes that can be fetched from one ServiceNow instance — 15."*
- These describe **different scopes** (per-table selection vs. per-instance total) but are easy to conflate, and Cisco does not reconcile them on the page.
- **This repo's stance:** plan around **10 attributes per table**; treat **15** as the per-instance ceiling; **verify the effective limit in your tenant** before committing a label design.

### B. Delete-entry interval: "10 continuous sync intervals" vs "48 consecutive sync intervals"
- The **"Processing ServiceNow records"** section states: *"Secure Workload will delete any entry not seen for 10 continuous sync intervals."*
- The **"Sync Interval Configuration"** section states: *"If an entry is not seen in 48 consecutive sync intervals, we go ahead and delete the entry."*
- Both agree on the **risk** (a prolonged ServiceNow outage can age out labels) but give **different default thresholds.**
- **This repo's stance:** do not rely on a specific default; **set the *Delete entry interval* explicitly** and validate the actual behavior in your tenant. Size it against your worst-case acceptable ServiceNow outage.

## What this repo intentionally does NOT assert

- **A fixed label key/namespace for ServiceNow attributes.** The CSW 4.0 ServiceNow connector documentation does not pin down a specific prefix the way the DNS *external orchestrator* does (`orchestrator_system/dns_name`). Rather than invent one, this repo tells you to **read the actual label keys from the Inventory view** after the first sync and use those. (See [`04-using-the-labels.md`](./04-using-the-labels.md).)
- **Exact UI menu paths for every release.** UI labels shift between versions. Paths here (e.g., **Manage → Connectors**, **Troubleshoot → Maintenance Explorer**, **Platform → Tenants**) match CSW 4.0; confirm in your build.
- **Release-specific limit values for non-4.0 deployments.** The limits in [`06-limitations-and-faq.md`](./06-limitations-and-faq.md) are the **4.0** values. 3.7/3.9/3.10 use the same architecture but may differ in specifics — check the matching User Guide.
- **The full `servicenow_cleanup_annotations` argument string.** Cisco documents the command shape (`servicenow_cleanup_annotations?args=…`) but not a complete copy-paste example with all arguments. Treat the cleanup workflow as a **Customer-Support-assisted** action and confirm the exact arguments for your release.

## How to re-verify before a customer engagement

1. Open the **User Guide that matches the customer's deployed CSW version** (On-Prem or SaaS).
2. Go to **Configure and Manage Connectors → Connectors for Inventory Enrichment → ServiceNow Connector.**
3. Re-confirm: the **Edge** appliance requirement, **roles** (`cmdb_read` / `web_service_admin`), **ports** (443), **sync defaults**, the **limits table**, and the **MFA-unsupported** note.
4. Resolve the two inconsistencies above **empirically in the tenant** (attribute max and delete interval) and note the observed values in the engagement record.

---

*Validation performed June 2026 against the CSW 4.0 On-Premises User Guide. This is enablement material, not official Cisco documentation — the User Guide for the customer's release is always authoritative.*
