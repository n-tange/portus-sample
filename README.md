# PortusでプライベートDockerレジストリを構築する手順

PortusとはDockerレジストリに認証機構とWebUIを付与するRailsアプリケーションです。  
オンプレミスでDockerレジストリを運用する際に、有効なソリューションとなります。
当リポジトリはこのPortusを構築するテンプレートと手順を提供します。

## Portus
* 公式 : http://port.us.org/
* Github : https://github.com/SUSE/Portus

## 参考サイト
* http://blog.geeko.jp/syuta-hashimoto/1787

## 概要
* この手順で構築するPortusのバージョンは 2.3 です。
* 公式のGithubから、テンプレートをダウンロードし設定を変更して構築します。

## 動作環境
* CentOS7+
* Docker 18+
* git

## 手順概要
* テンプレートのダウンロード
* Firewallの設定
* 設定ファイルを編集（マシンFQDNを設定）
* 自己証明書の作成、および、インストール
* Potusの起動

## 構築手順
#### テンプレートのダウンロード
当リポジトリにすでにテンプレートがありますので、それをそのまま使用すれば、Portus v2.3の構築が可能です。  
別途テンプレートをダウンロードする場合は、公式のGithubからPortusのリポジトリをクローンします。  
クローンする際に、オプションを指定してバージョン2.3のみを取得します。
* --depth 1 で最新版のみを取得し、データ量を減らす
* --b v2.3 でv2.3ブランチを指定

```bash
$ git clone --depth 1 -b v2.3 https://github.com/SUSE/Portus
```

クローンしたリポジトリPortus配下のディレクトリ[examples/compose]がテンプレートのディレクトリになります。  
このcomposeディレクトリをコピーし、別途配置、命名してください。composeディレクトリが実行コンテキストになります。

```
.
|-- config.yml #Portusの設定ファイル
|-- docker-compose.insecure.yml
|-- docker-compose.yml
|-- .env
|-- nginx
|   `-- nginx.conf
|-- README.md
|-- registry
|   |-- config.yml
|   `-- init
`-- secrets
    |-- portus.crt
    |-- portus.key
    `-- san_openssl.cnf
```
