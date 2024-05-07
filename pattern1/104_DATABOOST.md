# DataBoost

https://cloud.google.com/spanner/docs/databoost/databoost-overview

When using DataBoost, dedicated machine resources are created and processed for one query.
You can execute high-load queries without affecting the Spanner Instance, so it can be used conveniently for batch job execution.
The available query must be [Partition Query](https://cloud.google.com/spanner/docs/reads?hl=en#read_data_in_parallel).
Roughly speaking, queries that can be executed with Partition Query are queries that can be completed in each Split.

This time, to easily use DataBoost, we will use DataBoost from BigQuery to execute [Federated Query](https://cloud.google.com/bigquery/docs/cloud-spanner-federated-queries).

```
properties1=$(echo '{"database":"projects/'$CLOUDSDK_CORE_PROJECT'/instances/'$CLOUDSDK_SPANNER_INSTANCE'/databases/'$DB1'", "useParallelism":true, "useDataBoost": true}')
connection_name1=$(printf "spanner_%s_%s" $CLOUDSDK_SPANNER_INSTANCE $DB1)
bq mk --project_id $CLOUDSDK_CORE_PROJECT --connection --connection_type='CLOUD_SPANNER' --location='us-central1' \
--properties=$properties1 $connection_name1
```

# JOIN

Queries executed against Spanner with DataBoost are executed as PartitionQuery.
Therefore, there are restrictions on the queries that can be executed.
In the case of pattern 1, Users and Orders are not interleaved, so if you try to JOIN, the Row will span another Partition, so it cannot be executed.

```
# This query cannot be executed
bq query --use_legacy_sql=false << EOS
SELECT * FROM EXTERNAL_QUERY(
  '$CLOUDSDK_CORE_PROJECT.us-central1.$connection_name1',
  'SELECT Users.UserID,Orders.OrderID FROM Users JOIN Orders ON Users.UserID = Orders.UserID') AS UserOrders
EOS
```

An error message like the following is returned

```
Error while reading data, error message: Error accessing Cloud Spanner.
Query is not root partitionable since it does not have a DistributedUnion at the root.
Please check the conditions for a query to be root-partitionable.
File: SELECT Users.UserID,Orders.OrderID FROM Users JOIN Orders ON Users.UserID = Orders.UserID
```