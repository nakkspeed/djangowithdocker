# Django with Docker

Docker上で動作するPython（django）アプリ開発環境のサンプル。

## 参考
- Docker公式： [Quickstart: Compose and Django](https://docs.docker.com/compose/django/)
- django公式： [Getting started](https://docs.djangoproject.com/ja/1.11/intro/)

## 手順

### Docker Compose 準備と起動

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

2. requirements.txt

    ``` text
    Django>=2.0,<3.0
    psycopg2
    ```

3. docker-compose.yml

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

4. Docker Compose 実行

    プロジェクトフォルダ上で以下のコマンドを実行すると、PythonとPostgreSQLのコンテナが起動する。
    
    ``` shell
    sudo docker-compose run web django-admin.py startproject mysite
    ```
    
    Linuxの場合、コンテナ内のdjangoが自動生成したファイルの所有者がrootユーザーとなるため、そのままでは編集できない。以下のコマンドで所有権を自分に付け替える。(以降、自動生成の都度所有権の付け替えが必要。。。ちょっと面倒)
    
    ``` shell
    sudo chown -R $USER:$USER .
    ```

5. djangoへの設定(mysite/settings.py)
 
    `DATABASE` を、PostgreSQLコンテナへの接続設定に書き換える。
    
    ``` python
    # Database
    # https://docs.djangoproject.com/en/1.11/ref/settings/#databases
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

6. Dockerの起動

    以下のコマンドでコンテナを起動する。
    
    ``` shell
    docker-compose up
    ```
    `http://localhost:8000` で、起動を確認できる。
    
    停止は`Ctrl+C`。