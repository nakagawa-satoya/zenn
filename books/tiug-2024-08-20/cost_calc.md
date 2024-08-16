---
title: "コスト計算"
---
# コスト計算
## まずは物理最適化
![物理最適化](/images/tiug-2024-08-20/physical_optimization.png)
### 参考
https://docs.pingcap.com/ja/tidbcloud/sql-physical-optimization

:::message
今回はコストモデルにフォーカスしていきます。
:::

## コストモデル
TiDB のコストモデルは Ver1,とVer2 がある
### ソースコードVer1 
```go
// GetPlanCostVer1 calculates the cost of the plan if it has not been calculated yet and returns the cost.
func (p *PhysicalSelection) GetPlanCostVer1(taskType property.TaskType, option *optimizetrace.PlanCostOption) (float64, error) {
	costFlag := option.CostFlag
	if p.planCostInit && !hasCostFlag(costFlag, costusage.CostFlagRecalculate) {
		return p.planCost, nil
	}

	var selfCost float64
	var cpuFactor float64
	switch taskType {
	case property.RootTaskType, property.MppTaskType:
		cpuFactor = p.SCtx().GetSessionVars().GetCPUFactor()
	case property.CopSingleReadTaskType, property.CopMultiReadTaskType:
		cpuFactor = p.SCtx().GetSessionVars().GetCopCPUFactor()
	default:
		return 0, errors.Errorf("unknown task type %v", taskType)
	}
	selfCost = getCardinality(p.children[0], costFlag) * cpuFactor
	if p.fromDataSource {
		selfCost = 0 // for compatibility, see https://github.com/pingcap/tidb/issues/36243
	}

	childCost, err := p.children[0].GetPlanCostVer1(taskType, option)
	if err != nil {
		return 0, err
	}
	p.planCost = childCost + selfCost
	p.planCostInit = true
	return p.planCost, nil
}
```
#### バージョン指定
TiDB v6.2.0 から ver2 がデフォルトで導入され、Ver1はデフォルトでは使用されていません。
tidb_cost_model_version 変数を設定してバージョン切り替えができるようです。

### コスト計算処理 Ver_2
```go
// GetCost computes the cost of projection operator itself.
func (p *PhysicalProjection) GetCost(count float64) float64 {
	sessVars := p.SCtx().GetSessionVars()
	cpuCost := count * sessVars.GetCPUFactor()
	concurrency := float64(sessVars.ProjectionConcurrency())
	if concurrency <= 0 {
		return cpuCost
	}
	cpuCost /= concurrency
	concurrencyCost := (1 + concurrency) * sessVars.GetConcurrencyFactor()
	return cpuCost + concurrencyCost
}
```

### ソースコード
現在はこちらが有効なようです。
```go
// GetPlanCostVer2 calculates the cost of the plan if it has not been calculated yet and returns the cost.
func (p *basePhysicalPlan) GetPlanCostVer2(taskType property.TaskType, option *optimizetrace.PlanCostOption) (costusage.CostVer2, error) {
	if p.planCostInit && !hasCostFlag(option.CostFlag, costusage.CostFlagRecalculate) {
		return p.planCostVer2, nil
	}
	childCosts := make([]costusage.CostVer2, 0, len(p.children))
	for _, child := range p.children {
		childCost, err := child.GetPlanCostVer2(taskType, option)
		if err != nil {
			return costusage.ZeroCostVer2, err
		}
		childCosts = append(childCosts, childCost)
	}
	if len(childCosts) == 0 {
		p.planCostVer2 = costusage.NewZeroCostVer2(costusage.TraceCost(option))
	} else {
		p.planCostVer2 = costusage.SumCostVer2(childCosts...)
	}
	p.planCostInit = true
	return p.planCostVer2, nil
}
```
:::message alert
これは１つのタイプで、パラメータ違いで同様の処理が多い
:::

### コスト集計処理 Ver2
```go
// SumCostVer2 sum the cost up of all the passed args.
func SumCostVer2(costs ...CostVer2) (ret CostVer2) {
	if len(costs) == 0 {
		return
	}
	for _, c := range costs {
		ret.cost += c.cost
		if c.trace != nil {
			if ret.trace == nil { // init
				ret.trace = &CostTrace{make(map[string]float64), ""}
			}
			for factor, factorCost := range c.trace.factorCosts {
				ret.trace.factorCosts[factor] += factorCost
			}
			if ret.trace.formula != "" {
				ret.trace.formula += " + "
			}
			ret.trace.formula += "(" + c.trace.formula + ")"
		}
	}
	return ret
}
```


## コストを見る
今回は論理最適化のコスト計算結果を見ていきます。 
### 論理最適化コストの取得
```bash
trace plan <sql> 
```

を実行すると
```bash
<tmpdir>/optimize_trace/<number>/<trace planの結果のzipファイル名 
```
にplan実行結果が確認できます。

:::message
tmpdir は go の　os.TempDir() のパスになります。下記は os.TempDir() の説明です。

On Unix systems, it returns $TMPDIR if non-empty, else /tmp. 
On Windows, it uses GetTempPath, returning the first non-empty value from %TMP%, %TEMP%, %USERPROFILE%, or the Windows directory. On Plan 9, it returns /tmp.
:::


```json:実行結果のJoin Reorderをしているところ
    "steps": [
      {
        "action": "join order becomes ((cn*c)*cl) from original ((c*cn)*cl)",
        "reason": "join cost during reorder: [[((cn*c)*cl), cost:14756.664870689656],[(cn*c), cost:8520.073275862069],[c, cost:4079],[cl, cost:984],[cn, cost:239]]",
        "type": "Projection",
        "id": 7,
        "index": 0
      }
    ],
```