---
layout: post
title: "hUGETrackerでゲームボーイBGMを作成してROMで再生する"
date: 2026-06-15
categories: [gameboy-dev]
tags: [gameboy, rgbds, hugetracker, audio, music]
---

## はじめに

前回の記事では、RGBDSで作成したゲームボーイROMに日本語フォントを表示するところまで進めました。
PNGで作成したフォントを2bppへ変換し、`charmap` を使ってソースコード内に日本語文字列を書けるようにしました。

今回はその続きとして、ゲームボーイでBGMを鳴らすことに挑戦します。

画面に文字を出せるようになると、次に欲しくなるのは音です。
ゲームボーイには独特の音源があり、矩形波、波形メモリ、ノイズを使って音楽や効果音を鳴らせます。
ただし、レジスタを直接叩いて曲を作るのは初心者にはかなり大変です。

そこで今回は、Game Boy向けの作曲ツールである `hUGETracker` と、ROM上で曲を再生するための `hUGEDriver` を使いました。
hUGETrackerでBGMを作成し、RGBDS形式でエクスポートして、自作ROMに組み込んで再生するところまでを記録します。

この記事では、単に手順だけを書くのではなく、実際に作業中に躓いた点も残しておきます。
特に日本語キーボードでの入力、Instrumentの音量調整、`hUGE_init` の引数、`hUGE_dosound` を呼ぶタイミングは、初心者が詰まりやすいポイントだと感じました。

## hUGETrackerとは

`hUGETracker` は、Game Boy向けの音楽を作成できるトラッカー形式の作曲ツールです。
画面上に音符やエフェクトを入力して、ゲームボーイの音源に近い形で曲を作れます。

hUGETrackerで作成した曲は、`hUGEDriver` と組み合わせることで実際のROM上で再生できます。
RGBDS向けの `.asm` ファイルとしてエクスポートできるため、これまで作ってきたRGBDSプロジェクトにも組み込みやすいです。

また、サンプル曲も付属しています。
最初から自分で曲を作る前に、サンプルを開いて再生できるか確認できるので、導入時の確認にも使いやすいです。

## hUGETrackerの導入

今回はWindows版のhUGETrackerを使用しました。
GitHub ReleasesからWindows向けのビルドをダウンロードし、展開して起動しました。

まずは起動確認です。
起動直後に画面が表示され、サンプル曲を読み込んで再生できることを確認しました。

![hUGETrackerを起動した直後の画面]({{ site.baseurl }}/assets/images/2026-06-15-hugetracker-bgm-playback/hugetracker_startup.png)

ここでサンプル曲が鳴れば、ツール自体の導入はひとまず成功です。
最初は画面にたくさんの列や数字が並んでいて難しそうに見えましたが、最低限見る場所はそれほど多くありません。

主に触るのは次のあたりです。

- `Duty 1`、`Duty 2`、`Wave`、`Noise` の各チャンネル
- Instrument設定
- Patternの音符入力欄
- 再生ボタン
- Export機能

最初は全部を理解しようとせず、サンプルを鳴らしてから少しずつ触るのが良さそうです。

## 日本語キーボードで最初に躓いた点

最初に大きく躓いたのは、音符の入力でした。

hUGETrackerでは、PCキーボードを鍵盤のように使って音符を入力します。
しかし、初期状態ではなぜか `B-8` しか入力できず、他の音がうまく入りませんでした。

最初は使い方を間違えているのかと思いました。
Patternの行を選択し、Instrumentも選び、音符を入力しているつもりなのに、期待した音階になりません。
そこでマニュアルを確認しながら、キー入力とOctave設定の関係を調べました。

原因は、日本語キーボードと英語キーボードの配列差異でした。
hUGETrackerの標準キーマップは英語キーボードを前提にしているため、日本語キーボードでは一部のキー配置が合わず、想定どおりに音符を入力できませんでした。

また、Octave設定も重要でした。
Octaveを適切に設定すると、同じキーでも入力される音域が変わります。
最初はこの関係が分かっておらず、入力された音が極端に高かったり、意図しない音になったりしていました。

ここで、標準キーマップをそのまま使うのは諦めて、Custom Key Mapを設定することにしました。

## Custom Key Mapの設定

日本語キーボードでは標準配列が使いづらかったため、Custom Key Mapを利用して独自配列を作成しました。

最終的には、以下のような配列にしました。

```text
  S D   G H J
Z X C   V B N M
```

下段を白鍵、上段を黒鍵として使うイメージです。

対応は次のようにしました。

- `Z` = C
- `S` = C#
- `X` = D
- `D` = D#
- `C` = E
- `V` = F
- `G` = F#
- `B` = G
- `H` = G#
- `N` = A
- `J` = A#
- `M` = B

![日本語キーボード向けに設定したCustom Key Map]({{ site.baseurl }}/assets/images/2026-06-15-hugetracker-bgm-playback/custom_keymap.png)

この設定にすると、キーボードの左下あたりを鍵盤として扱えるようになります。
Octave設定とも連動しており、Octaveを変更すると同じキー配置のまま音域を上下できます。

この設定を作ってから、音符入力がかなり楽になりました。
日本語キーボードでhUGETrackerを使う場合、Custom Key Mapは最初に設定しておくと良いと思います。

## Instrumentの作成

次にInstrumentを作成しました。

今回作成した音色は以下です。

- `Lead`
- `Base`
- `Coin`
- `Jump`
- `Laser`

まずはメロディ用の `Lead` と、伴奏用の `Base` を作りました。

`Lead` の設定は以下です。

- Duty 50%
- Start Vol 15
- Direction Down
- Change 3

`Base` の設定は以下です。

- Duty 25%
- Start Vol 13
- Direction Down
- Change 5

![作成したInstrument設定]({{ site.baseurl }}/assets/images/2026-06-15-hugetracker-bgm-playback/instruments.png)

最初は伴奏の音量が大きすぎて、メロディがほとんど聞こえなくなりました。
単純に `Start Vol` を下げれば解決すると思っていましたが、実際にはそれだけではうまくいきませんでした。

重要だったのは `Change` です。
`Start Vol` は音の開始時の音量ですが、`Change` は音量変化の速さに関係します。
同じ開始音量でも、減衰の仕方が違うと聞こえ方がかなり変わります。

試しながら確認したところ、今回の設定では `Change` は次のような感覚でした。

- `1` が最も急激に減衰する
- `7` が最も緩やかに減衰する

つまり、短く切れる音にしたい場合は小さめの値、長く残る音にしたい場合は大きめの値が向いていました。

伴奏を目立たせすぎないようにするには、`Start Vol` だけでなく `Change` も合わせて調整する必要があります。
最終的には、実際に曲を再生しながら、メロディが前に出るように耳で調整しました。

## きらきら星を作ってみる

練習曲として「きらきら星」を作ってみました。

最初からオリジナル曲を作ると、作曲の問題なのか、ツールの使い方の問題なのか、ROM組み込みの問題なのか分からなくなります。
そこで、誰でも知っていて音の間違いに気付きやすい曲を選びました。

構成は次のようにしました。

- `Duty 1` にメロディ
- `Duty 2` に伴奏
- `Noise` に簡単なリズム
- `Wave` は試したが最終的には未使用

![きらきら星のPattern入力画面]({{ site.baseurl }}/assets/images/2026-06-15-hugetracker-bgm-playback/patterns.png)

`Duty 1` にはメインメロディを入力しました。
`Duty 2` には、メロディを邪魔しない程度の伴奏を入れました。

途中で `Wave` チャンネルもベースとして使えないか試しました。
しかし、期待したほど低音の支えにならず、音色によってはノイズっぽく聞こえました。
Wave用の波形をきちんと作り込めば使えるのかもしれませんが、今回は初心者向けの練習として、無理に使わないことにしました。

一方で、`Noise` チャンネルはリズム用としてかなり便利でした。
いくつか音程を試したところ、`C-8` 付近が扱いやすく感じました。
短いノイズ音を一定間隔で入れるだけでも、曲らしさがかなり増します。

メロディだけだと単なる音階の確認に近い印象でしたが、伴奏とリズムを入れると一気にBGMらしくなりました。

## RGBDS形式でエクスポート

曲ができたら、RGBDS向けにエクスポートします。

hUGETrackerのメニューから `Export RGBDS .asm` を選択しました。
このとき、Song descriptorを入力する必要があります。

今回は次の名前にしました。

```text
bgm_twinkle
```

出力ファイル名は以下にしました。

```text
bgm_twinkle.asm
```

![RGBDSエクスポート時に入力したSong descriptor]({{ site.baseurl }}/assets/images/2026-06-15-hugetracker-bgm-playback/song_descriptor.png)

ここで入力したSong descriptorは、RGBDS側から曲データを参照するときのラベル名になります。
今回であれば、初期化時に `bgm_twinkle` を指定します。

```rgbasm
ld hl, bgm_twinkle
call hUGE_init
```

ファイル名とラベル名は別物ですが、初心者のうちは同じ名前にしておくと混乱しにくいです。

## RGBDSプロジェクトへの組み込み

エクスポートした曲を、前回までに作っていたRGBDSプロジェクトへ組み込みました。

追加したファイルは以下です。

```text
include/
├─ hUGE.inc
└─ hUGE_note_table.inc

src/
├─ hUGEDriver.asm
└─ music/
   └─ bgm_twinkle.asm
```

`hUGE.inc` と `hUGE_note_table.inc` は、hUGEDriverを使うために必要なincludeファイルです。
`hUGEDriver.asm` が再生処理本体で、`bgm_twinkle.asm` がhUGETrackerからエクスポートした曲データです。

`main.asm` 側では、必要なファイルを読み込みます。
プロジェクトの構成によって書き方は変わりますが、例としては以下のようになります。

```rgbasm
INCLUDE "hardware.inc"
INCLUDE "hUGE.inc"

INCLUDE "src/music/bgm_twinkle.asm"
INCLUDE "src/hUGEDriver.asm"
```

Makefileで複数の `.asm` ファイルを個別にアセンブルしてリンクしている場合は、`INCLUDE` ではなくオブジェクトファイルとして扱う構成でも良いです。
今回はまず動かすことを優先し、分かりやすい形で組み込みました。

## hUGEDriver導入時に躓いた点

hUGEDriverを導入するときにも、いくつか躓きました。

まず、`hUGE_note_table.inc` が必要でした。
最初は `hUGE.inc` と `hUGEDriver.asm` だけあれば動くと思っていましたが、アセンブル時に参照エラーが出ました。
ドライバ側のソースを確認すると、ノートテーブル用のincludeが必要なことが分かりました。

次に、`hUGE_init` に渡すレジスタを勘違いしました。

当初は、曲データのアドレスを `DE` レジスタで渡すのだと思っていました。
しかし、実際のドライバソースを確認すると、`HL` レジスタで渡す必要がありました。

正しい初期化コードは以下です。

```rgbasm
ld hl, bgm_twinkle
call hUGE_init
```

このように、曲データのラベルを `HL` に入れてから `hUGE_init` を呼びます。

動かないときは、サンプルコードだけでなく、実際のドライバソースを読むのが大事だと感じました。
特にアセンブリでは、どのレジスタで値を渡すかがずれるだけで正常に動きません。

## VBlank割り込み対応

次に躓いたのが、`hUGE_dosound` を呼ぶタイミングです。

最初はメインループから `hUGE_dosound` を呼んでいました。

```rgbasm
.mainLoop
    call hUGE_dosound
    halt
    jr .mainLoop
```

これでも一応音は鳴りました。
しかし、曲がおかしく聞こえました。
テンポが安定せず、hUGETracker上で聞いた曲と比べると再生タイミングに違和感がありました。

原因は、`hUGE_dosound` を安定した周期で呼べていなかったことです。
hUGEDriverの再生処理は、基本的に一定間隔で呼ばれる前提です。
ゲームボーイでは、VBlank割り込みに合わせて呼ぶのが自然です。

そこで、VBlank割り込みから `hUGE_dosound` を呼ぶように修正しました。

例としては、以下のような形です。

```rgbasm
SECTION "VBlank Interrupt", ROM0[$0040]
    jp VBlankHandler

SECTION "VBlank Handler", ROM0
VBlankHandler:
    push af
    push bc
    push de
    push hl

    call hUGE_dosound

    pop hl
    pop de
    pop bc
    pop af
    reti
```

割り込みを使う場合は、初期化時にVBlank割り込みを有効にします。

```rgbasm
ld a, %00000001 ; VBlank割り込みを有効化
ldh [rIE], a
ei
```

この修正で、再生品質が大きく改善しました。
ROM上でも「きらきら星」として認識できるレベルになり、hUGETrackerで再生した内容にかなり近づきました。

メインループから呼んでも音が出るため、最初は問題に気付きにくいです。
しかし、BGMとして安定して鳴らすには、VBlank割り込みから呼ぶ構成にした方が良さそうです。

## 完成

最終的に、hUGETrackerで作成した「きらきら星」をROM上で再生できました。

![ROM上でBGM再生を確認している画面]({{ site.baseurl }}/assets/images/2026-06-15-hugetracker-bgm-playback/rom_playback.png)

今回実現できたことは以下です。

- hUGETrackerで作曲
- Instrumentを作成
- Patternへメロディ、伴奏、リズムを入力
- RGBDS形式へエクスポート
- hUGEDriverをRGBDSプロジェクトへ組み込み
- ROM上でBGMを再生

前回までは画面表示が中心でしたが、今回は音が鳴るようになりました。
文字が表示され、さらにBGMが鳴ると、一気にゲームらしさが出ます。

## まとめ

hUGETrackerは、Game Boy向けの作曲ツールとして非常に使いやすいと感じました。
最初はトラッカー画面に慣れる必要がありますが、サンプル曲を再生しながら少しずつ触ると理解しやすいです。

hUGEDriverを使うことで、RGBDSプロジェクトへの組み込みも比較的簡単でした。
ただし、`hUGE_note_table.inc` の追加、`hUGE_init` への `HL` 渡し、`hUGE_dosound` をVBlank割り込みから呼ぶ点は、最初に確認しておくと詰まりにくいです。

また、日本語キーボードではCustom Key Mapの設定が非常に有効でした。
標準配列でうまく音符を入力できない場合は、早めに独自配列を作った方が作業しやすくなります。

今後は、今回作った「きらきら星」だけでなく、以下にも挑戦したいです。

- オリジナル曲の作成
- 効果音の作成
- ゲーム本編への組み込み

前回の日本語フォント表示に続いて、今回はBGM再生まで進められました。
画面と音の両方が動き始めたので、次は実際のゲームらしい処理に組み込んでいきたいと思います。
