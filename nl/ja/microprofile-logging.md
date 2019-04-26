---

copyright:
  years: 2019
lastupdated: "2019-04-04"

keywords: java logging, log level java, debug java, json log java, json log help, kibana liberty, liberty messages

subcollection: java

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}
{:tip: .tip}
{:note: .note}
{:important: .important}

# ロギング
{: #mp-logging}

MicroProfile アプリケーションでのロギングに推奨される方法は、Java の JSR-47 ロギング標準です。次のインポートから始めます。

```java
import java.util.logging.Level;
import java.util.logging.Logger;
```
{: codeblock}

次に、クラス・レベルでロガー・インスタンスのインスタンス化を行います。

```java
private static Logger logger = Logger.getLogger(PortfolioService.class.getName());
```
{: codeblock}

コードに、演算の前、演算の後、演算の中で `logger` インスタンスへの呼び出しを追加します。`Logger` インターフェースのメソッドにはそれぞれ、記録される情報の重要度、すなわち「レベル」を示す名前が付けられます。

```java
logger.info("Creating portfolio for "+owner);
logger.warning("Unable to send message to JMS provider. Continuing without notification of change in loyalty level.");
```
{: codeblock}

このログ・レベルは、これらのメッセージがコンソールに出力されるときに表示されます。

```
[INFO] Creating portfolio for John

[WARNING] Unable to send message to JMS provider. Continuing without notification of change in loyalty level.
```
{: screen}

ログ・レベルによって、ユーザーはアプリケーションが書き込むログを動的に選択できる柔軟性を持つことができます。これにより、ユーザーは概要のアプリケーション状態と詳細なデバッグ内容の両方を記述するログ・コードを事前に作成しておき、それが必要になるまではそうした冗長なデバッグ内容をフィルターで除外することができます。ログ・レベル `info` が一般に最小の出力レベルで、その後、`fine`、`finer`、`finest`、`debug` と続きます。

ログ・エントリーに複数のコード行が必要な場合や、文字列の連結など費用のかかる演算が含まれる場合は、ログ・レベルを使用可能にできるかどうかを判別するためのテストでそれらを保護することを検討する必要があります。こうすることで、アプリケーションが最終的には除外されることになるログ・メッセージの作成に貴重な時間を割かないようにすることができます。以下の例では、メッセージ出力を作成する前に、目的とするログ・レベル `fine` を有効にすることを検証しています。

```java
if (logger.isLoggable(Level.FINE)) {
    StringWriter writer = new StringWriter();
    exception.printStackTrace(new PrintWriter(writer));
    logger.fine(writer.toString());
}
```
{: codeblock}

ログ・レベル、および構成の詳細について詳しくは、[WebSphere Liberty のトラブルシューティング・ガイド](https://www.ibm.com/support/knowledgecenter/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/rwlp_logging.html){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン") および [java.util.logging API 資料](https://docs.oracle.com/javase/8/docs/api/java/util/logging/package-summary.html){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン") を参照してください。

## Liberty を使用した JSON ロギング
{: #mp-json-logging}

Liberty は JSON 形式のロギングをサポートしています。これを有効にした場合、ログ・メッセージは JSON 形式でコンソールに書き込まれます。`server.xml` で以下のロギング・スタンザを使用して、これを有効にします。

```xml
<logging consoleLogLevel="INFO" consoleFormat="json" consoleSource="message,trace,accessLog,ffdc" />
```
{: codeblock}

上記のコンソール・リソースのリストに `accessLog` は含まれていますが、これらのログがコンソールに書き込まれる前に HTTP アクセス・ロギングを有効にする必要があることに気を付けてください。以下のスニペットは、`server.xml` で `accessLogging` サブエレメントを `httpEndpoint` エレメントに追加する方法を示しています。

```xml
<httpEndpoint id="defaultHttpEndpoint" host="\*" httpPort="9080" httpsPort="9443">
  <accessLogging
    filepath="${server.output.dir}/logs/http_defaultEndpoint_access.log"
    logFormat='%h %u %t "%r" %s %b %D %{User-agent}i'>
  </accessLogging>
</httpEndpoint>
```
{: codeblock}

ここで、このコードをアプリケーションに追加する際に、

```java
if (logger.isLoggable(Level.AUDIT)) {
    logger.audit("Initialization complete");
}
```

ログに、以下のような出力が表示されます。

```json
{ "type":"liberty_message",
  "host":"trader-54b4d579f7-4zvzk",
  "ibm_userDir":"\/opt\/ol\/wlp\/usr\/",
  "ibm_serverName":"defaultServer",
  "ibm_datetime":"2018-06-21T19:23:21.356+0000",
  "ibm_threadId":"00000028",
  "module":"com.trader.Main",
  "loglevel":"AUDIT",
  "message":"Initialization complete"}
```
{: codeblock}

### JSON ログ出力の読み取り
{: #mp-json-log-output}

完全な JSON 出力はログ・ストレージとログ検索に役立ちますが、読み取るのはそれほど容易ではありません。`kubectl` を使用して、端末ウィンドウでログの内容を確認する必要があります。幸い、これに役立つ `jq` という名前のコマンド行ツールがあります。

`jq` ではユーザーがログ内容を絞り込み、必要なフィールド (複数可) に焦点を当てることができます。例えば、`message` フィールドだけを調べ、その他のすべてを除外する場合は、以下のようにします。

```
kubectl logs trader-54b4d579f7-4zvzk -n stock-trader -c trader | grep message | jq .message -r
```
{: pre}

Liberty には、JSON 形式ではないプリミティブ型のコンソール・メッセージが多少あります。`grep` を使用することで、`jq` がメッセージ・フィールドを含む行だけを解析するようにできます。

## 追加フィーチャー
{: #mp-log-features}

`logger.info` や `logger.fine` をいつ使うかなど、ログ・レベルを使用する指針は、各組織やプロジェクトで決定する事項になります。ただし、ほとんどすべてのプロジェクトでこれらのインターフェースは必要であり、有益だと考えられます。

ベスト・プラクティスの 1 つとして、`server.xml` の関連する各フィールドで環境変数を使うという手段があります (環境変数は Kube 構成マップやシークレットでポッドに渡します)。これによって、Docker イメージをビルドし直して再デプロイする必要なく、ロギング構成を変更できます。

例えば、環境変数を使用して非常にきめ細かいログ属性を設定するには、前出の例の以下のスタンザを変更します。

```xml
<logging consoleLogLevel="INFO" consoleFormat="json" consoleSource="message,trace,accessLog,ffdc" />
```
{: codeblock}

これを、次のように変更します。

```xml
<logging consoleLogLevel="${env.LOG_LEVEL}" consoleFormat="${env.LOG_FORMAT}" consoleSource="${env.LOG_SOURCE}" />
```
{: codeblock}

さらに代替方法として、`WLP_LOGGING_CONSOLE_FORMAT` 環境変数を使用する方法があります。これについては、[ロギングおよびトレースの資料](https://www.ibm.com/support/knowledgecenter/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/rwlp_logging.html){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン") を参照してください。これは、上記の例に似ています。`WLP_LOGGING_CONSOLE_FORMAT` 変数を `basic` (デフォルト) または `json` に設定できます。

## Liberty の Kibana ダッシュボード
{: #liberty-kibana}

新しい JSON ロギング・フィーチャーと並んで、Liberty には事前構築された Kibana ダッシュボードが用意されています ([GitHub からダウンロードできます](https://www.ibm.com/support/knowledgecenter/en/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/twlp_icp_json_logging.html){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン"))。インストールするには、リンクの指示に従ってください。2 つの新しいダッシュボードが利用可能です。

![Kibana ダッシュボード](images/microprofile-logging-image4.png "Kibana ダッシュボード")

問題判別のためにこのダッシュボードをクリックすると、次のように表示されます。

![Kibana ダッシュボードの詳細](images/microprofile-logging-image5.png "Kibana ダッシュボードの詳細")

このダッシュボードは対話式です。例えば、**「Liberty メッセージ (Liberty Message)」**ウィジェットの凡例で **INFO** をクリックすると、以下の**「Liberty メッセージ検索 (Liberty Messages Search)」**ウィジェットでフィルター処理が行われ、`loglevel=INFO` メッセージだけを取り出すことができます。ダッシュボードは Liberty ベースのすべてのマイクロサービスのログ・データを統合し、他のシステム・ログはフィルター処理で除外します。

さらに、Liberty Helm チャートと関連付けられた別の Kibana ダッシュボードや Grafana ダッシュボードもあります。これらは [Libertyクラウド・パックの拡張機能](https://github.com/IBM/charts/tree/master/stable/ibm-websphere-liberty/ibm_cloud_pak/pak_extensions/dashboards){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン") として使用可能です。

## 次の手順
{: #mp-logging-next-steps notoc}

アペンダー、ログ・レベル、構成の詳細を指定してログ・メッセージをカスタマイズする方法について詳しくは、公式サイトの[Spring Boot 解説書のロギングに関する情報](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-logging.html){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン") を参照してください。

各デプロイメント環境でのログの表示方法について詳しくは、次の各ページを参照してください。

* [Kubernetes のログ](https://kubernetes.io/docs/concepts/cluster-administration/logging/){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン")
* [{{site.data.keyword.openwhisk}} のログおよびモニタリング](/docs/openwhisk?topic=cloud-functions-openwhisk_logs#openwhisk_logs)
* [{{site.data.keyword.cloud_notm}} Log Analysis](/docs/services/CloudLogAnalysis?topic=cloudloganalysis-log_analysis_ov#log_analysis_ov)
* [{{site.data.keyword.cloud_notm}} Private ELK スタック](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_2.1.0.2/manage_metrics/logging_elk.html){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン")