# #1 ユーザー登録

## 変更したファイル

accounts/models.py

accounts/admin.py

accounts/urls.py

accounts/views.py

templates/accounts

## ユーザーモデルの設計でやること

### 1. モデルの定義

accounts/models.py

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    email = models.EmailField(max_length=254)
```

※AbstractUserを継承する理由…後からユーザーモデルを拡張するときに便利(なぜ便利かは後で調べる)。プロフィール機能の課題があった時の名残。

### 2. adminページに登録(これは後でも大丈夫)

accounts/admin.py

```python
from django.contrib import admin
from .models import User

# Register your models here.
admin.site.register(User)
```

これで[http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/)でユーザーモデルの管理を行うことができる。

### 3. AUTH_USER_MODELの設定

mysite/settings.py

```python
AUTH_USER_MODEL = "accounts.User"
```

### 4. migrationして終わり

## ユーザー登録ページの作成

### 1. urlの設定

(1)クラスベースビューでやった場合(推奨)

accounts/urls.py

```python
from django.contrib.auth import views as auth_views
from django.urls import path, include

from . import views

app_name = "accounts"
urlpatterns = [
    path('signup/', views.SignUpView.as_view(), name='signup'),
    # path('login/', auth_views.LoginView.as_view(), name='login'),
    # path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    # path('<str:username>/', views.UserProfileView.as_view(), name='user_profile'),
    # path('<str:username>/follow/', views.FollowView.as_view(), name='follow'),
    # path('<str:username>/unfollow/', views.UnFollowView, name='unfollow'),
    # path('<str:username>/following_list/', views.FollowingListView.as_view(), name='following_list'),
    # path('<str:username>/follower_list/', views.FollowerListView.as_view(), name='follower_list'),
]
```

※as_view()とは…

### 2. Viewの定義

(1)クラスベースビューでやった場合(推奨)

accounts/views.py

```python
from django.shortcuts import render
from django.views.generic import CreateView
# Create your views here.
from .models import User

class SignUpView(CreateView):
    template_name = "accounts/signup.html"
		form_class = SignUpForm
		success_url = reverse_lazy("tweets:home")
```

### 3. Formの定義

```python
from django.contrib.auth.forms import UserCreationForm
from .models import User

class SignUpForm(UserCreationForm):
    class Meta:
        model = User
        fields = (
            "username",
            "email",
        )
```

ここのMetaは何？(後で調べる)

### 4. Templateの作成

templates/accounts/signup.html

```html
{% extends "base.html" %}

{% block content %}
<div>
    <h2>ユーザー登録</h2>
    <div>
        <form method="post">
            {% csrf_token %}
            {{ form.as_p }}
            <button type="submit">登録</button>
        </form>
    </div>
</div>
{% endblock %}
```

## ユーザー登録と同時にログインするには

### form_valid()メソッドをオーバーライドする

accounts/views.py

```python
class SignUpView(CreateView):
    template_name = "accounts/signup.html"
    form_class = SignUpForm
    success_url = reverse_lazy("tweets:home")

    def form_valid(self, form):
        response = super().form_valid(form)
        username = form.cleaned_data["username"]
        password = form.cleaned_data["password1"]
        user = authenticate(self.request, username=username, password=password)
        login(self.request, user)
        return response
```

### response = super.form_valid(form)とは？

### authenticate(self.request, username=username, password=password)は何?

ユーザーのバリデーションチェックを行っている

ユーザーが存在しない場合、以降のコードをスキップする

```python
@sensitive_variables("credentials")
def authenticate(request=None, **credentials):
    """
    If the given credentials are valid, return a User object.
    """
    for backend, backend_path in _get_backends(return_tuples=True):
        backend_signature = inspect.signature(backend.authenticate)
        try:
            backend_signature.bind(request, **credentials)
        except TypeError:
            # This backend doesn't accept these credentials as arguments. Try
            # the next one.
            continue
        try:
            user = backend.authenticate(request, **credentials)
        except PermissionDenied:
            # This backend says to stop in our tracks - this user should not be
            # allowed in at all.
            break
        if user is None:
            continue
        # Annotate the user object with the path of the backend.
        user.backend = backend_path
        return user

    # The credentials supplied are invalid to all backends, fire signal
    user_login_failed.send(
        sender=__name__, credentials=_clean_credentials(credentials), request=request
    )
```

### login()は何？

```python
def login(request, user, backend=None):
    """
    Persist a user id and a backend in the request. This way a user doesn't
    have to reauthenticate on every request. Note that data set during
    the anonymous session is retained when the user logs in.
    """
    session_auth_hash = ""
    if user is None:
        user = request.user
    if hasattr(user, "get_session_auth_hash"):
        session_auth_hash = user.get_session_auth_hash()

    if SESSION_KEY in request.session:
        if _get_user_session_key(request) != user.pk or (
            session_auth_hash
            and not constant_time_compare(
                request.session.get(HASH_SESSION_KEY, ""), session_auth_hash
            )
        ):
            # To avoid reusing another user's session, create a new, empty
            # session if the existing session corresponds to a different
            # authenticated user.
            request.session.flush()
    else:
        request.session.cycle_key()

    try:
        backend = backend or user.backend
    except AttributeError:
        backends = _get_backends(return_tuples=True)
        if len(backends) == 1:
            _, backend = backends[0]
        else:
            raise ValueError(
                "You have multiple authentication backends configured and "
                "therefore must provide the `backend` argument or set the "
                "`backend` attribute on the user."
            )
    else:
        if not isinstance(backend, str):
            raise TypeError(
                "backend must be a dotted import path string (got %r)." % backend
            )

    request.session[SESSION_KEY] = user._meta.pk.value_to_string(user)
    request.session[BACKEND_SESSION_KEY] = backend
    request.session[HASH_SESSION_KEY] = session_auth_hash
    if hasattr(request, "user"):
        request.user = user
    rotate_token(request)
    user_logged_in.send(sender=user.__class__, request=request, user=user)
```

## テンプレートに変数を表示させる(DTL)

### デフォルトで使える変数

- 「request」…リクエストオブジェクト
- 「user」…サイトにアクセスしているユーザー
- 「perms」…サイトにアクセスしているユーザーのパーミッション
- 「messages」…フラッシュメッセージ

### 具体的には？

```html
ユーザー名：{{ user.username }}
メールアドレス：{{ user.email }}
```

## テンプレートをmanage.pyと同階層にするとき

mysite/settings.py
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"], #ここでtemplatesフォルダの場所を指定
        "APP_DIRS": True, #Trueならアプリケーション直下に置いても持ってきてくれる
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
