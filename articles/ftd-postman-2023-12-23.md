---
title: "Postman Monitor のリージョンをCloudflare Worker で調べる"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["postman", "cloudflare"]
published: true
publication_name: ftd_tech_blog
---
# Postman Monitor のリージョンをCloudflare Worker で調べる

:::message
この記事は、Postman Advent Calendar 2023 25日目の記事です。
:::

@[card](https://qiita.com/advent-calendar/2023/postman)

## こちらの記事の派生です。
https://zenn.dev/ftd_tech_blog/articles/ftd-cloudflare-2023-12-17

リージョンを指定して、APIの速度が測れる環境を作るか悩んでた時に、Postman Monitor で測れることが分かったので、 そちらを試してみました。


## ただ
![image](/images/2023-12-17/region.png)

:::message
リージョンがどこの都市とか細かい情報がないんです。
:::

速度測りたいのに、どこからのリクエストかわからないんじゃ測り用がない。
ってことでどこから来ているのか調べました。

## 用意したもの
### Collection
@[card](https://www.postman.com/ftechno-dev/workspace/presentation/collection/16629267-dfb3160c-717f-4c54-9898-e009b86c9d37)

### Monitor 
@[card](https://www.postman.com/ftechno-dev/workspace/presentation/monitor/Postman-Monitor--------------~1eea0c51-9248-47d0-b976-017e7e893ae0)

### プログラム
こちらにアップしてあります。
@[card](https://gitlab.com/future-techno-developers/public/cloudflare-worker-collection/momento-cache-worker)

```Typescript: index.ts
app.get("/", async (c) => {
	console.log("country:", c.req.raw.cf.country);
	console.log("city:", c.req.raw.cf.city);

	return new Response(
		JSON.stringify({
			version: "0.0.1",
			country: c.req.raw.cf.country,
			city: c.req.raw.cf.city,
			continent: c.req.raw.cf?.continent,
		}),
	);
});
```
- Honoで書いてます
- Cloudflare Worker は cf にcountry, cityが入っている。これはデータセンターの位置を表してます。

### デプロイ
こちらにデプロイしてあるので、手元で叩いてみても良いです。
https://momento-cache-worker.future-techno-developers.workers.dev/

:::message
Cloudflare Worker のセットアップ方法は割愛します。
:::

## 結果
結果は wrangler コマンドを使います。これは記事読者には権限ないですが、プロジェクトコピーしてデプロイしてみるなどで試してみてください。

```Bash
 wrangler tail --format=pretty momento-cache-worker -c config/wrangler.dev.toml   
```

```bash:モニター実行したログ
GET https://momento-cache-worker.future-techno-developers.workers.dev/ - Ok @ 12/23/2023, 3:56:25 PM
  (log) country: US
  (log) city: Boardman
GET https://momento-cache-worker.future-techno-developers.workers.dev/ - Ok @ 12/23/2023, 3:56:25 PM
  (log) country: US
  (log) city: Boardman
GET https://momento-cache-worker.future-techno-developers.workers.dev/ - Ok @ 12/23/2023, 3:56:25 PM
  (log) country: CA
  (log) city: Montreal
GET https://momento-cache-worker.future-techno-developers.workers.dev/ - Ok @ 12/23/2023, 3:56:25 PM
  (log) country: US
  (log) city: Ashburn
GET https://momento-cache-worker.future-techno-developers.workers.dev/ - Ok @ 12/23/2023, 3:56:25 PM
  (log) country: GB
  (log) city: London
GET https://momento-cache-worker.future-techno-developers.workers.dev/ - Ok @ 12/23/2023, 3:56:25 PM
  (log) country: US
  (log) city: Ashburn
GET https://momento-cache-worker.future-techno-developers.workers.dev/ - Ok @ 12/23/2023, 3:56:25 PM
  (log) country: SG
  (log) city: Singapore
GET https://momento-cache-worker.future-techno-developers.workers.dev/ - Ok @ 12/23/2023, 3:56:26 PM
  (log) country: DE
  (log) city: Frankfurt am Main
GET https://momento-cache-worker.future-techno-developers.workers.dev/ - Ok @ 12/23/2023, 3:56:26 PM
  (log) country: BR
  (log) city: São Paulo
```
### まとめ

| リージョン           |Country|City|
|-----------------|---|---|
| US(West)        |US|Boardman|
| Canada(Central) |CA|Montreal|
| US(East)        |US|Ashburn|
| United Kingdom  |GB|London|
| Asian Pacific   |SG|Singapore|
| Europe(Central) |DE|Frankfurt am Main|
| South America   |BR|São Paulo|

:::message
BoardmanとAshburn は同じUSで出ましたが、東海岸と西海岸の都市なので、そうだろうということで記載してます。
:::
## お気持ち

https://twitter.com/xiombatsg/status/1734241175161430023
https://twitter.com/xiombatsg/status/1734527547658932684

## 最後に
Asian Pacificがシンガポールって出た時に思わず呟くほどの衝撃でした。Postman 日本法人が立ち上がった今！是非東京リージョンを作って欲しいです。

## 番外編
ちなみ、ブラウザとアプリでもリージョンが異なります。
遅いなって思ったら気にしてみても良いとは思います。

:::message
大多数のケースではアプリで利用した方が良いと思います。
:::

### ブラウザは US, アプリは近いところから（私の手元では JP,Tokyo)
#### ブラウザ
![image](/images/2023-12-17/browser.png)

```json
{"version":"0.0.1","country":"US","city":"Ashburn","continent":"NA"}
```
#### アプリ
![image](/images/2023-12-17/appli.png)

```json
{"version":"0.0.1","country":"JP","city":"Tokyo","continent":"AS"}
```

:::message
VS Code は試していませんが、おそらくアプリと同じ挙動をすると思います。
:::



## 参考
@[card](https://qiita.com/rikosaita/items/9a80cd3213b624ae1a3c)

