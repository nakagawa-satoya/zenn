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
npx zenn new:article --slug ftd-kotlin-2023-12-01 --title "Kotlin Mutliplatform Watch party & Recap" --type tech --publication-name ftd_tech_blog 
npx zenn new:article --slug ftd-ochacafe-2023-12-05 --title "OCI Cache with Redisでキャッシュ管理" --type tech --publication-name ftd_tech_blog 
npx zenn new:article --slug ftd-postman-2024-01-29 --title "フロントエンドにこそおすすめしたいPostman" --type tech --publication-name ftd_tech_blog 
npx zenn new:article --slug ftd-nuxt-2024-02-29 --title "Nuxt Fontを試してみた" --type tech --publication-name ftd_tech_blog 
 ```
