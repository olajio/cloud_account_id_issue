You can hand it over like this:

### Summary of Investigation – `cloud.account.id` Ingested as Floating Point / Scientific Notation

We recently observed some values of `cloud.account.id` appearing in Discover in floating point/scientific notation format (example: `3.6973397917E10`) instead of the expected AWS account ID string format (example: `036973397917`).

The following items were reviewed as part of the investigation:

---

### Data Reviewed

#### Discover Export Data

Reviewed:

* `us_east_1.csv`
* `us_east_2.csv`

These files contained the actual ingested documents showing both:

* Correct values:

  * `036973397917`
* Incorrect/scientific notation values:

  * `3.6973397917E10`

---

### Data Streams Reviewed

Reviewed settings for:

* `filebeat-8.19.5-mt-qa`
* `filebeat-8.19.5-mt-uat`

Files reviewed:

* `filebeat-8.19.5-mt-qa_data_stream_settings`
* `filebeat-8.19.5-mt-uat_data_stream_settings`

No settings were identified that would cast or transform `cloud.account.id` into a numeric datatype.

---

### Index Templates Reviewed

Reviewed:

* `filebeat-8.19.5-mt-qa_index_template`
* `filebeat-8.19.5-mt-uat_index_template`

Also reviewed component template:

* `filebeat_mt_default_component`

Findings:

* `cloud.account.id` is mapped as a `keyword`
* No dynamic template, runtime field, script, or field formatter was identified that would convert the field into a float/double/scientific notation representation

---

### Ingest Pipelines Reviewed

Reviewed all related pipelines from:

* `pipelines.txt`

Pipelines reviewed include:

* `filebeat_mt_default`
* `enrich_cloud_name`
* `accounts_lookup`
* `calculate_lag`
* `enrich_user_type`

Analysis of pipeline behavior:

* `cloud.account.id` is only used as an enrich lookup key
* No `convert`, `script`, `set`, `rename`, `foreach`, or painless processor modifies the value
* No processor converts the field into numeric format
* The enrich pipeline only enriches `cloud.account.name`

Example:

* `accounts_lookup` pipeline uses:

  * `field: cloud.account.id`
  * `target_field: cloud.account`

This only performs lookup/enrichment and does not rewrite the original ID value.

---

### Enrich Policies Reviewed

Reviewed:

* `accounts-policy-v1`
* `user-type`

Findings:

* `accounts-policy-v1` matches:

  * source index: `accounts`
  * match field: `id`
  * enrich field: `name`
* No enrichment field overwrites or modifies `cloud.account.id`

---

## Analysis

At this time, no evidence was found in:

* data stream settings
* index templates
* component templates
* ingest pipelines
* enrich policies

that would explain conversion of:

* `036973397917`
  into:
* `3.6973397917E10`

Since the field is mapped as `keyword`, Elasticsearch will store whatever representation it receives at ingest time.

This strongly suggests the value is already being serialized or emitted in scientific notation before it reaches Elasticsearch ingest pipelines.

---

## Pattern Observed

The issue appears to affect only a subset of ingested data and is not globally consistent.

Observed behavior:

* Some documents contain the correct AWS account ID string
* Other documents contain scientific notation representation

Most impacted data currently appears in:

* `.ds-filebeat-8.19.5-mt-uat-*`

A strong correlation was observed by Kubernetes/EC2 node hostname.

Some nodes consistently ingest:

* correct string values

while others consistently ingest:

* scientific notation values

This suggests:

* node-specific behavior
* source-side serialization differences
* or a recent change in upstream processing/runtime/library behavior

rather than an Elasticsearch-side mapping issue.

---

## Conclusion

Current findings indicate this is most likely occurring upstream of Elasticsearch ingest.

The value appears to already be transformed into numeric/scientific notation before entering the Elasticsearch ingest pipeline.

No Elasticsearch ingest pipeline, enrich policy, index template, or mapping reviewed during this investigation appears responsible for the conversion.

---

## Recommended Areas for Further Investigation

Since this behavior started recently, focus should be on identifying recent upstream changes affecting only a subset of nodes/services.

Recommended areas to investigate:

### 1. Filebeat / Agent Changes

Check for:

* recent Filebeat upgrades
* custom processors
* decode_json_fields behavior
* script processors
* ECS transformations
* autodiscover changes

Especially compare:

* affected nodes
  vs
* unaffected nodes

---

### 2. Source Application / Log Generation

Investigate whether the original application logs/events already contain:

* `3.6973397917E10`

before Filebeat ingestion.

This is currently one of the strongest possibilities.

---

### 3. Serialization / JSON Encoding Libraries

Look for recent changes involving:

* Golang JSON marshaling
* Python serialization
* Java/Jackson serialization
* CSV/Excel export handling
* numeric coercion
* scientific notation formatting

Potential scenario:

* account ID treated as numeric instead of string
* leading zero dropped
* value auto-converted to exponential notation

---

### 4. Node-Specific Runtime Differences

Since behavior correlates strongly with specific nodes:

* compare Filebeat configs
* compare application versions
* compare container images
* compare environment variables
* compare runtime libraries/JDK/Python/Golang versions

between:

* impacted nodes
  and
* healthy nodes

---

### 5. Intermediate Processing Layers

Investigate any recent changes in:

* Logstash
* Kafka consumers/producers
* FluentBit/Fluentd
* Lambda transformations
* custom ETL processors
* stream processing logic

especially anything that may deserialize/re-serialize JSON payloads.

---

### 6. Raw Event Validation

Recommended next step:
Capture and compare raw events at multiple stages:

1. original application log
2. Filebeat harvested event
3. pre-ingest Elasticsearch payload
4. indexed document

This should identify the exact stage where the conversion first occurs.
