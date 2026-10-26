---
title: Mac アプリ版 Tailscale の CLI をターミナルから使えるようにした
date: 2026-10-26
tags:
  - tailscale
  - mac
  - claude
  - all-AI-generated
---

雑なメモ。この記事は Claude が全部書いた。

Mac に Tailscale.app は入れているが、ターミナルで `tailscale` と打つと `command not found` になる。

## 原因

CLI の本体はアプリの中にもう入っている。

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale version
```

これで普通にバージョンが出る。パスが通っていないだけだった。

## やったこと

`~/.local/bin`（もともと PATH に入っている）に、アプリ内の本体を呼ぶだけのスクリプトを置いた。

```sh
#!/bin/sh
exec /Applications/Tailscale.app/Contents/MacOS/Tailscale "$@"
```

```bash
chmod +x ~/.local/bin/tailscale
```

これで `tailscale status` や `tailscale ssh blog-raspi` が使える。

## メモ

- シンボリックリンクではなくスクリプトにしたのは、Mac アプリ版の本体はリンク経由で呼ぶとうまく動かないことがあるため。
- `brew install tailscale` は入れない。アプリと別のデーモンが立って二重になる。
- アプリを更新してもパスは変わらないので、スクリプトはそのままで動く。
