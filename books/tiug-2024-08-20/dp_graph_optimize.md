---
title: "論理最適化"
---
# 論理最適化
![論理最適化](/images/tiug-2024-08-20/logic_optimization.png)


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
## DP Graph
`pkg/planner/core/rule_join_reorder_dp.go` で dpGraphメソッドをコールし、サブグラフ単位で、Joinしている

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