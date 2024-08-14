---
title: "付録：Package の読み方"
---
# Packageの読み方
TiDB はビルドツールで Bazel (https://bazel.build/?hl=ja)を使用しています。

```text
load("@io_bazel_rules_go//go:def.bzl", "go_library")

go_library(
    name = "base",
    srcs = [
    //　ここに記述されたgoファイルがgo_library ようにビルドされます
        "misc_base.go",
        "plan_base.go",
        "task_base.go",
    ],
    importpath = "github.com/pingcap/tidb/pkg/planner/core/base",
    visibility = ["//visibility:public"],
    deps = [
    //　ここに記述されたpackage パスは、依存関係としてビルド時に参照されます。
    // ファイル個別の指定はは、それぞれのimport 句で指定されています.
    
        "//pkg/expression",
        "//pkg/kv",
        "//pkg/planner/context",
        "//pkg/planner/funcdep",
        "//pkg/planner/property",
        "//pkg/planner/util/costusage",
        "//pkg/planner/util/optimizetrace",
        "//pkg/types",
        "//pkg/util/collate",
        "//pkg/util/execdetails",
        "//pkg/util/tracing",
        "@com_github_pingcap_tipb//go-tipb",
    ],
)

```


