---
title: "Airflowで依存関係の修正漏れを防ぐ"
emoji: "🌈"
type: "tech" # tech: 技術記事 / idea: アイデア記事。どちらでもOK
topics: ["Airflow", "CI", "ExternalTaskSensor", "DataEngineering"]
publication_name: "luup_developers"
published: true # 公開設定（trueにしてmainブランチにマージすると一般公開開始）
---

こんにちは。Data Engineeringチームの河野([@matako1124](https://twitter.com/matako1124)) です！

Airflowで依存関係の設定にExternalTaskSensorを使っているのですが、ExternalTaskSensorを使用する時は、以下2つを注意しなければいけません。

1. 実行時間が同じでなければいけない
2. DAG名、Task名も正確に記載しないといけない

これを間違えてしまうと、Pokingし続けてしまいます。

特にリファクタでDAG名やTask名が変わるといったことはよく起こりうると思います。
今回は「DAG名、Task名が正確に記載されていなければ、CIで落とし、修正漏れを防ぐ」事例をご紹介します。

## 注意

- **執筆に当たり細心の注意を払っておりますが、不十分な説明や誤りがある可能性もございます。**
- **記事内で紹介しているコードは部分的なものであり、参考程度にご参照ください。**

## 目次

- どういう時に困ったか
- CIへの追加実装事例
- 終わりに

## どういう時に困ったか

実装例を取り上げながら、どういう時に困ったか説明します。
現状BigQueryへのデータ処理は、1つのwrapファイルにまとめており、そのメソッドを呼び出して処理を行なっています。
例えば、以下は主にMasterテーブルを作成する際に呼び出しているメソッドで、1〜3までをTaskGroupで1つのTaskにまとめています。

1. テーブルを削除
2. テーブルを新規作成
3. テーブルにデータをロードする

```python
def delete_create_load(
    self,
    dataset_id: str,
    table_name: str,
    table_description: str,
    partition_setting: dict = None,
) -> TaskGroup:

    schema_relative_path = "{}/src/schema".format(self.dags_folder)
    sql_relative_path = "/src/sql"
    table_resource = {
        "schema": {"fields": [json_file_load(schema_path, table_name)]},
        "description": table_description,
    }

    with TaskGroup(
        group_id="delete_create_load_table_{}".format(table_name)
    ) as delete_create_load_trans:
        delete_table = BigQueryDeleteTableOperator(
            task_id="delete_table",
            deletion_dataset_table="{}.{}.{}".format(
                self.project_id, dataset_id, table_name
            ),
            ignore_if_missing=True,
        )

        create_empty_table = BigQueryCreateEmptyTableOperator(
            task_id="create_empty_table",
            project_id=self.project_id,
            dataset_id=dataset_id,
            table_id=table_name,
            table_resource=table_resource,
        )

        load_table = BigQueryInsertJobOperator(
            task_id="load_table",
            configuration={
                "query": {
                    "query": f"{{% include '{'{}/{}.sql'.format(sql_path, table_name)}' %}}",
                    "useLegacySql": False,
                    "writeDisposition": "WRITE_EMPTY",
                    "destinationTable": {
                        "projectId": self.project_id,
                        "datasetId": dataset_id,
                        "tableId": table_name,
                    },
                }
            },
        )

    delete_table >> create_empty_table >> load_table
    return delete_create_load_trans
```

上記メソッドをDAGから呼びます。これをDAG_Aとします。

```python
from src.udf.wrap_bq_udf import WrapUnityBqUdf

wrap_unity_bq_udf = WrapUnityBqUdf(
    config_unity_dag.project_id,
    config_unity_dag.bucket_name,
    config_unity_dag.dags_folder,
)

with airflow.DAG(
    dag_id="DAG_A",
    default_args=config_unity_dag.config_default_args(
        owner, execution_timeout_minutes=120
    ),
    ...
    tags=["dwh"],
) as dag:

    cities_master = wrap_unity_bq_udf.delete_create_load(
        dataset_id=dataset_id,
        table_name="cities_master",
        table_description="Luup定義シティのマスタテーブル",
    )

```

DAG_Aで作成したcities_masterのStatusをExternalTaskSensorでキャッチします。これをDAG_Bとします。

```python
from airflow.sensors.external_task import ExternalTaskSensor

with airflow.DAG(
    os.path.splitext(os.path.basename(__file__))[0],
    default_args=config_unity_dag.config_default_args(owner),
    ...
    tags=["datamart"],
) as dag:

    wait_for_dwh_load_cities_master = ExternalTaskSensor(
        task_id="wait_for_dwh_load_cities_master",
        external_dag_id="DAG_A",
        external_task_id="delete_create_load_table_cities_master.load_table",
    )
```

このように定義されていた場合に、大本のTaskGroupのgroup_idの命名を以下のように変更したとします。

```python
group_id=table_name
```

↓

```python
group_id="delete_create_load_table_{}".format(table_name)
```

group_idを変えるとTaskの命名も変わるので、メソッドを呼び込んでいるDAG_Aだけでなく、ExternalTaskSensorで指定していたexternal_task_idの命名(DAG_B)も修正しないといけません。

例えば、table_name="cities_master"だとすると

```python
wait_for_dwh_load_cities_master = ExternalTaskSensor(
    ...
    external_task_id="cities_master.load_table",
)
```

を以下のように修正しないと、Pokingし続けてエラーで落ちてしまいます。

```python
wait_for_dwh_load_cities_master = ExternalTaskSensor(
    ...
    external_task_id="delete_create_load_table_cities_master.load_table",
)
```

上記のdelete_create_loadメソッドはいろんなDAGで使われているので、気軽にリファクタできなくなってしまいました。

## CIへの追加実装事例

そこで、TaskSensorに指定しているExternalDagId、ExternalTaskIdが存在するかチェックする処理をCIに組み込みました。
以下のようなDagValidationをCIに組み込んでいて、ここにExternalDagId、ExternalTaskIdが存在するかチェックする処理を追加しました。

```python
from airflow.sensors.external_task import ExternalTaskSensor

    def test_check_external_task_existence(self):
        ## TaskSensorに指定しているExternalDagId、ExternalTaskIdが存在するかチェックする
        external_task_sensor = [
            task
            for dag in self.dagbag.dags.values()
            for task in dag.tasks
            if isinstance(task, ExternalTaskSensor)
        ]
        for task in external_task_sensor:
            external_dag_id = task.external_dag_id
            external_task_id = task.external_task_id
            self.assertIn(
                external_dag_id, self.dagbag.dags, f"{external_dag_id} not found"
            )
            self.assertIn(
                external_task_id,
                self.dagbag.get_dag(external_dag_id).task_ids,
                f"{external_task_id} not found in {external_dag_id}",
            )
```

これでDAG名やTask名を変更しても、修正漏れが存在した場合は気付けるようになりました。

## 終わりに

CIの実装は後に回されがちですが、なるべく早く実装することで開発スピードや効率性が大きく変わってくると思います。
あとから見返したら、CIを早い段階で豊富にしたおかげでhotfixの数が激減したなんてこともよく聞きます。
ぜひご参考になれば幸いです。
