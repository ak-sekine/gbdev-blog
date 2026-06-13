---
layout: post
title: "WSL2でRGBDS開発環境を構築する"
date: 2026-06-07
categories:
  - gameboy-dev
tags:
  - gameboy
  - rgbds
  - wsl2
  - vscode
---

ゲームボーイ向けソフト開発を始めるために、Windows 11 + WSL2環境へRGBDSを導入しました。

この記事では、実際に環境構築した際の手順と、途中で遭遇したトラブルの解決方法をまとめます。

RGBDSにはパッケージマネージャーやビルド済みバイナリを使う方法もありますが、今回はWSL2上でソースコードからビルドします。

## 開発環境

- Windows 11
- WSL2 (Ubuntu)
- Visual Studio Code
- RGBDS

## 基本ツールのインストール

まずはUbuntuのパッケージを更新し、RGBDSのビルドに必要なパッケージをまとめてインストールします。

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y build-essential git make bison pkg-config libpng-dev
```

## RGBDSのインストール

RGBDSをソースコードからビルドします。

```bash
mkdir -p ~/tools
cd ~/tools

git clone https://github.com/gbdev/rgbds.git
cd rgbds

git checkout v1.0.1

make -j$(nproc)
sudo make install
```

`master`ブランチは開発中の状態になるため、この記事では安定版のタグを指定しています。別のバージョンを使いたい場合は、RGBDSのリリースページでタグ名を確認してから `git checkout` してください。

### ビルドエラー発生

最初のビルドでは、`bison` を入れていなかったため以下のエラーが発生しました。

```text
src/bison.sh: 7: bison: not found
src/bison.sh: 8: bison: not found
make: *** [Makefile:154: src/asm/parser.cpp] Error 127
```

この場合は不足しているパッケージをインストールします。

```bash
sudo apt update
sudo apt install -y bison
```

インストール後に再度ビルドします。

```bash
cd ~/tools/rgbds

make clean
make -j$(nproc)
sudo make install
```

## インストール確認

```bash
rgbasm -V
rgblink -V
rgbfix -V
rgbgfx -V
```

## 開発用ディレクトリの作成

```bash
mkdir -p ~/gbdev/hello
cd ~/gbdev/hello
```

## Visual Studio Codeとの連携

```bash
code .
```

しかし私の環境では以下のエラーが発生しました。

```text
Exec format error
```

### VS Code Serverを再構築

```bash
rm -rf ~/.vscode-server
```

PowerShellで以下を実行します。

```powershell
wsl --shutdown
```

その後、再度起動します。

```bash
cd ~/gbdev/hello
code .
```

## フォルダをエクスプローラーから開く

WSL2上のファイルは、Windowsのエクスプローラーからも参照できます。
作成したプロジェクトフォルダをWindows側のツールで確認したい場合は、アドレスバーに以下のようなパスを入力します。

```text
\\wsl$\Ubuntu\home\<ユーザー名>\gbdev\hello
```

## まとめ

WSL2を利用することで、Windows上でも快適にゲームボーイ開発環境を構築できます。

今回特に苦労したのは以下の2点でした。

- RGBDSビルド時の依存パッケージ不足
- VS CodeのWSL連携トラブル

次回はRGBDSで最初のゲームボーイROMを作成し、エミュレータ上で動作させるところまで進めます。
