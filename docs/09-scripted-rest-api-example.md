# 09 — Scripted REST API: IP + Calculated Fields for the CSW Connector

> **When to use this instead of a Database View.**  
> Use a Scripted REST API when you need any of the following:
> - **Calculated fields** (Groovy/JavaScript `type: calculated` fields on a CMDB table) — Database Views silently exclude them; a Scripted REST API can compute them server-side.
> - **Multi-hop reference resolution** — walking `owned_by → sys_user → email` or resolving a chain of references that a WHERE clause can't express.
> - **Derived/business-logic labels** — risk tier, compliance scope, asset criticality, or any value computed from multiple CMDB fields.
> - **Joins across more than two tables** where the intermediate reference chain makes a Database View WHERE clause impractical.
>
> If none of the above apply, prefer the Database View approach in [`08-no-ip-create-a-view.md`](./08-no-ip-create-a-view.md) — it requires no code and is easier to maintain.

---

## How the CSW connector consumes a Scripted REST API

The CSW ServiceNow connector reads data through the **Table API** (`/api/now/table/{tableName}`).  
A Scripted REST API sits at a **different base path** (`/api/{namespace}/{api_name}/{resource}`) but must return the **same JSON envelope** the connector expects:

```json
{
  "result": [
    { "ip_address": "10.1.2.3", "name": "app-server-01", "risk_tier": "high", ... },
    { "ip_address": "10.1.2.4", "name": "app-server-02", "risk_tier": "critical", ... }
  ]
}
```

One row per IP. Each row must contain a field the connector will use as the **IP key** (you select this during connector configuration — see [`03-configuration.md`](./03-configuration.md) Step 3).

---

## Step 1 — Create the Scripted REST API in ServiceNow

1. Navigate to **System Web Services → Scripted REST APIs → New**.
2. Fill in:

| Field | Example value |
|---|---|
| **Name** | `CSW CMDB IP Export` |
| **API ID** | `csw_cmdb_ip_export` (auto-generated; lowercase, no spaces) |
| **Default version** | `v1` |

3. **Submit** to save the API record.
4. In the **Resources** related list, click **New** to add a resource:

| Field | Value |
|---|---|
| **Name** | `records` |
| **HTTP method** | `GET` |
| **Relative path** | `/records` |
| **Authentication** | `Require Authentication` |

5. Paste the JavaScript below into the **Script** field.
6. **Submit** and note the full endpoint path shown in the API record header — it will look like:  
   `https://YOURINSTANCE.service-now.com/api/x_cisco_csw/csw_cmdb_ip_export/v1/records`

---

## Step 2 — The Script

The script below queries `cmdb_ci_appl` (application CIs) joined to `cmdb_ci_network_adapter` for IPs. It exports the IP key plus standard CMDB attributes — including an **`app_id`** application business key (hostname is `name`, owner is `owned_by`) — and four **calculated / derived fields** that a Database View cannot expose. Adapt the CI table, field names, and business logic to match your CMDB schema.

```javascript
(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {

    // ── Pagination ──────────────────────────────────────────────────────────
    var limit  = parseInt(request.queryParams.sysparm_limit)  || 500;
    var offset = parseInt(request.queryParams.sysparm_offset) || 0;

    // ── Query CIs ───────────────────────────────────────────────────────────
    // Change 'cmdb_ci_appl' to the CMDB table that holds your attributes.
    var ciGr = new GlideRecord('cmdb_ci_appl');
    ciGr.addActiveQuery();
    ciGr.chooseWindow(offset, offset + limit);
    ciGr.query();

    var records = [];

    while (ciGr.next()) {

        var ciSysId = ciGr.getUniqueValue();

        // ── Get active IPs for this CI via network adapter ─────────────────
        var naGr = new GlideRecord('cmdb_ci_network_adapter');
        naGr.addQuery('cmdb_ci', ciSysId);
        naGr.addQuery('operational_status', 1);   // 1 = Operational
        naGr.addNotNullQuery('ip_address');
        naGr.query();

        while (naGr.next()) {

            records.push({

                // ── Required: IP key ───────────────────────────────────────
                ip_address: naGr.getValue('ip_address'),

                // ── Standard CMDB fields ───────────────────────────────────
                sys_id:           ciSysId,
                name:             ciGr.getValue('name'),

                // Application identifier (business key) — adjust to your schema.
                // Many CMDBs use the record 'number' (e.g. APP0001234) or a
                // custom field such as 'u_app_id' / 'u_application_id'.
                app_id:           ciGr.getValue('u_app_id') || ciGr.getValue('number') || '',

                mac_address:      naGr.getValue('mac_address')      || '',
                operational_status: naGr.getDisplayValue('operational_status'),

                // Reference fields — getDisplayValue() gives human-readable string
                environment:      ciGr.getDisplayValue('u_environment'),
                support_group:    ciGr.getDisplayValue('support_group'),
                business_service: ciGr.getDisplayValue('business_service'),
                location:         ciGr.getDisplayValue('location'),
                owned_by:         ciGr.getDisplayValue('owned_by'),

                // ── Calculated / derived fields ────────────────────────────
                // These four fields CANNOT be returned by a Database View.

                // 1. risk_tier: derived from environment + vulnerability score
                risk_tier:          _riskTier(ciGr),

                // 2. owner_email: walk owned_by reference → sys_user.email
                owner_email:        _ownerEmail(ciGr),

                // 3. compliance_scope: inferred from business service name
                compliance_scope:   _complianceScope(ciGr),

                // 4. asset_criticality: composite score (env + data class)
                asset_criticality:  _assetCriticality(ciGr)
            });
        }
    }

    // ── Return Table-API-compatible envelope ────────────────────────────────
    response.setStatus(200);
    response.setContentType('application/json');
    response.getStreamWriter().writeString(JSON.stringify({ result: records }));

})(request, response);


// ─────────────────────────────────────────────────────────────────────────────
// Helper functions
// Adapt field names and logic to match your CMDB schema.
// ─────────────────────────────────────────────────────────────────────────────

/**
 * _riskTier
 * Derives a simple risk tier from the CI's environment and vulnerability score.
 * Replace u_vulnerability_score with your actual vulnerability/risk field name.
 */
function _riskTier(ciGr) {
    var env  = (ciGr.getValue('u_environment') || '').toLowerCase();
    var vuln = parseInt(ciGr.getValue('u_vulnerability_score')) || 0;

    if (env === 'production' && vuln > 7) return 'critical';
    if (env === 'production')             return 'high';
    if (env === 'staging')                return 'medium';
    return 'low';
}

/**
 * _ownerEmail
 * Walks the owned_by reference field to sys_user and returns the email.
 * A two-hop reference: cmdb_ci_appl.owned_by → sys_user.email.
 * A Database View WHERE clause cannot traverse this; a Scripted REST API can.
 */
function _ownerEmail(ciGr) {
    var userSysId = ciGr.getValue('owned_by');
    if (!userSysId) return '';

    var userGr = new GlideRecord('sys_user');
    if (userGr.get(userSysId)) {
        return userGr.getValue('email') || '';
    }
    return '';
}

/**
 * _complianceScope
 * Infers compliance frameworks from the business service display name.
 * Returns a comma-separated string suitable for a CSW label value.
 */
function _complianceScope(ciGr) {
    var bs  = (ciGr.getDisplayValue('business_service') || '').toLowerCase();
    var loc = (ciGr.getDisplayValue('location')         || '').toLowerCase();

    var scopes = [];
    if (bs.indexOf('payment') >= 0  || bs.indexOf('finance') >= 0)  scopes.push('pci-dss');
    if (bs.indexOf('health')  >= 0  || bs.indexOf('patient') >= 0)  scopes.push('hipaa');
    if (loc.indexOf('eu')     >= 0  || loc.indexOf('europe') >= 0)  scopes.push('gdpr');
    if (bs.indexOf('federal') >= 0  || bs.indexOf('gov')     >= 0)  scopes.push('fedramp');

    return scopes.length ? scopes.join(',') : 'none';
}

/**
 * _assetCriticality
 * Computes a composite criticality rating from environment and data classification.
 * Replace u_data_classification with your actual field name.
 */
function _assetCriticality(ciGr) {
    var score = 0;
    var env   = (ciGr.getValue('u_environment')        || '').toLowerCase();
    var dc    = (ciGr.getValue('u_data_classification') || '').toLowerCase();

    if (env === 'production') score += 3;
    else if (env === 'staging' || env === 'uat') score += 1;

    if (dc === 'restricted')   score += 3;
    else if (dc === 'confidential') score += 2;
    else if (dc === 'internal')     score += 1;

    if (score >= 5) return 'critical';
    if (score >= 3) return 'high';
    if (score >= 1) return 'medium';
    return 'low';
}
```

---

## Step 3 — Test the endpoint

```bash
# Replace with your instance URL, namespace, and API ID.
# Per the CSW User Guide, the connector account uses 'web_service_admin' for Scripted REST APIs
# (in addition to 'cmdb_read' for the underlying tables); see docs/00 and docs/02.
curl -u "csw_integration_user:PASSWORD" \
  "https://YOURINSTANCE.service-now.com/api/x_cisco_csw/csw_cmdb_ip_export/v1/records?sysparm_limit=5" \
  | python3 -m json.tool
```

Expected response shape:

```json
{
  "result": [
    {
      "ip_address": "10.10.1.25",
      "sys_id": "abc123...",
      "name": "payments-api-01",
      "app_id": "APP0001234",
      "mac_address": "00:50:56:ab:12:34",
      "operational_status": "Operational",
      "environment": "production",
      "support_group": "Platform Engineering",
      "business_service": "Payment Processing",
      "location": "US-East DC1",
      "owned_by": "Jane Smith",
      "risk_tier": "high",
      "owner_email": "jane.smith@example.com",
      "compliance_scope": "pci-dss",
      "asset_criticality": "critical"
    }
  ]
}
```

Verify:
- `ip_address` is a valid IPv4 or IPv6 string on every row.
- `app_id` is populated — if it comes back empty, your CMDB stores the application key in a different field; update the `app_id` line (see *Adapting this script*).
- Reference fields (`support_group`, `business_service`, etc.) return display names, not `sys_id`s.
- Calculated fields (`risk_tier`, `owner_email`, `compliance_scope`, `asset_criticality`) appear and contain non-empty values for at least one known CI.

---

## Step 4 — Grant the integration user access

The CSW service account needs:

| Permission | Where to grant |
|---|---|
| `web_service_admin` role (Cisco-documented requirement for Scripted REST APIs) | **User Administration → Users → {csw_integration_user} → Roles** — see [`00-official-references.md`](./00-official-references.md) and [`02-prerequisites.md`](./02-prerequisites.md) |
| Read on `cmdb_ci_appl` | Included in `cmdb_read` — see [`02-prerequisites.md`](./02-prerequisites.md) |
| Read on `cmdb_ci_network_adapter` | Included in `cmdb_read` |
| Read on `sys_user` (for `_ownerEmail`) | `itil` role includes this; or add a specific read ACL |

> **Scope note.** If the Scripted REST API is created in a **scoped application** (e.g., `x_cisco_csw`), the script runs in that scope. If it reads tables outside that scope (like `cmdb_ci_appl` in the global scope), set the application's **Cross-scope access** policy to `Allow` for those tables, or create the API in the global application scope.

---

## Step 5 — Point the CSW connector at the Scripted REST API

The CSW connector's "table" configuration points to a Table API path by default. To use a Scripted REST API you configure the **base URL** and **table name** fields to target your resource:

1. In the CSW connector configuration (see [`03-configuration.md`](./03-configuration.md)):
   - Set the **ServiceNow instance URL** to `https://YOURINSTANCE.service-now.com`.
   - For the **table name** field, enter the Scripted REST API resource path relative to the instance root:  
     `api/x_cisco_csw/csw_cmdb_ip_export/v1/records`
2. On the attribute-selection step, select **`ip_address`** as the IP key.
3. Select the additional attribute fields you want as CSW labels (`app_id`, `risk_tier`, `compliance_scope`, `asset_criticality`, `owner_email`, `environment`, etc.).
4. Set **Additional REST API url params** to:
   ```
   sysparm_limit=500
   ```
   The script handles `sysparm_limit` and `sysparm_offset` natively, so the connector's standard pagination will work.
5. **Review & apply**, then verify labels on a known IP per [`03-configuration.md`](./03-configuration.md) Step 6.

---

## Pagination behavior

The script reads `sysparm_limit` and `sysparm_offset` from query parameters, matching the Table API contract the CSW connector expects for large datasets:

| Query parameter | Default | Behavior |
|---|---|---|
| `sysparm_limit` | 500 | Max rows per call |
| `sysparm_offset` | 0 | Starting row (0-indexed) |

For large CMDB estates, set `sysparm_limit` to a value your ServiceNow instance can handle without hitting the transaction quota (typically 500–1000 rows per call is safe).

---

## Adapting this script to your schema

| What to change | Where in the script |
|---|---|
| Source CI table | `new GlideRecord('cmdb_ci_appl')` → replace with your table |
| Application ID field | `app_id:` line → point to your `number` / `u_app_id` / `u_application_id` field |
| Attribute field names | The `ciGr.getValue(...)` / `ciGr.getDisplayValue(...)` calls |
| Risk tier logic | `_riskTier()` — adjust thresholds and field names |
| Owner email walk | `_ownerEmail()` — change if ownership is stored differently |
| Compliance scope keywords | `_complianceScope()` — match your business service naming convention |
| Asset criticality weights | `_assetCriticality()` — tune score thresholds and classification values |

---

## Caveats

- **Performance.** The inner `while (naGr.next())` loop runs one GlideRecord query per CI. For estates with tens of thousands of CIs, consider using `GlideAggregate` or a pre-built work table. Monitor the ServiceNow transaction log for quota warnings.
- **Scope.** If you need fields from tables in a restricted scope, the API must be created in a scope with cross-scope read access configured.
- **Versioning.** The API endpoint path includes `/v1/`. If the schema changes, create a `/v2/` resource rather than editing the existing one — CSW connectors that read `/v1/` will continue to work until you migrate them.
- **Error handling.** Add try/catch blocks around GlideRecord queries in production; a malformed record should log an error and `continue` rather than aborting the entire response.

---

**Related pages:** [08 — Database View (no IP)](./08-no-ip-create-a-view.md) ·
[03 — Configuration](./03-configuration.md) ·
[02 — Prerequisites](./02-prerequisites.md) ·
[06 — Limitations & FAQ](./06-limitations-and-faq.md)
