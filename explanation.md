**Finding:** I don’t see anything in the attached mapping/templates or ingest pipelines that would convert `cloud.account.id` into floating/scientific notation.

The pipeline only:

* expands `cloud.account.id` if it arrives as dotted field
* runs an enrich lookup against `cloud.account.id`
* writes `cloud.account.name`
* calculates lag / enriches user type
  It does **not** cast, script, convert, or set `cloud.account.id`. 

The enrich policy also only matches account `id` from the `accounts` index and enriches `name`; it does not overwrite the ID. 

**So yes — this can happen if the data was already ingested that way.** Since `cloud.account.id` is `keyword`, Elasticsearch will store the value as a keyword term, but it will not “repair” the original representation if the source sent it as `3.6973397917E10`.

Pattern found in the CSVs:

| File            |      Region |                  Good value |           Bad/scientific value | Impacted |
| --------------- | ----------: | --------------------------: | -----------------------------: | -------: |
| `us_east_1.csv` | `us-east-1` | `036973397917` = 2,066 rows | `3.6973397917E10` = 1,986 rows |     ~49% |
| `us_east_2.csv` | `us-east-2` | `036973397917` = 1,493 rows | `3.6973397917E10` = 3,763 rows |     ~72% |

The affected data is **only in UAT** from:

`.ds-filebeat-8.19.5-mt-uat-2026.05.06-000592`

The strongest pattern is by **Kubernetes node**. Some nodes consistently send the good 12-digit string, while others send scientific notation.

Most impacted nodes include:

`us-east-1` scientific:

* `ip-10-132-54-247.ec2.internal`
* `ip-10-132-100-204.ec2.internal`
* `ip-10-132-191-106.ec2.internal`
* `ip-10-132-55-104.ec2.internal`
* `ip-10-132-179-202.ec2.internal`

`us-east-2` scientific:

* `ip-10-107-126-179.us-east-2.compute.internal`
* `ip-10-107-45-240.us-east-2.compute.internal`
* `ip-10-107-62-58.us-east-2.compute.internal`
* `ip-10-107-59-179.us-east-2.compute.internal`

**Conclusion:** This looks like a source-side or shipper-side serialization issue on a subset of nodes, not an Elasticsearch mapping/template/pipeline issue. Somewhere before Elasticsearch, `036973397917` is being treated as a number, losing the leading zero, and then being serialized as scientific notation: `3.6973397917E10`.
