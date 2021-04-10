---
title: GitHub ActionsのためのDockerfileサポート
shortTitle: Docker
intro:
  "Dockerコンテナアクション用の`Dockerfile`を作成する際には、いくつかのDockerの命令がGitHub
  Actionsやアクションのメタデータファイルとどのように関わるのかを知っておく必要があります。"
product: "{% data reusables.gated-features.actions %}"
redirect_from:
  - /actions/building-actions/dockerfile-support-for-github-actions
versions:
  free-pro-team: "*"
  enterprise-server: ">=2.22"
---

{% data variables.product.prodname_actions %} の支払いを管理する
{% data variables.product.prodname_dotcom %}は、macOS ランナーのホスト
に[MacStadium](https://www.macstadium.com/)を使用しています。

### Dockerfile の命令について

`Dockerfile`には、Docker コンテナの内容と起動時の動作を定義する命令と引数が含ま
れています。 Docker がサポートしている命令に関する詳しい情報については、Docker
のドキュメンテーション中の
「[Dockerfile のリファレンス](https://docs.docker.com/engine/reference/builder/)」
を参照してください。

### Dockerfile の命令とオーバーライド

Docker の命令の中には GitHub Action と関わるものがあり、アクションのメタデータフ
ァイルは Docker の命令のいくつかをオーバーライドできます。 予期しない動作を避け
るために、Dockerfile が{% data variables.product.prodname_actions %}とどのように
関わるかについて馴染んでおいてください。

#### USER

Docker アクションはデフォルトの Docker ユーザ（root）で実行されなければなりませ
ん。 `GITHUB_WORKSPACE`にアクセスできなくなってしまうので、`Dockerfile`中で
は`USER`命令を使わないでください。 詳しい情報については、
「[環境変数の利用](/actions/configuring-and-managing-workflows/using-environment-variables)」
と、Docker のドキュメンテーション中
の[USER のリファレンス](https://docs.docker.com/engine/reference/builder/#user)を
参照してください。

#### FROM

`Dockerfile`ファイル中の最初の命令は`FROM`でなければなりません。これは、Docker
のベースイメージを選択します。 詳しい情報については、Docker のドキュメンテーショ
ン中
の[FROM のリファレンス](https://docs.docker.com/engine/reference/builder/#from)を
参照してください。

`FROM`引数の設定にはいくつかのベストプラクティスがあります。

- 公式の Docker イメージを使うことをおすすめします。 たとえば`python`や`ruby`で
  す。
- バージョンタグが存在する場合は使ってください。メジャーバージョンも含めることが
  望ましいです。 たとえば`node:latest`よりも`node:10`を使ってください。
- [Debian](https://www.debian.org/)オペレーティングシステムに基づく Docker イメ
  ージを使うことをおすすめします。

#### WORKDIR

{% data variables.product.product_name %}は、ワーキングディレクトリのパスを環境
変数の`GITHUB_WORKSPACE`に設定します。 `Dockerfile`中では`WORKDIR`命令を使わない
ことをおすすめします。 アクションが実行される前に
、{% data variables.product.product_name %}は`GITHUB_WORKSPACE`ディレクトリを
、Docker イメージ内にあったその場所になにがあってもその上にマウントし
、`GITHUB_WORKSPACE`をワーキングディレクトリとして設定します。 詳しい情報につい
ては
「[環境変数の利用](/actions/configuring-and-managing-workflows/using-environment-variables)」
と、Docker のドキュメンテーション中
の[WORKDIR のリファレンス](https://docs.docker.com/engine/reference/builder/#workdir)を
参照してください。

#### ENTRYPOINT

アクションのメタデータファイル中で`entrypoint`を定義すると、それは`Dockerfile`中
で定義された`ENTRYPOINT`をオーバーライドします。 詳しい情報については
「[{% data variables.product.prodname_actions %}のメタデータ構文](/actions/creating-actions/metadata-syntax-for-github-actions/#runsentrypoint)」
を参照してください。

Docker の`ENTRYPOINT`命令には、*shell*形式と*exec*形式があります。 Docker
の`ENTRYPOINT`のドキュメンテーションは、`ENTRYPOINT`の*exec*形式を使うことを勧め
ています。 *exec*および*shell*形式に関する詳しい情報については、Docker のドキュ
メンテーション中
の[ENTRYPOINT のリファレンス](https://docs.docker.com/engine/reference/builder/#entrypoint)を
参照してください。

*exec*形式の`ENTRYPOINT`命令を使うようにコンテナを設定した場合、アクションのメタ
データファイル中に設定された`args`はコマンドシェル内では実行されません。 アクシ
ョンの`args`に環境変数が含まれている場合、その変数は置換されません。 たとえば、
以下の*exec*形式は`$GITHUB_SHA`に保存された値を出力せず、代わり
に`"$GITHUB_SHA"`を出力します。

```
ENTRYPOINT ["echo $GITHUB_SHA"]
```

変数の置換をさせたい場合は、*shell*形式を使うか、直接シェルを実行してください。
たとえば、以下の*exec*形式を使えば、シェルを実行して環境変数`GITHUB_SHA`に保存さ
れた値を出力できます。

```
ENTRYPOINT ["sh", "-c", "echo $GITHUB_SHA"]
```

アクションのメタデータファイルに定義された`args`を、`ENTRYPOINT`中で*exec*形式を
使う Docker コンテナに渡すには、`ENTRYPOINT`命令から呼ぶ`entrypoint.sh`というシ
ェルスクリプトを作成することをおすすめします。

##### *Dockerfile*の例

```
# コードを実行するコンテナイメージ
FROM debian:stretch-20210329-slim

# アクションリポジトリからコンテナのファイルシステムパスの`/`にコードファイルをコピー
COPY entrypoint.sh /entrypoint.sh

# Dockerコンテナの起動時に`entrypoint.sh`を実行
ENTRYPOINT ["/entrypoint.sh"]
```

##### *entrypoint.sh*ファイルの例

上の Dockerfile を使って、{% data variables.product.product_name %}はアクション
のメタデータファイルに設定された`args`を、`entrypoint.sh`の引数として送ります。
`#!/bin/sh`[シバン](<https://ja.wikipedia.org/wiki/シバン_(Unix)>)を`entrypoint.sh`フ
ァイルの先頭に追加し、システムの[POSIX](https://ja.wikipedia.org/wiki/POSIX)準拠
のシェルを明示的に使ってください。

```sh
#!/bin/sh

# `$*`は`array`内で渡された`args`を個別に展開するか、
# を空白で区切られた文字列中の`args`を分割します。
sh -c "echo $*"
```

コードは実行可能になっていなければなりません。 `entrypoint.sh`ファイルをワークフ
ロー中で使う前に、`execute`権限が付けられていることを確認してください。 この権限
は、ターミナルから以下のコマンドで変更できます。

```sh
chmod +x entrypoint.sh
```

`ENTRYPOINT`シェルスクリプトが実行可能ではなかった場合、以下のようなエラーが返さ
れます。

```sh
Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "exec: \"/entrypoint.sh\": permission denied": unknown
```

#### CMD

アクションのメタデータファイル中で`args`を定義すると、`args`は`Dockerfile`中で指
定された`CMD`命令をオーバーライドします。 詳しい情報については
「[{% data variables.product.prodname_actions %}のメタデータ構文](/actions/creating-actions/metadata-syntax-for-github-actions#runsargs)」
を参照してください。

`Dockerfile`中で`CMD`を使っているなら、以下のガイドラインに従ってください。

{% data reusables.github-actions.dockerfile-guidelines %}

### サポートされている Linux の機能

{% data variables.product.prodname_actions %}は、Docker がサポートするデフォルト
の Linux の機能をサポートします。 機能の追加や削除はできません。 Docker がサポー
トするデフォルトの Linux の機能に関する詳しい情報については、Docker のドキュメン
テーション中の
「[ Runtime privilege and Linux capabilities](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)」
を参照してください。 Linux の機能についてさらに学ぶには、Linux の man-page の
"[ Overview of Linux capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html)"
を参照してください。
