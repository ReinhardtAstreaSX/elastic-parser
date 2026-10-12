Yes, this is the better route. Instead of rerouting, add processors to `logs-cef.log@custom` that map the Vault CEF fields to the `cyberarkpas.audit.*` and ECS fields. The events stay in `logs-cef.log-*`, so there is no write to a second data stream. That removes the permission problem from last time.

One consequence is that Elastic's prebuilt CyberArk detection rules and dashboards read `logs-cyberarkpas.audit*`. They will not see your events until you add `logs-cef.log-*` to a copy of each rule's index patterns.

## What the Vault CEF looks like

CyberArk's Vault CEF has a fixed layout. A sample from the web:

`CEF:0|Cyber-Ark|Vault|VERSION|106|Update File Category|5|act="Update File Category" suser=USERNAME fname=Root\...xml shost=HOSTIP duser= externalId= app= reason= cs1Label="Affected User Name" cs2Label="Safe Name" cs3Label="Device Type" cs4Label="Database" cs5Label="Other info" cn1Label="Request Id" cn2Label="Ticket Id" msg=`[[1]]

Elastic's integration expects the same data as JSON. Its fields are `Issuer`, `Action`, `SourceUser`, `TargetUser`, `Safe`, `File`, `Station`, `MessageID`, `Desc`, `Severity`, `Reason` and others[]. The integration docs list the resulting fields: `cyberarkpas.audit.action`, `issuer`, `safe`, `station`, `target_user`, `message_id` and more[].[2][3]

## Field mapping

| CEF source | Target field |
|---|---|
| Device event class ID | `cyberarkpas.audit.message_id`, `event.code` |
| CEF name | `cyberarkpas.audit.desc`, `cyberarkpas.audit.message` |
| `act` | `cyberarkpas.audit.action` |
| `suser` | `cyberarkpas.audit.issuer`, `user.name` |
| `fname` | `cyberarkpas.audit.file` |
| `shost` (usually the client IP) | `cyberarkpas.audit.station`, `source.ip` |
| `cs1` (Affected User Name) | `cyberarkpas.audit.target_user` |
| `cs2` (Safe Name) | `cyberarkpas.audit.safe` |
| `reason` | `cyberarkpas.audit.reason` |
| `cn1` (Request Id) | `cyberarkpas.audit.ca_properties.sequence_id`, or keep it in `other` |

## Pipelines

I haven't seen your field names. The `cef.extensions.*` names below follow the usual `decode_cef` naming, such as `deviceAction` for `act`. Check them against one real document first. Each `set` uses `ignore_empty_value`, so missing fields are skipped.

```json
PUT _ingest/pipeline/cef-cyberark-vault-map
{
  "description": "Map CyberArk Vault CEF to cyberarkpas.audit.* + ECS (in place, no reroute)",
  "processors": [
    { "set": { "tag": "cv_message_id", "field": "cyberarkpas.audit.message_id", "copy_from": "cef.device.event_class_id", "ignore_empty_value": true } },
    { "set": { "tag": "cv_event_code", "field": "event.code", "copy_from": "cef.device.event_class_id", "ignore_empty_value": true } },
    { "set": { "tag": "cv_desc", "field": "cyberarkpas.audit.desc", "copy_from": "cef.name", "ignore_empty_value": true } },
    { "set": { "tag": "cv_message", "field": "cyberarkpas.audit.message", "copy_from": "cef.name", "ignore_empty_value": true } },
    { "set": { "tag": "cv_action", "field": "cyberarkpas.audit.action", "copy_from": "cef.extensions.deviceAction", "ignore_empty_value": true } },
    { "set": { "tag": "cv_issuer", "field": "cyberarkpas.audit.issuer", "copy_from": "cef.extensions.sourceUserName", "ignore_empty_value": true } },
    { "set": { "tag": "cv_user", "field": "user.name", "copy_from": "cef.extensions.sourceUserName", "ignore_empty_value": true } },
    { "set": { "tag": "cv_file", "field": "cyberarkpas.audit.file", "copy_from": "cef.extensions.fileName", "ignore_empty_value": true } },
    { "set": { "tag": "cv_station", "field": "cyberarkpas.audit.station", "copy_from": "cef.extensions.sourceHostName", "ignore_empty_value": true } },
    { "set": { "tag": "cv_target_user", "field": "cyberarkpas.audit.target_user", "copy_from": "cef.extensions.deviceCustomString1", "ignore_empty_value": true } },
    { "set": { "tag": "cv_safe", "field": "cyberarkpas.audit.safe", "copy_from": "cef.extensions.deviceCustomString2", "ignore_empty_value": true } },
    { "set": { "tag": "cv_reason", "field": "cyberarkpas.audit.reason", "copy_from": "cef.extensions.reason", "ignore_empty_value": true } },
    { "set": { "tag": "cv_vendor", "field": "cyberarkpas.audit.vendor", "value": "Cyber-Ark" } },
    { "set": { "tag": "cv_product", "field": "cyberarkpas.audit.product", "value": "Vault" } },
    { "set": { "tag": "cv_version", "field": "cyberarkpas.audit.version", "copy_from": "cef.device.version", "ignore_empty_value": true } },
    { "convert": { "tag": "cv_src_ip", "field": "cyberarkpas.audit.station", "target_field": "source.ip", "type": "ip", "ignore_missing": true, "ignore_failure": true } },
    { "append": { "tag": "cv_related_ip", "field": "related.ip", "value": "{{{source.ip}}}", "if": "ctx.source?.ip != null", "allow_duplicates": false } },
    { "append": { "tag": "cv_related_user", "field": "related.user", "value": "{{{user.name}}}", "if": "ctx.user?.name != null", "allow_duplicates": false } },
    { "append": { "tag": "cv_tag", "field": "tags", "value": "cyberarkpas-audit-mapped", "allow_duplicates": false } }
  ],
  "on_failure": [
    { "append": { "field": "tags", "value": "cyberark_vault_map_error" } },
    { "set": { "field": "error.message", "value": "{{{_ingest.on_failure_message}}}" } }
  ]
}

PUT _ingest/pipeline/logs-cef.log@custom
{
  "description": "CEF custom hook: map CyberArk Vault in place",
  "processors": [
    {
      "pipeline": {
        "tag": "call_cyberark_vault_map",
        "name": "cef-cyberark-vault-map",
        "if": "def v = ctx.cef?.device?.vendor ?: ctx.observer?.vendor; def p = ctx.cef?.device?.product ?: ctx.observer?.product; if (v == null || p == null) { return false; } return v.toString().replaceAll('[^A-Za-z0-9]','').equalsIgnoreCase('cyberark') && p.toString().trim().equalsIgnoreCase('Vault');"
      }
    }
  ],
  "on_failure": [
    { "append": { "field": "tags", "value": "cef_custom_pipeline_error" } },
    { "set": { "field": "error.message", "value": "{{{_ingest.on_failure_message}}}" } }
  ]
}
```

Splitting the mapping into its own pipeline keeps the `@custom` hook small. You can also reuse the mapping pipeline elsewhere.

## Test it

1. Run `POST _ingest/pipeline/logs-cef.log@custom/_simulate` with one real Vault CEF document, and make sure the mapped fields appear.
2. In production, query `tags: "cyberarkpas-audit-mapped"` in `logs-cef.log-*`. Then check for `cyberark_vault_map_error` and `cef_custom_pipeline_error`.
3. Open one document and compare it with the integration's example event. Look at `event.code`, `cyberarkpas.audit.action`, `user.name` and `source.ip`.
4. Add the `event.category`, `event.type` and `event.action` values you need. The native pipeline derives these from the audit code, and this one doesn't. You can add a lookup by `event.code` using a `script` or `enrich` processor.

## Alternatives

| Approach | Pros | Cons |
|---|---|---|
| In-place mapping (above) | No permissions change, quick | Rules and dashboards need your index pattern added |
| Fix reroute permissions (Custom Logs workaround) | Uses the integration's own dashboards | Broad API key, and CEF still needs mapping |
| Switch the Vault to the Elastic JSON XSL | Native parsing, rules work as shipped | Needs a change on the Vault side and a new input |
| Rebuild JSON and call the integration pipeline | Reuses its logic | Brittle: the pipeline name is versioned and expects its own syslog and JSON input, so I would avoid it |

Native JSON is the strongest long-term option. Use in-place mapping if you can't change the Vault or open another port.

Paste one sanitised Vault CEF document from Discover, with every `cef.*` field visible. I'll correct the source field names, which is the part most likely to be wrong.

Citations:
[1] CyberArk Log format coming into Splunk https://community.splunk.com/t5/All-Apps-and-Add-ons/CyberArk-Log-format-coming-into-Splunk/m-p/304445
[2] [CyberArk PAS] Add support for monitoring data #11320 - GitHub https://github.com/elastic/integrations/issues/11320
[3] CyberArk Privileged Access Security | Elastic integrations https://www.elastic.co/docs/reference/integrations/cyberarkpas
[4] Security Information and Event Management applications | Idira Docs https://docs.cyberark.com/pam-self-hosted/latest/en/content/pasimp/integrating-with-siem-applications.htm?Highlight=hostname%20qradar
[5] Security Information and Event Management (SIEM) Applications https://docs.cyberark.com/pam-self-hosted/latest/en/content/pasimp/dv-integrating-with-siem-applications.htm
[6] CyberArk PAM Self-Hosted - Cortex Marketplace https://cortex.marketplace.pan.dev/marketplace/details/CyberArkEPV/
[7] V 2.0 CyberArk Vault Audit Events - LogRhythm Documentation https://docs.logrhythm.com/devices/docs/syslog-cyberark-v-2-0-cyberark-vault-audit-events
[8] Monitor the Vault in SIEM Applications Using Syslog - CyberArk Docs https://docs.cyberark.com/pam-self-hosted/latest/en/content/pasimp/monitoring-the-vault-using-syslog.htm
[9] Vault Audit Action Codes - CyberArk Docs https://docs.cyberark.com/pam-self-hosted/latest/en/content/pasref/vault%20audit%20action%20codes.htm
[10] Audit log structure | CyberArk Docs https://docs.cyberark.com/secrets-manager-sh/13.6/en/content/operations/services/audit/dap-auditlog-structure.htm
[11] SIEM Integration : r/CyberARk - Reddit https://www.reddit.com/r/CyberARk/comments/67vsnx/siem_integration/
[12] Vault Syslog CEF Format Parameter Inconsistency with Arcsight https://community.cyberark.com/s/article/Vault-Syslog-CEF-Format-Parameter-Inconsistency-with-Arcsight
[13] How to set up rsyslog to handle Vault Syslog - CyberArk https://community.cyberark.com/s/article/How-to-set-up-rsyslog-to-handle-Vault-Syslog
[14] How to integrate CyberArk with SIEM platforms - LinkedIn https://www.linkedin.com/posts/vishalmaurya121_cyberark-siem-integration-activity-7338941716655181824-TPYF
