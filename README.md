## The pipelines

The Painless conditions don't call `.toString()`. They check `instanceof String`, so a null or non-string value returns false instead of raising a Java error. I've kept the mapping in two sub-pipelines, so you can fix one without touching the other.

```json
PUT _ingest/pipeline/cef-cyberark-pta-map
{
  "description": "Map CyberArk PTA CEF to cyberark_pta ECS shape (in place)",
  "processors": [
    { "set": { "tag": "pta_code", "field": "event.code", "copy_from": "cef.device.event_class_id", "ignore_empty_value": true } },
    { "set": { "tag": "pta_type", "field": "cyberark_pta.log.event_type", "copy_from": "cef.device.event_class_id", "ignore_empty_value": true } },
    { "set": { "tag": "pta_reason", "field": "event.reason", "copy_from": "cef.name", "ignore_empty_value": true } },
    { "set": { "tag": "pta_sev", "field": "event.severity", "copy_from": "cef.severity", "ignore_empty_value": true } },
    { "convert": { "tag": "pta_sev_long", "field": "event.severity", "type": "long", "ignore_missing": true, "ignore_failure": true } },
    {
      "script": {
        "tag": "pta_labels",
        "lang": "painless",
        "on_failure": [ { "append": { "field": "tags", "value": "cyberark_pta_map_error" } } ],
        "source": "def ext = ctx.cef?.extensions; if (ext == null) { return; } for (int i = 1; i <= 6; i++) { def lbl = ext.get('deviceCustomString' + i + 'Label'); def val = ext.get('deviceCustomString' + i); if (!(lbl instanceof String) || !(val instanceof String)) { continue; } if (lbl.equalsIgnoreCase('EventID')) { if (ctx.event == null) { ctx.event = new HashMap(); } ctx.event.id = val; } else if (lbl.equalsIgnoreCase('PTAlink')) { if (ctx.event == null) { ctx.event = new HashMap(); } ctx.event.reference = val; } }"
      }
    },
    { "dissect": { "tag": "pta_suser", "field": "cef.extensions.sourceUserName", "pattern": "%{source.user.name}@%{source.user.domain}", "ignore_missing": true, "ignore_failure": true } },
    { "dissect": { "tag": "pta_duser", "field": "cef.extensions.destinationUserName", "pattern": "%{destination.user.name}@%{destination.user.domain}", "ignore_missing": true, "ignore_failure": true } },
    { "set": { "tag": "pta_src_email", "field": "source.user.email", "copy_from": "cef.extensions.sourceUserName", "if": "ctx.source?.user?.domain != null", "ignore_failure": true } },
    { "set": { "tag": "pta_dst_email", "field": "destination.user.email", "copy_from": "cef.extensions.destinationUserName", "if": "ctx.destination?.user?.domain != null", "ignore_failure": true } },
    { "append": { "field": "tags", "value": "cyberark_pta_cef_mapped", "allow_duplicates": false } }
  ],
  "on_failure": [
    { "append": { "field": "tags", "value": "cef_custom_pipeline_error" } },
    { "set": { "field": "error.message", "value": "{{{ _ingest.on_failure_message }}}" } }
  ]
}
```

```json
PUT _ingest/pipeline/cef-cyberark-vault-map
{
  "description": "Best-effort map of CyberArk Vault CEF to cyberarkpas.audit / ECS (in place)",
  "processors": [
    { "set": { "tag": "v_code", "field": "event.code", "copy_from": "cef.device.event_class_id", "ignore_empty_value": true } },
    { "set": { "tag": "v_msgid", "field": "cyberarkpas.audit.message_id", "copy_from": "cef.device.event_class_id", "ignore_empty_value": true } },
    { "set": { "tag": "v_desc", "field": "cyberarkpas.audit.desc", "copy_from": "cef.name", "ignore_empty_value": true } },
    { "set": { "tag": "v_msg", "field": "cyberarkpas.audit.message", "copy_from": "cef.name", "ignore_empty_value": true } },
    { "set": { "tag": "v_ver", "field": "cyberarkpas.audit.version", "copy_from": "cef.device.version", "ignore_empty_value": true } },
    { "set": { "tag": "v_vendor", "field": "cyberarkpas.audit.vendor", "copy_from": "cef.device.vendor", "ignore_empty_value": true } },
    { "set": { "tag": "v_product", "field": "cyberarkpas.audit.product", "copy_from": "cef.device.product", "ignore_empty_value": true } },
    { "set": { "tag": "v_action", "field": "cyberarkpas.audit.action", "copy_from": "cef.extensions.deviceAction", "ignore_empty_value": true } },
    { "set": { "tag": "v_issuer", "field": "cyberarkpas.audit.issuer", "copy_from": "cef.extensions.sourceUserName", "ignore_empty_value": true } },
    { "set": { "tag": "v_srcuser", "field": "cyberarkpas.audit.source_user", "copy_from": "cef.extensions.sourceUserName", "ignore_empty_value": true } },
    { "set": { "tag": "v_user", "field": "user.name", "copy_from": "cef.extensions.sourceUserName", "override": false, "ignore_empty_value": true } },
    { "set": { "tag": "v_tgtuser", "field": "cyberarkpas.audit.target_user", "copy_from": "cef.extensions.destinationUserName", "ignore_empty_value": true } },
    { "set": { "tag": "v_file", "field": "cyberarkpas.audit.file", "copy_from": "cef.extensions.filename", "ignore_empty_value": true } },
    { "set": { "tag": "v_station", "field": "cyberarkpas.audit.station", "copy_from": "cef.extensions.sourceHostName", "ignore_empty_value": true } },
    { "set": { "tag": "v_reason", "field": "cyberarkpas.audit.reason", "copy_from": "cef.extensions.reason", "ignore_empty_value": true } },
    { "set": { "tag": "v_action_ecs", "field": "event.action", "value": "authentication_success", "if": "ctx.event?.code == '7'", "override": false } },
    {
      "script": {
        "tag": "v_labels",
        "lang": "painless",
        "on_failure": [ { "append": { "field": "tags", "value": "cyberark_vault_map_error" } } ],
        "source": "def ext = ctx.cef?.extensions; if (ext == null) { return; } Map known = ['affected user name': 'cyberarkpas.audit.target_user', 'safe name': 'cyberarkpas.audit.safe', 'device type': 'cyberarkpas.audit.ca_properties.device_type', 'database': 'cyberarkpas.audit.ca_properties.database']; boolean partial = false; for (int i = 1; i <= 6; i++) { def lbl = ext.get('deviceCustomString' + i + 'Label'); def val = ext.get('deviceCustomString' + i); if (!(lbl instanceof String) || !(val instanceof String) || val.isEmpty()) { continue; } String key = lbl.toLowerCase(); String target = known.get(key); if (target == null) { if (ctx.labels == null) { ctx.labels = new HashMap(); } ctx.labels.put('cyberark_' + key.replace(' ', '_'), val); partial = true; continue; } String[] parts = target.splitOnToken('.'); Map cur = ctx; for (int j = 0; j < parts.length - 1; j++) { def nxt = cur.get(parts[j]); if (!(nxt instanceof Map)) { nxt = new HashMap(); cur.put(parts[j], nxt); } cur = nxt; } cur.put(parts[parts.length - 1], val); } if (partial) { if (ctx.tags == null) { ctx.tags = new ArrayList(); } ctx.tags.add('cyberark_vault_cef_partial'); }"
      }
    },
    { "append": { "field": "tags", "value": "cyberark_vault_cef_mapped", "allow_duplicates": false } }
  ],
  "on_failure": [
    { "append": { "field": "tags", "value": "cef_custom_pipeline_error" } },
    { "set": { "field": "error.message", "value": "{{{ _ingest.on_failure_message }}}" } }
  ]
}
```

Replace the old reroute version of the hook with this one:

```json
PUT _ingest/pipeline/logs-cef.log@custom
{
  "description": "Dispatch CyberArk CEF events to in-place mapping pipelines",
  "processors": [
    {
      "pipeline": {
        "name": "cef-cyberark-pta-map",
        "tag": "dispatch_pta",
        "if": "def v = ctx.cef?.device?.vendor; def p = ctx.cef?.device?.product; return v instanceof String && p instanceof String && v.replace('-','').equalsIgnoreCase('cyberark') && p.equalsIgnoreCase('PTA');"
      }
    },
    {
      "pipeline": {
        "name": "cef-cyberark-vault-map",
        "tag": "dispatch_vault",
        "if": "def v = ctx.cef?.device?.vendor; def p = ctx.cef?.device?.product; return v instanceof String && p instanceof String && v.replace('-','').equalsIgnoreCase('cyberark') && p.equalsIgnoreCase('Vault');"
      }
    }
  ],
  "on_failure": [
    { "append": { "field": "tags", "value": "cef_custom_pipeline_error" } },
    { "set": { "field": "error.message", "value": "{{{ _ingest.on_failure_message }}}" } }
  ]
}
```

I used triple-brace Mustache in the `on_failure` blocks. Elastic's own changelog fixed a bug with the same pattern, so it avoids HTML escaping of the message[].

## Testing and next steps

1. Pull one real PTA and one real Vault document from `logs-cef.log-*`. Run `POST _ingest/pipeline/logs-cef.log@custom/_simulate` with each. Check that `cyberarkpas.audit.*` and `event.*` are populated, and that unmatched labels appear under `labels.*`.
2. Vault event codes map to `event.action`, `event.category` and `event.type` in the native pipeline. I only included code `7`, because it is the documented example, which maps to `authentication_success`[]. Build the full lookup from your own event codes, in the same style. [elastic](https://www.elastic.co/docs/reference/integrations/cyberarkpas)
3. Point your detections at `logs-cef.log-*` with a filter such as `tags: cyberark_vault_cef_mapped`. For PTA, use `tags: cyberark_pta_cef_mapped`.
4. Alert on `tags: cef_custom_pipeline_error`, `cyberark_vault_map_error` and `cyberark_vault_cef_partial`. The last one tells you which Vault labels you haven't mapped yet.

If you paste one sanitised Vault CEF event, I'll replace my remembered field layout with your actual one. I can then add the full `event.code` to `event.action`, `event.category` and `event.type` lookup.
