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
world サンプルで実行してみる
```go
mysql> explain ANALYZE   select * from city as c   join  country as cn on c.CountryCode = cn.Code   join countrylanguage as cl on c.CountryCode = cl.CountryCode;
```

| id                             | estRows                  | actRows | task      | access object | execution info                                                                                                                                                                                                                                                                                                                                                                                                | operator info                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | memory   | disk    |
| -------------------------------- |--------------------------| --------- | ----------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| ---------- | --------- |
| Projection_13                  | 5252.59                  | 30670   | root      |               | time:18.5ms, loops:34, RU:5.448218, Concurrency:5                                                                                                                                                                                                                                                                                                                                                             | world.city.id, world.city.name, world.city.countrycode, world.city.district, world.city.population, world.country.code, world.country.name, world.country.continent, world.country.region, world.country.surfacearea, world.country.indepyear, world.country.population, world.country.lifeexpectancy, world.country.gnp, world.country.gnpold, world.country.localname, world.country.governmentform, world.country.headofstate, world.country.capital, world.country.code2, world.countrylanguage.countrycode, world.countrylanguage.language, world.countrylanguage.isofficial, world.countrylanguage.percentage   | 5.59 MB  | N/A     |
| └─HashJoin_24                  | 5252.59                  | 30670   | root      |               | time:18.3ms, loops:34, build_hash_table:{total:1.94ms, fetch:1.09ms, build:855.4µs}, probe:{concurrency:5, total:76.5ms, max:18.3ms, probe:51.1ms, fetch and wait:25.4ms}                                                                                                                                                                                                                                     | inner join, equal:[eq(world.city.countrycode, world.countrylanguage.countrycode)]                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | 122.8 KB | 0 Bytes |
| │  ├─TableReader_71(Build) | 984.00  | 984     | root      |               | time:983µs, loops:2, cop_task: {num: 3, max: 522.3µs, min: 244.3µs, avg: 347.8µs, p95: 522.3µs, tot_proc: 2.22µs, tot_wait: 225.7µs, copr_cache_hit_ratio: 1.00, build_task_duration: 1.12µs, max_distsql_concurrency: 1}, rpc_info:{Cop:{num_rpc:3, total_time:1ms}}                                                                                                                                         | data:TableFullScan_70                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | 62.5 KB  | N/A     |
| │  │ └─TableFullScan_70         | 984.00                   | 984     | cop[tikv] | table:cl      | tikv_task:{proc max:6ms, min:2ms, avg: 3.33ms, p80:6ms, p95:6ms, iters:11, tasks:3}, scan_detail: {get_snapshot_time: 186.4µs, rocksdb: {block: {}}}, time_detail: {total_process_time: 2.22µs, total_wait_time: 225.7µs, tikv_wall_time: 405.1µs}                                                                                                                                                            | keep order:false, stats:pseudo                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | N/A      | N/A     |
| │  └─HashJoin_65(Probe)         | 4202.07                  | 4079    | root      |               | time:6.15ms, loops:6, build_hash_table:{total:1.58ms, fetch:818.9µs, build:763.8µs}, probe:{concurrency:5, total:28.8ms, max:6.09ms, probe:8.21ms, fetch and wait:20.6ms}                                                                                                                                                                                                                                     | inner join, equal:[eq(world.country.code, world.city.countrycode)]                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | 323.0 KB | 0 Bytes |
| │  │  ├─TableReader_67(Build)    | 239.00                   | 239     | root      |               | time:325.8µs, loops:2, cop_task: {num: 2, max: 545.1µs, min: 240.7µs, avg: 392.9µs, p95: 545.1µs, tot_proc: 1.98µs, tot_wait: 162.2µs, copr_cache_hit_ratio: 1.00, build_task_duration: 7.93µs, max_distsql_concurrency: 1}, rpc_info:{Cop:{num_rpc:2, total_time:750.1µs}}                                                                                                                                   | data:TableFullScan_66                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | 79.4 KB  | N/A     |
| │  │  │ └─TableFullScan_66       | 239.00                   | 239     | cop[tikv] | table:cn      | tikv_task:{proc max:5ms, min:2ms, avg: 3.5ms, p80:5ms, p95:5ms, iters:4, tasks:2}, scan_detail: {get_snapshot_time: 139.9µs, rocksdb: {block: {}}}, time_detail: {total_process_time: 1.98µs, total_wait_time: 162.2µs, tikv_wall_time: 331.1µs}                                                                                                                                                              | keep order:false, stats:pseudo                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | N/A      | N/A     |
|  └─TableReader_69(Probe)    | 4079.00                  | 4079    | root      |               | time:5.47ms, loops:6, cop_task: {num: 5, max: 3.41ms, min: 206.9µs, avg: 924.5µs, p95: 3.41ms, max_proc_keys: 367, p95_proc_keys: 367, tot_proc: 923.9µs, tot_wait: 306.5µs, copr_cache_hit_ratio: 0.80, build_task_duration: 1.68µs, max_distsql_concurrency: 1}, rpc_info:{Cop:{num_rpc:5, total_time:4.57ms}}                                                                                              | data:TableFullScan_68                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | 178.7 KB | N/A     |
|       └─TableFullScan_68       | 4079.00                  | 4079    | cop[tikv] | table:c       | tikv_task:{proc max:3ms, min:1ms, avg: 1.8ms, p80:3ms, p95:3ms, iters:22, tasks:5}, scan_detail: {total_process_keys: 367, total_process_keys_size: 25484, total_keys: 368, get_snapshot_time: 233µs, rocksdb: {delete_skipped_count: 367, key_skipped_count: 734, block: {}}}, time_detail: {total_process_time: 923.9µs, total_wait_time: 306.5µs, total_kv_read_wall_time: 1ms, tikv_wall_time: 1.65ms}    | keep order:false                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | N/A      | N/A     |
