# GKE に Django+React をデプロイする

## やりたいこと

Docker で環境構築して k8s にデプロイしたい。
バックエンドは DjangoRestFramework で RestAPI を配信して、フロントエンドは React(TypeScript)で実装したい。

## 環境

```sh
node --version
v12.14.1

npm --version
6.13.7

python --version
Python 3.7.4

docker --version
Docker version 19.03.8
```

1. アプリケーションを作成する
   ローカルで起動できることを確認する

2. GCP でプロジェクトを作成する
   CloudSQL, CloudStorage を使ってローカルからクラウドのマネージドサービスを利用できるようにする。

3. Docker イメージを作成する
   イメージを作成する。.env は省いた方が良いがこれはまだ対応していない。

4. GKE のクラスタを作成する
   プロジェクトを開始して GKE クラスタを作成する。

5. Deployment を作成する
   backend の Secret を作成してデータベースに利用。

6. Service を作成する
   backend と frontend

7. Ingress を作成する

8. ヘルスチェックに対応する
   ヘルスチェックに対応する。
   静的アドレスに追加する。200 を返すように view を追加。

9. DNS に対応する
   ドメインを取得して ingress に host を追加。

10. HTTPS 化
    https://qiita.com/watiko/items/71d78a4d31eee02cb47d

## まずはローカルで始める

### ディレクトリを作成する。

```sh
# プロジェクトフォルダの作成
$ mkdir gke-django-tutorial
$ cd gke-django-tutorial
# ディレクトリを作成する
$\gke-django-tutorial\mkdir backend
$\gke-django-tutorial\mkdir frontend
```

### Backend の開発を始める

backend は Djang-rest-framework で RestAPI を作成します。
まずは backend から環境を作成してみます。

```sh
# backendディレクトリに移動
$\gke-django-tutorial\cd backend
# Pythonの仮想環境作成
$\gke-django-tutorial\backend\python -m venv venv
# 仮想環境の有効化
$\gke-django-tutorial\backend\vnev\Scripts\activate
# Pythonパッケージのインストール
(venv)$\gke-django-tutorial\backend\python -m install --upgrade pip setuptools
(venv)$\gke-django-tutorial\backend\python -m install django djangorestframework python-dotenv
# Djangoのプロジェクトを始める
(venv)$\gke-django-tutorial\backend\django-admin startproject config .
```

backend ディレクトリ下で`django-admin startprject config .`とすることで
`config`という Django プロジェクトフォルダが作成されました。

ローカルサーバーが起動するかどうか確認しましょう。

```sh
(venv)$\gke-django-tutorial\backend\python manage.py runserver
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 17 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.
April 27, 2020 - 11:22:06
Django version 3.0.5, using settings 'config.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CTRL-BREAK.
```

開発用サーバーが起動したので`http://localhost:8000/`にアクセスすると`The install worked successfully!`の画面が確認できます。

#### settings.py

`config/settings.py`を編集して基本的な設定を盛り込みます。
settings.py の秘匿すべき情報は`.env`ファイルに記述して公開しないようにします。
python-dotenv を使って`.env`に記載された情報を利用するように変更しましょう。

```sh
# .envファイルの作成
(venv)$\gke-django-tutorial\backend\type null > .env
```

```python:config.settins.py
# config/settings.py

"""
Django settings for config project.

Generated by 'django-admin startproject' using Django 3.0.5.

For more information on this file, see
https://docs.djangoproject.com/en/3.0/topics/settings/

For the full list of settings and their values, see
https://docs.djangoproject.com/en/3.0/ref/settings/
"""

import os
from dotenv import load_dotenv  # 追加

# Build paths inside the project like this: os.path.join(BASE_DIR, ...)
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
PROJECT_DIR = os.path.basename(BASE_DIR)  # 追加

# .envの読み込み
load_dotenv(os.path.join(BASE_DIR, '.env'))  # 追加

# Quick-start development settings - unsuitable for production
# See https://docs.djangoproject.com/en/3.0/howto/deployment/checklist/

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = os.getenv('SECRET_KEY')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = os.getenv('DEBUG')

ALLOWED_HOSTS = ["*"]  # 変更


# Application definition

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'config.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],  # 変更
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

WSGI_APPLICATION = 'config.wsgi.application'


# Database
# https://docs.djangoproject.com/en/3.0/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}


# Password validation
# https://docs.djangoproject.com/en/3.0/ref/settings/#auth-password-validators

AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]


# Internationalization
# https://docs.djangoproject.com/en/3.0/topics/i18n/

LANGUAGE_CODE = 'ja'  # 変更

TIME_ZONE = 'Asia/Tokyo'

USE_I18N = True

USE_L10N = True

USE_TZ = True


# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/3.0/howto/static-files/

STATIC_URL = '/static/'


# 開発環境下で静的ファイルを参照する先
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')] # 追加

# 本番環境で静的ファイルを参照する先
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles') # 追加

# メディアファイルpath
MEDIA_URL = '/media/' # 追加

```

```sh:.env
# .env
SECRET_KEY = 'nz8t((i_mali=%0+z_g+uc**9dgky8)74slvs&6h$=9b^8t7$@'
DEBUG = False
```

#### アプリケーションを追加する

todo アプリケーションを作っていきましょう。

```sh
(venv)$\gke-django-tutorial\backend\python manage.py startapp todo
```

`config/settings.py` の`INSTALLED_APPS`に`todo`と`rest_framework`を追加します。

```python
# config/settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # 3rd party
    'rest_framework',

    # Local
    'todo.apps.TodoConfig',
]

# 追加
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny',
    ]
}

```
`rest_framework.permissions.AllowAny`は django-rest-framework が暗黙的に決めているデフォルトの設定`'DEFAULT_PERMISSION_CLASSES'`を解除するためのものです。
この設定はまだよくわかってないのですがとりあえず前に進みます。

#### todo/models.py

`todo`アプリケーション model を作成します。

```python
# todo/models.py
from django.db import models


class Todo(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()

    def __str__(self):
        return self.title

```

`todo/admin.py`に作成したモデルを追加します。

```python
# todo/admin.py
from django.contrib import admin
from .models import Todo


admin.site.register(Todo)
```

マイグレーションします。

```sh
(venv)$\gke-django-tutorial\backend\python manage.py makemigrations
Migrations for 'todo':
  todo\migrations\0001_initial.py
    - Create model Todo

(venv)$\gke-django-tutorial\backend\python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions, todo
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying sessions.0001_initial... OK
  Applying todo.0001_initial... OK
```

#### createsuperuser

管理者ユーザーを作成します。

```sh
(venv)$\gke-django-tutorial\backend\python manage.py createsuperuser
ユーザー名 (leave blank to use 'you'): admin
メールアドレス: XXXXX@gmail.com
Password:
Password (again):
Superuser created successfully.
```

開発用サーバーを起動して`http://localhost:8000/admin`にアクセスすると
Django管理サイトが表示されます。
先ほど設定したユーザー名、パスワードを入力してログインしてみましょう。

作成したアプリケーション`Todo`のテーブルを確認することができます。
2，3コアイテムを追加しておきましょう。

#### URLs

`config/urls.py`に todo へのルーティングを追加します。

```python
# config/urls.py

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('todo.urls'))  # 追加
]

```

#### todo/urls.py

`todos/urls.py`を作成します。

```sh
(venv)$\gke-django-tutorial\backend\type null > todo\urls.py
```

```python
# todo/urls.py
from django.urls import path, include
from .views import ListTodo, DetailTodo

urlpatterns = [
    path('<int:pk>', DetailTodo.as_view()),
    path('', ListTodo.as_view())
]
```

#### todo/selializers.py

モデルインスタンスを簡単にjson形式に変換するためのシリアライザーを作成します。

```sh
(venv)$\gke-django-tutorial\backend\type null > todo\serializers.py
```

```python
# todo/serializers.py
from rest_framework import serializers
from .models import Todo


class TodoSerializer(serizers.ModelSerializer):
    class Meta:
        model = Todo
        fields = ('id', 'title', 'body')

```

`fields = ('id', 'title', 'text')`での`id`は PrimaryKey を指定しない場合、
Django によって自動的に追加されます。

#### todo/views.py

Django Rest Frameworkで`views.py`を作成する場合は`rest_framework.generics`のAPIViewを継承します。

```python
# todo/views.py

from django.shortcuts import render
from rest_framework import generics
from .models import Todo
from .serializers import TodoSerializer


class ListTodo(generics.ListAPIView):
    queryset = Todo.objects.all()
    serializer_class = TodoSerializer


class DetailTodo(generics.RetrieveAPIView):
    queryset = Todo.objects.all()
    serializer_class = TodoSerializer
```

router など設定できていませんが、とりあえずは Todo アイテムを API として使用できるようになりました。
開発サーバーで`http://127.0.0.1:8000/api/`にアクセスすると APIview を確認することができます。

ここまでは Django でよくあるローカル環境での開発です。

#### CORS

CORS(Cross-Origin Resource Sharing)は React と Django を連携させる場合、
React を起動した`localhost:3000`は Django の API サーバー`localhost:8000`と
json のやり取りを行わせるためのものです。
`django-cors-headers`をインストールしましょう。

```sh
(venv)$\gke-django-tutorial\backend\python -m pip install django-cors-headers
```

`config/settings.py`を更新します。

```python
# config/settings.py

# Application definition

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # 3rd party
    'rest_framework',
    'corsheaders',

    # Local
    'todos.apps.TodosConfig',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMidddleware',  # 追加
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

##################
# rest_framework #
##################

REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny',
    ]
}

CORS_ORIGIN_WHITELIST = (
    'http://localhost:3000',
)
```

#### local_settings.py

`config/settings.py`は本番環境に使用することを考慮し、`config/local_settings.py`を作成してローカル開発用に分けておきます。
GKEデプロイ時にはCloudSQLを使用し、ローカルではsqlite3を使用するように、settings.pyを分けておくことで設定値を書き換えずに済みます。

```sh
# ファイルの作成
(venv)$\gke-django-tutorial\backend\type null > config/local_settings.py
```

```python
# config/local_settings.py
from .settings import *

DEBUG = True

ALLOWED_HOSTS = ['*']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}
```

`config/local_settings.py`を使って開発用サーバーを起動しておきます。

```sh
(venv)$\gke-django-tutorial\backend\python manage.py runserver --settings config.local_settings
```

#### Tests

テストを書きます。

```python
# todos/test.py

from django.test import TestCase
from .models import Todo


class TodoModelTest(TestCase):

    @classmethod
    def setUpTestData(cls):
        Todo.objects.create(title="first todo", body="a body here")

    def test_title_content(self):
        todo = Todo.objects.get(id=1)
        excepted_object_name = f'{todo.title}'
        self.assertEqual(excepted_object_name, 'first todo')

    def test_body_content(self):
        todo = Todo.objects.get(id=1)
        excepted_object_name = f'{todo.body}'
        self.assertEqual(excepted_object_name, 'a body here')

```

```sh
(venv)$\gke-django-tutorial\backend\ python manage.py test
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
..
----------------------------------------------------------------------
Ran 2 tests in 0.007s

OK
Destroying test database for alias 'default'...
```

うまくいったようです。

#### 静的ファイル

配信後に管理者機能のcssが反映されるように静的ファイルを集約しておきます。

```sh
# 静的ファイル用ディレクトリ
(venv)$\gke-django-tutorial\backend\mkdir static
# 静的ファイルの集約
(venv)$\gke-django-tutorial\backend\python manage.py collectstatic
```

### Frontendの開発を進める

新しいコマンドプロンプトを開いてReactのプロジェクトを開始していきます。

```sh
# ディレクトリ下にReactプロジェクトをたてる
$\gke-django-tutorial\frontend\npx create-react-app .
# Reactの開発用サーバーを立ち上げる
$\gke-django-tutorial\frontend\yarn start
yarn run v1.22.0
$ react-scripts start
i ｢wds｣: Project is running at http://192.168.11.8/
i ｢wds｣: webpack output is served from
i ｢wds｣: Content not from webpack is served from C:\Users\masayoshi\docker_project\gke-django-tutorial_v2\frontend\public
i ｢wds｣: 404s will fallback to /
Starting the development server...
Compiled successfully!

You can now view frontend in the browser.

  Local:            http://localhost:3000
  On Your Network:  http://192.168.11.8:3000

Note that the development build is not optimized.
To create a production build, use yarn build.
```

`http://localhost:3000`にアクセスするとReactのWelcomeページが確認できます。

APIをリクエストするのには`axios`を使います。

```sh
# ライブラリのインストール
$\gke-django-tutorial\frontend\yarn add axios
```

#### App.js
APIのエンドポイントは以下のような形でAPIを返してきます。

```javascript
// src/App.js
import React from 'react';
import axios from "axios";
import './App.css';

class App extends Component {
  state = {
    todo: []
  };

  componentDidMount() {
    this.getTodos();
  }

  getTodos() {
    axios
      .get("http://localhost:8000/api/")
      .then(res => {
        this.setState({ todo: res.data });
      })
      .catch(err => {
        console.log(err);
      });
  }
  render() {
    return (
      <div>
        {this.state.todo.map(item => (
          <div>
            <h1>{item.title}</h1>
            <p>{item.body}</p>
          </div>
        ))}
      </div>
    );
  }
}

export default App;

```

これローカル環境でfronendからbarckendへのapiを叩いてtodoリスト一覧を表示させることができました。

### Docker化

次はこれをDocker化していきます。
frontend, backendそれぞれにDockerfileを作成してbackendコンテナ、frontendコンテナを作成します。

docker-composeで立ち上げられるところまでを考えていきます。

#### backendのDocker化
