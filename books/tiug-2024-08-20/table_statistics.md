---
title: "テーブル統計"
---

Trace ログ ここから追うt
https://docs.pingcap.com/tidb/stable/sql-statement-trace

## コンパイラからOptimzer を読む
- コードの場所
- 流れがよくわからないので、Trace でselect 分をおう
- exec


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



pkg/planner/funcdep/doc.go

FDSet ,FDEdge ってきたが、これがツリーを構成する各ノード？


Function Dependence に関する参考資料
[Exploiting Functional Dependence in Query Optimization](https://cs.uwaterloo.ca/research/tr/2000/11/CS-2000-11.thesis.pdf)

