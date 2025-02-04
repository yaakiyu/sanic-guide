# Sanicアプリケーション

## インスタンス化

---:1
最も基本的な構成要素は、`Sanic()`インスタンスです。これは必須ではありませんが、カスタマイズする場合はこれを`server.py`というファイルでインスタンス化します。
:--:1
```python
# /path/to/server.py

from sanic import Sanic

app = Sanic("My Hello, world app")
```
:---

## アプリケーションコンテキスト

ほとんどのアプリケーションでは、コード・ベースのさまざまな部分でデータやオブジェクトを共有/再利用する必要があります。最も一般的な例はDB接続です。

---:1
v21.3より前のバージョンのSanicでは、これは通常、属性をアプリケーションインスタンスにアタッチすることによって行われていました。
:--:1
```python
# 21.3では非推奨として警告を出力する
app = Sanic("MyApp")
app.db = Database()
```
:---

---:1
これにより、名前の競合に関する潜在的な問題が発生したり、[要求コンテキスト](./request.md#context)オブジェクトv 21.3では、アプリケーションレベルのコンテキストオブジェクトが導入されました。
:--:1
```python
# アプリケーションにオブジェクトを添付する正しい方法
app = Sanic("MyApp")
app.ctx.db = Database()
```
:---

## アプリケーションの登録

---:1

Sanicインスタンスをインスタンス化すると、後でSanicアプリケーションレジストリから取得できます。これは、他の方法ではアクセスできない場所からSanicインスタンスにアクセスする必要がある場合などに便利です。
:--:1
```python
# ./path/to/server.py
from sanic import Sanic

app = Sanic("my_awesome_server")

# ./path/to/somewhere_else.py
from sanic import Sanic

app = Sanic.get_app("my_awesome_server")
```
:---

---:1

存在しないアプリケーションで`Sanic.get_app("non-existing")`を呼び出すと、デフォルトで`SanicException'が発生します。代わりに、その名前の新しいSanicインスタンスを返すようにメソッドに強制できます。
:--:1
```python
app = Sanic.get_app(
    "non-existing",
    force_create=True,
)
```
:---

---:1
Sanicインスタンスが**1つしか**登録されていない場合、引数なしで`Sanic.get_app ()`を呼び出すと、そのインスタンスが返されます。
:--:1
```python
Sanic("My only app")

app = Sanic.get_app()
```
:---

## 構成

---:1
Sanicは、`Sanic`インスタンスの`config`属性に設定を保持します。設定は、ドット表記**または**辞書の**どちらかを**使用して変更できます。
---:1
```python
app = Sanic('myapp')

app.config.DB_NAME = 'appdb'
app.config['DB_USER'] = 'appuser'

db_settings = {
    'DB_HOST': 'localhost',
    'DB_NAME': 'appdb',
    'DB_USER': 'appuser'
}
app.config.update(db_settings)
```
:---

::: tip Heads up
構成キーは大文字で _なければなりません_ 。 しかし、これは主に規約によるもので、ほとんどの場合小文字で動作します。
```
app.config.GOOD = "yay!"
app.config.bad = "boo"
```
:::

There is much [more detail about configuration](/guide/deployment/configuration.md) later on.


## カスタマイズ

Sanicアプリケーションインスタンスは、インスタンス化時にさまざまな方法でアプリケーションのニーズに合わせてカスタマイズできます。

### カスタムな構成
---:1

この最も単純なカスタム設定の方法は、独自のオブジェクトを直接Sanicアプリケーションのインスタンスに渡すことです。

カスタム設定オブジェクトを作成する場合は、Sanicの`Config`オプションをサブクラス化して、その動作を継承することを強くお勧めします。 このオプションを使用して、プロパティを追加することも、独自のカスタムロジックセットを追加することもできます。

*v21.6で追加*
:--:1
```python
from sanic.config import Config

class MyConfig(Config):
    FOO = "bar"

app = Sanic(..., config=MyConfig())
```
:---

---:1
この機能の有用な例は、 [supported](../deployment/configuration.md#using-sanic-update-config)とは異なる形式の設定ファイルを使用する場合です。
:--:1
```python
from sanic import Sanic, text
from sanic.config import Config

class TomlConfig(Config):
    def __init__(self, *args, path: str, **kwargs):
        super().__init__(*args, **kwargs)

        with open(path, "r") as f:
            self.apply(toml.load(f))

    def apply(self, config):
        self.update(self._to_uppercase(config))

    def _to_uppercase(self, obj: Dict[str, Any]) -> Dict[str, Any]:
        retval: Dict[str, Any] = {}
        for key, value in obj.items():
            upper_key = key.upper()
            if isinstance(value, list):
                retval[upper_key] = [
                    self._to_uppercase(item) for item in value
                ]
            elif isinstance(value, dict):
                retval[upper_key] = self._to_uppercase(value)
            else:
                retval[upper_key] = value
        return retval

toml_config = TomlConfig(path="/path/to/config.toml")
app = Sanic(toml_config.APP_NAME, config=toml_config)
```
:---
### カスタム コンテキスト
---:1
By default, the application context is a [`SimpleNamespace()`](https://docs.python.org/3/library/types.html#types.SimpleNamespace) that allows you to set any properties you want on it. ただし、代わりに任意のオブジェクトを渡すこともできます。

*Added in v21.6*
:--:1
```python
app = Sanic(..., ctx=1)
```

```python
app = Sanic(..., ctx={})
```

```python
class CustomContext:
    ...

app = Sanic(..., ctx=MyContext())
```
:---
### カスタムリクエスト
---:1
独自の 「Request」 クラスを用意し、デフォルトの代わりにそれを使用するようにSanicに指示すると便利な場合があります。 たとえば、デフォルトの`request.id`ジェネレータを変更する場合です。

::: tip 重要

クラスのインスタンスではなく、*クラス*を渡すことを覚えておくことが重要です。

:::
:--:1
```python
import time

from sanic import Request, Sanic, text


class NanoSecondRequest(Request):
    @classmethod
    def generate_id(*_):
        return time.time_ns()


app = Sanic(..., request_class=NanoSecondRequest)


@app.get("/")
async def handler(request):
    return text(str(request.id))
```
:---

### カスタムエラー処理

---:1
詳細については、[exception handling](../best-practices/exceptions.md#custom-error-handling)を参照してください。
:--:1
```python
from sanic.handlers import ErrorHandler

class CustomErrorHandler(ErrorHandler):
    def default(self, request, exception):
        ''' エラーハンドラが割り当てられていないエラーを処理する '''
        # あなた独自のエラーハンドリングロジック...
        return super().default(request, exception)

app = Sanic(..., error_handler=CustomErrorHandler())
```
:---

### Custom dumps function

---:1 
It may sometimes be necessary or desirable to provide a custom function that serializes an object to JSON data.
:--:1
```python
import ujson

dumps = partial(ujson.dumps, escape_forward_slashes=False)
app = Sanic(__name__, dumps=dumps)
```
:---

---:1
Or, perhaps use another library or create your own.
:--:1
```python
from orjson import dumps

app = Sanic(__name__, dumps=dumps)
```
:---

### Custom loads function

---:1
Similar to `dumps`, you can also provide a custom function for deserializing data.

*Added in v22.9*
:--:1
```python
from orjson import loads

app = Sanic(__name__, loads=loads)
```
:---

::: new NEW in v23.6
### Custom typed application

The correct, default type of a Sanic application instance is:

```python
sanic.app.Sanic[sanic.config.Config, types.SimpleNamespace]
```

It refers to two generic types:

1. The first is the type of the configuration object. It defaults to `sanic.config.Config`, but can be any subclass of that.
2. The second is the type of the application context. It defaults to `types.SimpleNamespace`, but can be **any object** as show above.

Let's look at some examples of how the type will change.

---:1
Consider this example where we pass a custom subclass of `Config` and a custom context object.

:--:1
```python
from sanic import Sanic
from sanic.config import Config

class CustomConfig(Config):
    pass

app = Sanic("test", config=CustomConfig())
reveal_type(app) # N: Revealed type is "sanic.app.Sanic[main.CustomConfig, types.SimpleNamespace]"
```
```
sanic.app.Sanic[main.CustomConfig, types.SimpleNamespace]
```
:---

---:1
Similarly, when passing a custom context object, the type will change to reflect that.
:--:1
```python
from sanic import Sanic

class Foo:
    pass

app = Sanic("test", ctx=Foo())
reveal_type(app)  # N: Revealed type is "sanic.app.Sanic[sanic.config.Config, main.Foo]"
```
```
sanic.app.Sanic[sanic.config.Config, main.Foo]
```
:---

---:1
Of course, you can set both the config and context to custom types.
:--:1
```python
from sanic import Sanic
from sanic.config import Config

class CustomConfig(Config):
    pass

class Foo:
    pass

app = Sanic("test", config=CustomConfig(), ctx=Foo())
reveal_type(app)  # N: Revealed type is "sanic.app.Sanic[main.CustomConfig, main.Foo]"
```
```
sanic.app.Sanic[main.CustomConfig, main.Foo]
```
:---

This pattern is particularly useful if you create a custom type alias for your application instance so that you can use it to annotate listeners and handlers.

```python
# ./path/to/types.py
from sanic.app import Sanic
from sanic.config import Config
from myapp.context import MyContext
from typing import TypeAlias

MyApp = TypeAlias("MyApp", Sanic[Config, MyContext])
```

```python
# ./path/to/listeners.py
from myapp.types import MyApp


def add_listeners(app: MyApp):
    @app.before_server_start
    async def before_server_start(app: MyApp):
        # do something with your fully typed app instance
        await app.ctx.db.connect()
```

```python
# ./path/to/server.py
from myapp.types import MyApp
from myapp.context import MyContext
from myapp.config import MyConfig
from myapp.listeners import add_listeners

app = Sanic("myapp", config=MyConfig(), ctx=MyContext())
add_listeners(app)
```

*Added in v23.6*

### Custom typed request

Sanic also allows you to customize the type of the request object. This is useful if you want to add custom properties to the request object, or be able to access your custom properties of a typed application instance.

The correct, default type of a Sanic request instance is:

```python
sanic.request.Request[
    sanic.app.Sanic[sanic.config.Config, types.SimpleNamespace],
    types.SimpleNamespace
]
```

It refers to two generic types:

1. The first is the type of the application instance. It defaults to `sanic.app.Sanic[sanic.config.Config, types.SimpleNamespace]`, but can be any subclass of that.
2. The second is the type of the request context. It defaults to `types.SimpleNamespace`, but can be **any object** as show above in [custom requests](#custom-requests).

Let's look at some examples of how the type will change.

---:1
Expanding upon the full example above where there is a type alias for a customized application instance, we can also create a custom request type so that we can access those same type annotations.

Of course, you do not need type aliases for this to work. We are only showing them here to cut down on the amount of code shown.
:--:1
```python
from sanic import Request
from myapp.types import MyApp
from types import SimpleNamespace

def add_routes(app: MyApp):
    @app.get("/")
    async def handler(request: Request[MyApp, SimpleNamespace]):
        # do something with your fully typed app instance
        results = await request.app.ctx.db.query("SELECT * FROM foo")
```
:---

---:1
Perhaps you have a custom request object that generates a custom context object. You can type annotate it to properly access those properties with your IDE as shown here.
:--:1
```python
from sanic import Request, Sanic
from sanic.config import Config

class CustomConfig(Config):
    pass

class Foo:
    pass

class RequestContext:
    foo: Foo

class CustomRequest(Request[Sanic[CustomConfig, Foo], RequestContext]):
    @staticmethod
    def make_context() -> RequestContext:
        ctx = RequestContext()
        ctx.foo = Foo()
        return ctx

app = Sanic(
    "test", config=CustomConfig(), ctx=Foo(), request_class=CustomRequest
)

@app.get("/")
async def handler(request: CustomRequest):
    # Full access to typed:
    # - custom application configuration object
    # - custom application context object
    # - custom request context object
    pass
```
:---

See more information in the [custom request context](./request.md#custom-request-context) section.

*Added in v23.6*
:::
