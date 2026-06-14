---
layout: post
title: "RGBDSでゲームボーイに日本語フォントを表示する"
date: 2026-06-14 10:00:00 +0900
category: gameboy-dev
tags: [gameboy, rgbds, font, japanese, charmap]
---

前回の記事では、RGBDSで最初のゲームボーイROMを作成し、BGBやSameBoyで起動できるところまで確認しました。

今回はその続きとして、ゲームボーイ画面に日本語フォントを表示します。
PNG画像で作成したフォントをゲームボーイ用の2bppタイルデータへ変換し、RGBDSの `charmap` を使ってソースコード内に日本語文字列を直接書けるようにします。

## 今回やること

この記事では、以下の流れで日本語表示を行います。

- PNG画像からゲームボーイ用2bppフォントデータを作る
- `rgbgfx` を使ってPNGを2bppへ変換する
- `font_tiles.inc` でタイル番号を定義する
- `charmap.inc` で文字とタイル番号を対応付ける
- フォントデータをVRAMへ転送する
- BGマップへ文字列を書き込む
- 日本語文字列をソースコード内に直接記述する
- 濁点・半濁点を上付きタイルとして扱う

## フォント画像の準備

まず、ゲームボーイで表示するためのフォント画像を用意しました。

ドットエディタにはPixeloramaを使いました。
フォントには「NUきなこもち」を使用し、ゲームボーイ風の4色パレットに合わせて画像を作成しています。

使用したパレットは以下です。

```text
#E0F8D0
#88C070
#346856
#081820
```

ゲームボーイの背景タイルは8×8ドット単位で扱うため、フォント画像も8×8タイルで構成しました。
ひらがな、カタカナ、英数字をタイル単位で並べておき、後でそれぞれにタイル番号を割り当てます。

この段階では、PNG画像はまだゲームボーイが直接読める形式ではありません。
次に `rgbgfx` を使って、ゲームボーイ用の2bppデータへ変換します。

## PNGを2bppへ変換する

RGBDSには、PNG画像をゲームボーイ用のタイルデータへ変換する `rgbgfx` が含まれています。

最初は、Makefileに以下のようなルールを書いていました。

```makefile
$(GFX_DIR)/%.2bpp: $(ASSETS_DIR)/%.png
	mkdir -p $(dir $@)
	rgbgfx -o $@ $<
```

この書き方でも2bppファイルは生成できます。
しかし、画像内の色順が期待どおりにならず、表示したときの濃淡が想定とずれてしまいました。

そこで最終的には、`-c` オプションで4色の順番を明示しました。

```makefile
$(GFX_DIR)/%.2bpp: $(ASSETS_DIR)/%.png
	mkdir -p $(dir $@)
	rgbgfx -c "#E0F8D0,#88C070,#346856,#081820" -o $@ $<
```

これで、PNG上の色とゲームボーイ側の色番号を意図した順番で対応付けられます。

### rgbgfx -c embeddedで失敗した

途中で以下の指定も試しました。

```bash
rgbgfx -c embedded
```

これはPNGに埋め込まれたパレット情報を使う指定です。
しかし、今回作成したPNGには埋め込みパレットが存在しなかったため失敗しました。

そのため、今回は `-c "#E0F8D0,#88C070,#346856,#081820"` のように、Makefile側で色を直接指定する形にしました。

## font_tiles.incでタイル番号を定義する

次に、フォント画像内の各文字がどのタイル番号に対応するかを定義します。

`font_tiles.inc` には多数の文字を定義しますが、記事では一部だけ抜粋します。

```rgbasm
DEF TILE_END   EQU $00
DEF TILE_SPACE EQU $0B

DEF TILE_A EQU $10
DEF TILE_B EQU $11

DEF TILE_HIRA_A EQU $30 ; あ
DEF TILE_HIRA_I EQU $31 ; い
DEF TILE_HIRA_U EQU $32 ; う

DEF TILE_DAKUTEN        EQU $B0
DEF TILE_HANDAKUTEN     EQU $B1
DEF TILE_SUP_DAKUTEN    EQU $FE
DEF TILE_SUP_HANDAKUTEN EQU $FF
```

最初は古い書き方を参考にして、以下のように `EQU` だけで定義していました。

```rgbasm
TILE_A EQU $10
```

しかし、今回使っているRGBDSではこの形式だとエラーになりました。
最新版では以下のように `DEF` を付けます。

```rgbasm
DEF TILE_A EQU $10
```

また、最初は空白文字用の `TILE_SPACE` を `$00` にしていました。
しかし、文字列の終端にも `0` を使っていたため、空白を出したい場所で文字列が終わってしまいました。

そこで、文字列終端を表す `TILE_END` を `$00`、空白文字を表す `TILE_SPACE` を `$0B` に分離しました。

```rgbasm
DEF TILE_END   EQU $00
DEF TILE_SPACE EQU $0B
```

これで、空白と文字列終端を別の意味として扱えます。

## charmap.incで文字とタイル番号を対応付ける

RGBDSの `charmap` を使うと、ソースコード内に書いた文字を任意のバイト列へ変換できます。

たとえば、以下のように文字とタイル番号を対応付けます。

```rgbasm
charmap "A", TILE_A
charmap "B", TILE_B

charmap " ", TILE_SPACE

charmap "あ", TILE_HIRA_A
charmap "い", TILE_HIRA_I
charmap "う", TILE_HIRA_U
```

これにより、ソースコード内に `"あいう"` と書くと、アセンブル時に対応するタイル番号へ変換されます。

濁点付きの文字は、1文字を2タイルへ展開するようにしました。

```rgbasm
charmap "が", TILE_HIRA_KA, TILE_SUP_DAKUTEN
charmap "ぱ", TILE_HIRA_HA, TILE_SUP_HANDAKUTEN
```

この例では、`"が"` は「か」のタイルと上付き濁点タイルの2つに展開されます。
同じように、`"ぱ"` は「は」のタイルと上付き半濁点タイルの2つに展開されます。

濁点・半濁点を文字ごとに別タイルとして全部作る方法もあります。
ただし今回は、基本の文字タイルと上付き記号タイルを組み合わせることで、必要なタイル数を抑える方針にしました。

## フォントデータの読み込み

`main.asm` では、まず必要な定義ファイルを読み込みます。

```rgbasm
INCLUDE "hardware.inc"
INCLUDE "include/font_tiles.inc"
INCLUDE "include/charmap.inc"
```

`hardware.inc` はゲームボーイのレジスタ定義を使うためのファイルです。
`font_tiles.inc` はタイル番号の定義、`charmap.inc` は文字とタイル番号の対応付けに使います。

フォントデータ本体は、`INCBIN` でROM内へ読み込みます。

```rgbasm
FontTiles:
    INCBIN "build/gfx/font.2bpp"
FontTilesEnd:
```

ここで指定するパスは、Makefileが実際に出力する2bppファイルの場所と一致している必要があります。
今回は `rgbgfx` の出力先を `build/gfx/font.2bpp` にしたため、`INCBIN` も同じパスに合わせました。

起動時にはLCDを止めた状態で、フォントデータをVRAMへ転送します。
その後、BGマップ全体を `TILE_SPACE` で初期化してから、文字列を書き込みます。

処理の流れは以下のようになります。

```rgbasm
    ld de, FontTiles
    ld hl, _VRAM
    ld bc, FontTilesEnd - FontTiles
    call CopyBytes

    ld hl, _SCRN0
    ld bc, 32 * 32
    ld a, TILE_SPACE
    call FillBytes

    ld de, TextHello
    ld hl, _SCRN0 + 32 * 8 + 4
    call PrintText
```

`CopyBytes` でROM上のフォントデータをVRAMへコピーし、`FillBytes` でBGマップを空白タイルで埋めています。
最後に `PrintText` ルーチンで、指定した位置へ文字列を書き込みます。

文字列は以下のように、ソースコード内へ直接日本語で書けるようになりました。

```rgbasm
TextHello:
    db "こんにちは がっこう!", 0
```

末尾の `0` は文字列終端です。
`PrintText` はこの値を見つけるまで、1バイトずつBGマップへ書き込んでいきます。

## 日本語表示を確認

SameBoyでROMを実行し、日本語文字列が表示されることを確認しました。

今回は「こんにちは がっこう！」という文字列を表示しています。
濁点付きの「が」も、基本文字と上付き濁点タイルの組み合わせで表示できました。

これで、RGBDSのソースコード内に日本語文字列を直接書き、ゲームボーイ画面へ表示する流れができました。

![日本語フォント表示に成功した画面]({{ site.baseurl }}/assets/images/2026-06-14-rgbds-japanese-font/01-japanese-font-result.png)

## ハマったポイント

今回の作業では、いくつか詰まりやすい点がありました。

### EQU単独ではエラーになった

古いサンプルでは以下のような定義を見かけます。

```rgbasm
TILE_A EQU $10
```

しかし、今回のRGBDSでは `EQU` 単独の定義がエラーになりました。
以下のように `DEF ... EQU ...` の形式へ修正しました。

```rgbasm
DEF TILE_A EQU $10
```

### INCBINのパスが出力先と違っていた

`INCBIN` に指定したパスと、Makefileが生成する2bppファイルの場所が一致していないと、アセンブル時にファイルが見つかりません。

今回は、実際の出力先に合わせて以下のように修正しました。

```rgbasm
INCBIN "build/gfx/font.2bpp"
```

### ld hl, deは使えなかった

ルーチン内で `de` の値を `hl` へ移したい場面がありました。
しかし、RGBDSでは以下のような命令は使えません。

```rgbasm
ld hl, de
```

そのため、上位バイトと下位バイトを分けて代入しました。

```rgbasm
ld h, d
ld l, e
```

### TILE_SPACEと文字列終端が衝突した

最初は `TILE_SPACE` を `$00` にしていました。
しかし、文字列終端も `0` だったため、空白文字を出した時点で `PrintText` が終了してしまいました。

そこで、文字列終端と空白タイルを分けました。

```rgbasm
DEF TILE_END   EQU $00
DEF TILE_SPACE EQU $0B
```

これで、文字列内の空白を正しく表示できます。

### rgbgfx -c embeddedが失敗した

PNGに埋め込みパレットがある前提で、以下を試しました。

```bash
rgbgfx -c embedded
```

しかし、今回のPNGには埋め込みパレットが存在しなかったため失敗しました。
最終的には、Makefileで4色を明示しました。

```makefile
rgbgfx -c "#E0F8D0,#88C070,#346856,#081820" -o $@ $<
```

### VS Codeで日本語文字にUnicode警告が表示された

`main.asm` に日本語文字列を直接書くと、VS CodeでUnicode関連の警告が表示されることがあります。

日本語文字を意図して使っている場合は、設定で日本語ロケールを許可できます。

```json
"editor.unicodeHighlight.allowedLocales": {
    "ja": true
}
```

日本語以外の紛らわしい文字まで無条件に許可するのではなく、必要なロケールだけ許可しておくと安心です。

## まとめ

今回は、PNGで作成した日本語フォントをゲームボーイ用の2bppタイルデータへ変換し、画面に表示するところまで進めました。

`rgbgfx` を使うことで、PNGフォントをゲームボーイ用タイルへ変換できました。
また、`charmap` によって、RGBDSのソースコードへ日本語文字列を直接書けるようになりました。

濁点・半濁点については、基本文字と上付きタイルを組み合わせることで表示できました。
これで、日本語表示の基盤が完成しました。

今後は、この仕組みを使ってメッセージ表示やゲーム画面へ応用していきます。
