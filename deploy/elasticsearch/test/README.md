# Elasticsearch Index Settings

## Working in progress

- *parse: json*

    Optional: Specify whether to forward structured JSON log entries as JSON objects in the structured field. The log entry must contain valid structured JSON; otherwise, OpenShift Logging removes the structured field and instead sends the log entry to the default index, __app-00000x__.

## Kibana

This is a clipboard about Kibana version 6.8 on Openshift configured by the following operators:

- Red Hat OpenShift Logging 5.4.0
- OpenShift Elasticsearch Operator 5.4.0


### Kibana usefull commands

```bash
# print all the indeces
GET /_cat/indices?v

# How much memory is used per index?0
GET /_cat/indices?v&h=i,tm&s=tm:desc

# Which index has the largest number of documents?
GET /_cat/indices?v&s=docs.count:desc

# create an index
PUT /type_index_name
{
  "aliases": {
    "logs_write": {}
  }
}

# delete an index
DELETE /type_index_name

# getting the index mapping
GET /app-test-logging-*/_mapping/_doc?&include_type_name=true
GET /app-000001/_mapping/_doc?&include_type_name=true
```

#### Getting the field mapping'

https://www.elastic.co/guide/en/elasticsearch/reference/6.8/indices-get-field-mapping.html

#### Getting the field's conflits list

```bash
GET app-*/_mapping/field/conflictedFieldname
```




### Kibana Structured fields list

It follows the default preset of the _structured_ fields list like generated by the logforwarder json parsing engine:

    structured.@timestamp
    structured.application
    structured.error.message
    structured.error.stacktrace
    structured.error.type
    structured.level
    structured.location.class
    structured.location.file
    structured.location.line_number
    structured.location.method
    structured.logger_name
    structured.message
    structured.thread_name

### elastichsearch indexes template model

```json
{
  "mappings": {
    "_doc": {
      "dynamic_templates": [
        {
          "longs_as_strings": {
            "match_mapping_type": "string",
            "match":   "long_*",
            "unmatch": "*_text",
            "mapping": {
              "type": "long"
            }
          }
        }
      ]
    }
  }
}
```

### Security Exception

Referring to:

- https://access.redhat.com/solutions/5538221

__date__

```"format": "yyyy-MM-dd HH:mm:ss,SSSZ||yyyy-MM-dd'T'HH:mm:ss.SSSZ||yyyy-MM-dd'T'HH:mm:ss.SSSSSSZ||yyyy-MM-dd'T'HH:mm:ssZ||dateOptionalTime"```

## Elasticsearch

### Using policies to manage index rollover

The rollover action enables you to automatically roll over to a new index based on the index size, document count, or age. When a rollover is triggered, a new index is created, the write alias is updated to point to the new index, and all subsequent updates are written to the new index.

Rolling over to a new index based on size, document count, or age is preferable to time-based rollovers. Rolling over at an arbitrary time often results in many small indices, which can have a negative impact on performance and resource usage.

> IMPORTANT:
  New indices created via rollover will not automatically inherit the policy used by the old index, and will not use any policy by default. Therefore, it is highly recommended to apply the policy via index template, including a Rollover alias setting, for your indices which specifies the policy you wish to use for each new index.


#### Set the indexes rollover by Elasticsearch Operator

You can set the following limits:

* MaxSize: maxSize
* MaxDocs: maxDoc
* MaxAge:  maxAge

Elasticsearch Operator Snippet:

```yaml
apiVersion: logging.openshift.io/v1
kind: Elasticsearch
...
spec:
  indexManagement:
    mappings:
      - aliases:
          - app
          - logs.app
        name: app
        policyRef: app-policy
      - aliases:
          - infra
          - logs.infra
        name: infra
        policyRef: infra-policy
      - aliases:
          - audit
          - logs.audit
        name: audit
        policyRef: audit-policy
    policies:
      - name: app-policy
        phases:
          delete:
            minAge: 1d
            pruneNamespacesInterval: 30m
          hot:
            actions:
              rollover:
                maxAge: 1h
        pollInterval: 15m
      - name: infra-policy
        phases:
          delete:
            minAge: 7d
            pruneNamespacesInterval: 30m
          hot:
            actions:
              rollover:
                maxAge: 8h
        pollInterval: 15m
      - name: audit-policy
        phases:
          delete:
            minAge: 7d
            pruneNamespacesInterval: 30m
          hot:
            actions:
              rollover:
                maxAge: 8h
        pollInterval: 15m
```

#### Set/Get the ILM Policy (it doesn't work on Openshift)

Create a new ILM Policy

```
./ilm_policy.sh
```

The output will be likely as follows:

```{"_index":"ilm","_type":"policy","_id":"ilm_dedalus_policy","_version":1,"result":"created","_shards":{"total":2,"successful":1,"failed":0},"_seq_no":0,"_primary_term":1}```

Get the ILM Policy

```bash
GET /ilm/policy/ilm_dedalus_policy
```

### Making the tasks automatic

#### Get the elastricsearch running POD

```bash
es_pod=$(oc -n openshift-logging get pods -l component=elasticsearch --no-headers | head -1 | cut -d" " -f1)
```

#### Creating index templates

* by the _curl_ command:

```bash
curl -XPUT -k -H "X-Forwarded-For: 127.0.0.1" -H "Authorization: Bearer $token" https://es-openshift-logging.apps.sno-cluster.okd-sno.dedalus.red.com/_template/dedalus_es_template -H 'Content-Type: application/json' -d'
{
    "index_patterns": ["app-*"],
    "settings": {
      "number_of_shards": 2
    },
    "version": 1,
    "mappings": {
        "_doc": {
            "_source": { "enabled": true },
            "properties": {
              "structured.@timestamp": {
                  "type": "date",
                  "format": "strict_date_optional_time"
              },
              "created_at": {
                "type": "date",
                "format": "strict_date_hour_minute"
              }
            }
        }
    }
}'
```

* by elasticsearch pod:

```bash
oc exec -n openshift-logging -c elasticsearch ${es_pod} -- es_util --query=_template/dedalus_es_template -XPUT -d'
{
    "index_patterns": ["app-*"],
    "settings": {
      "number_of_shards": 2
    },
    "version": 1,
    "mappings": {
        "_doc": {
            "_source": { "enabled": true },
            "properties": {
              "structured.@timestamp": {
                  "type": "date",
                  "format": "strict_date_optional_time"
              },
              "created_at": {
                "type": "date",
                "format": "strict_date_hour_minute"
              }
            }
        }
    }
}'

# Getting a specific Template
oc exec -n openshift-logging -c elasticsearch ${es_pod} -- es_util --query=_template/dedalus_es_template

# Delete Template
oc exec -n openshift-logging -c elasticsearch ${es_pod} -- es_util --query=_template/dedalus_es_template -XDELETE

# Getting All Templates
oc exec -n openshift-logging -c elasticsearch ${es_pod} -- es_util --query=_template | jq "[.]"
```

### Index pattern

It follows the commands to create the Index Pattern by Elasticsearch API:

>> working in progress

```bash
# create the default index pattern
oc exec -n openshift-logging -c elasticsearch elasticsearch-cdm-owd8rzjq-1-5854579bd6-46mwp -- es_util --query=api/index_patterns/default -XPOST   -d'
{
    "attributes": {
        "title": "app-test-logging-*",
        "timeFieldName": "@timestamp"
    }
}'

# set the advanced settings on kibana
oc exec -n openshift-logging -c elasticsearch elasticsearch-cdm-owd8rzjq-1-5854579bd6-46mwp -- es_util --query=api/kibana/settings -XPOST   -d'
{
    "changes": {
        "defaultIndex": "default-app-index"
    }
}'
```