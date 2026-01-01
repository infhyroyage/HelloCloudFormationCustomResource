# CloudFormation カスタムリソース + Lambda ハンズオン

## 概要

このハンズオンでは、CloudFormation の**カスタムリソース**を Lambda 関数と連携させる方法を学びます。

カスタムリソースは、CloudFormation の標準リソースタイプでは対応できない処理を Lambda 関数で実装し、スタックのライフサイクル(作成・更新・削除)と連携させる機能です。

このデモでは、Lambda 関数内で boto3 を使って S3 バケット一覧を取得し(`aws s3 ls` 相当)、CloudWatch Logs に出力します。

## アーキテクチャ

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│                 │    │                 │    │                 │
│  CloudFormation │───▶│  Lambda関数     │───▶│ CloudWatch Logs │
│                 │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                      │
        │                      │ cfnresponse.send()
        │                      │
        └──────────────────────┘
              SUCCESS/FAILED
```

## 前提条件

- AWS アカウントを持っていること
- IAM ユーザーに CloudFormation、Lambda、IAM、CloudWatch Logs へのアクセス権限があること
- AWS CLI がインストール・設定済みであること(CLI 版を実施する場合)

## ファイル構成

```
HelloCloudFormationCustomResource/
├── custom-resource-demo.yaml  # CloudFormationテンプレート
├── README.md                  # このファイル
└── LICENSE
```

## 作成されるリソース

| リソース         | 説明                                                                           |
| ---------------- | ------------------------------------------------------------------------------ |
| Lambda 関数      | カスタムリソースのロジックを実行(S3 バケット一覧の取得と CloudWatch Logs 出力) |
| IAM ロール       | Lambda 関数の実行ロール(CloudWatch Logs への書き込みと S3 読み取り権限)        |
| カスタムリソース | Lambda 関数を呼び出す CloudFormation リソース                                  |

---

## 手順 1: AWS マネジメントコンソールでデプロイする場合

1. **CloudFormation コンソールを開く**

   - https://console.aws.amazon.com/cloudformation/ にアクセス

2. **スタック作成**

   - 「スタックの作成」 > 「新しいリソースを使用(標準)」をクリック

3. **テンプレートのアップロード**

   - 「テンプレートファイルのアップロード」を選択
   - `custom-resource-demo.yaml` を選択
   - 「次へ」をクリック

4. **スタック設定**

   - スタック名: `CustomResourceDemo`
   - 「次へ」をクリック

5. **オプション設定**

   - そのまま「次へ」をクリック

6. **確認と作成**

   - 「AWS CloudFormation によって IAM リソースが作成される場合があることを承認します」にチェック
   - 「送信」をクリック

7. **動作確認**
   - 「イベント」タブでカスタムリソースの作成状況を確認
   - CREATE_COMPLETE になることを確認
   - 「出力」タブでカスタムリソースからの応答を確認(S3BucketCount など)

---

## 手順 2: AWS CLI でデプロイする場合

### スタック作成

```bash
# スタックを作成
aws cloudformation create-stack \
  --stack-name CustomResourceDemo \
  --template-body file://custom-resource-demo.yaml \
  --capabilities CAPABILITY_IAM

# スタック作成完了を待機
aws cloudformation wait stack-create-complete \
  --stack-name CustomResourceDemo

# スタックの出力を確認
aws cloudformation describe-stacks \
  --stack-name CustomResourceDemo \
  --query "Stacks[0].Outputs"
```

### スタック更新

```bash
# テンプレートを変更して更新
aws cloudformation update-stack \
  --stack-name CustomResourceDemo \
  --template-body file://custom-resource-demo.yaml \
  --capabilities CAPABILITY_IAM

# スタック更新完了を待機
aws cloudformation wait stack-update-complete \
  --stack-name CustomResourceDemo
```

### スタック削除

```bash
# スタックを削除
aws cloudformation delete-stack --stack-name CustomResourceDemo

# 削除完了を待機
aws cloudformation wait stack-delete-complete \
  --stack-name CustomResourceDemo
```

---

## CloudWatch Logs で動作確認

1. https://console.aws.amazon.com/cloudwatch/ にアクセス
2. 左メニューから「ロググループ」を選択
3. `/aws/lambda/CustomResourceDemo-CustomHandler` を選択
4. ログストリームで Lambda 関数の実行ログを確認
   - `受信イベント`
   - `=== S3バケット一覧===`
   - `バケット数: X`
   - 各バケットの作成日時と名前の一覧

---

## 学習ポイント

### テンプレートの重要な要素

| 項目           | 説明                                                  |
| -------------- | ----------------------------------------------------- |
| `ServiceToken` | カスタムリソースが Lambda 関数を呼び出すための ARN    |
| `cfnresponse`  | Lambda 関数から CloudFormation へ結果を返すモジュール |
| `RequestType`  | `Create` / `Update` / `Delete` のいずれか             |
| `Fn::GetAtt`   | カスタムリソースの出力値を取得する関数                |

### カスタムリソースの仕組み

1. CloudFormation がスタック操作(作成/更新/削除)を開始
2. カスタムリソースに到達すると、`ServiceToken`で指定された Lambda 関数を呼び出し
3. Lambda 関数はイベントを受け取り、カスタム処理を実行
   - このデモでは boto3 を使って S3 バケット一覧を取得
   - 結果を CloudWatch Logs に出力
4. `cfnresponse.send()`で CloudFormation に SUCCESS/FAILED を返答
5. CloudFormation は応答を受け取り、スタック操作を続行

### cfnresponse モジュール

`ZipFile`プロパティでインラインコードを記述する場合、`cfnresponse`モジュールが自動的に利用可能になります。

```python
import cfnresponse

# 成功時
cfnresponse.send(event, context, cfnresponse.SUCCESS, response_data)

# 失敗時
cfnresponse.send(event, context, cfnresponse.FAILED, {"Error": "エラーメッセージ"})
```

### boto3 による AWS サービス操作

Lambda 関数内では boto3 を使って AWS の各種サービスを操作できます。

```python
import boto3

# S3 クライアントを作成
s3_client = boto3.client('s3')

# バケット一覧を取得
response = s3_client.list_buckets()
for bucket in response['Buckets']:
    print(f"{bucket['CreationDate']} {bucket['Name']}")
```

**注意**: Lambda 関数の IAM ロールに適切な権限が必要です。

---

## 参考資料

- [Lambda-backed custom resources - AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources-lambda.html)
- [cfn-response module](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-lambda-function-code-cfnresponsemodule.html)
- [Create custom provisioning logic with custom resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)

---

## ライセンス

このプロジェクトは LICENSE ファイルに記載されたライセンスの下で提供されています。
