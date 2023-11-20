---
title: "Google Cloud Composer(Airflow)でGoogle Driveに定期的にデータを送ってみる"
emoji: "🏊‍♀️"
type: "tech" # tech: 技術記事 / idea: アイデア記事。どちらでもOK
topics: ["Airflow", "GoogleCloudComposer", "DataEngineering", "GoogleDrive"]
publication_name: "luup_developers"
published: true # 公開設定（trueにしてmainブランチにマージすると一般公開開始）
---

こんにちは。Data Engineeringチームの河野([@matako1124](https://twitter.com/matako1124)) です！

この記事では、Google Cloud Composer(Airflow)の中でGoogle Driveに定期的にデータを送るための実装方法をご紹介したいと思います。実装当初、Google Driveにデータを定期的に送るような事例は多いのかなと思っていたのですが、意外と世に記事がでていなかったので執筆してみました。

なお、Luupのデータ基盤については以下の記事をご参照ください。
[冪等性を担保したGoogle Cloud Composerの設計と実装](https://zenn.dev/luup/articles/data-kouno-20220723)

## 注意

- **執筆に当たり細心の注意を払っておりますが、内容に誤りや説明が不十分な箇所がある可能性がございます。**
- **記事内で紹介しているコードは部分的なものであり、参考程度にご参照ください。**

## 目次

- なぜGoogle Driveに定期的にデータを送りたいのか
- 実装方法
- まとめ
- 終わりに

## なぜGoogle Driveに定期的にデータを送りたいのか

Luupでは、BIツールにRedashとData Portalを採用していますが、アウトプットとしてBIツールではなく、SpreadSheetやExcel、CSVで出力したい部署なども存在しています。

今まではデータチーム宛に抽出依頼をもらい、毎月データを落として渡すというフローで手動対応していたのですが、出力フォーマットが決まっている中で毎月同じ作業をするのは、依頼する方も対応する方も面倒です。

これは自動化したほうがいいだろうということで、Google Cloud Composer(Airflow)上で定期的にデータを自動で送る実装を行いました。

## 実装方法

Luupでは、組織全体でGoogle Workspaceを利用しています。Google Workspaceを利用している前提での実装前準備として、**Google Workspaceの管理コンソールからdomain wide delegationの設定をしないといけません**。
Google Cloud Composer(Airflow)は付与しているサービスアカウントを元に色々な実行を行なっているのですが、サービスアカウントでGoogle Driveに送ったファイルは他のユーザーがみることはできません。要するに実行自体は成功しますが、Google Driveに存在しないような状態になってしまいます。
これを解決するためにdomain wide delegation設定を行う必要があり、対象のサービスアカウントで実行したものは、実行の際にオプションで設定したLuup組織のGoogleアカウント名で送ることができるようになります。

英語ではありますが、詳細な設定方法は以下の記事が参考になるかと思います。
[How to Upload Files to Google Drive using Airflow](https://towardsdatascience.com/how-to-upload-files-to-google-drive-using-airflow-73d961bbd22)

Google Cloud Composer(Airflow)を使用していれば、実装自体は数行のコードで実装できます。
注意点としては、[GCSToGoogleDriveOperator](https://airflow.apache.org/docs/apache-airflow-providers-google/stable/_api/airflow/providers/google/suite/transfers/gcs_to_gdrive/index.html)というOperatorも存在するのですが、なぜかdomain wide delegation設定を行なっても反映されず、うまく動かないという状態になってしまっているため、[GoogleDriveHook](https://airflow.apache.org/docs/apache-airflow-providers-google/stable/_api/airflow/providers/google/suite/hooks/drive/index.html)というGoogleDriveAPIを直接叩いてくれる方を使用してください。

以下、DAGファイルの一例です。
GoogleDriveHookの`gcp_conn_id`には対象のサービスアカウントのIDを指定し、`delegate_to`にGoogleアカウント名(メールアドレス)を指定してください。

```python
import airflow
from airflow.operators.python_operator import PythonOperator
from airflow.providers.google.suite.hooks.drive import GoogleDriveHook

...

    def export_gcs_to_drive(folder_name: str, table_name: str, data_division: str):
        hook = GoogleDriveHook(
            gcp_conn_id="airflow_to_google_workspace",
            delegate_to="k.t@",
        )
        exec_gcs_to_drive = hook.upload_file(
            "test_data_folder/export/{}-{}.csv".format(
                table_name, data_division
            ),
            "{}/{}-{}.csv".format(folder_name, table_name, data_division),
        )
        return exec_gcs_to_drive

    export_gcs_to_drive_ride_data = PythonOperator(
        task_id="export_gcs_to_drive_ride_data",
        python_callable=export_gcs_to_drive,
        op_args=[
            "bq_export/test",
            "ride_data",
            "2022-09-01",
        ],
    )

    export_gcs_to_drive_ride_data

```

## まとめ

毎月月初に前月分のデータが欲しい、ExcelやCSVで欲しいといった要望に対して、毎月手動で行なったとしても数分でできると思います。
ですが、依頼側と作業側のコミュニケーションと作業時間を毎月なくせると考えたら自動化するべきではないでしょうか。
データ基盤にGoogle Cloud Composer(Airflow)を採用されている方は、ぜひ参考にしていただけると幸いです。

## 終わりに

Luupでのデータ基盤の構築はまだまだ走り始めたばかりで、やるべきことは山のようにあります。
電動マイクロモビリティの位置情報データを扱えるという面では、非常に難しいものではあるものの、大量のデータをどう整合性担保して管理していくか、どういう構造にすればデータ活用がより浸透していくか等、面白いことが満載な環境だと私は思っています。

もしLuupでのデータ基盤構築、データ活用に少しでもご興味がある方がいらっしゃればお気軽にご連絡をお待ちしております。
https://recruit.luup.sc/
