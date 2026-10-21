---
title: RTX 3050 が PCIe から落ちる不具合に対処した
date: 2026-10-21
tags:
  - linux
  - gpu
---

自分用のメモ。
Ubuntu のnvidia gpuマシンで、定期的にgpuが認識しなくなったりnvidia-smiが動かなくなることがあった。
どうやら2枚挿している GPU のうち RTX3050 の方だけが認識しなくなっていたらしい。（主にclaudeが）対処した経過を残しておく。

## 環境

- GPU: RTX 3060 (CPU 直結のスロット) と RTX 3050 (チップセット側の2番目の x4 スロット)
- マザーボード: ASRock B550 Phantom Gaming 4
- ドライバ: 555.42.06
- 用途: Ollama でローカル LLM を動かしている

## 症状

Ollama に `/api/generate` を投げた1秒後に、RTX3050が認識されなくなった。カーネルログにはこう残っていた。

```
NVRM: Xid (PCI:0000:04:00): 79, GPU has fallen off the bus.
```

過去のログをさかのぼると、同じ Xid 79 が 2026-06-17 にも出ていた。どちらも落ちるのは 3050 (04:00.0) で、3060 は正常だった。モデルをロードするときの負荷の変化がきっかけになっていたっぽい？

落ちた状態では `lspci` が `Unknown header type 7f` を返す。ドライバではなく PCIe のリンクで切れている。

## 原因の見立て

Xid 79 は、チップセット経由で使っている2枚目の GPU でよく報告されている。挙がっていた原因は次のとおり。

- 補助電源ケーブルの接触不良や容量不足
- スロットの信号品質 (x4 は余裕が少ない)
- ASPM などの省電力の状態遷移

手を付けやすい省電力から順に潰すことにした。

## やったこと

### 1. BIOS の確認

BIOS の Advanced → AMD PBS にある `PM L1 SS` はもともと Disabled だった。ここは変更なし。

この状態でos側からpciの状態を見てみる。
```bash
sudo lspci -vvv -s 04:00.0 | grep -E "LnkCtl:|L1SubCtl1:|LnkSta:"
```

```
LnkCtl: ASPM L1 Enabled; ...
LnkSta: Speed 2.5GT/s (downgraded), Width x4 (downgraded)
L1SubCtl1: PCI-PM_L1.2- PCI-PM_L1.1- ASPM_L1.2- ASPM_L1.1-
```

L1 サブステートは切れていたが、ASPM L1 は有効のままだった。
どうして、、、、、、こういうのってBIOS側が優先されるもんじゃないの？？？？

### 2. ASPM を実行時に切る

これは再起動なしでも反映されるらしい。

```bash
echo performance | sudo tee /sys/module/pcie_aspm/parameters/policy
```

`LnkCtl` が `ASPM Disabled` に変わったので、カーネル側から切れることが分かった。

### 3. 起動オプションで固定する

`/etc/default/grub` の `GRUB_CMDLINE_LINUX_DEFAULT` に `pcie_aspm.policy=performance` を足し、`sudo update-grub` した。
`pcie_aspm=off` はBIOSの設定を書き換えるほどの威力がない、らしい。
（そもそもBIOSの設定が上書きされていたのはさておき）

### 4. Persistence Mode

nvidiaのgpuでは、persistence modeというのを有効化すると、電源状態を固定することができるらしい。

`nvidia-persistenced` は動いていたが、Persistence Mode は Disabled のままだった。モデルのロードとアンロードのたびに電源状態が遷移するのを減らしたいので、有効にして永続化する方針にした。

Ubuntu のパッケージのユニットは `--no-persistence-mode` 付きで起動することがある。その場合は `systemctl edit nvidia-persistenced` で `ExecStart` を上書きして、このオプションを外す。

```
[Service]
ExecStart=
ExecStart=/usr/bin/nvidia-persistenced --user nvidia-persistenced --verbose
```

## 所感
基本的にこのエラーは電源の不安定性に起因するものらしく、よくあるATXマザボに無理やり2枚のgpuを載せた際、しょぼい方のpcieスロットに刺さってるgpuが認識しなくなってしまう。
あんまり分かってなかったが、メインのx16レーンはcpuに直結している一方で、サブのx4レーンはチップセットを経由するらしい。
そのせいで電源管理の事情もどうやら違ってくるらしい。
確か補助電源の配線は一応気を使ったはずで、電源ユニットの根本から別のケーブルでそれぞれに供給しているはずだ。
組んだ時に一応気を使っておいて良かったな、それでもこういう不具合が起きてくるんだな、という学びになった。
一般家庭用pcで頑張ろうとするとこういうところで引っかかるんだなぁ、多分サーバー用とかの豪華なcpuとかチップセットだとこうはならんのやろなぁ、とも思った。
（一旦これで様子を見るが、それでも改善しなければ次は電源ユニット交換くらいしか打つ手がない、、、、）
