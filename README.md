# Django with Docker

Docker上で動作するPython（django）アプリ開発環境のサンプル。

## 参考
- Docker公式： [Quickstart: Compose and Django](https://docs.docker.com/compose/django/)
- django公式： [Getting started](https://docs.djangoproject.com/ja/2.0/intro/)

## 手順

### Docker Compose 準備と起動

ここでは django 公式チュートリアルに従い、 django + PostgreSQL の構成でWebアプリケーションを構築する。

1. docker-compose.yml

    Docker Compose を使うことで、複数のコンテナ (ここでは django と PostgreSQL) を、ひとまとまりのアプリケーションとして一括管理できる。

    ``` yaml
    version: '3'
    
    services:
      db:
        image: postgres
        volumes:
          - postgres-pysample:/var/lib/postgresql/data
        ports:
          - "5432:5432"
      web:
        build: .
        command: python3 manage.py runserver 0.0.0.0:8000
        volumes:
          - .:/code
        ports:
          - "8000:8000"
        depends_on:
          - db
    volumes:
      postgres-pysample:
    ```

    - `version` : 文法のバージョン。サンプルなので、最新の3を指定する。

    - `services` : アプリケーションを構成するサービス (コンテナ) の設定を列挙する。サービス名は任意。
        
        - `db` : DBサーバーのサービスなので、`db` とした。
            
            - `image` : 他のコンテナイメージをそのまま利用する場合は、`image` タグで指定する。ここでは、Docker 公式の PostgreSQL サーバーのイメージを使用。

            - `volumes` : DBデータをコンテナ外の領域にマウントする。ここでは、後述のルートノードの `volumes` で定義する `postgres-pysample` に PostgreSQL のデータ領域をマウントする。 (マウントしないと、コンテナ起動の都度データが飛ぶ)

            - `ports` : `"ホストPCのポート:コンテナのポート"` を指定して割り当てる。IDE等からDBを直接参照できると便利なので、PostgreSQLのデフォルトポートを割り当てておく。

        - `web` : django Webサーバーのサービスなので、`web` とした。
            - build : 既存の Docker イメージでなく、Doclerfile からビルドしたイメージからコンテナを作成する場合に、Dockerfile 配置ディレクトリを指定する。今回は単一のWebアプリケーションのため、作業ディレクトリに Dockerfile (および django アプリのファイル) を配置する。

            - `command` : サービス起動時にコンテナのOSに実行させるコマンド。django Webサーバーをポート8000で起動する。

            - `volumes` : ホストPCの作業ディレクトリをコンテナの django ルートディレクトリ (Dockerfile で設定) にマウントする。これでホストPCからコンテナのWebアプリケーションを直接編集できる。

            - `ports` : ホストPCのポート (8000) を、`command` で起動するWebサーバーのポート (8000) にリンクする。これでホストPCのブラウザから、 `localhost:8000` でコンテナのWebアプリを実行できる。

            - `depends_on` : 上述の `db` サービスへの依存を宣言する。これで`web` サービスから `db` サービスにアクセスできる。

    - `volumes` : Docker ミドルウェアが管理するストレージを定義する。ホストPCのディレクトリにもマウントできるが、DBのデータ領域などホストPCから直接参照する用がなければ、こちらに使い慣れておいたほうが良い。

        - `postgres-pysample` : `db`のDBデータ領域用に定義した。

1. Dockerfile

    ``` dockerfile
    FROM python:3
    ENV PYTHONUNBUFFERED 1
    RUN mkdir /code
    WORKDIR /code
    ADD requirements.txt /code/
    RUN pip install -r requirements.txt
    ADD . /code/
    ```

1. requirements.txt

    ``` text
    Django>=2.0,<3.0
    psycopg2
    ```

1. Docker Compose 実行

    プロジェクトフォルダ上で以下のコマンドを実行すると、PythonとPostgreSQLのコンテナが起動する。
    
    ``` shell
    sudo docker-compose run web django-admin.py startproject mysite .
    ```
    
    Linuxの場合、コンテナ内のdjangoが自動生成したファイルの所有者がrootユーザーとなるため、そのままでは編集できない。以下のコマンドで所有権を自分に付け替える。(以降、自動生成の都度所有権の付け替えが必要。。。ちょっと面倒)
    
    ``` shell
    sudo chown -R $USER:$USER .
    ```

1. djangoへの設定(mysite/settings.py)

    `DATABASE` を、PostgreSQLコンテナへの接続設定に書き換える。
    
    ``` python
    
    # Database
    # https://docs.djangoproject.com/en/2.0/ref/settings/#databases

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'NAME': 'postgres',
            'USER': 'postgres',
            'HOST': 'db',
            'PORT': 5432,
        }
    }
    ```
    
    言語とタイムゾーンの設定を行う。
    
    ``` python
    LANGUAGE_CODE = 'ja-jp'
    ```
    
    ``` python
    TIME_ZONE = 'Asia/Tokyo'
    ```

1. Dockerの起動

    以下のコマンドでコンテナを起動する。
    
    ``` shell
    docker-compose up -d
    ```
    `http://localhost:8000` で、起動を確認できる。
    
    停止。

    ``` shell
    docker-compose down
    ```