---
title: "データカタログにNotionを選択した理由"
emoji: "📗"
type: "tech" # tech: 技術記事 / idea: アイデア記事。どちらでもOK
topics: ["データカタログ", "DataCatalog", "Notion", "DataEngineering"]
publication_name: "luup_developers"
published: true # 公開設定（trueにしてmainブランチにマージすると一般公開開始）
published_at: 2022-12-09 00:00 # 未来の日時を指定すればその日時に予約公開されます
---

この記事は、[Luup Advent Calendar](https://adventar.org/calendars/7944)の9日目の記事です。

こんにちは。Data Engineeringチームの河野([@matako1124](https://twitter.com/matako1124)) です！

最近データカタログを導入したのですが、ツールの選定方法と実装方法についてご紹介したいと思います。結論から言うと、Luupでは社内ドキュメントとしてNotionをどの部署も使用しているため、Notionをデータカタログとして使おうという選択にしました。

## 注意

- **執筆に当たり細心の注意を払っておりますが、不十分な説明や誤りがある可能性もございます。**
- **記事内で紹介しているコードは部分的なものであり、参考程度にご参照ください。**

## 目次

- データカタログを導入する目的
- ツール選定方法
- 実装方法
- まとめ
- 終わりに

## データカタログを導入する目的

Luupでは、BIツールにはRedash、DataWarehouseにはBigQueryを採用しています。そのためデータ抽出にはSQLスキルが必須です。

その上で、以下のような情報が載っているドキュメントを用意することは必須だと思いました。

1. どこにどういう形で欲しいデータが保存されているか
2. どのデータが正確なのか
3. わからない時、だれに聞けばいいのか(作成者は誰か)
4. 同じ目的で作成されたテーブルが存在しているか
5. どれくらいの頻度、時間でデータが更新されているか

Luupには、Redashでクエリを書く非エンジニアが多く、データ抽出したいというメンバーにわざわざGCPアカウントを付与して管理する必要がありません。とはいえ、BigQueryにどんなテーブルがどういうスキーマ情報で保存されているのか見えない状態になっているため、データカタログに存在しているスキーマ情報も重要になってきます。(Redashからはテーブル名だけ見えるようになっています)

以上の理由により、データカタログ導入を決定しました。

## ツール選定方法

上記で挙げた項目を満たすために求められるツールの要件として、以下6つを定めました。

- シンプルなUI
- テキスト検索できる
- 非エンジニアでも使える
- アクセス管理が容易
- コード管理できる(APIが提供されているか)
- カスタマイズ性豊富

実はNotionをデータカタログとして実装する前に、検証としてGoogle Data Catalogを一部のメンバーだけ試しに使用していたのですが、すべてのメンバーにGCPアカウントを付与して管理しないといけない点がネックでした。
Notionであれば全社員がすでに活用しているので、アクセス権限周りも問題なく、APIが用意されているのでカスタマイズ性も豊富であると判断しました。何より**参入障壁が低くて済む**というのは大きいなと思います。どんないいツールを導入したとしても使ってもらわないと意味がないですからね。
他ツールも色々と探してみたのですが、あまりいいツールが見つかりませんでした。内製で作るという選択肢も上がったのですが、運用する工数を考えたときに厳しいと思い、断念しました。

|     |  Notion  |  Google Data Catalog  |
|---- | ---- | ---- |
| シンプルなUI | ○ | ○ |
| テキスト検索できる | ○ | ○ |
| すぐに誰でも使える | ◎ | ○ |
| アクセス管理が用意 | ◎ | △ |
| コード管理できる | ○ | ○ |
| カスタマイズ性豊富 | ◎ | ◎ |

## 実装方法

[冪等性を担保したGoogle Cloud Composerの設計と実装](https://zenn.dev/luup/articles/data-kouno-20220723)で紹介しているとおり、Luupのデータ基盤はGoogle Cloud Composerを軸に動いています。なので今回も、Google Cloud Composerの環境下に作りました。

アウトプットイメージは以下です。
![datacatalog1](/images/data-kouno-20221209/datacatalog1.png)
![datacatalog2](/images/data-kouno-20221209/datacatalog2.png)

以下のNotion APIのDocumentを参考に実装を進めていきます。
https://developers.notion.com/docs/create-a-notion-integration
サンプルコードも豊富で、説明も丁寧なので簡単に実装できました。

以下、コード一例です。

```python:notion_api.py
# Notionのフォーマットに変換するメソッド
def format_standard_property_value(self, property_name: str, value: str):
    if property_name == "title":
        return {"title": [{"text": {"content": value}}]}
    elif property_name == "rich_text":
        return {"rich_text": [{"text": {"content": value}}]}
    elif property_name == "table_text":
        return {"type": "text", "text": {"content": value}}
    elif property_name == "status":
        return {"status": {"name": value}}
    elif property_name == "select":
        return {"select": {"name": value}}
    else:
        raise Exception("無効なproperty_name({})が選択されています".format(property_name))

# NotionのDatabaseを初期化するメソッド
def clear_database(self, database_id: str):
    notion = Client(auth=self.notion_apitoken)
    db = notion.databases.query(**{"database_id": database_id})
    for index in range(len(db["results"])):
        page_id = db["results"][index]["id"]
        notion.pages.update(**{"page_id": page_id, "archived": True})
    return

# Databaseにページを追加するメソッド
def create_pages_in_datacatalog_database(
    self, database_id: str, datacatalog_file_path: str, schema_file_path: str
):
    with open(datacatalog_file_path) as datacatalog_json:
        datacatalog_value_json = json.load(datacatalog_json)
        table_name = datacatalog_value_json["table_name"]
        dataset_name = datacatalog_value_json["dataset_name"]
        description = datacatalog_value_json["description"]
        creator = datacatalog_value_json["creator"]
        purpose = datacatalog_value_json["purpose"]
        sql_file = datacatalog_value_json["sql_file"]
        status = datacatalog_value_json["status"]
        update_frequency = datacatalog_value_json["update_frequency"]
        update_time = datacatalog_value_json["update_time"]
        use_case = datacatalog_value_json["use_case"]
        keyword_list = datacatalog_value_json["keyword"]
        try:
            partition = datacatalog_value_json["partition"]
        except KeyError:
            partition = "Null"

    # スキーマリストをNotionのテーブル形式にする
    schema_table_list = [
        {
            "type": "table_row",
            "table_row": {
                "cells": [
                    [self.format_standard_property_value("table_text", "name")],
                    [self.format_standard_property_value("table_text", "type")],
                    [self.format_standard_property_value("table_text", "mode")],
                    [
                        self.format_standard_property_value(
                            "table_text", "description"
                        )
                    ],
                ]
            },
        }
    ]
    if "udf" not in schema_file_path:
        with open(schema_file_path) as schema_json:
            schema_value_json = json.load(schema_json)
        for column in schema_value_json:
            table_row = {
                "type": "table_row",
                "table_row": {
                    "cells": [
                        [
                            self.format_standard_property_value(
                                "table_text", column["name"]
                            )
                        ],
                        [
                            self.format_standard_property_value(
                                "table_text", column["type"]
                            )
                        ],
                        [
                            self.format_standard_property_value(
                                "table_text", column["mode"]
                            )
                        ],
                        [
                            self.format_standard_property_value(
                                "table_text", column["description"]
                            )
                        ],
                    ]
                },
            }
            schema_table_list.append(table_row)

    notion = Client(auth=self.notion_apitoken)
    notion.pages.create(
        **{
            "parent": {"database_id": database_id},
            "properties": {
                "テーブル名": self.format_standard_property_value("title", table_name),
                "データセット名": self.format_standard_property_value(
                    "rich_text", dataset_name
                ),
                "テーブルの説明": self.format_standard_property_value(
                    "rich_text", description
                ),
                "作成者": self.format_standard_property_value("rich_text", creator),
                "作成目的": self.format_standard_property_value("rich_text", purpose),
                "パーティション": self.format_standard_property_value(
                    "rich_text", partition
                ),
                "SQL": {
                    "rich_text": [
                        {
                            "text": {
                                "content": sql_file,
                                "link": {"url": sql_file},
                            }
                        }
                    ]
                },
                "使用可否": self.format_standard_property_value("status", status),
                "更新頻度": self.format_standard_property_value(
                    "select", update_frequency
                ),
                "更新時間": self.format_standard_property_value(
                    "rich_text", update_time
                ),
                "使用例": self.format_standard_property_value("rich_text", use_case),
                "キーワードタグ": {
                    "multi_select": [{"name": keyword} for keyword in keyword_list]
                },
            },
            "children": [
                {
                    "object": "block",
                    "type": "heading_2",
                    "heading_2": self.format_standard_property_value(
                        "rich_text", "Schema"
                    ),
                },
                {
                    "object": "block",
                    "type": "table",
                    "table": {
                        "table_width": 4,
                        "has_column_header": True,
                        "has_row_header": False,
                        "children": schema_table_list,
                    },
                },
            ],
        }
    )
    return
```

```python:dag.py
with airflow.DAG(
    ...
) as dag:

    clear_datacatalog_database = PythonOperator(
        task_id="clear_datacatalog_database",
        python_callable=datacatalog_service.clear_database,
        op_args=[datacatalog_database_id],
    )

    datacatalog_files_path = "{}/src/datacatalog/**/*".format(
        config_unity_dag.dags_folder
    )
    for datacatalog_file_path in glob.glob(datacatalog_files_path):
        create_pages_in_datacatalog_database = PythonOperator(
            task_id="create_pages_in_datacatalog_database_{}".format(
                os.path.splitext(os.path.basename(datacatalog_file_path))[0]
            ),
            python_callable=datacatalog_service.create_pages_in_datacatalog_database,
            op_args=[
                datacatalog_database_id,
                datacatalog_file_path,
                datacatalog_file_path.replace(
                    "datacatalog", "schema"
                ),  ## schemaファイルを指定
            ],
        )

        (clear_datacatalog_database >> create_pages_in_datacatalog_database)
```

## まとめ

データカタログは、データエンジニアリング界隈でホットな題材かと思い、紹介してみました。
エンジニアというよりは、データを見たい人(社員全員)が対象になるので、より使いやすく見やすいものを基準にツールを選びました。
実装方法もjsonファイルを追加するだけでNotionのDatabaseにPageが追加されるように作り、エンジニアだけでなく、アナリストやサイエンティストもCommitしやすいようにしました。
データカタログは**みんなで作る精神**が重要ですからね。

## 終わりに

Luupでのデータ基盤構築、データ活用に少しでもご興味がある方もしくは、うちはこのツールをデータカタログに使っているよ等ありましたら、ぜひ情報交換という形でもお話しできたら嬉しいです。
https://recruit.luup.sc/
