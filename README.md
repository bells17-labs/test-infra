# test-infra

## Cluster

- digitalocean kubernetes を利用
- kubernetes version: 1.18.6
- 以下のコマンドでは作成したクラスターのkubeconfigをこのリポジトリトップに `kubeconfig.yaml` として置いてある前提ですすめる

## Prow

基本
https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md
をベースに必要最低限prowを十分に使えるようにしたものなのでこれも参考になる

### ツール周りのセットアップ

- Prowを利用するためのドメインを用意する
  - このドメインのためのSSL証明書を用意する

- [Bot用のGithubアカウントを用意する](https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md#github-bot-account)
  - private repositoryでもprowが作れるようにssh keyを登録する
  ```
  ssh-keygen -t rsa -b 4096 -f id_rsa
  ```
  - GithubアカウントのPersonal Access Tokenを生成する
    - 以下の権限を付与する
      - `repo`
      - `admin:org_hook`
      - `admin:org`
    - Access TokenのSecret用のファイルを生成する
    ```sh
    echo -n "<生成したGithub Token>" > github-token
    ```
  - [prow用のOauth Appを生成する](https://github.com/kubernetes/test-infra/blob/master/prow/cmd/deck/github_oauth_setup.md)
    - URLは予め用意したドメインを利用する
    - 以下のように入力する
      - Application name: `prow`
      - Homepage URL: `https://<用意したドメイン>/`
      - Authorization callback URL: `https://<用意したドメイン>/github-login/redirect`
    - Oauth AppのSecret用のファイルを生成する
    ```sh
    cat <<EOS > github-oauth-config
    client_id: <APP_CLIENT_ID>
    client_secret: <APP_CLIENT_SECRET>
    redirect_url: <用意したドメイン>/github-login/redirect
    final_redirect_url: <用意したドメイン>/pr
    EOS

    openssl rand -out github-cookie -base64 32
    ```

- [GithubのWebhook用のSecretを生成する](https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md#create-the-github-secrets)
```sh
openssl rand -hex 20 > github-secret
```

- テスト動作用のGithubのOrganizationの作成
  - 作成したOrganization内にstage-botというチームを作成
  - チームにbotアカウントを招待する
  - OrganizationのWebhookを設定
    - Payload URL: `https://<用意したドメイン>/hook`
    - Content type: `application/json`
    - Secret: <生成したgithub-secret>
    - SSL verification: Enable SSL verification
    - Which events would you like to trigger this webhook?: Send me everything.

- リポジトリの作成
  - 作成したOrganizationにprowを管理するためのtest-infraリポジトリを作成
  - 作成したリポジトリに作成したチームをadmin権限で招待する

- [GCSの用意](https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md#configure-a-gcs-bucket)
  - GCSのバケットを作成(バケット名は全世界で一意にする必要があるので注意)
  - GCS操作用のService Accountの作成
  - Service Accountの認証キーの発行
  ```sh
  bucket=<作成するバケット名>
  gcloud iam service-accounts create prow-gcs-publisher
  identifier="$(  gcloud iam service-accounts list --filter 'name:prow-gcs-publisher' --format 'value(email)' )"
  gsutil mb "gs://${bucket}/" # step 2
  gsutil iam ch allUsers:objectViewer "gs://${bucket}" # step 3
  gsutil iam ch "serviceAccount:${identifier}:objectAdmin" "gs://${bucket}" # step 4
  gcloud iam service-accounts keys create --iam-account "${identifier}" service-account.json # step 5
  ```

### Secret情報の登録

```
kubectl create secret generic oauth-token --from-file=oauth=github-token
kubectl create secret generic hmac-token --from-file=hmac=github-secret
kubectl create secret generic github-oauth-config --from-file=secret=github-oauth-config
kubectl create secret generic cookie --from-file=secret=github-cookie
kubectl create ns test-pods
kubectl create secret generic gcs-credentials --from-file=service-account.json
kubectl create secret generic gcs-credentials --from-file=service-account.json -n test-pods
kubectl create secret generic ssh-secret --from-file=id_rsa=id_rsa -n test-pods
kubectl create secret generic oauth-token --from-file=oauth=github-token -n test-pods
```

### Config情報の登録

```
cd prow
make apply-config
kubectl create cm job-config --from-file=<作成したOrg>--test-infra.yaml=./jobs/<作成したOrg>/test-infra/<作成したOrg>--test-infra.yaml
```

### テンプレートの編集

TODO

必要そうなパラメータ

- GCS Bucket Name
- Github Org


### デプロイ

```
make deploy
```

### ロードバランサーIPとドメインの紐付け

ロードバランサーIPとドメインの紐付けを行う

### 確認

#### deck

設定したドメインにアクセスして画面が表示されればOK

#### hook

issueの作成などを行い、Webhookが成功しているかを確認する

## Peribolos

以下のように実行する

```
docker run -it --rm \
  -v (pwd)/org.yaml:/Users/<user>/org.yaml \
  -v (pwd)/../github-token:/Users/<user>/github-token \
  gcr.io/k8s-prow/peribolos:v20200722-b942f8218d \
    --github-token-path ~/github-token \
    --config-path ~/org.yaml \
    --fix-org \
    --fix-org-members \
    --fix-teams \
    --fix-team-members \
    --required-admins=bells17 \
    --required-admins=necobot \
    --min-admins=2 \
    ## --confirm=true # --confirm=trueをつけないと実行されない
```

- https://github.com/kubernetes/test-infra/tree/master/prow/cmd/peribolos
- https://github.com/kubernetes/org/blob/master/admin/update.sh
- https://github.com/kubernetes/org/blob/master/admin/BUILD.bazel
