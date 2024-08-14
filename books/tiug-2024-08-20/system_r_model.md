---
title: "Planner のSystem Rを読む"
---
# System Rおさらい

System R の構造がTiDB ではどう表現されているのかを見ましょう


## 前回も使った参考資料
https://www.docswell.com/s/kumagi/KENNPE-selinger-optimizer

### System R のOptimizer 概要
![SystemRのOptimizer概要](/images/tiug-2024-08-20/system_r.png)


### Left Deep Tree
![Left Deep Tree](/images/tiug-2024-08-20/left_deep_tree.png)
:::message alert
事前にTiDBの実装箇所が特定できなかった。
:::


### DPを用いた網羅的探索 
![DPを用いた網羅的探索](/images/tiug-2024-08-20/DPGraph.png)

### Interesting Order 
![Interesting Order](/images/tiug-2024-08-20/interesting_order.png)

:::message alert
事前にTiDBの実装箇所が特定できなかった。
:::

### コスト計算
![Cost Model](/images/tiug-2024-08-20/cost_model.png)
