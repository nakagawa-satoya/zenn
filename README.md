# Zenn CLI

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

# 記事作成
```bash
npx zenn new:article
```

# 記事プレビュ
```bash
npx zenn preview
```

# Markdown ガイド
https://zenn.dev/zenn/articles/markdown-guide

# Diagram
mermaid で記載します

## PlantUML
- PlantUMLを書く手法はないのかな？

## CLI Guide
https://zenn.dev/zenn/articles/zenn-cli-guide

## Publication
``` bash
npx zenn new:article --publication_name ftd_tech_blog
```

## slugは入れた方がいい
``` bash
npx zenn new:article --slug what-is-slug
```


## index
```bash
npx zenn new:article --slug ftd-llam-2023-10-10 --title "ChatGPT/LangChainによるチャットシステム構築実践入門 から" --type tech --publication-name ftd_tech_blog 
npx zenn new:article --slug ftd-tiug-2024-07-16 --title "TiDBユーザグループ「TiUG」TiDBソースコード輪読会 #1 アフターレポート" --type tech --publication-name tidb_user_group 
 ```
https://tiug.connpass.com/event/319256/


## book

```
 npx zenn new:book --slug tiug-2024-08-20 --title "TiDBソースコード輪読会 #2" --summary "TiDBソースコード輪読会 #2 の輪読ガイドです。" --price 0 --published false
 npx zenn new:book --slug tiug-2024-09-20 --title "TiDBソースコード輪読会 #3" --summary "TiDBソースコード輪読会 #3 の輪読ガイドです。" --price 0 --published false
 npx zenn new:book --slug tiug-2024-10-20 --title "TiDBソースコード輪読会 #4" --summary "TiDBソースコード輪読会 #4 の輪読ガイドです。" --price 0 --published false
 npx zenn new:book --slug truenas-startup --title "TrueNAS ではじめる" --summary "TiDBソースコード輪読会 #2 の輪読ガイドです。" --price 0 --published false
 npx zenn new:book --slug webgpu-renderfarm --title "TrueNAS ではじめる" --summary "TiDBソースコード輪読会 #2 の輪読ガイドです。" --price 0 --published false

```
