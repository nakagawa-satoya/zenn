---
title: "Left Deep Tree"
---

# Left Deep Tree
![Left Deep Tree](/images/tiug-2024-08-20/left_deep_tree.png)

`pkg/planner/core/logical_plans.go` の `ExtractFD` あたりでやっていそう。

## ExtractFD
```go
// ExtractFD implements the logical plan interface, extracting the FD from bottom up.
// 1:
// In most of the cases, using FDs to check the only_full_group_by problem should be done in the buildAggregation phase
// by extracting the bottom-up FDs graph from the `p` --- the sub plan tree that has already been built.
//
// 2:
// and this requires that some conditions push-down into the `p` like selection should be done before building aggregation,
// otherwise, 'a=1 and a can occur in the select lists of a group by' will be miss-checked because it doesn't be implied in the known FDs graph.
//
// 3:
// when a logical agg is built, it's schema columns indicates what the permitted-non-agg columns is. Therefore, we shouldn't
// depend on logicalAgg.ExtractFD() to finish the only_full_group_by checking problem rather than by 1 & 2.
func (la *LogicalAggregation) ExtractFD() *fd.FDSet {
	// basically extract the children's fdSet.
	fds := la.logicalSchemaProducer.ExtractFD()
	// collect the output columns' unique ID.
	outputColsUniqueIDs := intset.NewFastIntSet()
	notnullColsUniqueIDs := intset.NewFastIntSet()
	groupByColsUniqueIDs := intset.NewFastIntSet()
	groupByColsOutputCols := intset.NewFastIntSet()
	// Since the aggregation is build ahead of projection, the latter one will reuse the column with UniqueID allocated in aggregation
	// via aggMapper, so we don't need unnecessarily maintain the <aggDes, UniqueID> mapping in the FDSet like expr did, just treating
	// it as normal column.
	for _, one := range la.Schema().Columns {
		outputColsUniqueIDs.Insert(int(one.UniqueID))
	}
	// For one like sum(a), we don't need to build functional dependency from a --> sum(a), cause it's only determined by the
	// group-by-item (group-by-item --> sum(a)).
	for _, expr := range la.GroupByItems {
		switch x := expr.(type) {
		case *expression.Column:
			groupByColsUniqueIDs.Insert(int(x.UniqueID))
		case *expression.CorrelatedColumn:
			// shouldn't be here, intercepted by plan builder as unknown column.
			continue
		case *expression.Constant:
			// shouldn't be here, interpreted as pos param by plan builder.
			continue
		case *expression.ScalarFunction:
			hashCode := string(x.HashCode())
			var (
				ok             bool
				scalarUniqueID int
			)
			if scalarUniqueID, ok = fds.IsHashCodeRegistered(hashCode); ok {
				groupByColsUniqueIDs.Insert(scalarUniqueID)
			} else {
				// retrieve unique plan column id.  1: completely new one, allocating new unique id. 2: registered by projection earlier, using it.
				if scalarUniqueID, ok = la.SCtx().GetSessionVars().MapHashCode2UniqueID4ExtendedCol[hashCode]; !ok {
					scalarUniqueID = int(la.SCtx().GetSessionVars().AllocPlanColumnID())
				}
				fds.RegisterUniqueID(hashCode, scalarUniqueID)
				groupByColsUniqueIDs.Insert(scalarUniqueID)
			}
			determinants := intset.NewFastIntSet()
			extractedColumns := expression.ExtractColumns(x)
			extractedCorColumns := expression.ExtractCorColumns(x)
			for _, one := range extractedColumns {
				determinants.Insert(int(one.UniqueID))
				groupByColsOutputCols.Insert(int(one.UniqueID))
			}
			for _, one := range extractedCorColumns {
				determinants.Insert(int(one.UniqueID))
				groupByColsOutputCols.Insert(int(one.UniqueID))
			}
			notnull := util.IsNullRejected(la.SCtx(), la.schema, x)
			if notnull || determinants.SubsetOf(fds.NotNullCols) {
				notnullColsUniqueIDs.Insert(scalarUniqueID)
			}
			fds.AddStrictFunctionalDependency(determinants, intset.NewFastIntSet(scalarUniqueID))
		}
	}

	// Some details:
	// For now, select max(a) from t group by c, tidb will see `max(a)` as Max aggDes and `a,b,c` as firstRow aggDes,
	// and keep them all in the schema columns before projection does the pruning. If we build the fake FD eg: {c} ~~> {b}
	// here since we have seen b as firstRow aggDes, for the upper layer projection of `select max(a), b from t group by c`,
	// it will take b as valid projection field of group by statement since it has existed in the FD with {c} ~~> {b}.
	//
	// and since any_value will NOT be pushed down to agg schema, which means every firstRow aggDes in the agg logical operator
	// is meaningless to build the FD with. Let's only store the non-firstRow FD down: {group by items} ~~> {real aggDes}
	realAggFuncUniqueID := intset.NewFastIntSet()
	for i, aggDes := range la.AggFuncs {
		if aggDes.Name != "firstrow" {
			realAggFuncUniqueID.Insert(int(la.schema.Columns[i].UniqueID))
		}
	}

	// apply operator's characteristic's FD setting.
	if len(la.GroupByItems) == 0 {
		// 1: as the details shown above, output cols (normal column seen as firstrow) of group by are not validated.
		// we couldn't merge them as constant FD with origin constant FD together before projection done.
		// fds.MaxOneRow(outputColsUniqueIDs.Union(groupByColsOutputCols))
		//
		// 2: for the convenience of later judgement, when there is no group by items, we will store a FD: {0} -> {real aggDes}
		// 0 unique id is only used for here.
		groupByColsUniqueIDs.Insert(0)
		for i, ok := realAggFuncUniqueID.Next(0); ok; i, ok = realAggFuncUniqueID.Next(i + 1) {
			fds.AddStrictFunctionalDependency(groupByColsUniqueIDs, intset.NewFastIntSet(i))
		}
	} else {
		// eliminating input columns that are un-projected.
		fds.ProjectCols(outputColsUniqueIDs.Union(groupByColsOutputCols).Union(groupByColsUniqueIDs))

		// note: {a} --> {b,c} is not same with {a} --> {b} and {a} --> {c}
		for i, ok := realAggFuncUniqueID.Next(0); ok; i, ok = realAggFuncUniqueID.Next(i + 1) {
			// group by phrase always produce strict FD.
			// 1: it can always distinguish and group the all-null/part-null group column rows.
			// 2: the rows with all/part null group column are unique row after group operation.
			// 3: there won't be two same group key with different agg values, so strict FD secured.
			fds.AddStrictFunctionalDependency(groupByColsUniqueIDs, intset.NewFastIntSet(i))
		}

		// agg funcDes has been tag not null flag when building aggregation.
		fds.MakeNotNull(notnullColsUniqueIDs)
	}
	fds.GroupByCols = groupByColsUniqueIDs
	fds.HasAggBuilt = true
	// just trace it down in every operator for test checking.
	la.fdSet = fds
	return fds
}

```

:::message alert
追いきれなかったので、輪読会までに追加で調査予定
:::

