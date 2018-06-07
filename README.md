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
* CentOS7+ (vagrantで検証)
* Docker 18+
* git

## 手順概要
* テンプレートのダウンロード
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
|-- docker-compose.insecure.yml      # SSL無し用のdocker-composeファイル（今回未使用）
|-- docker-compose.yml               # SSL通信用のdocker-composeファイル
|-- .env                             # docker-compose用環境設定ファイル
|-- nginx
|   `-- nginx.conf                   # nginx設定ファイル
|-- README.md
|-- registry
|   |-- config.yml                   # docker registry設定ファイル
|   `-- init                         # docker registry起動時の初期化ファイル（証明書の登録を行う）
`-- secrets                          # 証明書格納ディレクトリ
    `-- .gitignore
```

### 設定ファイルを編集
##### docker-composeの設定
* .env

  docker-compose.ymlで使用される環境変数 MACHINE_FQDN の値を設定します。
  docker-compose.yml内の記述${MACHINE_FQDN}は各種設定ファイルの値よりこの値が優先的に設定されます。
  当リポジトリではサーバのIP 192.168.33.13 を MACHINE_FQDN として設定しています。
  ```
  MACHINE_FQDN=192.168.33.13
  ```

##### nginxの設定
* docker-compose.ymlの service:nginx セクション

  nginxによるリバースプロキシのポートをホストで公開するためのポート設定があります。
  ポートを変更する場合はこの値を変更してください。
  例）ホスト側で公開するHTTPポートを変更する場合、8080:80 とします。
  ```
  nginx:
    image: library/nginx:alpine
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./secrets:/secrets:ro
      - static:/srv/Portus/public:ro
    ports:
      - 80:80    # ここを変更　ホストのポート:コンテナのポート　の書式になっている
      - 443:443  # ここを変更　ホストのポート:コンテナのポート　の書式になっている
    links:
      - registry:registry
      - portus:portus
  ```

* nginx.conf

  http ＞ server の server_name に .envに指定した MACHINE_FQDN の値を設定します。
  ```
  http {
    ･･･
    ･･･
    server {
      listen 443 ssl http2;
      server_name 172.17.0.1;　# ここをMACHINE_FQDNの設定値に変更
      root /srv/Portus/public;
    ･･･
    ･･･
  }
  ```

##### Portusの設定
* docker-compose.ymlの service:portus セクション

  Portusのバージョンを2.3に変更します。headはmasterブランチを指しているため、更新されやすく動作しなくなる可能性があります。  
  バージョンを指定して安定した動作を確保します。
  ```
  services:
    portus:
      image: opensuse/portus:head # ここのheadを2.3に変更
      environment:
        - PORTUS_MACHINE_FQDN_VALUE=${MACHINE_FQDN}
  ･･･
  ･･･
  ```

  Portusの設定ファイルであるconfig.ymlをマウントさせます。これで、config.ymlによるPortusの設定が可能になります。  
  config.ymlは当リポジトリ上では、実行コンテキストのルートディレクトリにあります。  
  公式のGithubリポジトリ上では、/Portus/config/config.yml にあるので、コピーして配置するといいでしょう。
  ```
  services:
    portus:
  ･･･
  ･･･    
      volumes:
        - ./secrets:/certificates:ro
        - static:/srv/Portus/public
        - ./config.yml:/srv/Portus/config/config.yml # この行を追加する。ホスト：コンテナの形式で指定
  ･･･
  ･･･
  ```

* Portusのconfig.yml

  LDAP設定等、Portusの各種設定が可能。上記手順docker-compose.ymlのportusセクションでマウントの設定をすることで利用できます。  
  machine_fqdn valueに.envに指定した MACHINE_FQDN の値を設定します。  
  この設定は、docker-compose.ymlのPORTUS_MACHINE_FQDN_VALUE設定より優先度が低いため、必須設定ではありませんが、可読性のため設定を推奨します。
  ```
  ･･･
  ･･･
  # The FQDN of the machine where Portus is being deployed.
  machine_fqdn:
    value: "portus.test.lan" # ここをMACHINE_FQDNの設定値に変更
  ･･･
  ･･･
  ```

##### registryの設定
* docker-compose.ymlの service:registry セクション

  docker-registryのデータ格納ボリュームを設定可能。初期設定の場合、ホストの /var/lib/portus/registry がマウントされます。  
  この設定のままだと、コンテナを再作成した場合や別の実行コンテキストで作成した場合に、データが引継がれます。  
  データの引継ぎをしない場合は実行コンテキスト内にディレクトリを作成し、そのディレクトリをマウントするといいでしょう。
  ```
  services:
  ･･･
  ･･･
    registry:
    ･･･
    ･･･
    volumes:
      - /var/lib/portus/registry:/var/lib/registry # 通常はそのままでOK。実行コンテキストごとに切り替えたい場合、./registry_volume:/var/lib/registry などに変更する。
    ･･･
    ･･･
  ```

* registry/config.yml

  docker-registryの詳細設定。変更は不要。

* registry/init

  docker-registry起動時の設定shスクリプト。証明書の読み込みを行います。変更は不要です。  
  docker-compose.ymlの中でCOMMANDエントリで指定されています。

##### backgroundの設定
* docker-compose.ymlの service:background セクション

  backgroundはdocker-registryとPortusを連携するためのコンテナです。イメージはPortusをそのまま使用します。  
  Portusのバージョンを2.3に変更します。headはmasterブランチを指しているため、更新されやすく動作しなくなる可能性があります。  
  バージョンを指定して安定した動作を確保します。
  ```
  services:
  ･･･
  ･･･
    background:
      image: opensuse/portus:head # ここのheadを2.3に変更
  ･･･
  ･･･
  ```

##### dbの設定
* docker-compose.ymlの service:db セクション

  db(MariaDB)のデータ格納ボリュームを設定可能。初期設定の場合、ホストの /var/lib/portus/mariadb がマウントされます。
  これにより、コンテナが停止してもデータの永続化が可能になっています。変更は不要です。

### 自己証明書の作成、および、インストール
Portusを構築には、SSL証明書が必須となります。正規のSSL証明書がある場合はそれをそのまま利用できますが  
オンプレで構築する際には、正規の証明書が無い場合が多いため、自己証明書を作成してそれを利用します。

##### 自己証明書の作成
* SAN(Subject Alternatively Name)に対応した自己証明書作成のための設定ファイル変更

  CentOS7で、自己証明書を作成する場合、OpenSSLを使用します。  
  標準の設定ファイル /etc/pki/tls/openssl.cnf をコピーして実行コンテキストのsecretsディレクトリにコピーします。  
  コピーしたファイルは「san_openssl.cnf」等、区別できるように名前を変更して下さい。

  コピーした設定ファイル「san_openssl.cnf」の内容を変更します。  
  [ req ] カテゴリ内のreq_extensionsのコメントをはずします
  ```
  ■変更前
  [ req ]
  ･･･
  # req_extensions = v3_req # The extensions to add to a certificate request
  ･･･

  ■変更後
  [ req ]
  ･･･
  req_extensions = v3_req # The extensions to add to a certificate request  # コメントを外します。
  ･･･
  ```

  [ usr_cert ] カテゴリ内のauthorityKeyIdentifierに :always を追記します。
  ```
  ■変更前
  [ usr_cert ]
  ･･･
  authorityKeyIdentifier=keyid,issuer
  ･･･

  ■変更後
  [ usr_cert ]
  ･･･
  authorityKeyIdentifier=keyid,issuer:always # :alwaysを追記します。
  ･･･
  ```

  [ usr_cert ] カテゴリ内のsubjectAltNameのコメントをはずして、設定値を @alt_names に変更します。
  ```
  ■変更前
  [ usr_cert ]
  ･･･
  # subjectAltName=email:copy
  ･･･

  ■変更後
  [ usr_cert ]
  ･･･
  subjectAltName=@alt_names # コメントを外して、設定値を@alt_namesにします
  ･･･
  ```

  [ v3_req ] カテゴリ内にsubjectAltNameを追加し、設定値を @alt_names にします。
  ```
  [ v3_req ]

  # Extensions to add to a certificate request

  basicConstraints = CA:FALSE
  keyUsage = nonRepudiation, digitalSignature, keyEncipherment
  subjectAltName = @alt_names # この行を追加します
  ```

  [ v3_ca ] カテゴリの前に　[ alt_names ]カテゴリを追加して、
  [ alt_names ]カテゴリ内に DNS.1 を追加し、ドメイン名、もしくは IP.1 を追加し、IPアドレスを記載します。
  今回は MACHINE_FQDN をIPで設定しているため、IP.1 を記載します。
  ```
  [ alt_names ]                 # この2行を追加します。
  IP.1 = 192.168.33.13　　　　　 # 今回はIPで設定します。　　

  [ v3_ca ]
  ･･･
  ･･･
  ```

  [ v3_ca ] カテゴリ内のsubjectAltNameのコメントをはずして、設定値を @alt_names に変更します。
  ```
  ■変更前
  [ v3_ca ]
  ･･･
  # subjectAltName=email:copy
  ･･･

  ■変更後
  [ v3_ca ]
  ･･･
  subjectAltName=@alt_names # コメントを外して、設定値を@alt_namesにします
  ･･･
  ```

* 自己証明書と秘密鍵の作成

  実行コンテキスト内のsecretsディレクトリに移動し、以下のコマンドを実行します。  
  コマンドには上記で作成した設定ファイルを使用するようにパラメータを指定してあります。
  ```
  $ openssl req -newkey rsa:4096 -nodes -sha256 -keyout ./portus.key -x509 -days 365 -out ./portus.crt -config san_openssl.cnf
  ```

  コマンド実行後の入力値はCommon Name以外は空で問題ありません。  
  Common Nameのみ MACHINE_FQDN に設定した値を入力して下さい。
  ```
  Generating a 4096 bit RSA private key
  ..................................++
  .................................................++
  writing new private key to './portus.key'
  -----
  You are about to be asked to enter information that will be incorporated
  into your certificate request.
  What you are about to enter is what is called a Distinguished Name or a DN.
  There are quite a few fields but you can leave some blank
  For some fields there will be a default value,
  If you enter '.', the field will be left blank.
  -----
  Country Name (2 letter code) [XX]:
  State or Province Name (full name) []:
  Locality Name (eg, city) [Default City]:
  Organization Name (eg, company) [Default Company Ltd]:
  Organizational Unit Name (eg, section) []:
  Common Name (eg, your name or your server's hostname) []:192.168.33.13  # Common Name には MACHINE_FQDN を設定して下さい。
  Email Address []:
  ```

  成功すると、証明書ファイル portus.crt と 秘密鍵 portus.key が作成されます。  
  SANに対応した証明書が作成できているか検証するには以下のコマンドを実行し
  ```
  $ openssl x509 -text -noout -in portus.crt
  ```

  Subjectに、CN={MACHINE_FQDN} となっている  
  X509v3 Subject Alternative Name に IP Address:{MACHINE_FQDN} となっていることを確認します。
  ```
  ･･･
        Subject: C=XX, L=Default City, O=Default Company Ltd, CN=192.168.33.13
  ･･･
            X509v3 Subject Alternative Name:
                IP Address:192.168.33.13
  ･･･      
  ```

##### 自己証明書のインストール
* Portusサーバへのインストール

  Portusサーバへの自己証明書インストールは実行コンテキストの secrets ディレクトリにportus.crt、portus.keyが格納されていれば  
特に作業はありません。

* クライアントへのインストール

  Portusにイメージをpushするクライアントに作成した自己証明書をインストールする必要があります。  
  クライアントマシンにログインし、rootで「/etc/docker/certs.d/{MACHINE_FQDN}」というディレクトリを作成します。
  上記で作成した自己証明書 portus.crt を作成したディレクトリにコピーし、ca.crt という名前に変更してください。
  その後、dockerのプロセスを再起動し、作業完了です。
  ```
  以下作業例

  $ su -
  $ mkdir /etc/docker/certs.d/192.168.33.13/
  $ cp ./portus.crt /etc/docker/certs.d/192.168.33.13/ca.crt
  $ systemctl restart docker
  ```  

### Portusの起動
実行コンテキストで、以下のコマンドを実施し、Portusを起動します。
```
$ docker-compose up -d
```
起動後、https://{MACHINE_FQDN} にアクセスすると初回アクセス画面が表示されます。
