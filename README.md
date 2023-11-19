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


```bash
git push https://nakgawa.satoya@github.com/nakagawa-satoya/zenn.git 
```
git push git@github.com:nakagawa-satoya/zenn.git

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
 ```

npx zenn new:article --slug ftd-security-2023-11-11 --title "情報処理安全確保支援士実践講習" --type idea --publication-name ftd_tech_blog
npx zenn new:article --slug ftd-llam-2023-11-16 --title "【東京開催】もめんと Meet-up #7 ユースケース祭り！" --type tech --publication-name ftd_tech_blog 
