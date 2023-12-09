---
title: "Gitlab CI カタログをみんなで作ろう！"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gitlab", "ci", "IaC", "devops", "gitlabci"]  
published: true
publication_name: ftd_tech_blog
---
# Gitlab CI カタログをみんなで作ろう！
:::message
この記事は、GitLab Advent Calendar 2023 10日目の記事です。
:::
@[card](https://qiita.com/advent-calendar/2023/gitlab)

## 始まり
GitLab CI を使っているみんな、GitHub Actions が羨ましくないですか？
:::message
羨ましい
:::


### 特にGitHub MarketPlace
https://github.com/marketplace?type=actions

#### Marketplace良いところ
1. 公開されているので、自分で作らず、そのまま使える(Collaborative = Good!)
2. MarketPlace になっているので、探しやすい(Discoverable = Good!)
3. 汎用的に作られているので、使いやすい(Integrated = Good!)
4. コミュニティが勝手にメンテしてくれる(Mainance = Good!)


#### Package のページのように管理されているのすごい良いですよね
こんな感じになっている

https://github.com/marketplace/actions/setup-java-jdk

:::message
実際のところ私はGitHub Actions と GitLab CI を併用していたりします。
:::


## じゃぁGitHub Actionsでいいじゃん
GitLabはCI以外の魅力も多いので、GitHub よりGitLab推しです。

:::message
GitLab 最高！
:::
## そんなMarketPlace的なものが GitLab にもやっと来ました
https://twitter.com/xiombatsg/status/1707218241079398435

### こんな感じ
![image](/images/gitlab-ci/catalog.png =640x)

@[card](https://gitlab.com/explore/catalog)

#### ちょっと待って！え。。。しょぼくない？ 15件って嘘でしょう！？しかもカテゴリすらない
![image](/images/gitlab-ci/whats.png =320x)

:::message alert
圧倒的に足りません！みんな急いで作りましょう
:::

![image](/images/gitlab-ci/rapid.png =320x)


## 急いで作る
### レシピ
1. Nuxt3 のプロジェクトを作る
2. GitLab CI でnuxt generate する カタログを作る
3. リリースする
4. Nuxt3 のプロジェクトの.gitlab-ci.ymlに追加する
5. CI Done!

### 1. Nuxt3 のプロジェクトを作る
こちらの記事で使っているプロジェクトを利用します。
https://zenn.dev/ftd_tech_blog/articles/ftd-momento-2023-12-03

#### Gitlab リポジトリはこちら
@[card](https://gitlab.com/future-techno-developers/public/paapi-moment/paapi-momento)

:::message
作りたい人はそちらを参照してください。
:::

### 2. GitLab CI でnuxt generate する カタログを作る

#### マニュアル
@[card](https://gitlab-docs.creationline.com/ee/ci/components/)
@[card](https://gitlab-docs.creationline.com/ee/ci/components/catalog.html)

:::message
クリエーションラインさん。いつも助かってます！
:::

こんな感じ
```yml:Nuxt3/template.yml
spec:
  inputs:
    stage:
      default: build
      type: string
    image:
      default:  node:20-buster-slim
      type: string

---
# npm install & nuxt prepare
nuxt-prepare:
  image: $[[ inputs.image ]]
  stage: $[[ inputs.stage ]]
  script:
    - npm install
    - npx nuxt prepare
  cache:
    - key: caches-$CI_COMMIT_REF_NAME-$CI_PROJECT_ID-node-20-node_modules
      paths:
        - node_modules
      policy: push
    - key: caches-$CI_COMMIT_REF_NAME-$CI_PROJECT_ID-node-20-nuxt3
      paths:
        - .nuxt
      policy: push

# nuxt generate
nuxt-generate:
  image: $[[ inputs.image ]]
  stage: $[[ inputs.stage ]]
  needs: [ "nuxt-prepare" ]
  script:
    - npx nuxt generate
  cache:
    - key: caches-$CI_COMMIT_REF_NAME-$CI_PROJECT_ID-node-20-node_modules
      paths:
        - node_modules
      policy: pull
    - key: caches-$CI_COMMIT_REF_NAME-$CI_PROJECT_ID-node-20-nuxt3
      paths:
        - .nuxt
      policy: pull
```

:::message alert
inputsは階層構造が持てないので、分けたい場合はcomponentを分ける必要があるようです。stage分けたいjobをどうするか悩ましい
:::

### 3. リリースする
リリースはCIで作ってもいいけど、考慮することがそこそこあり、面倒なので今回はglabでリリースする
:::message
そもそもタグを手動でつけるなら、glab で両方作ればいいじゃん
:::
#### glab についてはこちらを参照
@[card](https://docs.gitlab.co.jp/ee/integration/glab/)

#### CI Lintだけは通す
```Bash
glab ci lint templates/Nuxt3/template.yml 
```
#### リリースする
```Bash
glab release create v0.0.1 -N "Nuxt3 generate Catalog test release"
```
:::message
glab release はデフォルトブランチをターゲットにリリースされるので、その点は注意
:::
#### カタログプロジェクトにする
![image](/images/gitlab-ci/setting.png)

:::message
設定＞一般> 可視性、プロジェクトの機能、権限>CI/CDカタログリソース

:::


#### あれ？リリースされない
##### これだとリリースされないので注意
1. 作ったプロジェクトが公開プロジェクトじゃなかった
2. project description の記載がないとリリースされないので追加する
3. 再度、タグを削除してトライ
```Bash
glab release create v0.0.1 -N "Nuxt3 generate Catalog test release"
```
#### カタログに登録された！
@[card](https://gitlab.com/explore/catalog/future-techno-developers/public/ci-catalog/catalog-template)

### 4. Nuxt3 のプロジェクトの.gitlab-ci.ymlに追加する
追加する内容。include で追加するだけ。
```yml:.gitlab-ci.yml
include:
  - component: 'gitlab.com/future-techno-developers/public/ci-catalog/catalog-template/Nuxt3@v0.0.1'
stages:
  - setup
  - build
```
:::message alert
nuxt-prepare job用にsetup stage も作りたかったんですが、inputにstageは一つしか入力できなさそうです。これならjobのstage を上書きして利用した方が良さそうです。
:::

### 5. CI Done!
Gitlab CI を実行しましょう。Done！ うまく動けば成功です。
#### こんな感じ
![image](/images/gitlab-ci/pipeline.png )

![image](/images/gitlab-ci/winner.png =320x)


## 終わり
### 感触とか
#### ほとんど社内で作成しているスクリプトをGitLab CI にできます。
そんなに手間がかからないです。
:::message
そのまんま持っていくと色々地雷がありそうな気がするので、それは今後の調査
:::
#### 正直 コンポーネントはカタログがなければ使わない・・・
include.projectと何が違うのかがわかりません。より公開しやすくなった？

#### キャッシュとかどうなんの？
include した時に、キャッシュをどうするかは課題がある。触ってもらうのが良いのか？
:::message
例えば npm install してnode_modules をキャッシュする Job にした場合、キャッシュ名はどうするのか？とか
:::

#### CIコンポーネントリポジトリは細かく分けた方が良さそう
カタログなので、npmとかnodeとかreact とか、そういう単位で分けた方が良さそうです。

#### ラベルは？
カテゴリわけされないけど、ラベルはつけられないの？

#### こうするとベター？？
1. 公開する単位は機能単位のプロジェクト名にしておくのが良さそうです(nuxt-catalog とか)
2. ルートにはtemplate.ymlをおかない。templatesディレクトリにコンポーネントディレクトリを作った方が後々良さそう。 
:::message
カタログ内で複数作ったり、テスト書いたりで、ymlが肥大化して、includeにしたくなるので、分けておいた方が良さそう
:::
3. 当たり前ですが、変数、引数にできるところ、共通化できるところは共通化した方がいいです。
4. 今回は同梱していませんが、CIカタログ中にテストプロジェクトも内包した方が良いです。
5. Readmeに inputsの内容、 jobの内容は記載しましょう。パラメータ変更して実行するケースの方が多いと思います。

### 今後やりたいこと
- 社内で使用しているCIスクリプトをカタログ化してpublicにしていく。
- GitLab CI カタログ増やしていく。
- ベストプラクティクスを共有していく。
:::message 
誰か一緒に増やしていく活動しましょう
:::
### みんなもっとGitLab を使っていこうぜ！

## 補足
### 情報がまじでないです。特に日本語。みんな発信していきましょう
下記は英語情報ですが、参考になりました。とはいえ誰かの作ったカタログ見るのが一番早いです。
https://about.gitlab.com/blog/2023/07/10/introducing-ci-components/
https://www.youtube.com/watch?v=Hn5buzm2epk

https://www.youtube.com/watch?v=Vw8-ce8LNBs

https://gitlab.com/guided-explorations/gitlab-ci-yml-tips-tricks-and-hacks/dry-repository-a-cheatsheet

https://gitlab.com/groups/gitlab-org/-/epics/7462
