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

# 記載
{: #mp-logging}

使用 MicroProfile 應用程式進行記載的建議方法是 Java 的 JSR-47 記載標準。從下列 import 開始：

```java
import java.util.logging.Level;
import java.util.logging.Logger;
```
{: codeblock}

其次，實例化類別層次中的日誌程式實例：

```java
private static Logger logger = Logger.getLogger(PortfolioService.class.getName());
```
{: codeblock}

在程式碼中，於作業之前、之後以及當中，新增呼叫至 `logger` 實例。`Logger` 介面的方法本身被命名為指出要記載之資訊的重要性或「層次」。

```java
logger.info("Creating portfolio for "+owner);
logger.warning("Unable to send message to JMS provider. Continuing without notification of change in loyalty level.");
```
{: codeblock}

當這些訊息輸出至主控台時，會顯示記載層次。

```
[INFO] Creating portfolio for John

[WARNING] Unable to send message to JMS provider. Continuing without notification of change in loyalty level.
```
{: screen}

記載層次可讓您彈性選擇應用程式要寫入的日誌。這可讓您寫入日誌碼，以說明高階應用程式狀態和詳細除錯內容，但會過濾掉更詳細的除錯內容，直到您需要它為止。記載層次 `info` 通常是最低的輸出層次，後面接著 `fine`、`finer`、`finest` 及 `debug`。

當日誌項目需要多行程式碼，或涉及諸如字串連結之類的昂貴作業時，您應該考慮使用測試來保護它們，以判定是否啟用記載層次。這樣可確保您的應用程式不會花費重要的時間，來建置最終只會被過濾掉的日誌訊息。下列範例會在嘗試建置訊息輸出之前，先驗證是否已啟用想要的日誌層次 `fine`。

```java
if (logger.isLoggable(Level.FINE)) {
    StringWriter writer = new StringWriter();
    exception.printStackTrace(new PrintWriter(writer));
    logger.fine(writer.toString());
}
```
{: codeblock}

如需記載層次和配置詳細資料的相關資訊，請參閱 [WebSphereLiberty 疑難排解手冊](https://www.ibm.com/support/knowledgecenter/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/rwlp_logging.html){: new_window} ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示")，以及 [java.util.logging API 文件](https://docs.oracle.com/javase/8/docs/api/java/util/logging/package-summary.html){: new_window} ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示")。

## 使用 Liberty 的 JSON 記載
{: #mp-json-logging}

Liberty 支援 JSON 格式的記載。啟用時，日誌訊息會以 JSON 格式寫入主控台。請在 `server.xml` 中使用下列記載段落來啟用此項目：

```xml
<logging consoleLogLevel="INFO" consoleFormat="json" consoleSource="message,trace,accessLog,ffdc" />
```
{: codeblock}

請注意，當上述主控台來源清單中包含 `accessLog` 時，必須先啟用 HTTP 存取記載，才能將那些日誌寫入主控台。下列 Snippet 顯示如何將 `accessLogging` 子元素新增至 `server.xml` 中的 `httpEndpoint` 元素：

```xml
<httpEndpoint id="defaultHttpEndpoint" host="\*" httpPort="9080" httpsPort="9443">
  <accessLogging
    filepath="${server.output.dir}/logs/http_defaultEndpoint_access.log"
    logFormat='%h %u %t "%r" %s %b %D %{User-agent}i'>
  </accessLogging>
</httpEndpoint>
```
{: codeblock}

現在，當您將此程式碼新增至應用程式時：

```java
if (logger.isLoggable(Level.AUDIT)) {
    logger.audit("Initialization complete");
}
```

您會在日誌中找到如下的內容：

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

### 讀取 JSON 日誌輸出
{: #mp-json-log-output}

完整 JSON 輸出對於日誌儲存和搜尋非常有用，但不容易閱讀。您可能需要使用 `kubectl`，在終端機視窗中檢查日誌的內容。幸好，有一個名稱為 `jq` 的指令行工具可以協助您。

`jq` 可讓您過濾掉並著重於您需要的欄位。例如，若您想要只看到 `message` 欄位，並過濾掉其他所有內容：

```
kubectl logs trader-54b4d579f7-4zvzk -n stock-trader -c trader | grep message | jq .message -r
```
{: pre}

Liberty 具有一些不是 JSON 格式的基本主控台訊息。使用 `grep` 可確保 `jq` 只會剖析包含訊息欄位的字行。

## 其他特性
{: #mp-log-features}

每個組織或專案必須針對使用記載層次的準則（例如，何時使用 `logger.info` 或 `logger.fine`）做出決策。不過，我們預期這些介面在幾乎所有專案中都是必要的，而且很有用。

最佳作法是在 `server.xml` 的每個相關欄位中使用環境變數（透過 Kubernetes ConfigMaps 或 Secrets 來進行的）。這可讓您變更記載配置，而不需要重建及重新部署 Docker 映像檔。

例如，若要使用環境變數來設定精細記載屬性，您將變更前一個範例中的段落：

```xml
<logging consoleLogLevel="INFO" consoleFormat="json" consoleSource="message,trace,accessLog,ffdc" />
```
{: codeblock}

至如下所示的內容：

```xml
<logging consoleLogLevel="${env.LOG_LEVEL}" consoleFormat="${env.LOG_FORMAT}" consoleSource="${env.LOG_SOURCE}" />
```
{: codeblock}

另一個替代方案是使用 `WLP_LOGGING_CONSOLE_FORMAT` 環境變數，如[記載和追蹤文件](https://www.ibm.com/support/knowledgecenter/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/rwlp_logging.html){: new_window} ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示") 所述。這類似於上述範例：您可以將 `WLP_LOGGING_CONSOLE_FORMAT` 變數設為 `basic`（預設值）或 `json`。

## Liberty 的 Kibana 儀表板
{: #liberty-kibana}

隨著新的 JSON 記載特性，Liberty 會提供預先建置的 Kibana 儀表格，[您可以從 GitHub 下載](https://www.ibm.com/support/knowledgecenter/en/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/twlp_icp_json_logging.html){: new_window} ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示")。請遵循鏈結中的指示來安裝它們。有兩個新的儀表板可用：

![Kibana 儀表板](images/microprofile-logging-image4.png "Kibana 儀表板"){: caption="圖 1. Kibana 儀表板" caption-side="bottom"}

當您按一下儀表板來進行問題判斷時，您可以看到這個項目：

![Kibana 儀表板詳細資料](images/microprofile-logging-image5.png "Kibana 儀表板詳細資料"){: caption="圖 2. Kibana 儀表板詳細資料" caption-side="bottom"}

儀表板是互動式的。例如，若您在 **Liberty 訊息**小組件的圖註中，按一下**資訊**，則下面的 **Liberty 訊息搜尋**小組件會自行過濾為只有 `loglevel=INFO` 訊息。儀表板會聯合來自所有 Liberty 型微服務的日誌資料，並過濾掉其他系統日誌。

另外還有其他 Kibana 和 Grafana 儀表板與 Liberty helm 圖表相關聯。這些儀表板可作為 [Liberty Cloud Pak](https://github.com/IBM/charts/tree/master/stable/ibm-websphere-liberty/ibm_cloud_pak/pak_extensions/dashboards){: new_window} ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示") 的延伸。

## 後續步驟
{: #mp-logging-next-steps notoc}

如需使用附加器、記載層次及配置詳細資料來自訂日誌訊息的相關資訊，請參閱官方的 [Spring Boot 記載參考資料](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-logging.html){: new_window} ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示")。

進一步瞭解在每一個部署環境中檢視日誌：

* [Kubernetes 日誌](https://kubernetes.io/docs/concepts/cluster-administration/logging/){: new_window} ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示")
* [{{site.data.keyword.openwhisk}} Logs & Monitoring](/docs/openwhisk?topic=cloud-functions-openwhisk_logs#openwhisk_logs)
* [{{site.data.keyword.cloud_notm}} Log Analysis](/docs/services/CloudLogAnalysis?topic=cloudloganalysis-log_analysis_ov#log_analysis_ov)
* [{{site.data.keyword.cloud_notm}} Private ELK stack](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_2.1.0.2/manage_metrics/logging_elk.html){: new_window} ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示")