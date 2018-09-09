# 3 Web表单

这是Flask Mega-Tutorial系列的第三部分，其中我将告诉您如何使用*Web表单*。

在[第2章](02 模板.md)中，我为应用程序的主页创建了一个简单的模板，并使用虚假对象作为我尚未拥有的东西的占位符，如用户或博客文章。在本章中，我将讨论我在此应用程序中仍然存在的众多漏洞之一，特别是如何通过Web表单接受用户的输入。

Web表单是任何Web应用程序中最基本的构建块之一。我将使用表单来允许用户提交博客帖子，以及登录应用程序。

在继续本章之前，请确保您已安装上一章中的*微博*应用程序，并且您可以毫无错误地运行它。

## Flask-WTF简介

为了处理这个应用程序中的Web表单，我将使用[Flask-WTF](http://packages.python.org/Flask-WTF)扩展，这是一个围绕[WTForms](https://wtforms.readthedocs.io/)包的薄包装，可以很好地与Flask集成。这是我向你呈现的第一个Flask扩展，但它不会是最后一个。扩展是Flask生态系统中非常重要的一部分，因为它们提供了Flask故意不会发表意见的问题的解决方案。

Flask扩展是随附安装的常规Python包`pip`。您可以继续在虚拟环境中安装Flask-WTF：

```
(venv) $ pip install flask-wtf
```

## 组态

到目前为止，应用程序非常简单，因此我不需要担心它的*配置*。但对于除最简单的应用程序之外的任何应用程序，您将发现Flask（以及可能还有您使用的Flask扩展）在如何执行操作方面提供了一些自由，并且您需要做出一些决定，并将其传递给框架作为配置变量列表。

应用程序有多种格式可指定配置选项。最基本的解决方案是将变量定义为键`app.config`，使用字典样式来处理变量。例如，你可以这样做：

```
app = Flask(__name__)
app.config['SECRET_KEY'] = 'you-will-never-guess'
# ... add more variables here as needed
```

虽然上面的语法足以为Flask创建配置选项，但我喜欢强制执行*关注点分离*的原则，所以不要将我的配置放在我创建应用程序的相同位置，而是使用一个稍微复杂一点的结构来允许我将我的配置保存在单独的文件中。

我非常喜欢的格式，因为它是非常可扩展的，是使用类来存储配置变量。为了保持组织良好，我将在一个单独的Python模块中创建配置类。您可以在下面看到此应用程序的新配置类，它存储在顶级目录的*config.py*模块中。

*config.py*：密钥配置

```
import os

class Config(object):
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'you-will-never-guess'
```

很简单吧？配置设置被定义为类中的类变量`Config`。由于应用程序需要更多配置项，因此可以将它们添加到此类中，稍后如果我发现需要有多个配置集，我可以创建它的子类。但是不要担心这一点。

`SECRET_KEY`我作为唯一配置项添加的配置变量是大多数Flask应用程序中的重要部分。Flask及其一些扩展使用密钥的值作为加密密钥，可用于生成签名或令牌。Flask-WTF扩展使用它来保护Web表单免受称为[跨站请求伪造](http://en.wikipedia.org/wiki/Cross-site_request_forgery)或CSRF（发音为“seasurf”）的令人讨厌的攻击。顾名思义，秘密密钥应该是秘密的，因为使用它生成的令牌和签名的强度取决于应用程序的可信维护者之外的任何人都不知道它。

密钥的值被设置为具有两个术语的表达式，由`or`运算符连接。第一个术语查找环境变量的值，也称为`SECRET_KEY`。第二个术语，只是一个硬编码的字符串。这是一种模式，您会看到我经常重复配置变量。我们的想法是首选来自环境变量的值，但如果环境没有定义变量，则使用硬编码字符串。在开发此应用程序时，安全性要求很低，因此您可以忽略此设置并使用硬编码字符串。但是当这个应用程序部署在生产服务器上时，我将在环境中设置一个独特且难以猜测的值，以便服务器具有其他人不知道的安全密钥。

现在我有一个配置文件，我需要告诉Flask阅读并应用它。这可以在使用以下`app.config.from_object()`方法创建Flask应用程序实例后立即完成：

*app / __ init__.py：Flask*配置

```
from flask import Flask
from config import Config

app = Flask(__name__)
app.config.from_object(Config)

from app import routes
```

我导入`Config`类的方式起初可能看起来很混乱，但是如果你看一下如何`Flask`从`flask`包中导入类（大写“F”）（小写“f”）你会注意到我在做同样的事情配置。小写的“config”是Python模块*config.py*的名称，显然具有大写“C”的那个是实际的类。

如上所述，可以使用字典语法访问配置项`app.config`。在这里，您可以看到使用Python解释器的快速会话，我在其中检查密钥的值是什么：

```
>>> from microblog import app
>>> app.config['SECRET_KEY']
'you-will-never-guess'
```

## 用户登录表

Flask-WTF扩展使用Python类来表示Web表单。表单类只是将表单的字段定义为类变量。

再次考虑到关注点分离，我将使用新的*app / forms.py*模块来存储我的Web表单类。首先，让我们定义一个用户登录表单，要求用户输入用户名和密码。表单还将包含“记住我”复选框和提交按钮：

*app / forms.py*：登录表单

```
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, BooleanField, SubmitField
from wtforms.validators import DataRequired

class LoginForm(FlaskForm):
    username = StringField('Username', validators=[DataRequired()])
    password = PasswordField('Password', validators=[DataRequired()])
    remember_me = BooleanField('Remember Me')
    submit = SubmitField('Sign In')
```

大多数Flask扩展使用`flask_<name>`其顶级导入符号的命名约定。在这种情况下，Flask-WTF的所有符号都在下面`flask_wtf`。这是`FlaskForm`从*app / forms.py*顶部导入基类的*地方*。

表示我用于此表单的字段类型的四个类是直接从WTForms包导入的，因为Flask-WTF扩展不提供自定义版本。对于每个字段，将在类中将对象创建为类变量`LoginForm`。每个字段都有一个描述或标签作为第一个参数。

`validators`您在某些字段中看到的可选参数用于将验证行为附加到字段。该`DataRequired`验证程序只是简单地检查该字段不会提交空。还有更多的验证器，其中一些将以其他形式使用。

## 表单模板

下一步是将表单添加到HTML模板，以便可以在网页上呈现。好消息是，`LoginForm`类中定义的字段知道如何将自己呈现为HTML，因此这项任务非常简单。您可以在下面看到登录模板，我将在文件*app / templates / login.html中*存储该*模板*：

*app / templates / login.html*：登录表单模板

```
{% extends "base.html" %}

{% block content %}
    <h1>Sign In</h1>
    <form action="" method="post" novalidate>
        {{ form.hidden_tag() }}
        <p>
            {{ form.username.label }}<br>
            {{ form.username(size=32) }}
        </p>
        <p>
            {{ form.password.label }}<br>
            {{ form.password(size=32) }}
        </p>
        <p>{{ form.remember_me() }} {{ form.remember_me.label }}</p>
        <p>{{ form.submit() }}</p>
    </form>
{% endblock %}
```

对于这个模板，我再次使用模板继承语句再次使用[第2章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-ii-templates)`base.html`所示的模板。我将使用所有模板实际执行此操作，以确保一致的布局，其中包括应用程序所有页面的顶部导航栏。`extends`

此模板需要将从`LoginForm`类中实例化的表单对象作为参数提供，您可以将其视为参考`form`。这个参数将由登录视图函数发送，我还没写过。

HTML `<form>`元素用作Web表单的容器。`action`表单的属性用于告知浏览器在提交用户在表单中输入的信息时应使用的URL。当操作设置为空字符串时，表单将提交到当前位于地址栏中的URL，该URL是在页面上呈现表单的URL。该`method`属性指定在将表单提交到服务器时应使用的HTTP请求方法。默认情况下是通过`GET`请求发送它，但几乎在所有情况下，使用`POST`请求都可以获得更好的用户体验，因为此类请求可以在请求正文中提交表单数据，`GET`请求将表单字段添加到URL，使浏览器地址栏变得混乱。该`novalidate`属性用于告知Web浏览器不对此表单中的字段应用验证，这有效地将此任务留给服务器中运行的Flask应用程序。使用`novalidate`完全是可选的，但是对于第一种形式，设置它是很重要的，因为这将允许您在本章后面测试服务器端验证。

该`form.hidden_tag()`模板参数生成一个隐藏字段，其中包括用来防止CSRF攻击形式的令牌。要使表单受保护，您需要做的就是包含此隐藏字段并`SECRET_KEY`在Flask配置中定义变量。如果您照顾这两件事，Flask-WTF会为您完成剩下的工作。

如果您以前编写过HTML Web表单，您可能会发现此模板中没有HTML字段很奇怪。这是因为表单对象中的字段知道如何将自己呈现为HTML。我需要做的就是包括`{{ form.<field_name>.label }}`我想要的字段标签，以及`{{ form.<field_name>() }}`我想要的字段。对于需要其他HTML属性的字段，可以将这些属性作为参数传递。此模板中的用户名和密码字段将`size`作为参数添加到`<input>`HTML元素作为属性。这是您还可以将CSS类或ID附加到表单字段的方法。

## 表格视图

在浏览器中看到此表单之前的最后一步是在应用程序中编写一个新的视图函数，该函数将呈现上一节中的模板。

因此，让我们编写一个映射到创建表单的*/ login* URL 的新视图函数，并将其传递给模板进行渲染。此视图函数也可以在*app / routes.py*模块中使用前一个：

*app / routes.py*：登录视图功能

```
from flask import render_template
from app import app
from app.forms import LoginForm

# ...

@app.route('/login')
def login():
    form = LoginForm()
    return render_template('login.html', title='Sign In', form=form)
```

我在这里做的是`LoginForm`从*forms.py*导入类，*从中*实例化一个对象，然后将其发送到模板。该`form=form`语法可能看起来很奇怪，但简单地使`form`在上面的行中创建（和示出右侧）对象的名称的模板`form`（在左侧示出）。这就是获取表单字段所需的全部内容。

为了便于访问登录表单，基本模板可以在导航栏中包含指向它的链接：

*app / templates / base.html*：导航栏中的登录链接

```
<div>
    Microblog:
    <a href="/index">Home</a>
    <a href="/login">Login</a>
</div>
```

此时，您可以运行该应用程序并在Web浏览器中查看该表单。运行应用程序后，键入`http://localhost:5000/`浏览器的地址栏，然后单击顶部导航栏中的“登录”链接以查看新的登录表单。很酷，对吗？

![登录表单](https://blog.miguelgrinberg.com/static/images/mega-tutorial/ch03-login-form.png)

## 接收表格数据

如果您尝试按下提交按钮，浏览器将显示“方法不允许”错误。这是因为到目前为止，上一节中的登录视图功能执行了一半的工作。它可以在网页上显示表单，但它没有处理用户提交的数据的逻辑。这是Flask-WTF使这项工作变得非常简单的另一个领域。以下是视图函数的更新版本，它接受并验证用户提交的数据：

*app / routes.py*：接收登录凭据

```
from flask import render_template, flash, redirect

@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        flash('Login requested for user {}, remember_me={}'.format(
            form.username.data, form.remember_me.data))
        return redirect('/index')
    return render_template('login.html', title='Sign In', form=form)
```

这个版本的第一个新东西是`methods`路径装饰器中的参数。这告诉Flask这个视图函数接受`GET`并`POST`请求，覆盖默认值，即只接受`GET`请求。HTTP协议声明`GET`请求是将信息返回给客户端的请求（在本例中为Web浏览器）。到目前为止，应用程序中的所有请求都属于这种类型。`POST`通常在浏览器向服务器提交表单数据时使用请求（实际上，`GET`请求也可用于此目的，但不建议使用此方法）。出现浏览器之前显示的“方法不允许”错误，因为浏览器尝试发送`POST`请求和应用程序未配置为接受它。通过提供`methods`参数，您告诉Flask应该接受哪些请求方法。

该`form.validate_on_submit()`方法完成所有表单处理工作。当浏览器发送`GET`接收带有表单的网页的请求时，此方法将返回`False`，因此在这种情况下，该函数会跳过if语句并直接在函数的最后一行呈现模板。

当浏览器`POST`由于用户按下提交按钮而发送请求时，`form.validate_on_submit()`将收集所有数据，运行附加到字段的所有验证器，如果一切正常，它将返回`True`，表明数据有效且可以由应用程序处理。但是，如果至少有一个字段未通过验证，则该函数将返回`False`，这将导致将表单呈现给用户，就像在`GET`请求案例中一样。稍后我将在验证失败时添加错误消息。

当`form.validate_on_submit()`回报`True`，登录视图功能调用了两个新的功能，从瓶进口。该`flash()`功能是向用户显示消息的有用方法。许多应用程序使用此技术让用户知道某些操作是否成功。在这种情况下，我将使用此机制作为临时解决方案，因为我还没有将用户登录为真实所需的所有基础结构。我现在能做的最好的事情是显示一条消息，确认应用程序已收到凭据。

登录视图功能中使用的第二个新功能是`redirect()`。此函数指示客户端Web浏览器自动导航到作为参数给出的其他页面。此视图函数使用它将用户重定向到应用程序的索引页。

当您调用该`flash()`函数时，Flask会存储该消息，但闪烁的消息不会神奇地出现在网页中。应用程序的模板需要以适用于站点布局的方式呈现这些闪烁的消息。我将把这些消息添加到基本模板，以便所有模板都继承此功能。这是更新的基本模板：

*app / templates / base.html*：基本模板中的闪烁消息

```
<html>
    <head>
        {% if title %}
        <title>{{ title }} - microblog</title>
        {% else %}
        <title>microblog</title>
        {% endif %}
    </head>
    <body>
        <div>
            Microblog:
            <a href="/index">Home</a>
            <a href="/login">Login</a>
        </div>
        <hr>
        {% with messages = get_flashed_messages() %}
        {% if messages %}
        <ul>
            {% for message in messages %}
            <li>{{ message }}</li>
            {% endfor %}
        </ul>
        {% endif %}
        {% endwith %}
        {% block content %}{% endblock %}
    </body>
</html>
```

在这里我使用的是`with`结构分配调用的结果`get_flashed_messages()`的`messages`变量，都在模板的上下文。该`get_flashed_messages()`函数来自Flask，并返回`flash()`之前已注册的所有消息的列表。后面的条件检查是否`messages`具有某些内容，在这种情况下，将`<ul>`每个消息呈现为一个元素作为`<li>`列表项。这种渲染风格看起来不太好，但是Web应用程序样式化的主题将在稍后出现。

这些闪烁消息的一个有趣属性是，一旦通过该`get_flashed_messages`函数请求它们，它们就会从消息列表中删除，因此它们在`flash()`调用函数后只出现一次。

这是再次尝试应用程序并测试表单如何工作的好时机。确保您尝试将用户名或密码字段为空的表单提交，以查看`DataRequired`验证程序如何暂停提交过程。

## 改进现场验证

附加到表单字段的验证程序可防止无效数据被接受到应用程序中。应用程序处理无效表单输入的方式是重新显示表单，让用户进行必要的更正。

如果您尝试提交无效数据，我确信您注意到虽然验证机制运行良好，但没有指示用户表单有问题，用户只需返回表单。下一个任务是通过在验证失败的每个字段旁边添加有意义的错误消息来改善用户体验。

事实上，表单验证器已经生成了这些描述性错误消息，因此缺少的是模板中用于呈现它们的一些额外逻辑。

以下是在用户名和密码字段中添加了字段验证消息的登录模板：

*app / templates / login.html*：登录表单模板中的验证错误

```
{% extends "base.html" %}

{% block content %}
    <h1>Sign In</h1>
    <form action="" method="post" novalidate>
        {{ form.hidden_tag() }}
        <p>
            {{ form.username.label }}<br>
            {{ form.username(size=32) }}<br>
            {% for error in form.username.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>
            {{ form.password.label }}<br>
            {{ form.password(size=32) }}<br>
            {% for error in form.password.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>{{ form.remember_me() }} {{ form.remember_me.label }}</p>
        <p>{{ form.submit() }}</p>
    </form>
{% endblock %}
```

我所做的唯一更改是在用户名和密码字段之后添加for循环，这些字段用于呈现验证器以红色添加的错误消息。作为一般规则，任何附加验证程序的字段都会在其下添加错误消息`form.<field_name>.errors`。这将是一个列表，因为字段可以附加多个验证器，并且多个可能提供错误消息以显示给用户。

如果您尝试使用空用户名或密码提交表单，现在您将收到一条红色的错误消息。

![表格验证](https://blog.miguelgrinberg.com/static/images/mega-tutorial/ch03-validation.png)

## 生成链接

登录表单现在相当完整，但在结束本章之前，我想讨论在模板和重定向中包含链接的正确方法。到目前为止，您已经看到了一些定义链接的实例。例如，这是基本模板中的当前导航栏：

```
    <div>
        Microblog:
        <a href="/index">Home</a>
        <a href="/login">Login</a>
    </div>
```

登录视图函数还定义了传递给`redirect()`函数的链接：

```
@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        # ...
        return redirect('/index')
    # ...
```

直接在模板和源文件中编写链接的一个问题是，如果有一天您决定重新组织链接，那么您将不得不在整个应用程序中搜索并替换这些链接。

为了更好地控制这些链接，Flask提供了一个名为的函数`url_for()`，它使用URL的内部映射来生成URL以查看函数。例如，`url_for('login')`返回`/login`并`url_for('index')`返回`'/index`。参数to `url_for()`是*端点*名称，它是视图函数的名称。

您可能会问为什么使用函数名称而不是URL更好。事实上，URL比视图函数名称更有可能发生变化，这些名称完全是内部的。第二个原因是，您将在后面了解到，某些URL中包含动态组件，因此手动生成这些URL需要连接多个元素，这很繁琐且容易出错。该`url_for()`还能够产生这些复杂的URL。

所以从现在开始，`url_for()`每次我需要生成应用程序URL时，我都会使用它。然后基本模板中的导航栏变为：

*app / templates / base.html*：使用url \ __（）函数进行链接

```
    <div>
        Microblog:
        <a href="{{ url_for('index') }}">Home</a>
        <a href="{{ url_for('login') }}">Login</a>
    </div>
```

这是更新的`login()`视图功能：

*app /* routes.py：对链接使用url \ __（）函数

```
from flask import render_template, flash, redirect, url_for

# ...

@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        # ...
        return redirect(url_for('index'))
    # ...
```