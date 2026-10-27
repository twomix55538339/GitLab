# 「社内ネットワークのセキュリティ機構（SSL可視化／インスペクション）」

## 1.全体像：社内ネットワークで起きていること

会社のPCからインターネット（GitLabやDocker Hub）へ通信するとき、多くの企業ではセキュリティ対策として中継機器（プロキシサーバー / ZscalerやBlue Coatなど）を挟みます。

```
[端末
↓ (1)
[社内プロキシサーバー]  ← ここで通信を中身ごと検査（SSLインスペクション）
↓ (2)
[インターネット (registry.gitlab.com / Docker Hub)]
このとき、以下の2つの壁にぶつかります。
```

1.プロキシ設定の壁:
Docker Engineが社内プロキシの場所（アドレス）を知らないため、そもそも社外へ通信が抜けられない。

2.企業Cert署名の壁（SSLエラー）:
プロキシサーバーがHTTPS通信を復号・検査するために、通信経路上で「会社独自のルート証明書（企業CA）」を使って証明書を差し替えて再署名する。
Docker側は「見知らぬ偽の証明書にすり替えられた（中間者攻撃の疑い）」と判断し、安全のために通信を切断してエラーを出す。

結果として、Dockerは x509: certificate signed by unknown authority などのエラーを出し、ログインやPullが一切通らなくなります。

## 2. なぜ「プロキー（プロキシ）設定」が必要か？

ブラウザは社内のプロキシ設定（PACファイルなど）を自動で読み取って通信できますが、Docker Engine（バックグラウンドで動くデーモン）は自動認識しません。
明示的に「インターネットへ出るにはこのプロキシを通せ」と設定する必要があります。

* 具体的に行うこと:
  Dockerデーモンやクライアントの設定に、社内プロキシのアドレス（[http://proxy.company.co.jp:8080](http://proxy.company.co.jp:8080) など）をHTTP_PROXY, HTTPS_PROXY, NO_PROXY として登録します。

## 3. なぜ「企業Cert（証明書）署名」が必要か？

ここが一番混乱しやすいポイントです。

* 通常のHTTPS: 端末 ⇆ GitLab の間で暗号化されるため、途中の機器は中身を盗み見できません。
* 企業のSSL可視化（インスペクション）:
  情報漏洩やマルウェア侵入を防ぐため、社内プロキシが通信を一度復号して中身を検査します。
  検査後、プロキシは「自社の認証局（企業CA）の秘密鍵」で署名し直した証明書をあなたのPCに送ります。

会社のWindows端末のOS側には、情報システム部によってあらかじめ「会社のルートCA証明書（企業Cert）」が配布・登録されているため、Chromeなどのブラウザでは警告が出ません。

しかし、Docker Engine（WSL2やLinux仮想マシン、Dockerコンテナ内）にはその企業Certが入っていません。 そのため、Dockerは「知らない機関が勝手に署名した怪しい証明書だ」と拒絶してしまうのです。

## 4. チームが検討している対策の具体策

チームが言っている対策は、要約すると以下の手順を指しています。

### ステップ1: 企業Cert（社内ルート証明書）をDockerに信頼させる

4-1.情報システム部などから、社内プロキシが使用しているルート証明書ファイル（company-ca.crt や .pem）を取得します。

4-2.これをDockerが読み込む信頼済み証明書ディレクトリに配置します。

* Linux / WSL2の場合:/etc/docker/certs.d/[registry.gitlab.com/ca.crt](https://registry.gitlab.com/ca.crt) に配置する、またはOSのルート証明書ストア（/usr/local/share/ca-certificates/）に追加して update-ca-certificates を実行。
* Docker Desktopの場合:
  設定画面の「Settings > Docker Engine」やOS側の証明書ストア（Windowsなら証明書マネージャーの「信頼されたルート証明機関」）と同期させる。

### ステップ2: Dockerのプロキシ設定を通す

Dockerの設定ファイル（~/.docker/config.json または Docker Desktopの「Settings > Resources > Proxies」）にプロキシサーバーのURLを設定します。

```
{
  "proxies": {
    "default": {
      "httpProxy": "http://proxy.example.com:8080",
      "httpsProxy": "http://proxy.example.com:8080",
      "noProxy": "localhost,127.0.0.1,.example.com"
    }
  }
}
```

まとめ

* プロキシ設定: インターネットへ出るための「トンネルの出口（社内ゲートウェイ）」をDockerに教える設定。
* 企業Cert署名: 社内ゲートウェイがパケット検査のために独自に署名し直しているため、その「社内CAの証明書」をDockerに信用させる設定。

この2つを正しく通して初めて、SSL/通信遮断エラーが解消され、前述した config.json による「パスワード無しの自動認証」が正常に完結するようになります。

# Windows環境のDocker Desktopにおいて、社内CAのルート証明書（企業Cert）を取り込み、Docker Engineに認識させる具体的な手順を解説します。

Docker Desktopは、Windows側の証明書ストアを参照する仕組みを持っていますが、バックグラウンドの仮想マシン（WSL 2）やレジストリ通信へ確実に反映させるには「Windows側でエクスポート → Docker/WSL 2の所定のディレクトリへ配置」を行うのが最も確実です。

## ステップ1: Windowsの証明書ストアから社内CA証明書をエクスポートする

社内PCには、情報システム部によって既に社内CA証明書が導入されています。これをファイルとして取り出します。

①証明書マネージャーを起動する

* キーボードの Windowsキー + R を押して「ファイル名を指定して実行」を開きます。
* certmgr.msc と入力して Enter を押します（「証明書 - 現在のユーザー」が起動します）。

管理者権限で端末全体を確認したい場合は certlm.msc を実行してください。

②社内CA証明書を探す

* 左側ツリーから 「信頼されたルート証明機関」 > 「証明書」 をクリックします。
* 一覧の中から、会社名やプロキシ製品名（例: Zscaler Root CA, BlueCoat, 社名が含まれるCA名）の証明書を見つけます。

③証明書をBase64形式でエクスポートする
必ず「Base 64 encoded X.509」を選択してください

* 該当の証明書を右クリックし、「すべてのタスク」 > 「エクスポート」 を選択します。
* ウィザードが進んだら、エクスポート形式の選択で 「Base 64 encoded X.509 (.CER)」 を選択します。
* ファイル名を company-ca.crt などとし、任意のフォルダ（例: C:\temp\company-ca.crt）に保存します。

## ステップ2: Dockerの証明書ディレクトリ（certs.d）へ配置する

Docker Engineは、対象レジストリ（registry.gitlab.com）と通信する際、所定のディレクトリ（certs.d）に配置された証明書を信頼済みCAとして自動的に読み込みます。

### 1. Windowsユーザーディレクトリ配下にフォルダを作成

PowerShellを起動し、以下のコマンドでフォルダを作成します。

```
PowerShell

# GitLabレジストリ用の証明書ディレクトリを作成
New-Item -ItemType Directory -Force -Path "$HOME\.docker\certs.d\registry.gitlab.com"
```

### 2. エクスポートした証明書を ca.crt という名前で配置

ステップ1で保存した証明書を、上記フォルダへファイル名を ca.crt に変更してコピーします。

```
PowerShell

Copy-Item -Path "C:\temp\company-ca.crt" -Destination "$HOME\.docker\certs.d\registry.gitlab.com\ca.crt"```
```

※Docker HubからのPull時にも証明書エラーが出る場合は、Docker Hub用にも同様にコピーします。

```
PowerShell

New-Item -ItemType Directory -Force -Path "$HOME\.docker\certs.d\docker.io"
Copy-Item -Path "C:\temp\company-ca.crt" -Destination "$HOME\.docker\certs.d\docker.io\ca.crt"
```

### ステップ3: WSL 2側のシステム信頼ストアに追加する（WSL 2バックエンド利用時）

Docker Desktopが「WSL 2 based engine」で動作している場合、WSL 2ディストリビューション側のOS信頼ストアにも証明書を入れておくと、コンテナ内部やCLIからの通信エラーを根本から防げます。
PowerShellからWSL（Ubuntuなど）に入って操作します。

```
Bash

# 1. WSLターミナルを開く
wsl

# 2. Windows側から証明書をUbuntuの信頼済みCAディレクトリへコピー
sudo cp /mnt/c/temp/company-ca.crt /usr/local/share/ca-certificates/company-ca.crt

# 3. OSのCAストアを更新
sudo update-ca-certificates
```

実行後に 1 added, 0 removed; done. と表示されれば、WSL環境全体で社内証明書が有効になります。

## ステップ4: Docker Desktopのプロキシ設定を確認・再起動する

証明書を配置したら、社内プロキシのアドレスが正しくDocker Desktopに登録されているか確認します。

4-1. Docker Desktop の画面を開きます。
4-2. 右上の 歯車アイコン（Settings） をクリックします。
4-3. 左メニューから 「Resources」 > 「Proxies」 を開きます。
4-4. 「Manual proxy configuration」 がオンになっていることを確認し、会社のプロキシ設定を入力します。

* Web Server (HTTP): [http://proxy.company.co.jp:8080](http://proxy.company.co.jp:8080)（社内指定の値）
* Secure Web Server (HTTPS): [http://proxy.company.co.jp:8080](http://proxy.company.co.jp:8080)
* Bypass for these hosts & domains: localhost,127.0.0.1,社内ドメイン

4-5. 右下の 「Apply & restart」 をクリックしてDocker Desktopを再起動します。

## ステップ5: 動作確認

再起動完了後、PowerShellまたはWSLのターミナルでログインおよびPullをテストします。

```
PowerShell

# ログイン確認（エラーが出ずに「Login Succeeded」になれば成功）
docker login registry.gitlab.com
```

証明書の警告（x509: certificate signed by unknown authority）が消え、認証プロンプトが正常に進めば社内CAの信頼設定は完了です。

# 手順通りに設定しても x509: certificate signed by unknown authority が解消しない場合のトラブルシューティング手順

x509: certificate signed by unknown authority が解消しない場合、主に「通信を遮断・署名している証明書と配置した証明書の不一致」「Docker Engine（デーモン）が参照する配置場所のズレ」「WSL 2仮想マシン内部への未反映」のいずれかが原因です。

以下の順で原因を切り分けて解決します。

## 1. 実際に通信をブロックしている証明書を直接取得する

ブラウザやWindows証明書ストアからエクスポートした証明書が、「Dockerの通信経路で実際にプロキシが差し込んでいる証明書」と異なっているケースが多発します。

PowerShellまたはWSLのターミナルから openssl コマンドを叩き、GitLabレジストリへ接続した際にプロキシが返してきている実際の証明書チェーンを直接取得して保存します。

実行コマンド（WSLまたはPowerShell）

```
Bash

# プロキシ経由で registry.gitlab.com に接続し、返ってきた証明書チェーンを表示・保存
openssl s_client -connect registry.gitlab.com:443 -proxy proxy.example.com:8080 -showcerts  /tmp/gitlab_certs.pem
```

* proxy.example.com:8080 は社内のプロキシURLとポートに置き換えてください。
* 出力された /tmp/gitlab_certs.pem をテキストエディタで開き、一番最後にある -----BEGIN CERTIFICATE----- から -----END CERTIFICATE-----（これがプロキシのルートCA）をコピーして、ca.crt として保存し直します。

## 2. Docker Engineデーモン側の参照ディレクトリを網羅する

Docker Desktop（特にWSL 2バックエンド）の場合、Windows側の %USERPROFILE%\.docker\certs.d\ だけでなく、WSL 2内のDockerデーモン自身が参照するディレクトリに置かないと認識されないことがあります。

WSLターミナルを開き、以下のすべての場所に証明書を配置します。

```
Bash

# 1. WSL内のLinuxシステム証明書ストアに配置
sudo cp /tmp/company-ca.crt /usr/local/share/ca-certificates/company-ca.crt
sudo update-ca-certificates

# 2. Dockerデーモン直下の certs.d に配置（ポート番号指定の有無両方をカバー）
sudo mkdir -p /etc/docker/certs.d/registry.gitlab.com
sudo mkdir -p /etc/docker/certs.d/registry.gitlab.com:443
sudo cp /tmp/company-ca.crt /etc/docker/certs.d/registry.gitlab.com/ca.crt
sudo cp /tmp/company-ca.crt /etc/docker/certs.d/registry.gitlab.com:443/ca.crt

# 3. Docker HubからのPull対策（docker.io）
sudo mkdir -p /etc/docker/certs.d/docker.io
sudo cp /tmp/company-ca.crt /etc/docker/certs.d/docker.io/ca.crt
```

## 3. docker-desktop 仮想マシンの内部へ直接証明書を注入する

Docker Desktopは、ユーザーのUbuntuディストリビューションとは別に、docker-desktop という専用の軽量Linux仮想マシン内部で dockerd を動かしています。

ここへ直接証明書を反映させる手順です。

3-1.PowerShellから docker-desktop ディストリビューションに root でログインします。

```
PowerShell

wsl -d docker-desktop -u root
```

3-2.Windows側の証明書を certs.d に直接コピーします。

```
Bash

mkdir -p /etc/docker/certs.d/registry.gitlab.com
cp /mnt/c/temp/company-ca.crt /etc/docker/certs.d/registry.gitlab.com/ca.crt
```

3-3.exit で抜けます。

## 4.daemon.json によるプロキシ・証明書設定の明示

Docker DesktopのGUI設定がデーモン側に正しく渡っていない場合があります。

4-1.Docker Desktopの Settings > Docker Engine を開きます。

4-2.JSONに以下のキーが正しく存在するか確認します（無ければ追記して Apply & restart）。

```
JSON

{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "proxies": {
    "http-proxy": "http://proxy.example.com:8080",
    "https-proxy": "http://proxy.example.com:8080",
    "no-proxy": "localhost,127.0.0.1"
  }
}
```

## 5. 最終検証：どこで落ちているかの特定

設定後、どのレイヤーで証明書エラーが出ているかを段階的にテストします。

① WSL内のcurlでプロキシ疎通と証明書チェック

```
Bash

curl -v -x http://proxy.example.com:8080 https://registry.gitlab.com/v2/
```

* HTTP 401 Unauthorized が返ってきた場合：証明書とプロキシは完全に通っています（401は認証待ちの正常応答）。
* SSL certificate problem: self signed certificate や unable to get local issuer certificate が出た場合：OS側の証明書ストア（update-ca-certificates）に配置した証明書がまだ間違っています。

② Docker CLIでの疎通チェック

```
Bash

docker login registry.gitlab.com
```

curlが通り、Docker側でも Username: の入力プロンプト（またはLogin Succeeded）が出れば解決です。

# Dockerビルド（docker build）実行時に社内CA証明書エラーでパッケージ取得が失敗する場合の対策

docker build 実行中に apt-get、pip、npm、curl などのパッケージ取得が社内CA証明書エラーで失敗する場合、「ホストOS側の信頼設定は、ビルド中のコンテナ内部には自動的に引き継がれない」という点が原因です。

完成するイメージに不要なファイルや社内認証情報を残さず、安全かつ確実にビルドを通す方法は主に3通りあります。

## 対策1: BuildKitのシークレットマウントを使う（最も推奨）

完成したイメージのレイヤーに社内証明書ファイルを焼き込まず、ビルド実行時の一時マウントとしてのみ証明書を渡すアプローチです。公開用イメージに社内証明書が混入するリスクを完全に防げます。

1-1. Dockerfileの記述例（Ubuntu / Debian系ベースの場合）
RUN --mount=type=secret を利用し、一時的にOSの証明書ディレクトリにマウントした状態で update-ca-certificates を走らせ、パッケージを導入します。

```
Dockerfile

# syntax=docker/dockerfile:1.4
FROM ubuntu:24.04

# ビルド時のみ証明書をマウントして信頼させ、パッケージをインストール
RUN --mount=type=secret,id=company-ca,target=/usr/local/share/ca-certificates/company-ca.crt \
    update-ca-certificates && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates \
        python3-pip && \
    rm -rf /var/lib/apt/lists/*
```

1-2. Python (pip) の場合
pip は環境変数 PIP_CERT で指定したパスの証明書を直接参照できます。

```
Dockerfile

# syntax=docker/dockerfile:1.4
FROM python:3.11-slim

COPY requirements.txt .

RUN --mount=type=secret,id=company-ca,target=/run/secrets/company-ca.crt \
    PIP_CERT=/run/secrets/company-ca.crt pip install --no-cache-dir -r requirements.txt
```

1-3. ビルド実行コマンド
ホスト側の証明書ファイルを id=company-ca として渡します。

```
Bash

docker build --secret id=company-ca,src=./company-ca.crt -t my-app:latest .
```

## 対策2: ビルド引数（ARG）＋マルチステージビルドで最後に削除する

イメージ公開の予定がない社内完結のプロジェクトで、Dockerfile内にシンプルに記述したい場合の手法です。

```
Dockerfile

FROM ubuntu:24.04

# 1. ホスト側から証明書を一時配置
COPY company-ca.crt /usr/local/share/ca-certificates/company-ca.crt

# 2. OSストアに登録してパッケージ取得
RUN update-ca-certificates && \
    apt-get update && \
    apt-get install -y curl python3-pip && \
    rm -rf /var/lib/apt/lists/* && \

# 3. ビルド終了時に証明書を削除（イメージに残さない）
    rm /usr/local/share/ca-certificates/company-ca.crt && \
    update-ca-certificates --fresh
```

## 対策3: 各言語・ツール固有の環境変数を渡す

Node.jsやPythonなど、OSの証明書ストアではなく独自のバンドル証明書を優先して参照するツールを使う場合は、ビルド時引数（ARG または ENV）でパスを教えてあげる必要があります。

| ツール / 言語	| 環境変数・オプション |	設定例 |
| ---- | ---- | ---- | 
| Node.js (npm / yarn)	| NODE_EXTRA_CA_CERTS | ENV NODE_EXTRA_CA_CERTS=/path/to/company-ca.crt |
| Python (pip / requests) | 	PIP_CERT または REQUESTS_CA_BUNDLE | 	ENV REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt |
| Git	| http.sslCAInfo |  	RUN git config --global http.sslCAInfo /path/to/company-ca.crt |
| Curl	| CURL_CA_BUNDLE | 	ENV CURL_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt |

注意：プロキシ設定のビルドコンテナへの伝播
証明書だけでなく、apt-get や pip が社内プロキシを通れずにタイムアウトや接続拒否になっているケースもあります。

Docker Desktopでは通常、ホストのプロキシ設定がビルドコンテナへ自動注入されますが、うまく動かない場合はビルドコマンド時に明示的に引数を渡します。

```
Bash

docker build \
  --build-arg HTTP_PROXY="http://proxy.example.com:8080" \
  --build-arg HTTPS_PROXY="http://proxy.example.com:8080" \
  --build-arg NO_PROXY="localhost,127.0.0.1" \
  --secret id=company-ca,src=./company-ca.crt \
  -t my-app:latest .
```

# GitLab CI/CD（Kaniko または Docker-in-Docker）のビルドジョブで、社内CA証明書とプロキシ設定を自動注入する設定方法

GitLab CI/CD上でコンテナイメージをビルドする際、社内CA証明書（企業Cert）とプロキシ設定をジョブへ自動注入する代表的な2つの方式（Kaniko と Docker-in-Docker）の具体設定を解説します。

共通の前提準備：GitLab CI/CD 変数の登録
リポジトリ（またはグループ）の Settings > CI/CD > Variables に以下を登録します。

| 変数名	 | タイプ | 内容・設定 |
| --- | --- | ---- |
| CORPORATE_CA_CERT |  	File	| 社内ルートCA証明書（company-ca.crt）のテキスト内容 |
| HTTP_PROXY | 	Variable | 	[http://proxy.example.com:8080](http://proxy.example.com:8080) |
| HTTPS_PROXY | 	Variable | 	[http://proxy.example.com:8080](http://proxy.example.com:8080) |
| NO_PROXY | 	Variable | 	localhost,127.0.0.1,.example.com,registry.gitlab.com |

## 方式1: Kaniko によるビルド（特権不要・セキュア・推奨）

Kanikoはデーモン不要のコンテナ内でビルドを完結させるため、GitLab Runnerに特権（privileged）モードを与えられない環境に最適です。

Kanikoがベースイメージを取得・Pushする通信、およびDockerfile内で実行される各コマンドの両方に証明書とプロキシを渡します。

.gitlab-ci.yml 設定例

```
YAML

build_kaniko:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.20.0-debug
    entrypoint: [""]
  variables:
    HTTP_PROXY: "$HTTP_PROXY"
    HTTPS_PROXY: "$HTTPS_PROXY"
    NO_PROXY: "$NO_PROXY"
    http_proxy: "$HTTP_PROXY"
    https_proxy: "$HTTPS_PROXY"
    no_proxy: "$NO_PROXY"
  before_script:
    # 1. GitLab Container Registry 認証情報の作成
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "$CI_REGISTRY_USER" "$CI_REGISTRY_PASSWORD" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json

    # 2. Kaniko 自身のルートCAストアに社内証明書を追加（BaseイメージのPull用）
    - cat "$CORPORATE_CA_CERT" >> /kaniko/ssl/certs/ca-certificates.crt
  script:
    - >-
      /kaniko/executor
      --context "$CI_PROJECT_DIR"
      --dockerfile "$CI_PROJECT_DIR/Dockerfile"
      --destination "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
      --build-arg HTTP_PROXY="$HTTP_PROXY"
      --build-arg HTTPS_PROXY="$HTTPS_PROXY"
      --build-arg NO_PROXY="$NO_PROXY"
      --build-arg http_proxy="$HTTP_PROXY"
      --build-arg https_proxy="$HTTPS_PROXY"
      --build-arg no_proxy="$NO_PROXY"
```

## 方式2: Docker-in-Docker（DinD）によるビルド

Docker CLIをそのまま利用できるため既存のビルドスクリプトと親和性が高い方式です（Runnerの privileged = true が必須）。

DinDでは、「クライアントコンテナ（docker:cli）」 と 「バックグラウンドで動くデーモンコンテナ（docker:dind）」 の両方に証明書とプロキシを認識させる必要があります。

.gitlab-ci.yml 設定例

```
YAML

build_dind:
  stage: build
  image: docker:24-cli
  services:
    - name: docker:24-dind
      # dindデーモンの起動オプションにプロキシと証明書適用を指定
      command: ["--insecure-registry=registry.gitlab.com"]
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    HTTP_PROXY: "$HTTP_PROXY"
    HTTPS_PROXY: "$HTTPS_PROXY"
    NO_PROXY: "$NO_PROXY"
  before_script:
    # 1. クライアント側のOS信頼ストアへ社内CAを追加
    - cp "$CORPORATE_CA_CERT" /usr/local/share/ca-certificates/company-ca.crt
    - update-ca-certificates

    # 2. Dockerデーモン（レジストリ通信用）の certs.d に証明書を配置
    - mkdir -p /etc/docker/certs.d/registry.gitlab.com
    - cp "$CORPORATE_CA_CERT" /etc/docker/certs.d/registry.gitlab.com/ca.crt
    - mkdir -p /etc/docker/certs.d/docker.io
    - cp "$CORPORATE_CA_CERT" /etc/docker/certs.d/docker.io/ca.crt

    # 3. GitLab Container Registry へログイン
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    # BuildKit を有効化してシークレットマウント機能で社内CAを渡す
    - export DOCKER_BUILDKIT=1
    - >-
      docker build
      --build-arg HTTP_PROXY="$HTTP_PROXY"
      --build-arg HTTPS_PROXY="$HTTPS_PROXY"
      --build-arg NO_PROXY="$NO_PROXY"
      --secret id=company-ca,src="$CORPORATE_CA_CERT"
      -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
```

Dockerfile 側での受け取り（共通）
CIジョブから渡されたプロキシ情報と証明書は、Dockerfile内で以下のように受け取ります。

```
Dockerfile

# syntax=docker/dockerfile:1.4
FROM ubuntu:24.04

# プロキシ設定の受け取り
ARG HTTP_PROXY
ARG HTTPS_PROXY
ARG NO_PROXY
ENV http_proxy=${HTTP_PROXY} \
    https_proxy=${HTTPS_PROXY} \
    no_proxy=${NO_PROXY}

# ビルド時のみ社内CAを適用してパッケージインストール
RUN --mount=type=secret,id=company-ca,target=/usr/local/share/ca-certificates/company-ca.crt \
    update-ca-certificates && \
    apt-get update && \
    apt-get install -y --no-install-recommends curl python3-pip && \
    rm -rf /var/lib/apt/lists/*
```

この構成により、CI/CD上で社内プロキシとSSLインスペクションを通過しつつ、生成される成果物イメージには社内証明書やパスワードを残さないセキュアなパイプラインが完成します。

# GitLab Runnerホスト側の config.toml で全ジョブにプロキシと社内CA証明書を一括注入する設定手順

GitLab Runnerホスト側の /etc/gitlab-runner/config.toml を構成すると、各リポジトリの .gitlab-ci.yml に個別に環境変数や証明書の設定を書くことなく、そのRunnerで実行されるすべてのジョブ・コンテナにプロキシ設定と社内CA証明書を一括で自動注入できます。

## 1. ホスト側での事前準備（証明書の配置）

GitLab Runnerが動作しているホストマシン（Linuxサーバー）の所定の場所に、社内CA証明書を配置します。

```
Bash

# 証明書用のディレクトリを作成
sudo mkdir -p /etc/gitlab-runner/certs

# 社内CA証明書（company-ca.crt）を配置
sudo cp /path/to/company-ca.crt /etc/gitlab-runner/certs/company-ca.crt
sudo chmod 644 /etc/gitlab-runner/certs/company-ca.crt
```

## 2. config.toml の編集

GitLab Runnerの設定ファイル（/etc/gitlab-runner/config.toml）を編集します。
主に environment（環境変数の一括定義）と volumes（コンテナへの証明書自動マウント）の2箇所を設定します。

```
Ini, TOML

concurrent = 4
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "corporate-docker-runner"
  url = "https://gitlab.example.com/"
  id = 1
  token = "glrt-xxxxxxxxxxxxxxxxxxxx"
  executor = "docker"

  # 1. 全ジョブにプロキシ環境変数とSSL証明書パスを一括注入
  environment = [
    "http_proxy=http://proxy.example.com:8080",
    "https_proxy=http://proxy.example.com:8080",
    "no_proxy=localhost,127.0.0.1,.example.com,registry.gitlab.com",
    "HTTP_PROXY=http://proxy.example.com:8080",
    "HTTPS_PROXY=http://proxy.example.com:8080",
    "NO_PROXY=localhost,127.0.0.1,.example.com,registry.gitlab.com",
    # 各言語・ツール向けの証明書パス指定
    "SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt",
    "REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt",
    "NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/company-ca.crt"
  ]

  [runners.custom_build_dir]
  [runners.cache]
    MaxUploadedArchiveSize = 0

  [runners.docker]
    tls_verify = false
    image = "ubuntu:24.04"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false

    # 2. ホストの証明書を起動する全コンテナ（ジョブ、サービス、dind）に自動マウント
    volumes = [
      "/cache",
      # Linux標準の証明書更新ディレクトリへ配置（ro = 読み取り専用）
      "/etc/gitlab-runner/certs/company-ca.crt:/usr/local/share/ca-certificates/company-ca.crt:ro",
      # Docker Engine / dind 用の certs.d 配下へ自動配置
      "/etc/gitlab-runner/certs/company-ca.crt:/etc/docker/certs.d/registry.gitlab.com/ca.crt:ro",
      "/etc/gitlab-runner/certs/company-ca.crt:/etc/docker/certs.d/docker.io/ca.crt:ro"
    ]

    shm_size = 0
```

## 3. 各設定項目の役割

| 項目	| 役割 |
| -- | -- |
| environment	| ジョブコンテナ起動時に、大文字・小文字両方のプロキシ環境変数を自動設定します。また、Python（REQUESTS_CA_BUNDLE）やNode.js（NODE_EXTRA_CA_CERTS）が参照する証明書パスも事前指定します。 |
| /usr/local/share/ca-certificates/...:ro | 	Debian/Ubuntu系ベースのコンテナで、ジョブ開始時に update-ca-certificates を叩くだけでOSレベルで即時信頼される場所に配置します。 |
| /etc/docker/certs.d/...:ro | 	Docker-in-Docker（dind）サービスを利用する際、dindデーモンがレジストリ（GitLabやDocker Hub）へアクセスする際の信頼済み証明書として自動認識させます。 |

## 4. 設定の反映と確認

### 4-1. GitLab Runner を再起動

```
Bash

sudo gitlab-runner restart
```

### 4-2. .gitlab-ci.yml 側での記述（簡略化の確認）

ホスト側で一括注入されているため、個々のパイプライン定義側ではプロキシや証明書を変数定義する必要がなくなります。

```
YAML

test_job:
  stage: test
  image: alpine:latest
  script:
    # ボリュームマウントされている証明書をOSストアに反映
    - cat /usr/local/share/ca-certificates/company-ca.crt >> /etc/ssl/certs/ca-certificates.crt
    # プロキシと証明書が通っていることを確認
    - wget -qO- https://registry.gitlab.com/v2/
```
