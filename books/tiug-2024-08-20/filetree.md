---
title: "付録：Planner ディレクトリ構成"
---

```bash
.
├── BUILD.bazel
├── optimize.go
├── OWNERS
├── cardinality          
│   ├── BUILD.bazel
│   ├── cross_estimation.go
│   ├── join.go
│   ├── main_test.go
│   ├── ndv.go
│   ├── pseudo.go
│   ├── row_count_column.go
│   ├── row_count_index.go
│   ├── row_count_test.go
│   ├── row_size.go
│   ├── row_size_test.go
│   ├── selectivity.go
│   ├── selectivity_test.go
│   ├── testdata
│   │   ├── cardinality_suite_in.json
│   │   └── cardinality_suite_out.json
│   ├── trace.go
│   └── trace_test.go
├── context
│   ├── BUILD.bazel
│   └── context.go
├── contextimpl
│   ├── BUILD.bazel
│   └── impl.go
├── core
│   ├── BUILD.bazel
│   ├── access_object.go
│   ├── base
│   │   ├── BUILD.bazel
│   │   ├── misc_base.go
│   │   ├── plan_base.go
│   │   └── task_base.go
│   ├── collect_column_stats_usage.go
│   ├── collect_column_stats_usage_test.go
│   ├── common_plans.go
│   ├── debugtrace.go
│   ├── encode.go
│   ├── exhaust_physical_plans.go
│   ├── explain.go
│   ├── expression_rewriter.go
│   ├── find_best_task.go
│   ├── flat_plan.go
│   ├── foreign_key.go
│   ├── fragment.go
│   ├── handle_cols.go
│   ├── hashcode.go
│   ├── hint_utils.go
│   ├── indexmerge_path.go
│   ├── indexmerge_unfinished_path.go
│   ├── initialize.go
│   ├── logical_plan_builder.go
│   ├── logical_plans.go
│   ├── main_test.go
│   ├── memtable_predicate_extractor.go
│   ├── metrics
│   │   ├── BUILD.bazel
│   │   └── metrics.go
│   ├── mock.go
│   ├── operator
│   │   └── baseimpl
│   │       ├── BUILD.bazel
│   │       └── plan.go
│   ├── optimizer.go
│   ├── partition_prune.go
│   ├── pb_to_plan.go
│   ├── physical_plans.go
│   ├── plan.go
│   ├── plan_cache.go
│   ├── plan_cache_lru.go
│   ├── plan_cache_param.go
│   ├── plan_cache_utils.go
│   ├── plan_cacheable_checker.go
│   ├── plan_cost_detail.go
│   ├── plan_cost_ver1.go
│   ├── plan_cost_ver2.go
│   ├── plan_to_pb.go
│   ├── planbuilder.go
│   ├── point_get_plan.go
│   ├── preprocess.go
│   ├── property_cols_prune.go
│   ├── recheck_cte.go
│   ├── resolve_indices.go
│   ├── rule_aggregation_elimination.go
│   ├── rule_aggregation_push_down.go
│   ├── rule_aggregation_skew_rewrite.go
│   ├── rule_build_key_info.go
│   ├── rule_collect_plan_stats.go
│   ├── rule_column_pruning.go
│   ├── rule_constant_propagation.go
│   ├── rule_decorrelate.go
│   ├── rule_derive_topn_from_window.go
│   ├── rule_eliminate_projection.go
│   ├── rule_generate_column_substitute.go
│   ├── rule_generate_column_substitute_test.go
│   ├── rule_inject_extra_projection.go
│   ├── rule_join_elimination.go
│   ├── rule_join_reorder.go
│   ├── rule_join_reorder_dp.go
│   ├── rule_join_reorder_greedy.go
│   ├── rule_max_min_eliminate.go
│   ├── rule_partition_processor.go
│   ├── rule_predicate_push_down.go
│   ├── rule_predicate_simplification.go
│   ├── rule_push_down_sequence.go
│   ├── rule_resolve_grouping_expand.go
│   ├── rule_result_reorder.go
│   ├── rule_semi_join_rewrite.go
│   ├── rule_topn_push_down.go
│   ├── runtime_filter.go
│   ├── runtime_filter_generator.go
│   ├── runtime_filter_generator_test.go
│   ├── scalar_subq_expression.go
│   ├── show_predicate_extractor.go
│   ├── stats.go
│   ├── stringer.go
│   ├── stringer_test.go
│   ├── task.go
│   ├── task_base.go
│   ├── testdata
│   │   ├── index_merge_suite_in.json
│   │   ├── index_merge_suite_out.json
│   │   ├── plan_cache_suite_in.json
│   │   ├── plan_cache_suite_out.json
│   │   ├── plan_suite_unexported_in.json
│   │   ├── plan_suite_unexported_out.json
│   │   ├── runtime_filter_generator_suite_in.json
│   │   └── runtime_filter_generator_suite_out.json
│   ├── tests
│   │   └── prepare
│   │       ├── BUILD.bazel
│   │       ├── main_test.go
│   │       └── prepare_test.go
│   ├── tiflash_selection_late_materialization.go
│   ├── trace.go
│   └── util.go
├── funcdep
│   ├── BUILD.bazel
│   ├── doc.go
│   ├── extract_fd_test.go
│   ├── fd_graph.go
│   ├── fd_graph_test.go
│   └── main_test.go
├── implementation
│   ├── BUILD.bazel
│   ├── base.go
│   ├── base_test.go
│   ├── datasource.go
│   ├── join.go
│   ├── main_test.go
│   ├── simple_plans.go
│   └── sort.go
├── memo
│   ├── BUILD.bazel
│   ├── expr_iterator.go
│   ├── expr_iterator_test.go
│   ├── group.go
│   ├── group_expr.go
│   ├── group_expr_test.go
│   ├── group_test.go
│   ├── implementation.go
│   └── main_test.go
├── pattern
│   ├── BUILD.bazel
│   ├── engine.go
│   ├── engine_test.go
│   ├── pattern.go
│   └── pattern_test.go
├── property
│   ├── BUILD.bazel
│   ├── logical_property.go
│   ├── physical_property.go
│   ├── stats_info.go
│   └── task_type.go
├── task
│   ├── BUILD.bazel
│   ├── task.go
│   ├── task_scheduler.go
│   ├── task_scheduler_test.go
│   └── task_test.go
├── tree.log
・・・以下輪読から対象外なので割愛
├── cascades
└── util
    ├── BUILD.bazel
    ├── byitem.go
    ├── coretestsdk
    │   ├── BUILD.bazel
    │   └── testkit.go
    ├── coreusage
    │   ├── BUILD.bazel
    │   ├── cast_misc.go
    │   └── correlated_misc.go
    ├── costusage
    │   ├── BUILD.bazel
    │   └── cost_misc.go
    ├── debugtrace
    │   ├── BUILD.bazel
    │   └── base.go
    ├── expression.go
    ├── fixcontrol
    │   ├── BUILD.bazel
    │   ├── fixcontrol_test.go
    │   ├── get.go
    │   ├── main_test.go
    │   ├── set.go
    │   └── testdata
    │       ├── fix_control_suite_in.json
    │       └── fix_control_suite_out.json
    ├── main_test.go
    ├── misc.go
    ├── null_misc.go
    ├── optimizetrace
    │   ├── BUILD.bazel
    │   └── opt_tracer.go
    ├── path.go
    └── path_test.go

```