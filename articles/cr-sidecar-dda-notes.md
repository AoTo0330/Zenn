---
title: "Cloud Run に Datadog Agent をサイドカーでデプロイする際の注意点" # 記事のタイトル
emoji: "🐶" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["gcp", "cloudrun", "googlecloud", "datadog"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

# TL;DR
この記事は [Cloud Run のサイドカーデプロイ](https://cloud.google.com/run/docs/deploying?hl=en#sidecars)を利用して Datadog Agent をデプロイする際の注意点をまとめたものです。

デプロイの際の手順やサイドカーデプロイの活用方法については、「[Cloud Run で Datadog Agent をサイドカーとして動かす](https://qiita.com/AoTo0330/items/35a840462f219596e39d)」でまとめています。
また、こちらの内容は **Datadog 公式にはサポートされていない**内容のため、ご注意ください。[Datadog 公式の Cloud Run ドキュメント](https://docs.datadoghq.com/ja/serverless/google_cloud_run/)では、専用のエージェントを用いた Cloud Run シングルコンテナのトレース・ログ・カスタムメトリクスの送信を**ベータ版で公開**しています。


:::message
2023/5/21 時点の情報を基に作成しています。各プラットフォームのアップデートにより、いくつかの点は変更される可能性があります。
:::

# イメージをビルドする際
## アプリケーションの Dockerfile
アプリケーションの Dockerfile には、言語に応じたトレーサーとそれに必要な環境変数を定義する必要があります。
この時、デフォルトの`localhost`で動作するため`DD_AGENT_HOST`を設定する必要はありません。

Python アプリケーションを例にとると、`ENV` で [Datadog の標準タグ](https://docs.datadoghq.com/ja/getting_started/tagging/#統合サービスタグ付け)に基づいたタグを環境変数に定義し、`RUN`で`requirements.txt`に記載された dd-tracer をインストールした上で、`CMD`で明示的にトレーサーを起動します。

```dockerfile:Dockerfile.python
FROM python:3

ENV DD_SERVICE="calendar"
ENV DD_ENV="dev"
ENV DD_VERSION="0.1.0"

WORKDIR /home

COPY requirements.txt /home
COPY /calendar /home/calendar

RUN pip install -r requirements.txt

# Run the application with Datadog
CMD ["ddtrace-run", "python", "calendar/app.py"] 
```

## Datadog Agent の Dockerfile
Datadog が公開している [Docker イメージ](https://github.com/DataDog/docker-dd-agent/blob/master/Dockerfile)では、既に `EXPOSE 8126/tcp`をしているため明示的にポートを解放する必要はありません。

この段階では、設定変更を行うために`datadog.yaml`を配置したり、`ENV`内で環境変数をしていして組み込むことができます。環境変数の設定は Cloud Run へのデプロイ時でも可能ですが、イメージビルドの際に組み込むことで設定が反映された状態での配布が可能となります。
```dockerfile:Dockerfile.dd
FROM gcr.io/datadoghq/agent:latest
COPY config.yaml /etc/datadog-agent/datadog.yaml

EXPOSE 8126

ENV DD_API_KEY=<DD_API_KEY>
ENV DD_SITE=datadoghq.com
ENV DD_APM_ENABLED=true
```
## アプリケーションの構成
Datadog のトレーサーはアプリケーションに直接手を加えることなく、トレースデータを自動計測(auto instrumentation)することが可能です。このデータはデフォルトで Unix ドメインソケット`/var/run/datadog/apm.socket`にトレースを送信しようとします。ソケットが存在しない場合、トレースは`http://localhost:8126`に送信されます。

この設定を変更したい場合は、`DD_TRACE_AGENT_URL`を使用して、`http:`または`unix:`で始まる形で送信先を指定することができます。
また、これに合わせる形で Datadog Agent のリッスンするポートは`EXPOSE`する必要があります。

# イメージをデプロイする際
## Cloud Run YAML の記載方法
Cloud Run YAML のリファレンスは公式ドキュメントがあります。

https://cloud.google.com/run/docs/reference/yaml/v1

Datadog Agent を正常に稼働させるためには、常にリソースを保つ必要があり、`spec: template: metadata: annotations: run.googleapis.com/cpu-throttling: "false"`で CPU throttling を無効化することで、 CPU を常時割り当てることができます。

これについては、「[新しい CPU 割り当てコントロールにより Cloud Run 上でさらに多様なワークロードを実行](https://cloud.google.com/blog/ja/products/serverless/cloud-run-gets-always-on-cpu-allocation?hl=ja)」で詳しい説明があります。

裏側では [Knative](https://knative.dev/docs/serving/services/configure-requests-limits-services/) を利用しているので、`resouces: request`と`resouces: limit`で明示的に CPU とメモリのサイズを指定することも可能です。

Datadog Agent を先に起動することで、トレースデータの送信を初めから確実に行うことができます。これは、`spec: template: metadata: annotations: container-dependencies: '{"app":["dd-agent"]}'`で依存関係を定義することで行え、3つ以上のコンテナについても定義可能です。

## Cloud Build でのデプロイ
Google Cloud Shell を利用している間、Shell 上の操作は利用しているユーザの IAM 権限に基づいて操作の[認可](https://cloud.google.com/shell/docs/auth)を行うことが可能です。しかし、Cloud Build で`cloudbuild.yaml`で Docker イメージのビルド・プッシュ・デプロイを定義していても、デプロイ時に権限エラーが表示されることがあります。

これはユーザーの権限があっても、 Cloud Build が持つ権限に Cloud Run へのデプロイ権限が付与されていない場合に発生するもので、その場合は Cloud Shell から直接デプロイをし直す必要があります。

# トレースデータを確認する際
## Datadog Serverless Monitoring
Datadog での Google Cloud リソースの Serverless Monitoring は Cloud Functions と Cloud Run について、ベータ版で提供されています。(2023/5/21 現在)
しかし、今回のようにサイドカーデプロイを行った Cloud Run リソースについては、正しくメトリクスデータが取得できずに Datadog の Serverless ページ上には表示されません。

メトリクスデータを併せて取得したい場合は、[Datadog 公式の Cloud Run ドキュメント](https://docs.datadoghq.com/ja/serverless/google_cloud_run/)に基づいて、専用のエージェントを用いた Cloud Run シングルコンテナデプロイを行う必要があります。

:::message
この機能は 2023/5/21 時点ではベータ版として提供されています。
:::

## Datadog APM
Datadog APM は送信された Datadog Agent のメトリクス情報を Infrastructure タブに自動的に表示しますが、前述の通りサイドカーデプロイの場合このデータは取得できません。

![](https://storage.googleapis.com/zenn-user-upload/370b02e04a43-20230521.png)

ビルドやデプロイの際に指定したタグ変数は、主にスパンタグとして適用され、表示の下部に JSON 形式にパースされて表示されます。

また、Cloud Run によるルートスパンは表示されず、アプリケーションのリクエストやハンドラをルートスパンとして表示します。
