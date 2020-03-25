WebHookの開発とかで、たまに[RequestBin](https://github.com/Runscope/requestbin)したくなります。

今上がってるやつをそのまま使うのはなんだかなーというので、ざっくりとbasic認証だけ追加してherokuにpushして使ったのでメモ。

変更済みのフォークはこちら。 https://github.com/sawanoboly/requestbin/tree/local/heroku/basicauth

ユーザ名とパスワードを環境変数から取るようにして、全部のリクエストの前に差し込むHookで認証を必須にしたつもり。

```diff:git_diff_master..local/heroku/basicauth
diff --git a/requestbin/__init__.py b/requestbin/__init__.py
index b7e978a..f6af00e 100644
--- a/requestbin/__init__.py
+++ b/requestbin/__init__.py
@@ -4,7 +4,7 @@ from cStringIO import StringIO
 
 from flask import Flask
 from flask_cors import CORS
-
+from flask_httpauth import HTTPBasicAuth
 
 class WSGIRawBody(object):
     def __init__(self, application):
@@ -35,6 +35,22 @@ class WSGIRawBody(object):
 
 
 app = Flask(__name__)
+auth = HTTPBasicAuth()
+
+users = {
+    os.environ.get('USER'):os.environ.get('PASS')
+}
+
+@auth.get_password
+def get_pw(username):
+    if username in users:
+        return users.get(username)
+    return None
+
+@app.before_request
+@auth.login_required
+def before_request():
+    pass
 
 if os.environ.get('ENABLE_CORS', config.ENABLE_CORS):
     cors = CORS(app, resources={r"*": {"origins": os.environ.get('CORS_ORIGINS', config.CORS_ORIGINS)}})
diff --git a/requirements.txt b/requirements.txt
index 38f400f..8f74bb8 100644
--- a/requirements.txt
+++ b/requirements.txt
@@ -14,3 +14,4 @@ python-dateutil==2.1
 gunicorn
 bugsnag
 blinker
+flask-httpauth
diff --git a/runtime.txt b/runtime.txt
index 4b38fc9..f27f1cc 100644
--- a/runtime.txt
+++ b/runtime.txt
@@ -1 +1 @@
-python-2.7.13
+python-2.7.15
```

これを公式のガイド[Deploy your own instance using Heroku](https://github.com/Runscope/requestbin#deploy-your-own-instance-using-heroku)でherokuにpushします。

あとは環境変数`USER`,`PASS`に値を入れておしまい。

```
$ heroku config:set set USER=foo PASS=bar
```

まあ、アプリケーション名ランダムで十分長くしておく、とかでも構わんのですが。最後に用事が済んだら`heroku redis:cli`からDBをFLUSHALLしとくとなんか安心。

補足

SaaSが良ければ requestbin alternative で検索すれば色々出てきてフリープラン付きもあるのでそれを使えばよいでしょう。だいたい自分がRequestBinしか名前を覚えてないのでこんなことに。
ほか、ngrokでローカルになんか立てるのも良いですね。

