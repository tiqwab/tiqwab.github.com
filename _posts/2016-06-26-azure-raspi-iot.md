---
layout: post
title:  "Microsoft Azure体験"
tags: "azure, raspberrypi, iot, d3"
comments: true
---

先日ふとMicrosoft Azureを試してみようと思い1ヶ月の無料サブスクリプション期間内で色々遊んでみました。クラウドを触るのは初めてでしたが、こういった自分で好きに使える環境があるのは便利で今後も使用することがありそうなので、この1か月間で行ったことを備忘録がてらメモしておこうと思います。  

今回やったことは以下のようにRaspberry Piによる室温測定からデータのグラフ化までの流れです。

1. [Raspberry Piによる室温測定](#anchor1)
2. [Raspberry PiからAzure IoT Hubへの接続準備](#anchor2)
3. [データ送信テスト](#anchor3)
4. [測定データのAzure SQL Databaseへの保存](#anchor4)
5. [測定データ取得用APIの作成](#anchor5)
6. [d3.jsを使用したグラフ描写](#anchor6)

---

<a id="anchor1"></a>

### 1. Raspberry Piによる室温測定

IoT界のHello Worldともいうべき室温測定を行いました。Raspberry Pi3と温湿度・気圧センサBME280を使用しています。手順は以下のように。

1. Raspberry PiのI2C有効化設定
2. 温度センサとI2C接続、測定値を扱いやすいデータに変換

#### Raspberry PiのI2C有効化設定

[Raspberry Pi の I2C を有効化する方法 (2015年版)][2]を参考にしました。ただ、一部足りない手順もあり、実際に行った手順の要約は以下のようになっています。

- `sudo raspi-config`からRaspberry PiのI2C有効化
- `/etc/modules`に`snd-bcm2835`, `i2c-dev`追加
- `sudo apt-get install i2c-tools python-smbus`で必要なコマンド、ライブラリ取得

#### 温度センサとI2C接続、測定値を扱いやすいデータに変換

[Raspberry Pi 2で温湿度・気圧センサのBME280をPythonから使う][1]を参考にしました。温度センサとRaspberyy Piとの配線方法は以下の通り(引用)。

> SDI      (BME280) -> GPIO2 P03 (Raspberry Pi SDA1)  
> SCK      (BME280) -> GPIO3 P05 (Raspberry Pi SCL1)  
> GND,SDO  (BME280) -> GND  P09 (Raspberry Pi)  
> Vio,CSB  (BME280) -> 3.3v P01 (Raspberry Pi)  

測定した生データの数値変換はスイッチサイエンス様が提供されているプログラムをもとにしましたが、JSONデータとして取得したかったため一部変更しました。

<a id="anchor2"></a>

### 2. Raspberry PiからAzure IoT Hubへの接続準備

Raspberry PiからAzure上のサーバにデータを送信するためのセットアップを行いました。[Azure IoT Hub for Java の使用][3]を参考にして以下の作業を行っています。

1. Azure IoT Hubの作成
2. デバイスIDの作成

デバイスIDの作成用のスクリプトは以下のよう(Groovy)。Microsoftからもそれ専用のプログラムが提供されているようなのですが、わざわざそれをダウンロードして行うまでもないように思います。登録されたデバイスに関する情報は、Azureポータル上からも確認できます。

```java
@Grab('com.microsoft.azure.iothub-java-client:iothub-java-service-client:1.0.6')
import com.microsoft.azure.iot.service.sdk.*
import com.microsoft.azure.iot.service.exceptions.*

// ARGS[0]: connectionString, ARGS[1]: deviceId
RegistryManager registryManager = RegistryManager.createFromConnectionString(args[0])
Device device = Device.createFromId(args[1], null, null)

try {
    device = registryManager.addDevice(device)

    println "Device created: ${device.getDeviceId()}"
    println "Device key: ${device.getPrimaryKey()}"
} catch (IotHubException e) {
    e.printStackTrace()
} catch (IOException e) {
    e.printStackTrace()
}
```

<a id="anchor3"></a>

### 3. データ送信テスト

はじめはインストール済みのPythonでテストを行うつもりでしたが、Python環境でのAzure SDKの準備が何だか面倒に見えたため、同様にインストール済みであったJava(というよりgroovy)で実行することにしました。

```bash
# Install libraries for groovy
$ wget http://central.maven.org/maven2/org/codehaus/groovy/groovy-all/2.4.7/groovy-all-2.4.7.jar
$ wget http://central.maven.org/maven2/org/apache/ivy/ivy/2.4.0/ivy-2.4.0.jar

# Shell script to execute groovy
$ cat ./send.sh
\#!/bin/sh
JAVA_OPTS="-Dgroovy.source.encoding=UTF-8 -Dfile.encoding-UTF-8"
java $JAVA_OPTS -cp ./groovy-all-2.4.7.jar:./ivy-2.4.0.jar groovy.ui.GroovyMain SendMessage.groovy "$1" "$2"

# Send message
$ ./send.sh
```

メッセージ送信はGroovyで以下のように。

```java
@Grab('com.google.code.gson:gson:2.3.1')
import com.google.gson.Gson
@Grab('com.microsoft.azure.iothub-java-client:iothub-java-device-client:1.0.7')
import com.microsoft.azure.iothub.*

/*
 * ARGS[0]: connectionString for device
 * ARGS[1]: messageContent
 */

String connectionString = args[0]
IotHubClientProtocol protocol = IotHubClientProtocol.HTTPS

DeviceClient client = new DeviceClient(connectionString, protocol)
client.open()

try {
    String content = args[1]
    println content

    Message msg = new Message(content)

    Object lockobj = new Object()
    client.sendEventAsync(msg, { status, context ->
        println "IoT Hub responded to message with status ${status.name()}"
        if (context != null) {
            synchronized (context) {
                context.notify()
            }
        }
    }, lockobj)

    synchronized (lockobj) {
        lockobj.wait()
    }
} finally {
    client.close()
}
```

<a id="anchor4"></a>

### 4. 測定データのAzure SQL Databaseへの保存

1. DBの準備
2. Stream Analyticsの準備
3. 実行

#### DBの準備

Azure Iot Hubへデータを送信するだけでは何もできないため、受信したデータをDBへ保存する環境を整えます。当初は[Azure DocumentDB][4]というNoSQL DBを使用するつもりでしたが、ネットで調べもあまり使用例が見当たらないこと、料金が最低でも月額2500円？ということからお試しで使うには敷居が高く感じたので、[Azure SQL Database][5]を使用しました。Azure SQL Databaseはストレージ容量とDTU(Database Transaction Unit)と呼ばれる秒間あたりの処理能力で料金が決定されるようで、ミニマムなものなら月額600円弱となっているみたいです。(2016/06/17現在)  

実際のDB作成はAzureポータル上で。(やや画面が古いですが、[Azure SQL データベース作成から、接続まで][6]を参考にしました。)

#### Stream Analyticsの準備

IoT Hubで受信したデータをSQL Databaseへ入れてあげるために、[Stream Analytics][7]ジョブを作成しました。Stream Analyticsはデバイスから受信したデータをリアルタイムで高スループットに処理するAzure上のジョブとのことです。Input, Output先を指定し、またSQL-likeなクエリを記述することでその間で簡単なデータの加工を行うことができるようです。  

Azureのポータル上で以下のような設定を行いました。

- inputに事前に作成したIoT Hubを指定
- outputに事前に作成したSQL Databaseを指定
- SQL-likeなクエリによるデータのマッピング、加工
    - `FROM`でinputに指定したendpoint名を記述
    - `INTO`でoutputに指定したendpoint名を記述
    - `SELECT`でinput内のカラム(JSONだとキーというべきか)を指定
      - input内とoutput内で名称が違うカラムが存在する場合は`AS`を使用

```
SELECT
    temperature,
    measurementTime AS measurement_time
INTO
    [output-temperature]
FROM
    [input-temperature]
```

#### 実行

DB上に適当なテーブルを作成したのちに、Stream Analyticsジョブを起動、Raspberry Piからメッセージを送信し動作確認。(`read_t.py`は室温測定値をJSON形式として出力するためのプログラム)

```bash
# Execute by cron
# Show Content of message and response from IoT Hub
$ cd ~/Workspace/azure && ./send.sh '<connectionString>' "`python ~/Workspace/BME280/Python27/read_t.py`"

{"measurementTime": "2016-06-18 12:11:22", "pressure": "1010.48", "temperature": "28.79", "humidity": "41.62"}
IoT Hub responded to message with status OK_EMPTY

# Confirm insertion to the table
# Use Groovy to connect and query the stored data
$ \`cat db-connect-script.txt\`
[[temperature:28.780, measurement_time:2016-06-18 12:11:01.0], [temperature:28.790, measurement_time:2016-06-18 12:11:22.0]]
```

<a id="anchor5"></a>

### 5. 測定データ取得用APIの作成

データをDBに保存するだけでは面白くないので、Node.jsでWeb APIを作成し、Azure Web Appでデプロイしました。Node.jsによるAzure Web Appの作成には以下の資料がとても役立ちました。

- [Azure App Service での Node.js Web アプリの使用][8]
- [Azure WebアプリにNode.js expressアプリをGitからデプロイ][9]

以下使用時に気づいた、思ったことメモ

- デプロイ元としてローカルのgitリポジトリやgithub上のリポジトリ等を指定することができる。
  - 少し試してみる分にはとても便利。
- `server.js`があると自動的に実行してくれる。
- `process.env.port`でAzureがセッティングするアプリケーション用のportが取得できる。
- `package.json`があればデプロイ時に依存パッケージをinstallしてから実行してくれる。
- Azure上から[該当Web App] -> [設定] -> [アプリケーションの設定] -> [アプリ設定]によりアプリケーション実行時の環境変数を設定できる。今回はDBへの接続情報等をここに記載した。
- CORS(Cross Origin Resource Sharing)の設定はAzure上で行える。

APIの実装自体は特に目新しい(Azureに特有な)点は無いはず。以下を参考にさせて頂きました。

- [Node.js + Express 4.x + MongoDB(Mongoose)でRESTfulなjsonAPIサーバの作成を丁寧に解説する][10]
- [node-mssql][11]

本筋に関係ない話ですが、SQL Serverからデータを取得するのに使用した`mssql`というパッケージで少し苦戦しました。DBにローカル基準での値を入れていたのですが、標準だとdatetimeカラムの値をUTCとみなして取得してくるようです。まあタイムゾーンを持たせないのならUTCで入れればいいという話ですね。

<a id="anchor6"></a>

### 6. d3.jsを使用したグラフ描写

上で作成したAPIを使用して、Webブラウザ上で測定データを見れるようにしました。素のデータをそのまま表示するのもなんなので、`d3.js`を使ってグラフとして表示させました。

<img
  src="/images/azure-raspi-iot/rt-api.png"
  title="graph of room temperature"
  alt="The image of rt-api"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

日ごと、月ごとのデータを表示させたりと動きも入れているのですが、いかんせんデータが平坦すぎて面白みのないことこの上ないです。ラズパイさん自身が熱いのか部屋に熱気がこもっているのか...  

これでとりあえずやろうと思っていたことは一通りでき、ちょうど1か月となりました。

### 今後確認したいこと

- このぐらいAzureを使用して結局月額だといくらになるのか。
  - 正直お金周りのことがわかりにくい気が
  - 値段の体系はもちろん公開されているし、Azureポータル上からも専用の画面はある
  - ただ思っていた値段と違ったり、よくわからないのがリソースにいたり(無料サブスクリプションとして勝手に入れられているのか？)、ちょっと理解できていない点もある

[1]: http://qiita.com/masato/items/027e5c824ae75ab417c1
[2]: https://blog.ymyzk.com/2015/02/enable-raspberry-pi-i2c/
[3]: https://azure.microsoft.com/ja-jp/documentation/articles/iot-hub-java-java-getstarted/
[4]: https://azure.microsoft.com/ja-jp/services/documentdb/
[5]: https://azure.microsoft.com/ja-jp/pricing/details/sql-database/
[6]: http://qiita.com/alingogo/items/848df1ae239f258a7500
[7]: https://azure.microsoft.com/ja-jp/services/stream-analytics/
[8]: https://azure.microsoft.com/ja-jp/documentation/articles/app-service-web-nodejs-get-started/
[9]: http://blog.hrendoh.com/deploy-node-js-express-app-to-azure-website-using-git/
[10]: http://qiita.com/shopetan/items/58a62a366aac4f5faa20
[11]: https://github.com/patriksimek/node-mssql
