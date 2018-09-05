数据库以有组织的方式存储应用程序数据。 之后，应用程序可以在需要时发出查询以检索数据的特定部分。很多应用都在使用关系型数据库，这种数据库使用 SQL 来操作数据。但是，近几年来，称为 NoSQL 的、基于文档或键值对的非关系型数据库也流行起来。

## 关系型数据库

关系数据库将数据存储在数据表中，数据表对应用程序中的各个实体进行建模。比如，一个订单管理应用中的数据库，应该会有 `customers(客户)`、`orders(订单)`和 `products(产品)`等数据表。

一个数据表拥有固定数量的字段和数量不定的行。字段用于描述数据的属性。比如，客户表应该有类似 `name(姓名)`、`address(地址)`和 `phone(电话)`等字段。表中的每一行都定义了一个实际的数据元素，它的值与字段一一对应。

数据表还可能有一个特殊的字段：`主键(primary key)`，用于给存储在表中的每一行数据做唯一标识。数据表还可能有`外键(foreign key)`字段，它引用本表或其它表的主键。这种连接被称为`关系(relationship)`

## NoSQL 数据库

没有遵循上一节中描述的关系模型的数据库统称为 NoSQL 数据库。一种常见的 NoSQL 数据库的组织模型是：使用`集合(collections)` 代表数据表，使用`文档(documents)`代表记录。

## Python 数据库框架

Python 对绝大部分数据库引擎提供了支持，包括开源和商业版本。Flask 没有限定使用哪种数据库，所以可以在 Flask 使用 PostgreSQL、MySQL、SQLite、Redis、MongoDB、CouchDB 或你喜欢的任何数据库。

我们将使用 Flask-SQLAlchemy 作为教程的数据库框架。它是一个对 SQLAlchemy 进行包装的 Flask 扩展。

## 使用 Flask-SQLAlchemy 管理数据库

使用以下命令安装该扩展及其所依赖的其它包：

```
(venv) $ pip install flask-sqlalcemy
```

在 Flask-SQLAlcemy 中，通过 URL 的形式来指定数据库。下表列示了常用的三种数据库的 URL。

| 数据库引擎           | URL                                                |
| -------------------- | -------------------------------------------------- |
| PostgreSQL           | `postgresql://username:password@hostname/database` |
| MySQL                | `mysql://username:password@hostname/database`      |
| SQLite(Linux/Mac OS) | `sqlite:////abspath/to/database`                   |
| SQLite(Windows)      | `sqlite:///c:/abspath/to/database`                 |

数据库的 URL 必须在 Flask 应用中，使用 `SQLALCHEMY_DATABASE_URI`来进行配置。Flask-SQLAlchemy 官方还建议，除非需要更改对象信号，否则请将 `SQLALCHEMY_TRACK_MODIFICATIONS` 设置为 `False`，以减少内存的占用。

> 【wrdll 注】SQLALCHEMY 是一个抽象包，它需要数据库的驱动程序来连接数据库。Python 内置了 SQLite 数据库的驱动，而其它数据库的驱动需要单独安装。下表列示了 PostgreSQL 和 MySQL 常用的驱动包。

| 数据库引擎 | 驱动                                                         |
| ---------- | ------------------------------------------------------------ |
| PostgreSQL | psycopg2-binary(新版需要使用该包，而不是旧版的 psycopg2 包，两者API完全一致) |
| MySQL      | pymysql(URL最好改为 `mysql+pymysql://username:password@hostname/database`) |

```
# 示例 5-1. hello.py: 配置数据库
import os
from flask_sqlalchemy import SQLAlchemy

basedir = os.path.abspath(os.path.dirname(__file__))

app = Flask(__name__)
# 使用 PostgreSQL。
# 用户名为 wrdll，密码为 wrdll.com，数据库服务器为 localhost，数据库名为 wrdll
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://wrdll:wrdll.com@localhost/wrdll'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)
```

> 原书使用的是 SQLite 数据库。而本站及本站稍后推出的视频教程都将使用 PostgreSQL。有关 PostgreSQL 的教程，请移步[PostgreSQL 轻松学](https://pg.sjk66.com/)。

## 模型定义

定义的模型必须继承自 `db.Model` 基类。

```python
# 示例 5-2. hello.py: 角色和用户的模型定义
class Role(db.Model):
    """角色模型"""
    __tablename__ = 'roles'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(64), unique=True)

    def __repr__(self):
        return '<Role %r>' % self.name

class User(db.Model):
    """用户模型"""
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, index=True)

    def __repr__(self):
        return '<User %r>' % self.username
```

`__tablename__` 定义了该模型在数据库中的表名，如果省略了 `__tablename__`，Flask-SQLAlchemy 将为该表生成默认的名字。其它的类变量定义了模型的属性，它们是 `db.Column` 的实例。`db.Column` 的第一个参数是数据类型。下表列示了常用的数据类型。

| 类型           | PYTHON 类型          | 说明                        |
| -------------- | -------------------- | --------------------------- |
| `Integer`      | `int`                | 常规整数，通常是32位        |
| `SmallInteger` | `int`                | 小整数，通常是16位          |
| `BigInteger`   | `int`或`long`        | 不限大小的整数              |
| `Float`        | `float`              | 浮点数                      |
| `Numeric`      | `decimal.Decimal`    | 实数                        |
| `String`       | `str`                | 变长字符串                  |
| `Text`         | `str`                | 不限长度的变长字符串        |
| `Unicode`      | `unicode`            | 变长Unicode字符串           |
| `UnicodeText`  | `unicode`            | 不限长度的变长Unicode字符串 |
| `Boolean`      | `bool`               | 布尔值                      |
| `Date`         | `datetime.date`      | 日期                        |
| `Time`         | `datetime.time`      | 时间                        |
| `DateTime`     | `datetime.datetime`  | 日期时间                    |
| `Interval`     | `datetime.timedelta` | 时间间隔                    |
| `Enum`         | `str`                | 枚举                        |
| `PickleType`   | 任何 Python 对象     | 自动序列化                  |
| `LargeBinary`  | `str`                | 二进制                      |

`db.Column` 还可以添加一些额外的属性。下表对一些常用的额外属性进行总结。

| 名称          | 说明                                            |
| ------------- | ----------------------------------------------- |
| `primary_key` | 如果设置为 `True`，那么该字段将成为表的主键     |
| `unique`      | 如果设置为`True`，那么该字段的值是唯一的        |
| `index`       | 如果设置为`True`，将为该字段创建索引            |
| `nullable`    | 如果设置为`False`，该字段的值不允许为空(`NULL`) |
| `default`     | 为字段设定默认值                                |

> 注意，Flask-SQLAlchemy 要求每个模型都有一个主键字段。主键字段通常使用 `id` 命名

## 关系

示例5-3展示如何为模型定义一对多关系。

```python
# 示例 5-3. hello.py: 数据库模型的关系
class Role(db.Model):
    # ...
    users = db.relationship('User', backref='role')

class User(db.Model):
    # ...
    role_id = db.Column(db.Integer, db.ForeignKey('roles.id'))
```

`db.ForeignKey` 用于定义外键，`db.relationship` 用于定义关系。`db.relationship` 中的 `backref` 是反向引用的意思。

【wrdll 注】外键是给数据库看的，关系是给 Python 看的。即是说，`db.ForeignKey` 引用的字段将在数据库中呈现，而 `db.relationship` 定义的关系，只在 Python 中可见。

> 如果你从 Github 克隆了示例代码，可以使用 `git checkout 5a` 签出本示例。

## 数据库操作

数据模型定义好之后，可以在 `flask shell` 中进行使用。使用该命令之前，请确认已将环境变量 `FLASK_APP` 设置成了 `hello.py`。

### 创建表

Flask-SQLAlchemy 提供了 `db.create_all()` 方法来创建所有实现了 `db.Model` 的模型类定义的数据表。

```
(venv) $ flask shell
>>> from hello import db
>>> db.create_all()
```

执行该命令后，Flask-SQLAlchemy 会在数据库中创建 `roles`(对应`Role`模型) 和`users` (对应`User`模型)表。

`db.drop_all()` 方法用来删除数据库里已存在的（与 Model 相关的）的数据表。所以，可以在创建表之前，先进行删除操作。

```
>>> db.drop_all()
>>> db.create_all()
```

### 插入数据

下面的例子创建了一些角色和用户数据。

```
>>> from hello import Role, User
>>> admin_role = Role(name='Admin')
>>> mod_role = Role(name='Moderator')
>>> user_role = Role(name='User')
>>> user_john = User(username='john', role=admin_role)
>>> user_susan = User(username='susan', role=user_role)
>>> user_david = User(username='david', role=user_role)
```

上面的代码只是创建了模型的几个实例，并没有保存到数据库。而数据库中，主键`id`的值是由数据库来自动生成的，所以，此时试图获取这些实例的 `id`，是获取不到的。

```
>>> print(admin_role.id)
None
>>> print(mod_role.id)
None
>>> print(user_role.id)
None
```

需要把这些实例保存到数据库中，才能获取到它们的`id`。保存到数据库中，使用 `db.session.add()` 或 `db.session.add_all()` 方法，然后调用 `db.session.commit()`。

逐个添加：

```
>>> db.session.add(admin_role)
>>> db.session.add(mod_role)
>>> db.session.add(user_role)
>>> db.session.add(user_john)
>>> db.session.add(user_susan)
>>> db.session.add(user_david)
```

批量添加：

```
>>> db.session.add_all([admin_role, mod_role, user_role, user_john, user_susan, user_david])
```

然后调用 `db.session.commit()` 写入数据库：

```
>>> db.session.commit()
```

此时，再打印它们的 `id`：

```
>>> print(admin_role.id)
1
>>> print(mod_role.id)
2
>>> print(user_role.id)
3
```

### 修改数据

```
>>> admin_role.name = 'Administrator'
>>> db.session.add(admin_role)
>>> db.session.commit()
```

### 删除数据

```
>>> db.session.delete(mod_role)
>>> db.session.commit()
```

### 查询数据

查询所有数据：

```
>>> Role.query.all()
[<Role 'Administrator'>, <Role 'User'>]
>>> User.query.all()
[<User 'john'>, <User 'susan'>, <User 'david'>]
```

查询指定数据：

```
>>> User.query.filter_by(role=user_role).all()
[<User 'susan'>, <User 'david'>]
```

获取对应的 SQL 语句：

```
>>> str(User.query.filter_by(role=user_role))
SELECT users.id AS users_id, users.username AS users_username,
users.role_id AS users_role_id \nFROM users \nWHERE :param_1 = users.role_id
```

获取单条记录：

```
>>> user_role = Role.query.filter_by(name='User').first()
```

Flask-SQLAlchemy 常用的过滤器：

| 过滤器        | 说明                                             |
| ------------- | ------------------------------------------------ |
| `filter()`    | 给原始查询添加过滤条件，并返回一个新的查询       |
| `filter_by()` | 给原始查询添加“相等”过滤条件，并返回一个新的查询 |
| `limit()`     | 限制原始查询返回的条数，并返回一个新的查询       |
| `offset()`    | 设置原始查询的偏移量，并返回一个新的查询         |
| `order_by()`  | 给原始查询添加排序条件，并返回一个新的查询       |
| `group_by()`  | 给原始查询添加分组条件，并返回一个新的查询       |

Flask-SQLAlchemy 查询常用的执行操作：

| 执行             | 说明                                            |
| ---------------- | ----------------------------------------------- |
| `all()`          | 获取所有数据                                    |
| `first()`        | 获取第一条数据                                  |
| `first_or_404()` | 获取第一条数据，如果没有数据，抛出404错误       |
| `get()`          | 通过主键获取一条数据                            |
| `get_or_404()`   | 通过主键获取一条数据，如果没有数据，抛出404错误 |
| `count()`        | 获取查询中的数据的条数                          |
| `paginate()`     | 对查询的数据进行分页                            |

在查询中，可以很方便的使用关系。

```
#  获取角色中的所有用户。一对多关系
>>> users = user_role.users
>>> users
[<User 'susan'>, <User 'david'>]
>>> users[0].role
<Role 'User'>
```

如果在定义关系的时候，传递 `lazy="dynamic"` 参数，那么关联的数据不会自动执行，而是返回一个查询对象。

```
# 示例 5-4. hello.py: 使用 dynamic 定义数据库关系
class Role(db.Model):
    # ...
    users = db.relationship('User', backref='role', lazy='dynamic')
    # ...
```

按这种方式定义的关系，返回的是一个查询，而不是数据集。需要手动调用一次查询的执行操作，如 `all()`：

```
>>> user_role.users.order_by(User.username).all()
[<User 'david'>, <User 'susan'>]
>>> user_role.users.count()
2
```

## 在视图函数中使用数据库

```python
# 示例 5-5. hello.py: 在视图函数中使用数据库
@app.route('/', methods=['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.name.data).first()
        if user is None:
            user = User(username=form.name.data)
            db.session.add(user)
            db.session.commit()
            session['known'] = False
        else:
            session['known'] = True
        session['name'] = form.name.data
        form.name.data = ''
        return redirect(url_for('index'))
    return render_template('index.html', form=form, name=session.get('name'), known=session.get('known', False))
```

对应的模板文件：

```html
<!-- 示例 5-6. templates/index.html: 在模板中自定义问候信息 -->
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

{% block title %}Flasky{% endblock %}

{% block page_content %}
<div class="page-header">
    <h1>你好, {% if name %}{{ name }}{% else %}陌生人{% endif %}!</h1>
    {% if not known %}
    <p>幸会!</p>
    {% else %}
    <p>很高兴再次见到你!</p>
    {% endif %}
</div>
{{ wtf.quick_form(form) }}
{% endblock %}
```

> 如果你从 Github 克隆了示例代码，可以使用 `git checkout 5b` 签出本示例。

## (将db和模型)集成到 Python Shell

```python
# 示例 5-7. hello.py: 添加 shell 上下文
@app.shell_context_processor
def make_shell_context():
    return dict(db=db, User=User, Role=Role)
```

现在，可以在 shell 中直接使用 `db` 和 `User`、`Role` 模型了，而不需要手动 import：

```
$ flask shell
>>> app
<Flask 'hello'>
>>> db
<SQLAlchemy engine='postgresql://wrdll:wrdll.com@localhost/wrdll'>
>>> User
<class 'hello.User'>
```

> 如果你从 Github 克隆了示例代码，可以使用 `git checkout 5c` 签出本示例。

## 使用 Flask-Migrate 迁移数据库

Flask-SQLAlchemy 只在数据表不存在的时候，才会创建数据表（`db.create_all()`）。如果是对一个已存在的模型进行修改，然后再调用`db.create_all()`，是没有任何效果的。此时只能先调用 `db.drop_all()`，然后再调用 `db._create_all()`。如果数据表中已经有记录了，那么这种操作将导致数据丢失。

Flask-Migrate 扩展可以在已存在的数据表发生改变时，不删除该表，而对它进行结构修改。这样既能将模型的改变体现到数据表中，又能保留数据表中已存在的数据。

首先，需要使用 pip 安装该扩展：

```
(venv) $ pip install flask-migrate
```

然后，对它进行初始化：

```
from flask_migrate import Migrate
# ...
migrate = Migrate(app, db)
```

### 初始化迁移仓库

使用 `flask db init` 命令，来初始化数据迁移仓库：

```
(venv) $ flask db init
  Creating directory /home/flask/flasky/migrations...done
  Creating directory /home/flask/flasky/migrations/versions...done
  Generating /home/flask/flasky/migrations/alembic.ini...done
  Generating /home/flask/flasky/migrations/env.py...done
  Generating “/home/flask/flasky/migrations/env.pyc...done
  Generating /home/flask/flasky/migrations/README...done
  Generating /home/flask/flasky/migrations/script.py.mako...done
  Please edit configuration/connection/logging settings in
  '/home/flask/flasky/migrations/alembic.ini' before proceeding.
```

这个命令会创建 `migrations` 目录，该目录就是数据迁移的仓库。

### 创建迁移脚本

使用 `flask db migrate` 创建迁移脚本。

```
(venv) $ flask db migrate -m "initial migration"
INFO  [alembic.migration] Context impl SQLiteImpl.
INFO  [alembic.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate] Detected added table 'roles'
INFO  [alembic.autogenerate] Detected added table 'users'
INFO  [alembic.autogenerate.compare] Detected added index
'ix_users_username' on '['username']'
  Generating /home/flask/flasky/migrations/versions/1bc
  594146bb5_initial_migration.py...done
```

### 更新数据库

使用 `flask db upgrade` 更新数据库。

```sql
(venv) $ flask db upgrade
INFO  [alembic.migration] Context impl SQLiteImpl.
INFO  [alembic.migration] Will assume non-transactional DDL.
INFO  [alembic.migration] Running upgrade None -> 1bc594146bb5, initial migration
```