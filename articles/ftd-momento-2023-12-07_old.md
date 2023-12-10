---
title: "Momento Cache + D1 を使って簡易絞り込み機能を作る"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Momento", "Hono", "Cloudflare","D1"]
published: false
publication_name: ftd_tech_blog
---

# Momento Cache で効率の良いCache 取得方法の模索

## キャッシュアルゴリズム 
### CPU
https://ja.wikipedia.org/wiki/%E3%82%AD%E3%83%A3%E3%83%83%E3%82%B7%E3%83%A5%E3%82%A2%E3%83%AB%E3%82%B4%E3%83%AA%E3%82%BA%E3%83%A0
- n-way set associative cache
- 2-way set associative cache
- direct mapped cache
- fully associative cache
- 
## キャッシュコヒーレンス
https://ja.wikipedia.org/wiki/%E3%82%AD%E3%83%A3%E3%83%83%E3%82%B7%E3%83%A5%E3%82%B3%E3%83%92%E3%83%BC%E3%83%AC%E3%83%B3%E3%82%B7

- ディレクトリーベース
- バススヌーピんぐ
- すなーフィン具

## 一貫性モデル
- MESI
- Write Once
- Synapse 
- Berkeley
- Firefly
- Dragon

## MESI
https://ja.wikipedia.org/wiki/MESI%E3%83%97%E3%83%AD%E3%83%88%E3%82%B3%E3%83%AB

M - Modified(変更): 当該キャッシュだけに存在し、主記憶上の値から変更されている(dirty)。他のCPUがこのキャッシュラインに相当する主記憶をリードするのを許可する前に、キャッシュ機構はこのキャッシュラインをいずれかの時点で主記憶に書き戻さなければならない。
E - Exclusive(排他): 当該キャッシュだけに存在するが、主記憶上の値と一致している(clean)。
S - Shared(共有): システム内の他のキャッシュにも同じキャッシュラインが存在している（主記憶とも内容が一致）。
I - Invalid(無効): このキャッシュラインは無効。

## 並列計算一貫性モデル
https://ja.wikipedia.org/wiki/%E4%B8%A6%E5%88%97%E8%A8%88%E7%AE%97#%E4%B8%80%E8%B2%AB%E6%80%A7%E3%83%A2%E3%83%87%E3%83%AB

## ChatGPT

レスポンス
KVS（Key-Value Store）の最適化に関する情報は、特定のKVSシステムに依存します。一般的なアドバイスとしては、KVSの最適化については以下のポイントを考慮することが重要です：

キーの設計: キーの設計は、KVSの性能に影響を与えます。キーは一意で、検索しやすいものであるべきです。

キャッシュの利用: キャッシュを活用して、アクセスパフォーマンスを向上させることができます。データの読み取り回数を減らすために、よくアクセスされるデータをキャッシュに保存します。

データの圧縮: データの圧縮を使用して、ストレージスペースを節約し、読み取り/書き込みの速度を向上させることができます。

インデックスの最適化: インデックスを適切に設計し、クエリのパフォーマンスを向上させることができます。

クエリの最適化: クエリを効率的に実行するために、クエリの最適化を行います。クエリプランナーやクエリチューニングツールを使用することがあります。

レプリケーションとシャーディング: 高可用性やスケーラビリティを実現するために、データのレプリケーションやシャーディングを考慮します。

ハードウェアの最適化: ハードウェアのリソース（CPU、メモリ、ストレージ）を適切に構成し、性能を最適化します。

KVSの種類や使用しているテクノロジーによって、最適化の具体的な方法は異なる可能性があります。詳細な情報を提供するには、具体的なKVSシステムに関する情報が必要です。どのKVSシステムについての情報が必要ですか？


## Redis
https://redis.io/docs/management/optimization/benchmarks/

https://redis.io/docs/manual/keyspace/

### Keyで簡単に検索するアイデア
- D1 で index テーブルを作る
### Sorted Set を使う

### Directory Base Cache 的に考えて、
キーが別であれば、cache テーブルを検索することを考える
- 一貫性の担保を cache テーブルで考えると楽

簡単な検索できるようにする
- どういう構造で格納するのが良いのか？

Woker(Searcher) - Momento Cache(Dictionary) 
- 基本的にDictionary に同一ルールセットのデータは詰め込む
- DictionaryGetFields を使って、複数キーを一度に取得する
- D1 にindex テーブルを作る
- select * from D1 where key = 'index' and value = 'value'
- Eviction されたkey が見つかったら、after worker で削除する

### Client Side Caching
https://redis.io/docs/manual/client-side-caching/

Client Side は Session Storage

