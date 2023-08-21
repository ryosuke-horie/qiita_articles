---
title: EC2を利用してNest.jsで作成したプロジェクトをデプロイする
tags:
  - Node.js
  - AWS
  - EC2
  - 個人開発
  - Nest.js
private: false
updated_at: '2023-08-13T00:24:43+09:00'
id: c7c692a1dbccc8ecf803
organization_url_name: null
slide: false
---
## 目次
1. この記事のゴール
2. 開発環境
3. 前提条件
4. docker-compose.ymlの作成
5. AWSでEC2インスタンスを立てる手順
6. EC2インスタンスにEC2 Instance ConnectでSSH接続して設定

## この記事のゴール
Nest.jsで開発したプロジェクトをAWS無料枠が適用できるEC2にデプロイし、アクセスできることがゴールです。
DBはPostgresをdocker-composeを活用して作成します。
セキュリティについての設定は省いた実装を行います。

## 開発環境
Nest.js : v9.3.0
Node.js : v16.20.2
*EC2のAmazon Linux2でNode.js v16を利用するように指示があるため従っています。

https://docs.aws.amazon.com/ja_jp/sdk-for-javascript/v2/developer-guide/setting-up-node-on-ec2-instance.html

## 前提条件
デプロイができるまでの前提条件です。
- ローカルの開発環境で動作することが確認できていること。
- TypeORMを活用しPostgresデータベースに接続できていること。
- AWSのアカウントを作成していること。
- GitHubでプロジェクトを管理していること。

## docker-compose.ymlの作成
```yaml:docker-compose.yml
version: '3.9'

services:
  postgres:
    image: postgres:12-alpine
    container_name: postgres
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=postgres
    volumes:
      - postgres:/var/lib/postgresql/data
    ports:
      - 5432:5432

  # pgAdminは動作確認用。本来はセキュリティ上ここに書かないほうがいいと思います。
  pgadmin:
    image: dpage/pgadmin4
    restart: always
    ports:
      - 81:80
    environment:
      PGADMIN_DEFAULT_EMAIL: test@example.com
      PGADMIN_DEFAULT_PASSWORD: password
    volumes:
      - ./docker/pgadmin:/var/lib/pgadmin
    depends_on:
      - postgres
    user: root

volumes:
  postgres:

```



## AWSでEC2インスタンスを立てる手順

### VPC関連
1. VPCを作成する。
「VPC」コンソールのメニューバーから「お使いのVPC」を選択し、「VPCを作成」のオレンジ色のボタンをクリック。
適当な名前を付けて、IPv4 CIDRに 192.168.0.0/16 を設定

2. サブネットを作成。
左のメニューバーからサブネットを選択し作成
パブリックサブネット用に192.168.10.0/24
を作成する。

3. インターネットゲートウェイの作成
左のメニューバーから「インターネットゲートウェイ」を選択して作成する。適当な名前を付けるのみ。

4. インターネットゲートウェイをVPCにアタッチ
「アクション」ボタンから「VPCにアタッチ」を選択し１でつくったVPCを関連付ける

5. ルートテーブルの作成
左のメニューバーから「ルートテーブル」を選択。
新しく作成する。このとき、VPCは１で作ったものを使用する。

6. ルートテーブルにサブネットを関連付ける
チェックをつけて下のメニューの「サブネットの関連付け」をクリック
パブリックサブネットとして作成したものを設定する。

7. ルートの設定
下のメニューの「ルート」を選択してルートを編集します。
送信先0.0.0.0./0でターゲットを３でつくったインターネットゲートウェイに設定して保存します。

### EC2インスタンスの作成
  1. 「EC2」から「インスタンスを起動」ボタンをクリック
  2. 名前は適当に設定する
  3. アプリケーションおよびOSイメージは「Amazon Linux」を選択
  4. AMIの設定はAmazon Linux２のものを選択する。
  5. インスタンスタイプは「t2.micro」を選択
  6. キーペアの部分では「新しいキーペアの作成」をクリックしキーペア名を入力して作成
  7. ネットワーク設定の「編集」をクリック
  8. VPCは前の手順で作成したものを選択
  9. サブネットも前の手順で作成した、パブリックサブネットのものを選択
  10. パブリック IP の自動割り当ては有効化を選択(インターネットに接続するため)
  11. セキュリティグループを作成する
  12. 「セキュリティグループルールを追加」ボタンをクリック
  13. タイプが「HTTP」でソースに「0.0.0.0/0」を記載する
  14. タイプが「カスタムTCP」でポートを「3000」、ソースには「0.0.0.0/0」
  15. （もしpgAdminをDockerで作成しポート81を利用するように設定しているなら）タイプが「カスタムTCP」でポートを「81」、ソースには「0.0.0.0/0」
  16. タイプが「SSH」でソースに「0.0.0.0/0」を設定
  17. インスタンスを起動をクリック
  18. EC2ダッシュボードの左側のメニューから「Elastic IP」を選択
  19. 「Elastic IP アドレスを割り当てる」をクリックしてElastic IPを取得する。
  20. 「アクション」をクリックし、「Elastic IPアドレスの関連付け」を行う

## EC2インスタンスにEC2 Instance ConnectでSSH接続して設定

https://docs.aws.amazon.com/ja_jp/sdk-for-javascript/v2/developer-guide/setting-up-node-on-ec2-instance.html

1. 上記リンクを参考にNode.jsをインストール
    ```bash
    sudo yum update

    # nvmのインストール
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
    # nvmの有効化
    . ~/.nvm/nvm.sh
    # Amazon linux2を利用しているためNode.js v16を選択
    nvm install 16
    # nvmでNode.jsを有効化
    nvm use 16
    # .bash_profileで毎回有効化させる
    vi ~/.bash_profile
    # 最終行にnvm use 16を記載する
    ```

2. gitのインストールとGit Clone
    ```bash
    # gitのインストール
    sudo yum install git
    # sshキーを生成
    cd ~/.ssh
    ssh-keygen
    # 公開鍵を表示しコピーしてGithubで設定する（省略）
    sudo cat id_rsa.pub

    # デプロイ先のディレクトリへ移動
    cd /var/www/
    # git clone
    git clone [Githubリポジトリに表示されるリポジトリパス]
    ```

3. Dockerのセットアップ

参考：

https://kacfg.com/aws-ec2-docker/

    ```
    # dockerのインストール
    sudo yum -y install docker
    # docker起動
    sudo service docker start
    # 確認
        sudo docker info
        # sudoなしでdockerを利用できるように
        sudo usermod -a -G docker ec2-user
    
        # 一度EC2 Instance Connectから抜けて入りなおす
        # sudoなしでDockerを利用できるか確認
        docker info
        # docker起動
        sudo systemctl start docker
        # 動作確認
        systemctl status docker
        # dockerの自動起動を有効化
        sudo systemctl enable docker
    
        # Docker Composeのバイナリファイルを格納するディレクトリを作成
        sudo mkdir -p /usr/local/lib/docker/cli-plugins
        # 変数VERにDocker Composeのバージョンを代入
        VER=2.4.1
        # バイナリファイルをダウンロード
        # 3行まとめて実行 
        sudo curl \
          -L https://github.com/docker/compose/releases/download/v${VER}/docker-compose-$(uname -s)-$(uname -m) \
          -o /usr/local/lib/docker/cli-plugins/docker-compose
        # バイナリファイルに実行権限を付与
        sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
        # /usr/bin/に/usr/local/lib/docker/cli-plugins/docker-composeへのシンボリックリンクを設定
        sudo ln -s /usr/local/lib/docker/cli-plugins/docker-compose /usr/bin/docker-compose
    ```

4. Nest.jsプロジェクトのセットアップ

    ```bash
    # git cloneで作成したプロジェクトのルートでdocker-composeを利用
    docker compose up -d

    # （任意）環境変数を利用しているなら.envを作成して記述
    vi .env
    # ライブラリのインストール
    npm i 
    # TypeORMでmigrationを実行(pakage.jsonでスクリプト等を用意していればそちらを利用してください。)
    npm run typeorm migration:run -- -d src/database/data-source.ts
    # プロジェクトを開始
    npm run start
    ```

ここまで実行すれば、「http://(パブリックIPアドレス):3000」 でプロジェクトをデプロイできます。
また、Dockerで作成したpgAdminへのアクセスは「「http://(パブリックIPアドレス):81」 でアクセスできます。

## まとめ
今回はAWSの無料枠を意識した構成でデプロイしてみました。
個人開発で利用するサービスのバックエンドをとりあえず動かしたい場合に利用できるのではないでしょうか。
セキュリティの面には全く気をつかっていないため実際に利用する際には手を加える必要があります。ご注意ください。
