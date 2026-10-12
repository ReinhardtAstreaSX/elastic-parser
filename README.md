# 1. What data stream and template is the CEF data actually in?
GET _data_stream/logs-cef.log-*?filter_path=data_streams.name,data_streams.template,data_streams.ilm_policy

# 2. Which pipelines do its backing indices use? (Expect logs-cef.log-<version> and .fleet_final_pipeline-1)
GET logs-cef.log-*/_settings?filter_path=*.settings.index.default_pipeline,*.settings.index.final_pipeline

# 3. Does the installed integration pipeline call @custom, and does yours exist?
GET _ingest/pipeline/logs-cef.log-*?filter_path=*.processors.pipeline
GET _ingest/pipeline/logs-cef.log@custom

# 4. Is your pipeline executing? (count should rise while events arrive)
GET _nodes/stats/ingest?filter_path=nodes.*.ingest.pipelines.logs-cef*

# 1. Where do the CyberArk events actually land?
GET logs-*/_search
{
  "size": 1,
  "_source": ["data_stream.*", "event.dataset", "cef.device.vendor", "cef.device.product"],
  "query": { "term": { "cef.device.vendor": "CyberArk" } }
}

# 2. Which pipelines does that backing index use?
GET <backing-index-or-data-stream>/_settings?filter_path=*.settings.index.default_pipeline,*.settings.index.final_pipeline

# 3. Does the integration pipeline call @custom?
GET _ingest/pipeline/logs-cef.log-*?filter_path=*.processors.pipeline