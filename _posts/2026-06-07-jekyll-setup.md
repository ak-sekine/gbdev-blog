---
layout: post
title: "WSL2でJekyll環境を構築しGitHub Pagesでブログを公開する"
date: 2026-06-07
categories: [blog]
tags: 
    - wsl2
    - jekyll
    - github-pages
    - setup
---

# WSL2でJekyll環境を構築しGitHub Pagesでブログを公開する

## はじめに

ゲームボーイ開発の記録を残すために、GitHub PagesとJekyllを使用したブログ環境を構築しました。

この記事では、Windows 11 + WSL2 (Ubuntu) 環境でJekyllを導入し、GitHub Pagesで公開するまでの手順をまとめます。

## 動作環境

- Windows 11
- WSL2
- Ubuntu
- Visual Studio Code
- GitHubアカウント

## 全体の流れ

1. WSL2をインストール
2. Rubyをインストール
3. Jekyllをインストール
4. ブログを作成
5. ローカルで確認
6. GitHubリポジトリ作成
7. GitHubへアップロード
8. GitHub Pagesを有効化
9. 公開確認

## WSL2の準備

WSL2が未導入の場合は管理者権限のPowerShellで以下を実行します。

```powershell
wsl --install
```

再起動後、Ubuntuを起動します。

Ubuntuが起動したらパッケージを最新化します。

```bash
sudo apt update
sudo apt upgrade
```

## Rubyのインストール

JekyllはRubyで作られているため、まずRubyをインストールします。

```bash
sudo apt install ruby-full build-essential zlib1g-dev
```

インストール後にバージョンを確認します。

```bash
ruby -v
```

表示例

```text
ruby 3.x.x
```

## Gemの設定

Ruby Gemsをユーザー領域へインストールするよう設定します。

```bash
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## Jekyllのインストール

JekyllとBundlerをインストールします。

```bash
gem install jekyll bundler
```

バージョンを確認します。

```bash
jekyll -v
bundle -v
```

## ブログ用ディレクトリを作成

任意の場所にブログ用ディレクトリを作成します。

```bash
mkdir ~/gbdev-blog
cd ~/gbdev-blog
```

## Jekyllサイトを作成

現在のディレクトリにJekyllサイトを作成します。

```bash
jekyll new . --force
```

以下のような構成になります。

```text
gbdev-blog/
├── _config.yml
├── _posts
├── index.md
├── Gemfile
└── ...
```

## ローカルで起動する

必要なライブラリをインストールします。

```bash
bundle install
```

ローカルサーバーを起動します。

```bash
bundle exec jekyll serve
```

ブラウザで以下を開きます。

http://localhost:4000

デフォルトのブログが表示されれば成功です。

## サイト設定を変更する

`_config.yml` を編集します。

```yaml
title: ゲームボーイ開発メモ
description: RGBDSを使ったゲームボーイ開発記録
baseurl: "/gbdev-blog"
url: "https://<GitHubのユーザーID>.github.io"
lang: ja
timezone: Asia/Tokyo

theme: minima
```

設定変更後はサーバーを再起動します。再起動後はbaseurlを付けたURLでアクセスする必要があるので注意してください。

http://localhost:4000/gbdev-blog

## 最初の記事を書く

`_posts` フォルダに記事を作成します。

```text
_posts/2026-06-07-first-post.md
```

内容例

```markdown
---
layout: post
title: "ブログ開始"
date: 2026-06-07
---

ゲームボーイ開発ブログを始めました。

RGBDSやドット絵制作について記録していきます。
```

## VS Codeで編集する

WSL2上で以下を実行します。

```bash
code .
```

Visual Studio CodeのRemote Development拡張機能を利用すると、WSL上のファイルをWindows側から快適に編集できます。

## GitHubリポジトリを作成する

GitHubで新しいリポジトリを作成します。

例

```text
gbdev-blog
```

公開設定は Public にします。

## Gitで管理する

初回のみ実行します。

```bash
git init
```

.gitignoreは以下のように変更します。

```gitignore
# Jekyll build output
_site/

# Jekyll cache
.jekyll-cache/
.jekyll-metadata

# Sass cache
.sass-cache/

# Bundler
.bundle/
vendor/bundle/

# VS Code
.vscode/

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
```

commitします。

```bash
git add .
git commit -m "Initial commit"
```

## GitHubへPushする

GitHubで表示されたURLを使用します。

```bash
git branch -M main
git remote add origin https://github.com/ユーザー名/gbdev-blog.git
git push -u origin main
```

## GitHub Pagesを有効化する

GitHubのリポジトリ画面で以下を開きます。

Settings → Pages

設定を以下にします。

Source: Deploy from a branch
Branch: main
Folder: /(root)

保存します。

## 公開を確認する

数分待つと以下のURLで公開されます。

https://ユーザー名.github.io/gbdev-blog/

## 記事を追加する

新しい記事を作成します。

例

_posts/2026-06-08-rgbds-install.md

更新後は以下を実行します。

```bash
git add .
git commit -m "Add new article"
git push
```

GitHub Pagesが自動で更新されます。

## おわりに

WSL2とJekyllを組み合わせることで、Linux環境そのままで快適にブログを運用できます。

GitHub Pagesを利用すれば無料で公開できるため、ゲームボーイ開発の記録や技術メモを残す用途におすすめです。
