---
title: "はじめに"
---

# はじめに
## 話の流れ
- 本書の構成
- 本書の目的
- 本書の対象読者
- 本書の前提条件
- 本書の参考文献
- 本書の謝辞
- 本書の執筆者
- 本書のライセンス
- 本書のお問い合わせ
- 本書の更新情報
- 本書の目次
- 本書の索引
- 本書の付録
  -
## 準備

https://docs.pingcap.com/ja/tidb/stable/tiup-overview
nakagawa@nakagawasatoshinarinoMac-mini zenn % curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
Dload  Upload   Total   Spent    Left  Speed
100 5365k  100 5365k    0     0  1763k      0  0:00:03  0:00:03 --:--:-- 1763k
WARN: adding root certificate via internet: https://tiup-mirrors.pingcap.com/root.json
You can revoke this by remove /Users/nakagawa/.tiup/bin/7b8e153f2e2d0928.root.json
Successfully set mirror to https://tiup-mirrors.pingcap.com
Detected shell: zsh
Shell profile:  /Users/nakagawa/.zshrc
/Users/nakagawa/.zshrc has been modified to add tiup to PATH
open a new terminal or source /Users/nakagawa/.zshrc to use it
Installed path: /Users/nakagawa/.tiup/bin/tiup
===============================================
Have a try:     tiup playground
===============================================
nakagawa@nakagawasatoshinarinoMac-mini zenn % tiup playground
zsh: command not found: tiup
nakagawa@nakagawasatoshinarinoMac-mini zenn % source /Users/nakagawa/.zshrc

tiup playground


🎉 TiDB Playground Cluster is started, enjoy!

Connect TiDB:    mysql --comments --host 127.0.0.1 --port 4000 -u root
TiDB Dashboard:  http://127.0.0.1:2379/dashboard
Grafana:         http://127.0.0.1:3000



mysql --comments --host 127.0.0.1 --port 4000 -u root < ./books/tiug-2024-07-16/world-db/world.sql


set @@global.tidb_enable_auto_analyze = OFF;


Trace ログ ここから追うt
https://docs.pingcap.com/tidb/stable/sql-statement-trace

## Planner のSystem Rモデル
- bonnenさんの
- https://www.docswell.com/s/bohnen/K4V3E6-tidb-query-execution#p16
## コンパイラからOptimzer を読む
- コードの場所
- 流れがよくわからないので、Trace でselect 分をおう
- exec

Plan Builder
pkg/planner/core/planbuilder.go

## 論理最適化を読む
## 物理プラン生成を読む
## コスト計算を読む
## 結合順序の最適化を読む
## アクセスパス選択を読む
## planner/core/optimizer.go
次回の輪読会では、System Rモデルの理論的背景を押さえつつ、TiDB Development Guideで言及されている重要なソースコード部分を実際に見ていきます。以下は、焦点を当てる予定のコード部分です：

プランナーの全体構造

ファイル: planner/core/optimizer.go
関数: func Optimize(ctx context.Context, sctx sessionctx.Context, node ast.Node, is infoschema.InfoSchema) (Plan, types.NameSlice, error)
説明: この関数はプランナーのエントリーポイントで、System Rモデルの最適化プロセスがどのように開始されるかを示しています。

## 読むソースコード
pkg/planner/optimize.go
pkg/planner/core/optimizer.go

論理最適化ルール

ファイル: planner/core/optimizer.go
変数: var optRuleList []logicalOptRule
説明: この変数は論理最適化ルールのリストを定義しており、System Rモデルの各種変換規則がどのように実装されているかを示しています。


物理プラン生成

ファイル: planner/core/find_best_task.go
関数: func (p *baseLogicalPlan) findBestTask(prop *property.PhysicalProperty) (task, error)
説明: この関数は、System Rモデルのコストベース最適化がどのように実装されているかを示しています。


コスト計算

ファイル: planner/core/cost_model.go
説明: このファイルには、System Rモデルのコスト計算に関連する様々な関数が含まれています。具体的にどの関数を見るかはセッション中に決定します。


結合順序の最適化

ファイル: planner/core/rule_join_reorder.go
説明: このファイルには、System Rモデルの核心的なアイデアの一つである結合順序の最適化に関する実装が含まれています。


アクセスパス選択

ファイル: planner/core/find_best_task.go
関数: func (ds *DataSource) findBestTask(prop *property.PhysicalProperty) (task, error)
説明: この関数は、System Rモデルのアクセスパス選択がどのように実装されているかを示しています。



準備すること：

進行役：上記のコード部分を詳細に分析し、System Rモデルの概念とどのように対応しているかを説明できるようにする。
参加者：TiDB Development Guideの関連部分を読み、上記のファイルに目を通しておく。



## テーブル統計

pkg/statistics

分析