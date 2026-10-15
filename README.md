確認項目	InfluxDB 2.xの例
接続先IP・ポート	192.168.1.100:8086
Organization	my-org
Bucket	ship_data
API Token	読み取り専用Token
Query Language	Flux
ネットワーク	Grafanaから接続可能
重要なのは、GrafanaがInfluxDBに直接クエリを送信する点です。
GrafanaがInfluxDBからデータを取得してグラフに表示するため、Grafana側にデータを複製する必要はありません。ただし、Grafana内部の一時的なキャッシュ等は別です。
一点だけ確認したいのですが、InfluxDBは2.xですか、それとも3.xですか？
ここが分かれば、InfluxDB側で準備する設定項目を確定できます。



補足
Organization（オーガニゼーション）は、InfluxDB内でデータやユーザー、アクセス権限を管理するための「組織」という単位です。
my-org は説明用の仮の名前で、実際にはInfluxDBに登録されているOrganization名を指定します。

Grafana接続に必要な情報と、InfluxDB側での確認方法をまとめます。
No.	　　確認する情報	　　　　確認方法
1	InfluxDBのバージョン	　　CLI・管理画面
パワーシェルに以下入力
influxd version

2	サーバーIP・ホスト名	　　OS・サーバー設定

3	ポート番号	　　　　　　　 設定ファイル・起動設定

4	HTTP / HTTPS	          サーバー設定

5	Organization	          CLI・管理画面（2.x）

6	Bucket / Database	      CLI・管理画面

7	API Token / 認証情報	    管理画面・CLI

8	Query Language	        バージョン・利用機能

9	Measurement	            クエリ・管理画面

10	Grafanaからの通信可否	ネットワーク確認
