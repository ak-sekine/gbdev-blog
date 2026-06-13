---
layout: post
title: "RGBDSで最初のゲームボーイROMを作る"
date: 2026-06-13 10:00:00 +0900
category: gameboy-dev
tags: [gameboy, rgbds, assembly, bgb]
---

前回の記事では、WSL2上にRGBDSを導入し、コマンドを実行できるところまで確認しました。

今回はその続きとして、最初のゲームボーイROMを作成します。
RGBDSで `hello.gb` を生成し、BGBで正常に起動できるところまで進めます。

ここで出てくる `hello` は、プロジェクト名とROMファイル名の意味です。
この記事の時点では、まだ文字表示は行いません。

## 作成する構成

まずは作業用ディレクトリを作成します。

```bash
mkdir -p ~/gbdev/hello
cd ~/gbdev/hello
```

最終的には以下のような構成にします。

```text
hello/
├── Makefile
├── include/
│   └── hardware.inc
├── obj/
│   └── main.o
├── src/
│   └── main.asm
└── hello.gb
```

`src` にはアセンブリのソースコードを置きます。
`include` にはRGBDS向けの定義ファイルを置きます。
`obj` にはビルド途中に生成される `.o` ファイルを置くようにします。

## hardware.incを取得する

ゲームボーイのレジスタ名や便利な定数を使うために、`hardware.inc` を取得します。

```bash
mkdir -p include
curl -L -o include/hardware.inc https://raw.githubusercontent.com/gbdev/hardware.inc/master/hardware.inc
```

この記事では `rLCDC` や `rBGP`、`LCDC_ON`、`LCDC_BG_ON` などの定義を `hardware.inc` から利用します。

## src/main.asmを作成する

ソースコード用のディレクトリを作成します。

```bash
mkdir -p src
```

`src/main.asm` を作成し、以下の内容を書きます。

```rgbasm
INCLUDE "hardware.inc"

SECTION "Header", ROM0[$100]
    jp Entry
    ds $150 - @, 0

SECTION "Main", ROM0[$150]
Entry:
    di
    ld sp, $FFFE

.waitVBlank
    ldh a, [rLY]
    cp 144
    jr c, .waitVBlank

    xor a
    ldh [rLCDC], a
    ldh [rSCX], a
    ldh [rSCY], a

    ld a, %11100100
    ldh [rBGP], a

    ld hl, _SCRN0
    ld bc, 32 * 32
.clearBG
    xor a
    ld [hli], a
    dec bc
    ld a, b
    or c
    jr nz, .clearBG

    ld a, LCDC_ON | LCDC_BG_ON
    ldh [rLCDC], a

.loop
    halt
    jr .loop
```

最初の `SECTION "Header", ROM0[$100]` は、ゲームボーイの起動後に実行される位置です。
`rgbfix` でROMヘッダーを補正するため、ここでは `$150` まで領域を空けています。
この記事では、背景を初期化してLCDを有効にするところまでを確認します。

## Makefileを作成する

次に `Makefile` を作成します。

```makefile
RGBASM := rgbasm
RGBLINK := rgblink
RGBFIX := rgbfix

SRC := src/main.asm
OBJ := obj/main.o
ROM := hello.gb

all: $(ROM)

$(ROM): $(OBJ)
	$(RGBLINK) -o $@ $<
	$(RGBFIX) -v -p 0xFF $@

$(OBJ): $(SRC) include/hardware.inc | obj
	$(RGBASM) -I include/ -o $@ $<

obj:
	mkdir -p obj

clean:
	rm -rf obj $(ROM)

.PHONY: all clean
```

`rgbasm` でオブジェクトファイルを作成し、`rgblink` でROMにリンクします。
最後に `rgbfix` を実行して、ゲームボーイROMとして必要なヘッダー情報を補正します。

## hello.gbを生成する

以下を実行します。

```bash
make
```

成功すると、プロジェクト直下に `hello.gb` が作成されます。

```text
hello.gb
```

ビルドし直したい場合は、以下で生成物を削除できます。

```bash
make clean
make
```

## BGBで実行する

Windows側でBGBを起動し、生成された `hello.gb` を開きます。

WSL2上のファイルは、エクスプローラーから以下のようなパスで参照できます。

```text
\\wsl$\Ubuntu\home\<ユーザー名>\gbdev\hello
```

このフォルダにある `hello.gb` をBGBへドラッグします。

## ハマったポイント

### Nintendoロゴで止まった

最初はBGBで起動しても、Nintendoロゴの画面から先へ進みませんでした。

![Nintendoロゴで止まった画面]({{ site.baseurl }}/assets/images/2026-06-13-rgbds-first-rom/01-nintendo-logo.png)

原因は、背景の初期化やLCD制御が不足していたことです。
背景マップを初期化してからLCDを有効化する処理を追加しました。

特に、LCDを有効にする部分では以下の定数を使いました。

```rgbasm
ld a, LCDC_ON | LCDC_BG_ON
ldh [rLCDC], a
```

古いサンプルでは `LCDCF_ON` や `LCDCF_BGON` が使われていることがあります。
しかし、今回使った `hardware.inc` では `LCDC_ON` と `LCDC_BG_ON` を使いました。

古い記事やサンプルコードを参考にする場合は、手元の `hardware.inc` に定義されている名前を確認したほうがよさそうです。

### main.oをBGBにドラッグしていた

もう一つの原因は、誤って `hello.gb` ではなく `main.o` をBGBにドラッグしていたことです。

その場合、BGB側でヘッダーチェックエラーが表示されました。

![main.oを開いたときのヘッダーチェックエラー]({{ site.baseurl }}/assets/images/2026-06-13-rgbds-first-rom/02-header-error.png)

`.o` ファイルはビルド途中のオブジェクトファイルで、ゲームボーイROMではありません。
BGBで開くのは、必ず `rgbfix` まで完了した `hello.gb` です。

間違えて開かないように、`Makefile` を修正して `.o` ファイルを `obj` ディレクトリへ分離しました。
これでプロジェクト直下には `hello.gb` が目立つようになり、開くファイルを間違えにくくなりました。

## 正常起動を確認

最終的に、BGB上でNintendoロゴの後に進み、ROMが正常に起動するところまで確認できました。

![正常に起動した画面]({{ site.baseurl }}/assets/images/2026-06-13-rgbds-first-rom/03-hello.png)

今回作成したROMでは、まだ文字や画像は表示していません。
それでも、RGBDSでソースコードを書き、ROMを生成し、エミュレータで正常起動を確認する一連の流れを確認できました。

次回は、日本語フォントを用意して、ゲームボーイ画面に日本語を表示するところまで進めます。
