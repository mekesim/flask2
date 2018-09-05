虽然将小型 Web 应用程序存储在单个脚本文件中会非常方便，但这种方法无法很好地进行扩展。随着应用程序复杂度的增加，使用单个源文件会出现问题。

与其他 Web 框架不同，Flask 不会为大型项目强加特定组织; 构建应用程序的方式完全由开发人员完成。在本章中，介绍了一种通过包和模块组织大型应用程序的可行方式。

## 项目结构

示例 7-1 展示了一个 Flask 应用的基本布局。

```
# 示例 7-1. 多文件 Flask 应用的基本结构
|-flasky
  |-app/
    |-templates/
    |-static/
    |-main/
      |-__init__.py
      |-errors.py
      |-forms.py
      |-views.py
    |-__init__.py
    |-email.py
    |-models.py
  |-migrations/
  |-tests/
    |-__init__.py
    |-test*.py
  |-venv/
  |-requirements.txt
  |-config.py
  |-flasky.py
```

这个结构有4个顶层目录：

- Flask 应用程序位于一个名为 `app` 的程序包内。
- 像以前一样，`migrations` 文件夹包含数据库迁移脚本。
- 单元测试写在 `tests` 目录里。
- 像以前一样，Python 虚拟环境存放在 `venv` 目录里。

这个结构还有一些新文件：

- `requirements.txt` 列出了该应用依赖的包。通过它可以很方便的在不同电脑上安装应用的依赖包。
- `config.py` 存储了配置信息。
- `flasky.py` 定义了 Flask 应用程序对象

为了帮助你完全理解这个结构，以下部分描述了将 `hello.py` 应用程序转换为这个结构的过程。

## 配置选项

应用程序通常需要几个配置集。最好的例子就是，要为开发环境、测试环境和生产环境配置不同的数据库连接。可以使用配置类的层次结构，而不是像 `hello.py` 使用的简单的 `app.config`字典式配置。示例7-2 展示了 `config.py` 文件，它实现了 `hello.py` 里的所有配置。

```
# 示例 7-2. config.py: 应用配置
import os

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'hard to guess string'
    MAIL_SERVER = os.environ.get('MAIL_SERVER', 'smtp.googlemail.com')
    MAIL_PORT = int(os.environ.get('MAIL_PORT', '587'))
    MAIL_USE_TLS = os.environ.get('MAIL_USE_TLS', 'true').lower() in \
        ['true', 'on', '1']
    MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
    MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')
    FLASKY_MAIL_SUBJECT_PREFIX = '[Flasky]'
    FLASKY_MAIL_SENDER = 'Flasky Admin <flasky@example.com>
    FLASKY_ADMIN = os.environ.get('FLASKY_ADMIN')
    SQLALCHEMY_TRACK_MODIFICATIONS = False

    @staticmethod
    def init_app(app):
        pass

class DevelopmentConfig(Config):
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = os.environ.get('DEV_DATABASE_URL') or \
        'postgresql://wrdll:wrdll.com@localhost/wrdll_dev'

class TestingConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = os.environ.get('TEST_DATABASE_URL') or \
        'postgresql://wrdll:wrdll.com@localhost/wrdll_testing'

class ProductionConfig(Config):
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
        'postgresql://wrdll:wrdll.com@localhost/wrdll'

config = {
    'development': DevelopmentConfig,
    'testing': TestingConfig,
    'production': ProductionConfig,
    'default': DevelopmentConfig,
}
```

> 注意：和原书代码不同，本站将数据库由 SQLite 换成了 PostgreSQL。

`Config` 基类定义了所有环境所需的通用配置。各子类定义了不同环境的配置，可以按照实际需求增加或删除子类。

为了使应用更具灵活性和安全性，有些配置项是从环境变量读取的。比如，`SECRET_KEY`。`SQLALCHEMY_DATABASE_URI` 变量的值则是根据不同的环境，配置成了不同的值。这非常重要，因为没人会希望在做单元测试的时候，对实际数据进行操作。

> 【wrdll 注】另一个原因是，开发环境一般是在本地电脑上，而生产环境是在远程服务器上。这两个环境的数据库连接大部分情况下是不同的。

为了让应用程序可以定制其配置，`Config` 及其子类可以定义一个接收应用对象作为参数的 `init_app` 方法。现在，`Config` 类实现了一个空的 `init_app` 方法。

在这个配置文件的底部，将不同的配置类放到了一个名为 `config` 的字典里，并将其中一个配置类(此例为 `DevelopmentConfig` 类)注册为了 `default`。

## 应用程序包

应用程序包是所有应用程序代码、模板和静态文件存放的地方。在这里，把它简单的命名为 `app`，当然，你也可以使用其它名字。`templates` 和 `static` 目录，现在放到了应用程序包（`app` 目录）内。数据库模型和电子邮件支持函数也都移到了应用程序包里，所以它们的路径分别变成了 `app/models.py` 和 `app/email.py`。

### 使用应用程序工厂

在单文件版本中创建应用程序的方式非常方便，但它有一个很大的缺点。由于应用程序是在全局范围内创建的，因此无法动态更改应用配置：在脚本运行时，应用程序实例已经创建完毕，因此对配置进行更改已经太晚了。动态更改应用配置对于单元测试尤为重要，因为有时需要在不同的配置下运行应用程序以获得更好的测试覆盖率。

此问题的解决方案是，将应用程序移到可以从脚本显式调用的`工厂函数(factory function)`中来延迟创建应用程序。这不仅使脚本有时间来设置配置，而且还能创建多个应用程序实例——另一件在测试过程中可能非常有用的东西。示例7-3中显示的应用程序工厂函数是在应用程序包构造器（`app/__init__.py`）中定义的。

```
# 示例 7-3. app/__init__.py: 应用程序包构造器
from flask import Flask, render_template
from flask_bootstrap import Bootstrap
from flask_mail import Mail
from flask_moment import Moment
from flask_sqlalchemy import SQLAlchemy
from config import config

bootstrap = Bootstrap()
mail = Mail()
moment = Moment()
db = SQLAlchemy()

def create_app(config_name):
    app = Flask(__name__)
    app.config.from_object(config[config_name])
    config[config_name].init_app(app)

    bootstrap.init_app(app)
    mail.init_app(app)
    moment.init_app(app)
    db.init_app(app)

    # 将路由和自定义错误页面放在这里

    return app
```

这构造器将当前使用的一些 Flask 扩展导入进来了，但是由于此时并没有应用程序对象，所以它们都没有初始化，而只是调用了它们各自不带参数的构造函数来实例化。我们需要在工厂函数里将它们进行初始化。`create_app()` 就是创建应用程序对象的工厂函数，这个函数需要一个参数，这个参数是在 `config.py` 定义的配置集的名称。通过 `app.config.from_object()` 方法，可以将对应的配置类中定义的类属性作为 Flask 的配置项。配置类是通过 `config.py` 中的 `config` 字典来获取的。一旦应用程序对象创建并配置完成，就可以调用每个扩展的 `init_app()` 方法来初始化这些扩展了。

### 使用蓝图(Blueprint)实现应用程序功能

转换到工厂函数模式后，路由的实现将会变得更复杂一些。在单文件应用中，由于应用对象 `app` 是一个全局变量，所以我们可以很容易地使用 `app.route` 装饰器来实现路由。但现在，应用对象是通过工厂函数 `create_app()` 在程序运行期间动态创建的，我们无法在应用程序对象创建之前，使用`app.route`来实现路由；同样的，使用 `app.errorhandler` 装饰器创建的自定义错误处理也存在这个问题。

幸运的是，通过 Flask 提供的 `蓝图(blueprint)` 可以解决这个问题。蓝图与应用程序类似，它也可以定义路由和错误处理程序。不同之处在于，当定义蓝图时，它们处于休眠状态，直到应用程序注册蓝图后，它们才会成为应用程序的一部分。使用全局作用域中定义的蓝图，应用程序的路由和错误处理程序的定义方式几乎与单脚本应用程序中的相同。

和应用程序一样，蓝图可以定义在单个文件中，也可以在多个文件中定义。为了实现最大的灵活性，将蓝图作为应用程序包内的子包。示例7-4展示了蓝图包的构造器，用于创建第一个蓝图。

```
# 示例 7-4. app/main/__init__.py: 创建 main 蓝图
from flask import Blueprint

main = Blueprint('main', __name__)

from . import views, errors
```

蓝图是 `Blueprint` 类的一个实例。这个类的构造函数有两个必填参数：蓝图的名称以及蓝图所在包或模块的位置。和应用程序对象一样，Python 的 `__name__` 可以为第二个参数提供正确的值。

路由保存在这个子包里的 `views` 模块中（`app/main/views.py`），错误处理程序则放在 `errors` 模块里（`app/main/errors.py`）。我们在蓝图创建后，将这两个模块进行了导入。注意，导入这两个模块的操作，必须放在最后进行，以避免发生循环导入的错误。原因在于，`views` 和 `errors` 两个模块也会导入 `main` 蓝图。

> `from . import <some-module>` 语法是 Python 的`相对导入`，`.` 用于在当前包中导入模块。与此类似的还有 `from .. import <some-module>` 语法，其中的 `..` 用于在上级包中导入模块。——类似于 Linux/Mac OS 的相对路径概念

在工厂函数 `create_app()` 中，将蓝图注册到应用程序中，示例7-5展示了此操作。

```
# 示例 7-5. app/__init__.py: 注册 main 蓝图
def create_app(config_name):
    # ...

    from .main import main as main_blueprint
    app.register_blueprint(main_blueprint)

    return app
```

示例7-6展示了错误处理函数。

```
# 示例 7-6. app/main/errors.py:  main 蓝图中的错误处理
from flask import render_template
from . import main

@main.app_errorhandler(404)
def page_not_found(e):
    return render_template('404.html'), 404

@main.app_errorhandler(500)
def internal_server_error(e):
    return render_template('500.html'), 500
```

注意，如果要在蓝图中定义应用级别的错误处理，需要使用 `蓝图.app_errorhandler` 装饰器，如果使用的是 `蓝图.errorhandler` 装饰器，定义是蓝图的级别的错误处理。

示例7-7显示了在蓝图中定义路由。

```
# Example 7-7. app/main/views.py: main 蓝图中定义的路由
from datetime import datetime
from flask import render_template, session, redirect, url_for
from . import main
from .forms import NameForm
from .. import db
from ..models import User

@main.route('/', methods=['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        # ...
        return redirect(url_for('.index'))
    return render_template('index.html',
                           form=form, name=session.get('name'),
                           known=session.get('known', False),
                           current_time=datetime.utcnow())
```

在蓝图中编写视图函数有两个不同之处：一是使用蓝图的装饰器来代替应用程序对象的装饰器，即，使用 `main.route` 来代替 `app.route`； 二是 `url_for()` 函数。该函数的第一个参数是视图函数的名称，默认是指应用程序定义的视图函数，而我们现在使用的是蓝图，所以需要指定蓝图名：`main.index`表示`main`蓝图下的`index()`函数。如果是当前蓝图下的视图函数，可以简单写为 `.视图函数`，所以当前蓝图下的 `index()`，可以简写为 `.index`。

为了完成项目结构的调整，表单类也需要移动到蓝图包中，保存到 `app/main/forms.py` 模块里。

## 应用程序脚本

应用程序对象的实例定义在顶层的 `flasky.py` 模块中。如示例7-8所示：

```
# 示例 7-8. flasky.py: 主入口脚本
import os
from app import create_app, db
from app.models import User, Role
from flask_migrate import Migrate

app = create_app(os.getenv('FLASK_CONFIG') or 'default')
migrate = Migrate(app, db)

@app.shell_context_processor
def make_shell_context():
    return dict(db=db, User=User, Role=Role)
```

这个脚本从创建一个应用程序对象开始。应用所需的配置由环境变量 `FLASK_CONFIG` 定义，如果没有定义这个环境变量，则使用 `default` 配置。之后，初始化 Flask-Migrate 扩展和 Shell 上下文。

由于主入口文件由 `hello.py` 变成了 `flasky.py`，所以需要在运行 flask 命令之前更新环境变量 `FLASK_APP` 的值。同时，我们开启 Flask 的调试模式，在 Linux/Mac OS 中，使用以下命令：

```
(venv) $ export FLASK_APP=flasky.py
(venv) $ export FLASK_DEBUG=1
```

在 Windows 中，使用以下命令：

```
(venv) $ set FLASK_APP=flasky.py
(venv) $ set FLASK_DEBUG=1
```

## requirements 文件

每个应用包含一个 `requirements.txt` 文件是一个很好的开发实践。该文件记录了应用所依赖的扩展包的名字和对应的版本。当在另一台电脑或在生产环境部署该应用时，可以快速搭建起所需要的 Python 环境。这个文件可以由 `pip freeze` 命令生成：

```
(venv) $ pip freeze > requirements.txt
```

当安装了新的依赖包或更新了依赖包时，请重新生成该文件。在需要搭建相同环境时，只需要在目标电脑上运行如下命令即可：

```
(venv) $ pip freeze -r requirements.txt
```

## 单元测试

这个应用非常小，所以没什么好测试的。但为了举例，示例7-9定义了两个简单的测试。

```
# 示例 7-9. tests/test_basics.py: 单元测试
import unittest
from flask import current_app
from app import create_app, db

class BasicsTestCase(unittest.TestCase):
    def setUp(self):
        self.app = create_app('testing')
        self.app_context = self.app.app_context()
        self.app_context.push()
        db.create_all()

    def tearDown(self):
        db.session.remove()
        db.drop_all()
        self.app_context.pop()

    def test_app_exists(self):
        self.assertFalse(current_app is None)

    def test_app_is_testing(self):
        self.assertTrue(current_app.config['TESTING'])
```

这个测试用例基于 Python 标准库里的 `unittest` 包编写。测试用例中的每个测试运行之前，都会运行 `setUp()` 方法；每个测试运行之后，都会运行 `tearDown()`方法。每个测试方法都以 `test_` 为前缀进行命名。

本例中的 `setUp()` 方法用来创建一个 Flask 应用程序对象，并为其指定使用了 `testing` 配置，之后激活应用程序上下文。然后调用了 Flask-SQLAlcemy 的 `create_all()` 方法来创建数据表。`tearDown()` 方法则将数据表和应用程序上下文进行移除。第一个测试 `test_app_exists()` 确保应用程序对象是存在的；第二个测试 `test_app_is_testing()` 确保应用程序使用的是 `testing` 配置。

为了让 `tests` 目录成为一个 Python 包，需要添加 `tests/__init__.py` 文件，哪怕该文件是空的。单元测试会在 `tests` 里扫描所有模块。

> 如果你从 Github 克隆了示例代码，可以使用 `git checkout 7a` 签出本示例。为了确保你的虚拟环境里安装了所有依赖包，请执行 `pip install -r requirements.txt` 命令。

可以在 `flasky.py` 里添加一个自定义命令来运行单元测试。示例7-10演示了如何添加一个自定义的 `test` 命令。

```
# 示例 7-10. flasky.py: 单元测试启动命令
@app.cli.command()
def test():
    """运行单元测试."""
    import unittest
    tests = unittest.TestLoader().discover('tests')
    unittest.TextTestRunner(verbosity=2).run(tests)
```

`app.cli.command` 装饰器让自定义命令变得非常简单。这个被装饰的函数的名字就是命令的名字（本例为 `test`），同时这个函数的文档字符，会成为该命令的帮助信息。

现在可以通过 `flask test` 命令来执行测试了：

```
(venv) $ flask test
test_app_exists (test_basics.BasicsTestCase) ... ok
test_app_is_testing (test_basics.BasicsTestCase) ... ok

.----------------------------------------------------------------------
Ran 2 tests in 0.001s

OK
----
```

## 设置数据库

重构之后的项目，优先从环境变量里读取数据库配置，如果没有配置环境变量，刚使用编码里的数据库配置。在开发环境，该环境变量为 `DEV_DATABASE_URL`，如果没有配置该环境变量，则使用本地服务器中的 `wrdll_testing` 数据库（ `postgresql://wrdll:wrdll.com@localhost/wrdll_testing`）。在测试或生产环境，则会使用不同的数据库。

无论是使用哪个数据库，通过 Flask-Migrate，只需要一条命令就可以将当前设置的数据库更新到最新状态：

```
(venv) $ flask db upgrade
```

## 运行应用程序

现在，重构已完成，可以启动应用程序了。请确保已更新了 `FLASK_APP` 环境变量，然后执行以下命令启动应用程序：

```
(venv) $ flask run
```

在每个 Shell 会话中都设置一遍 `FLASK_APP` 和 `FLASK_DEBUG` 环境变量，是一件非常乏味的事。如果使用的是 bash，可以将其写入 `~/.bashrc` 文件中，一劳永逸。

无论你是否相信，至此，你已经学完了第一部分。通过这一部分的内容，你学会了开发 Flask 应用所需的基本知识。但你可能不确定这些部分如何组合在一起形成一个真正的应用程序。第二部分（原书第8章-第14章）的目标是帮助你完成一个完整的应用程序的开发。