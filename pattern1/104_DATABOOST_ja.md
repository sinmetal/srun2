# DataBoost

https://cloud.google.com/spanner/docs/databoost/databoost-overview

DataBoostを利用すると1クエリのために専用のマシンリソースが作成され、処理される。
Spanner Instanceには影響を与えずに負荷が高いクエリを実行できるので、バッチジョブの実行などに便利に使える。
利用できるクエリは [Partition Query](https://cloud.google.com/spanner/docs/reads?hl=en#read_data_in_parallel) である必要がる。
Partition Queryで実行可能なクエリは大雑把に言うと各Splitで完結できるクエリ。

今回は簡単にDataBoostを使うためにBigQueryからDataBoostを利用して [Federated Query](https://cloud.google.com/bigquery/docs/cloud-spanner-federated-queries) を実行する。

```
properties1=$(echo '{"database":"projects/'$CLOUDSDK_CORE_PROJECT'/instances/'$CLOUDSDK_SPANNER_INSTANCE'/databases/'$DB1'", "useParallelism":true, "useDataBoost": true}')
connection_name1=$(printf "spanner_%s_%s" $CLOUDSDK_SPANNER_INSTANCE $DB1)
bq mk --project_id $CLOUDSDK_CORE_PROJECT --connection --connection_type='CLOUD_SPANNER' --location='us-central1' \
--properties=$properties1 $connection_name1
```

# JOIN

DataBoostでSpannerに対して実行するQueryはPartitionQueryとして実行される。
そのため、実行できるQueryに制限が存在する。
pattern1の場合、UsersとOrdersはインターリーブされてないので、JOINしようとすると別のPartitionにRowが跨ってしまうため、実行できない

```
# このクエリは実行できない
bq query --use_legacy_sql=false << EOS
SELECT * FROM EXTERNAL_QUERY(
  '$CLOUDSDK_CORE_PROJECT.us-central1.$connection_name1',
  'SELECT Users.UserID,Orders.OrderID FROM Users JOIN Orders ON Users.UserID = Orders.UserID') AS UserOrders
EOS
```

以下のようなエラーメッセージが返ってくる

```
Error while reading data, error message: Error accessing Cloud Spanner.
Query is not root partitionable since it does not have a DistributedUnion at the root.
Please check the conditions for a query to be root-partitionable.
File: SELECT Users.UserID,Orders.OrderID FROM Users JOIN Orders ON Users.UserID = Orders.UserID
```