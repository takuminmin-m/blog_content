---
title: Rails 8.1 と Ruby 4.0 に上げた
date: 2026-09-25
tags:
  - rails
  - ruby
---
しばらく放置していたこのブログのコードを、Ruby 4.0.7 と Rails 8.1 に上げました。

## 引っかかったところ

画像のウォーターマークが、実は一度も動いていませんでした。Active Job は `Pathname` を引数に取れないので、画像を保存した時点で例外になっていたのです。

```ruby
class Picture < ApplicationRecord
  OVERLAY = Rails.configuration.x.content_root.join("overlay.png").to_s

  has_one_attached :image do |attachable|
    attachable.variant :article, resize_to_limit: [ 1000, 1000 ],
      composite: [ OVERLAY, { gravity: "south_east" } ], preprocessed: true
  end
end
```

> テストがないと、こういう壊れ方には気づけない。

今は CI でテストと RuboCop、Brakeman が回るようになっています。
