---
title: "DPを用いた網羅的探索"
---
# DPを用いた網羅的探索
## まずは論理最適化
![論理最適化](/images/tiug-2024-08-20/logic_optimization.png)


[TiDB開発ガイド ルールベースの最適化](https://pingcap.github.io/tidb-dev-guide/understand-tidb/rbo.html0

## 論理最適化によるAST の書き換え

- サブクエリ関連の最適化
- カラムの剪定
- 相関サブクエリの非相関
- 最大/最小を排除
- Predicate Push Down
- パーティションのプルーニング
- TopN と Limit 演算子のプッシュダウン
- 結合したテーブルの再配置 **(今回はここのJoin Rule だけ見ていきます。)**
- ウィンドウ関数から TopN または Limit を導出する

## pkg/planner/core/optimizer.go
:::message
ここを具体的に実行していく流れのようです。
:::
```Go
var optRuleList = []logicalOptRule{
	&gcSubstituter{},
	&columnPruner{},
	&resultReorder{},
	&buildKeySolver{},
	&decorrelateSolver{},
	&semiJoinRewriter{},
	&aggregationEliminator{},
	&skewDistinctAggRewriter{},
	&projectionEliminator{},
	&maxMinEliminator{},
	&constantPropagationSolver{},
	&ppdSolver{},
	&outerJoinEliminator{},
	&partitionProcessor{},
	&collectPredicateColumnsPoint{},
	&aggregationPushDownSolver{},
	&deriveTopNFromWindow{},
	&predicateSimplification{},
	&pushDownTopNOptimizer{},
	&syncWaitStatsLoadPoint{},
	&joinReOrderSolver{},           // DP でのJoin管理などはここでやっている
	&columnPruner{}, // column pruning again at last, note it will mess up the results of buildKeySolver
	&pushDownSequenceSolver{},
	&resolveExpand{},
}
```

## Join 並べ替えソルバー
`pkg/planner/core/rule_join_reorder.go` の`joinReOrderSolver`からコールしている
`optimizeRecursive` で、DP による最適化を行っている。下記は処理中にDPソルバーかGreedyソルバーか選んでJoinを実行している

### 開発ガイド(Join Reorder )
> Join reorder tries to find the most efficient order to join several tables together. In fact, it's not a rule-based optimization. It makes use of statistics to estimate row counts of join results. We put join reorder in this stage out of convenience.

結合順序の変更は、複数のテーブルを結合する最も効率的な順序を見つけようとします。実際、これはルールベースの最適化ではありません。統計情報を使用して結合結果の行数を推定します。利便性のため、結合順序の変更をこのステージに配置します。

> Currently, we have implemented two join reorder algorithms: greedy and dynamic programming. The dynamic programming one is not mature enough now and is disabled by default. We focus on the greedy algorithm here.

現在、貪欲法と動的プログラミングの 2 つの結合並べ替えアルゴリズムを実装しています。動的プログラミングのアルゴリズムはまだ十分に成熟していないため、デフォルトでは無効になっています。ここでは貪欲法アルゴリズムに焦点を当てます。

> There are three files relevant to join reorder. rule_join_reorder.go contains the entry and common logic of join reorder. rule_join_reorder_dp.go contains the dynamic programming algorithm. rule_join_reorder_greedy.go contains the greedy algorithm.

結合順序付けに関連するファイルは 3 つあります。rule_join_reorder.go には、結合順序付けのエントリと共通ロジックが含まれています。rule_join_reorder_dp.go には、動的プログラミング アルゴリズムが含まれています。rule_join_reorder_greedy.go には、貪欲アルゴリズムが含まれています。

> At the beginning of join reorder, we extract "join groups" from the plan tree. A join group is some sub-trees connected together by inner Joins directly, which means there can't exist any other kind of operator between inner Joins. The join reorder algorithm optimizes one join group at a time. And join groups are optimized from bottom to top.

結合順序変更の開始時に、プラン ツリーから「結合グループ」を抽出します。結合グループとは、内部結合によって直接接続されたいくつかのサブツリーです。つまり、内部結合間に他の種類の演算子は存在できません。結合順序変更アルゴリズムは、一度に 1 つの結合グループを最適化します。結合グループは下から上に最適化されます。

> For every node in a join group, the row count is estimated. The join result row count is estimated using the simple and classic leftRowCount * rightRowCount / max(leftNDV, rightNDV) formula. Then two of them, which can get the minimum cost (calculated in (*baseSingleGroupJoinOrderSolver).baseNodeCumCost()), are chosen, connected by an inner join, then added into the join group. This process is repeated until all nodes in the join group are joined together.

結合グループ内のすべてのノードについて、行数が推定されます。結合結果の行数は、単純で標準的な leftRowCount * rightRowCount / max(leftNDV, rightNDV) 式を使用して推定されます。次に、最小コスト (*baseSingleGroupJoinOrderSolver).baseNodeCumCost() で計算) を取得できる 2 つの行が選択され、内部結合によって接続されてから、結合グループに追加されます。このプロセスは、結合グループ内のすべてのノードが結合されるまで繰り返されます。

### ソースコード
```goland
		if useGreedy {
			groupSolver := &joinReorderGreedySolver{
				baseSingleGroupJoinOrderSolver: baseGroupSolver,
			}
			p, err = groupSolver.solve(curJoinGroup, tracer)
		} else {
			dpSolver := &joinReorderDPSolver{
				baseSingleGroupJoinOrderSolver: baseGroupSolver,
			}
			dpSolver.newJoin = dpSolver.newJoinWithEdges
			p, err = dpSolver.solve(curJoinGroup, tracer)
		}

```
## DP
`pkg/planner/core/rule_join_reorder_dp.go` で dpGraphメソッドをコールし、サブグラフ単位で、コスト計算をしている

:::message 
ここをコードを読みながら掘り下げていく
:::

```go
// dpGraph is the core part of this algorithm.
// It implements the traditional join reorder algorithm: DP by subset using the following formula:
//
//	bestPlan[S:set of node] = the best one among Join(bestPlan[S1:subset of S], bestPlan[S2: S/S1])
func (s *joinReorderDPSolver) dpGraph(visitID2NodeID, nodeID2VisitID []int, _ []base.LogicalPlan,
	totalEqEdges []joinGroupEqEdge, totalNonEqEdges []joinGroupNonEqEdge, tracer *joinReorderTrace) (base.LogicalPlan, error) {
	nodeCnt := uint(len(visitID2NodeID))
	bestPlan := make([]*jrNode, 1<<nodeCnt)
	// bestPlan[s] is nil can be treated as bestCost[s] = +inf.
	for i := uint(0); i < nodeCnt; i++ {
		bestPlan[1<<i] = s.curJoinGroup[visitID2NodeID[i]]
	}
	// Enumerate the nodeBitmap from small to big, make sure that S1 must be enumerated before S2 if S1 belongs to S2.
	for nodeBitmap := uint(1); nodeBitmap < (1 << nodeCnt); nodeBitmap++ {
		if bits.OnesCount(nodeBitmap) == 1 {
			continue
		}
		// This loop can iterate all its subset.
		for sub := (nodeBitmap - 1) & nodeBitmap; sub > 0; sub = (sub - 1) & nodeBitmap {
			remain := nodeBitmap ^ sub
			if sub > remain {
				continue
			}
			// If this subset is not connected skip it.
			if bestPlan[sub] == nil || bestPlan[remain] == nil {
				continue
			}
			// Get the edge connecting the two parts.
			usedEdges, otherConds := s.nodesAreConnected(sub, remain, nodeID2VisitID, totalEqEdges, totalNonEqEdges)
			// Here we only check equal condition currently.
			if len(usedEdges) == 0 {
				continue
			}
			join, err := s.newJoinWithEdge(bestPlan[sub].p, bestPlan[remain].p, usedEdges, otherConds)
			if err != nil {
				return nil, err
			}
			// comment: ここでは、Joinした時のコストを計算に、besPlanに格納している
			curCost := s.calcJoinCumCost(join, bestPlan[sub], bestPlan[remain])
			tracer.appendLogicalJoinCost(join, curCost)
			if bestPlan[nodeBitmap] == nil {
				bestPlan[nodeBitmap] = &jrNode{
					p:       join,
					cumCost: curCost,
				}
			} else if bestPlan[nodeBitmap].cumCost > curCost {
				bestPlan[nodeBitmap].p = join
				bestPlan[nodeBitmap].cumCost = curCost
			}
		}
	}
	return bestPlan[(1<<nodeCnt)-1].p, nil
}

```

## Greedy
`pkg/planner/core/rule_join_reorder_dp.go` で サブグラフ単位で、コスト計算をしている

:::message
ここをコードを読みながら掘り下げていく
:::
```go
// solve reorders the join nodes in the group based on a greedy algorithm.
//
// For each node having a join equal condition with the current join tree in
// the group, calculate the cumulative join cost of that node and the join
// tree, choose the node with the smallest cumulative cost to join with the
// current join tree.
//
// cumulative join cost = CumCount(lhs) + CumCount(rhs) + RowCount(join)
//
//	For base node, its CumCount equals to the sum of the count of its subtree.
//	See baseNodeCumCost for more details.
//
// TODO: this formula can be changed to real physical cost in future.
//
// For the nodes and join trees which don't have a join equal condition to
// connect them, we make a bushy join tree to do the cartesian joins finally.
func (s *joinReorderGreedySolver) solve(joinNodePlans []base.LogicalPlan, tracer *joinReorderTrace) (base.LogicalPlan, error) {
	var err error
	s.curJoinGroup, err = s.generateJoinOrderNode(joinNodePlans, tracer)
	if err != nil {
		return nil, err
	}
	var leadingJoinNodes []*jrNode
	if s.leadingJoinGroup != nil {
		// We have a leading hint to let some tables join first. The result is stored in the s.leadingJoinGroup.
		// We generate jrNode separately for it.
		leadingJoinNodes, err = s.generateJoinOrderNode([]base.LogicalPlan{s.leadingJoinGroup}, tracer)
		if err != nil {
			return nil, err
		}
	}
	// Sort plans by cost
	slices.SortStableFunc(s.curJoinGroup, func(i, j *jrNode) int {
		return cmp.Compare(i.cumCost, j.cumCost)
	})

	// joinNodeNum indicates the number of join nodes except leading join nodes in the current join group
	joinNodeNum := len(s.curJoinGroup)
	if leadingJoinNodes != nil {
		// The leadingJoinNodes should be the first element in the s.curJoinGroup.
		// So it can be joined first.
		leadingJoinNodes := append(leadingJoinNodes, s.curJoinGroup...)
		s.curJoinGroup = leadingJoinNodes
	}
	var cartesianGroup []base.LogicalPlan
	for len(s.curJoinGroup) > 0 {
		newNode, err := s.constructConnectedJoinTree(tracer)
		if err != nil {
			return nil, err
		}
		if joinNodeNum > 0 && len(s.curJoinGroup) == joinNodeNum {
			// Getting here means that there is no join condition between the table used in the leading hint and other tables
			// For example: select /*+ leading(t3) */ * from t1 join t2 on t1.a=t2.a cross join t3
			// We can not let table t3 join first.
			// TODO(hawkingrei): we find the problem in the TestHint.
			// 	`select * from t1, t2, t3 union all select /*+ leading(t3, t2) */ * from t1, t2, t3 union all select * from t1, t2, t3`
			//  this sql should not return the warning. but It will not affect the result. so we will fix it as soon as possible.
			s.ctx.GetSessionVars().StmtCtx.SetHintWarning("leading hint is inapplicable, check if the leading hint table has join conditions with other tables")
		}
		cartesianGroup = append(cartesianGroup, newNode.p)
	}

	return s.makeBushyJoin(cartesianGroup), nil
}

```