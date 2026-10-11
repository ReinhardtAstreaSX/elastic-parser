Option A: Dev Tools Console (recommended)
Open Kibana, then Dev Tools > Console.
Paste the whole block, starting with PUT _ingest/pipeline/logs-cef.log@custom and ending at the closing }.
Run it. You should get {"acknowledged": true}.
Check it with GET _ingest/pipeline/logs-cef.log@custom.

Option B: Kibana UI
Go to Stack Management > Ingest Pipelines > Create pipeline > New pipeline.
Name it logs-cef.log@custom.
Click Import processors, or Load JSON in some versions. Paste only the body, with no PUT line:
json
{
  "processors": [ ...the three processors... ],
  "on_failure": [ ...the two on_failure processors... ]
}
Save. If the UI rejects the paste, the usual cause is the PUT line or a missing outer { }. The UI doesn't accept the description key in some versions, so enter the description in its own field.



PUT _ingest/pipeline/logs-cef.log@custom
{
  "description": "Reroute CyberArk PTA and Vault CEF events to their integration data streams",
  "processors": [
    {
      "set": {
        "field": "event.original_vendor_product",
        "copy_from": "cef.device.product",
        "ignore_empty_value": true,
        "ignore_failure": true,
        "tag": "debug_copy_product"
      }
    },
    {
      "reroute": {
        "tag": "reroute_cyberark_pta",
        "if": "def v = ctx.cef?.device?.vendor ?: ctx.observer?.vendor; def p = ctx.cef?.device?.product ?: ctx.observer?.product; if (v == null || p == null) { return false; } return v.toString().replaceAll('[^A-Za-z0-9]','').equalsIgnoreCase('cyberark') && p.toString().trim().equalsIgnoreCase('PTA');",
        "dataset": "cyberark_pta.events",
        "namespace": "{{data_stream.namespace}}"
      }
    },
    {
      "reroute": {
        "tag": "reroute_cyberark_vault",
        "if": "def v = ctx.cef?.device?.vendor ?: ctx.observer?.vendor; def p = ctx.cef?.device?.product ?: ctx.observer?.product; if (v == null || p == null) { return false; } return v.toString().replaceAll('[^A-Za-z0-9]','').equalsIgnoreCase('cyberark') && p.toString().trim().equalsIgnoreCase('Vault');",
        "dataset": "cyberarkpas.audit",
        "namespace": "{{data_stream.namespace}}"
      }
    }
  ],
  "on_failure": [
    { "append": { "field": "tags", "value": "cef_custom_pipeline_error" } },
    { "set": { "field": "error.message", "value": "{{{ _ingest.on_failure_message }}}" } }
  ]
}
