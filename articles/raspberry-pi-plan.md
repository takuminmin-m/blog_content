---
title: Raspberry Pi でブログを動かす計画
date: 2026-09-20
tags:
  - raspberry-pi
  - rails
---
家にある Raspberry Pi 3B+ でこのブログを動かし、Cloudflare Tunnel 経由で公開する予定です。

![机の上の Raspberry Pi](images/raspberry-pi.jpg)

## 構成

1. GitHub Actions で arm64 の Docker イメージをビルドする
2. Raspberry Pi が 5 分ごとに新しいイメージと記事の更新を確認する
3. Cloudflare Tunnel で、ルーターのポートを開けずに公開する

## 更新スクリプト

```bash
#!/bin/bash
set -euo pipefail

docker compose pull web
docker compose up -d
git -C content pull --ff-only && docker compose exec -T web bin/rails contents:sync
```

メモリが 1GB しかないので、ジョブは Puma の中で動かしてプロセスを増やさないようにします。
