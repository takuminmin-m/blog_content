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

## 記事を更新するとき

- 誤字や表現の修正：そのまま直す。履歴は git に残る。
- 内容の追記：`date` とファイル名は変えずに、`## 追記（2026-10-05）` のように日付付きで書き足す。
- 内容が大きく変わる：新しい記事を書き、古い記事の冒頭からリンクする。
