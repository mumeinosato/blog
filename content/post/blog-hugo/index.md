---
title: ブログをHugoに移行する
date: 2026-03-05
tags: 
    - Hugo
    - Cloudflare Workers
categories:
    - その他
---
## はじめに
今までWordPressでブログを運営してきましたが、Hugoに移行し実行環境も自宅サーバーからCloudflare Workersに移行することにしました。
理由としては、WordPressは機能が豊富で便利ですが、ほとんどの機能を使っていないことと、自宅サーバーの運用にコストがかかっていたためです。


<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">やっぱwebサーバーぐらいなら自宅サーバーよりクラウドに頼ったほうがよさそう<br>無料でいろいろできるのも増えてきたしその方がコストがかからない(労力も含めて)</p>&mdash; 無名の里 (@mumeinosato) <a href="https://twitter.com/mumeinosato/status/2028480675033321504?ref_src=twsrc%5Etfw">March 2, 2026</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## 移行作業
### Hugoの設定
#### Hugoのインストール
[公式サイトの手順](https://gohugo.io/installation/)に従ってインストールします。  
筆者はWindowsでWingetを使用してインストールしました。

```bash
winget install Hugo.Hugo.Extended
```

#### プロジェクトの初期化
以下のコマンドでHugoプロジェクトを初期化できます。
```bash
hugo new project <プロジェクト名>
```

#### テーマのインストール
今回は、[Stack](https://github.com/CaiJimmy/hugo-theme-stack/)を使用します。  
テーマを追加する方法としては、Git Submoduleを使用する方法と、テーマのファイルを直接プロジェクト内にコピーする方法があります。  
Gitで管理することを踏まえてGit Submoduleを使用する方法でインストールします。

```bash
git submodule add https://github.com/CaiJimmy/hugo-theme-stack/ themes/hugo-theme-stack
```

#### テーマの設定
[テンプレート](https://github.com/CaiJimmy/hugo-theme-stack-starter)を参考に設定をしていきます。

デフォルトの`hugo.toml`を削除し`config/_default/hugo.yaml`を作成します。

```yaml
baseurl: https://example.com/
languageCode: ja-JP
defaultContentLanguage: ja
title: "Hugo + Stack"
theme: "stack"
enableRobotsTXT: false

params:
  favicon: "/favicon.ico"

  dateFormat:
    published: "2006-01-02"
    lastUpdated: "2006-01-02 15:04"

  sidebar:
    avatar: "img/avatar.png"

  widgets:
    homepage:
      - type: search
      - type: archives
        params:
          limit: 5
      - type: categories
        params:
          limit: 10
      - type: tag-cloud
        params:
          limit: 10
    page:
      - type: toc

  comments:
    enabled: false

  opengraph:
    enabled: true

sitemap:
  changefreq: "weekly"
  priority: 0.5

permalinks:
  post: "/p/:slug/"
  page: "/:slug/"

markup:  
  goldmark:
    renderer:
      unsafe: true

  tableOfContents:  
    endLevel: 4
    ordered: true
    startLevel: 2
```

Stackではどうやら`posts`ではなく`post`を使う必要があるため、`content/posts`を`content/post`にリネームします。  
また投稿を作成する場合は`content/post/<一意に識別できる名前>/index.md`のように作成する必要があります。

`content/page`には左のサイドバーに表示するページを置く場所です。テンプレートから必要なものをコピーします。

ここで一度開発用サーバーを起動して動作を確認します。

```bash
hugo server -D
```

### Cloudflare Workersの設定
以下の2つのファイルをプロジェクトのルートに作成します。

```shell
# build.sh

#!/usr/bin/env bash

#------------------------------------------------------------------------------
# @file
# Builds a Hugo site hosted on a Cloudflare Worker.
#
# The Cloudflare Worker automatically installs Node.js dependencies.
#------------------------------------------------------------------------------

main() {

  DART_SASS_VERSION=1.97.3
  GO_VERSION=1.26.0
  HUGO_VERSION=0.155.3
  NODE_VERSION=24.13.1

  export TZ=Europe/Oslo

  # Install Dart Sass
  echo "Installing Dart Sass ${DART_SASS_VERSION}..."
  curl -sLJO "https://github.com/sass/dart-sass/releases/download/${DART_SASS_VERSION}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
  tar -C "${HOME}/.local" -xf "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
  rm "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
  export PATH="${HOME}/.local/dart-sass:${PATH}"

  # Install Go
  echo "Installing Go ${GO_VERSION}..."
  curl -sLJO "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz"
  tar -C "${HOME}/.local" -xf "go${GO_VERSION}.linux-amd64.tar.gz"
  rm "go${GO_VERSION}.linux-amd64.tar.gz"
  export PATH="${HOME}/.local/go/bin:${PATH}"

  # Install Hugo
  echo "Installing Hugo ${HUGO_VERSION}..."
  curl -sLJO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
  mkdir "${HOME}/.local/hugo"
  tar -C "${HOME}/.local/hugo" -xf "hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
  rm "hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
  export PATH="${HOME}/.local/hugo:${PATH}"

  # Install Node.js
  echo "Installing Node.js ${NODE_VERSION}..."
  curl -sLJO "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.xz"
  tar -C "${HOME}/.local" -xf "node-v${NODE_VERSION}-linux-x64.tar.xz"
  rm "node-v${NODE_VERSION}-linux-x64.tar.xz"
  export PATH="${HOME}/.local/node-v${NODE_VERSION}-linux-x64/bin:${PATH}"

  # Verify installations
  echo "Verifying installations..."
  echo Dart Sass: "$(sass --version)"
  echo Go: "$(go version)"
  echo Hugo: "$(hugo version)"
  echo Node.js: "$(node --version)"

  # Configure Git
  echo "Configuring Git..."
  git config core.quotepath false
  if [ "$(git rev-parse --is-shallow-repository)" = "true" ]; then
    git fetch --unshallow
  fi

  # Build the site
  echo "Building the site..."
  hugo build --gc --minify

}

set -euo pipefail
main "$@"
```

<p></p>

```toml
# wrangler.toml
# Configure Cloudflare Worker

name = "blog"
compatibility_date = "2026-03-02"

[build]
command = "chmod a+x build.sh && ./build.sh"

[assets]
directory = "./public"
not_found_handling = "404-page"
```

<p></p>

```json
# package.json
{
  "name": "blog",
  "version": "1.0.0",
  "description": "",
  "scripts": {
    "deploy": "wrangler deploy",
    "build": "hugo"
  },
  "dependencies": {
    "wrangler": "^4.69.0"
  }
}

```

ここまで来たらCloudflare Workersのプロジェクトを作成する際にGitHubのリポジトリを指定します。この際の注意点はビルドコマンドを指定しないことです。

## 参考
[Hugo 製の静的サイトを Cloudflare Pages から Cloudflare Workers に移行する](https://zenn.dev/chot/articles/deef1d5dba054b)  
[hosting-cloudflare-worker](https://github.com/jmooring/hosting-cloudflare-worker)