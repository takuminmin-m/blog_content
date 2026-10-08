# blog_content

[takuminmin-m/blog](https://github.com/takuminmin-m/blog) の記事・固定ページ・写真。アプリの `content/` に置き、`bin/rails contents:sync` で取り込む。

## 構成

```
articles/<YYYY-MM-DD>-<slug>.md     # 記事
articles/images/                    # 記事の画像（git 管理外）
gallery/                            # ギャラリーの写真（git 管理外）
static_pages/index.md               # トップの紹介文
static_pages/about.md               # /about
overlay.png                         # 写真に重ねる透かし
```

写真は git で管理しない。元データは Mac に置き、サーバーへは別途コピーする。

## 命名規則

### 記事 `articles/<YYYY-MM-DD>-<slug>.md`

- 例：`2026-09-27-building-this-blog.md` → `/articles/2026-09-27-building-this-blog`
- 日付は公開日で、front matter の `date` と同じ日にする。
- slug は半角の英小文字・数字・ハイフンだけ。ドット（`.`）は使わない（URL の拡張子扱いになり、記事が開けない）。
- 公開したらファイル名は変えない。URL が変わり、リンクが切れる。

```yaml
---
title: 記事タイトル
date: 2026-09-27
tags:
  - rails
---
```

### 記事の画像 `articles/images/<記事のファイル名>-<説明>.<拡張子>`

- 例：`2026-09-27-building-this-blog-home-page.png`
- 本文からは `![説明](images/2026-09-27-building-this-blog-home-page.png)` で参照する。
- 記事のファイル名を頭に付けるので、どの記事の画像か分かり、ギャラリーの写真とも名前が重ならない（写真はファイル名だけで区別されるため、重複は不可）。

### ギャラリー `gallery/<撮影日 YYYY-MM-DD>-<説明>.<拡張子>`

- 例：`2026-09-21-cosmos.jpg`。同じ日に複数あれば `2026-09-21-cosmos-2.jpg`。
- ファイル名の降順で並ぶので、撮影日の新しい順になる。
- 日付は手で付けなくてよい。カメラの名前のまま置いて `bin/rails contents:date_gallery` を実行すると、日付のない写真に EXIF の撮影日が付く（`P1200205.jpeg` → `2025-03-27-P1200205.jpeg`）。同じ日の写真はカメラの連番順に並ぶ。撮影日のない写真（PNG など）と、付けた名前がすでにある写真はそのまま残り、一覧に理由が出る。
- 名前を変えるのは Mac の元画像に対してだけ。サーバーへコピーする前に実行する（サーバー側で変えると元画像と名前がずれる）。

### 共通

- 拡張子は小文字の `.jpg` / `.png`。
- 元画像は配信されない（透かし入りの縮小版だけ）ので、長辺 3000px 程度に書き出してから置けば十分。

## サイトに反映する手順

サーバーは Raspberry Pi で、5分ごとに自動で更新を取り込む。記事と写真で手順が違う。

| 変えたもの | やること | 反映 |
| --- | --- | --- |
| 記事・固定の紹介文（`articles/*.md`、`static_pages/`） | このリポジトリを commit して push | 5分以内 |
| 写真（`articles/images/`、`gallery/`） | アプリのリポジトリで `deploy/bin/push-photos <ssh host>` | 実行した直後 |

### 記事（Markdown）

```bash
git add articles static_pages
git commit -m "add 記事名"
git push
```

Pi が5分ごとに pull して取り込む。記事の本文はリクエストのたびにファイルから読むので、本文だけの修正は pull されればすぐ変わる。タイトル・日付・タグを変えたときも、同じ更新で DB に取り込まれる。

### 写真

写真は git に入らないので、Mac から Pi に直接送る。アプリのリポジトリ（`~/Documents/blog`）で実行する。

```bash
deploy/bin/push-photos <ssh host>    # 例：deploy/bin/push-photos pi@blog.local
```

これが順に行うこと：

1. `bin/rails contents:date_gallery` で、日付のないギャラリーの写真に EXIF の撮影日を付ける（元画像のある Mac でしかできない）。
2. `rsync --delete` で `articles/images/` と `gallery/` を Pi に送る。**Mac で消した写真は Pi からも消える。**
3. Pi の `deploy/bin/update` を実行し、写真を DB に取り込む。タイマーを待たずに反映される。

日付を付けるだけで送らないときは、アプリのリポジトリで `bin/rails contents:date_gallery` を実行する。

### 記事に写真を載せるとき

記事と写真は別々に送るので、**写真を先に送ってから記事を push する**。順序が逆だと、記事が存在しない画像を参照している間は表示が崩れる。

1. `articles/images/` に画像を置く。
2. `deploy/bin/push-photos <ssh host>`
3. 記事を commit して push。

### うまく反映されないとき

- 写真の送信が `No photos in content/` で止まる：Mac の `content/` に写真がない。誤って Pi の写真を全消去しないための停止なので、`content/` が正しい場所にあるか確かめる。
- 記事が出ない：front matter に `title` と `date` があるか確かめる。足りないと取り込みが失敗する。
- 同じ名前の写真が2つある：写真はファイル名だけで区別される。`articles/images/` と `gallery/` の間でも重複させない。
- Pi の状況：`ssh <host>` して `journalctl -u blog-update -f` で更新のログを見る。

## 記事を更新するとき

- 誤字や表現の修正：そのまま直す。履歴は git に残る。
- 内容の追記：`date` とファイル名は変えずに、`## 追記（2026-10-05）` のように日付付きで書き足す。
- 内容が大きく変わる：新しい記事を書き、古い記事の冒頭からリンクする。
