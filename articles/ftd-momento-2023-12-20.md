---
title: "Momento CLI簡易版を作る"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["momento","CLI"]
published: true
publication_name: ftd_tech_blog
---
# Momento CLI簡易版を作る
:::message
この記事は、Momento Advent Calendar 2023 20日目の記事です。
:::

@[card](https://qiita.com/advent-calendar/2023/momento)

## なぜCLIを作るのか
Momentoをいじっている時にCLIでキャッシュ操作したかったが、
公式のCLIがSet/Get しか対応しておらず。Dictionary/ SortedSetsを確認するのに欲しかった。

## 現状
Sets/Dictionary/SortedSets の Get / Set / Fetch にのみ対応（優先低いので）

## ソースコード置き場
下記置いてあります。
@[card](https://gitlab.com/future-techno-developers/public/momento-tools/momento-cli)

## 使い方
```bash

Usage: index [options] [command]

Options:
  -h, --help                                          display help for command

Commands:
  cacheSets [options] <op> <name> <key>               Momento Cache Sets Type Cache
  cacheDictionary [options] <op> <name> <dic> <key>   Momento Cache Dictionary Type Cache
  cacheSortedSet [options] <op> <name> <set> <value>  Momento Cache SortedSet Type Cache
  help [command]                                      display help for command

```

:::message
とりあえず手元で使う程度にしていますが、整理してそのうちpackage にして配布します。
:::

### 使い方
#### Setsのキャッシュ設定例
```bash:
pnpm dev cacheSets Set <キャッシュ名> <key> -v <value> -c <config> 
```

#### Setsのキャッシュ取得方法
```bash:
pnpm dev cacheSets Get <キャッシュ名> <key> -c <config> 
```

#### Dictionaryのキャッシュ設定例
```bash:
pnpm dev cacheDictionary Set <キャッシュ名> <Dictionary名> <key> -v <value> -c <config> 
```

#### Dictionaryのキャッシュ取得方法
```bash:
pnpm dev cacheDictionary Get <キャッシュ名> <Dictionary名> <key>  -c <config> 
```

#### Dictionaryのキャッシュフェッチ方法
```bash:
pnpm dev cacheDictionary Fetch <キャッシュ名> <Dictionary名> <key>  -c <config> 
```

#### SortedSetのキャッシュ設定例
```bash:
pnpm dev cacheSortedSet Set <キャッシュ名> <Set名> <value> -s <score> -c <config> 
```

#### SortedSetのキャッシュ取得方法
```bash:
pnpm dev cacheSortedSet Get <キャッシュ名> <Set名> <value> -c <config> 
```

#### SortedSetのキャッシュフェッチ方法
```bash:
pnpm dev cacheSortedSet Fetch <キャッシュ名> <Set名> <key>  -c <config> 
```

## 今後
上でも書きましたが、手元でキャッシュを操作できるのは有用なので、マニュアルにある機能を簡単に使用できるようになったら、また記事にします。
