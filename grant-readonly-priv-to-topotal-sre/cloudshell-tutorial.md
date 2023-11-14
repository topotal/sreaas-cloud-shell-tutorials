# Topotal SREに対するreadonly権限付与

## はじめに

Topotal SRE(sre@topotal.com)に対して、特定のプロジェクト全体のリソースに対してreadonly権限を付与します。権限設定はCloud Deployment Managerを用いて行います。

一部のコマンドの実行にはプロジェクトの管理者権限が必要になります。

## 1. 対象プロジェクトの設定

Topotal SREに対して権限付与を行う対象のプロジェクトをCloud Shell上にセットします。

### 1.1. プロジェクトの一覧を表示する

```
gcloud projects list
```

### 1.2. プロジェクトを設定する

```
gcloud config set project YOUR_PROJECT_ID
```

上記のコマンドにより環境変数 `GOOGLE_CLOUD_PROJECT` がセットされます。

## 2. 各種APIの有効化

Cloud Deployment Managerの実行に必要なAPIの有効化を行います。
具体的には、Cloud Resorce Manager APIとCloud Resource Manager APIの有効化を行います。

### 2.1. Cloud Resource Manager APIの有効化

```
gcloud services enable cloudresourcemanager.googleapis.com
```

### 2.2. Deployment Manager APIの有効化

```
gcloud services enable deploymentmanager.googleapis.com
```

## 3. Deployment ManagerにIAMポリシーを設定するための権限を付与する

Deployment ManagerはGoogle APIサービスアカウントを使用して、他の Google API を呼び出し、Google Cloud Platform リソースを管理します。プロジェクトのGoogle APIサービスアカウントにroles/owner役割を付与して、構成で定義したIAMポリシーを適用できるようにする必要があります。

refs: [Deployment Manager に IAM ポリシーを設定するための権限を付与する](https://cloud.google.com/deployment-manager/docs/configuration/set-access-control-resources?hl=ja##granting_permission_to_set_iam_policies)

### 3.1. サービスアカウントの確認

```
gcloud projects get-iam-policy ${GOOGLE_CLOUD_PROJECT} --format=json |
  jq -r '.bindings | reduce .[].members as $item ([]; .+$item) | 
  .[] | select(startswith("serviceAccount"))' |
  grep @cloudservices.gserviceaccount.com | cut -d : -f 2
```

以下の形式のメールアドレスが設定された Google APIサービスアカウントが出力されれば次に進んでください。

`[PROJECT_NUMBER]@cloudservices.gserviceaccount.com`

### 3.2. roles/ownerの役割付与

コマンドライン中の `YOUR_SERVICE_ACCOUNT` の部分を先ほど確認したサービスアカウントのメールアドレスに置き換えた上で実行してください。

```
gcloud projects add-iam-policy-binding ${GOOGLE_CLOUD_PROJECT} \
  --member serviceAccount:YOUR_SERVICE_ACCOUNT --role roles/owner
```

## 4. readonly権限の付与

以下のコマンドを実行することで、Topotal SREが所属するGoogle Groups(`sre@topotal.com`)に対してプロジェクトレベルのreadonly権限(`roles/viewer`ロール)を付与します。

※ 付与対象のメンバーや権限内容の変更が必要な場合は、virtualProjectMember.yamlを修正してください。

```
gcloud deployment-manager deployments create grant-readonly-to-topotal-sre \
  --config grant-readonly-priv-to-topotal-sre/virtualProjectMember.yaml
```

### 4.1 動作確認

以下のコマンドの出力を確認し、`sre@topotal.com`に対して`roles/viewer`の役割が設定されていれば作業は完了です。

※ virtualProjectMember.yamlを書き換えた場合は、意図通りの設定が行われているかどうかを確認してください。

```
gcloud projects get-iam-policy ${GOOGLE_CLOUD_PROJECT}
```

### トラブルシューティング

なんらかの理由でdeploymentの作成に失敗した場合は、以下のコマンドを実行して作成済みのリソースを削除することで再実行が可能になります。

```
gcloud deployment-manager deployments delete grant-readonly-to-topotal-sre
```
