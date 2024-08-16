---
title: "事前準備"
---
# 準備
手元で動作確認できるようにTiDBをTiUPをセットアップするのを推奨します。

:::message
各回共通です。
:::


## 1. TiDBの環境構築 
### TiUP インストール
こちらの手順に従って、インストールしてください。
https://docs.pingcap.com/ja/tidb/stable/tiup-overview#install-tiup

:::message
Windows はWSL2 上のLinux環境でセットアップお願いします。
:::

- 1. インストール
```bash
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
```
- 2. source 反映
```bash
source ${your_shell_profile}
```
:::message
your_shell_profileは、適時 ~/.bashrc など環境にに変更してください。
また、不明点は公式を確認ください。
:::

- 3. tiup playground を実行
```bash
tiup playground
```
- 4. セットアップ完了したら、下記が出ているはず
```bash
🎉 TiDB Playground Cluster is started, enjoy!

Connect TiDB:    mysql --comments --host 127.0.0.1 --port 4000 -u root
TiDB Dashboard:  http://127.0.0.1:2379/dashboard
Grafana:         http://127.0.0.1:3000
```

Dashboard にアクセスして、Cluster が正常に動いているか確認しましょう


## 2. データを準備する
https://dev.mysql.com/doc/index-other.html

### 取得するデータ
![data](/images/tiug-2024-08-20/data.png)

:::message 
取得したらCluster に流し込んでください。
:::

```bash
mysql --comments --host 127.0.0.1 --port 4000 -u root < world.sql
```

## ここまで来たら準備は完了です。

## 番外編 Plan Replayer を実行する方法
Planを解析するツール
https://docs.pingcap.com/ja/tidb/stable/sql-plan-replayer

:::message
統計情報の取得に使用します。
:::
