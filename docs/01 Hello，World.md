# 1 Hello，World！

欢迎！您即将开始学习如何使用[Python](https://python.org/)和[Flask](http://flask.pocoo.org/)框架创建Web应用程序。上面的视频将为您提供本教程内容的概述。在第一章中，您将学习如何设置Flask项目。在本章结束时，您将在计算机上运行一个简单的Flask Web应用程序！

本教程中提供的所有代码示例都托管在GitHub存储库中。从[GitHub下载](https://github.com/miguelgrinberg/microblog)代码可以节省大量的输入，但我强烈建议您自己输入代码，至少在前几章中是这样。一旦你对Flask和示例应用程序更加熟悉，如果输入过于繁琐，你可以直接从GitHub访问代码。

在每章的开头，我将为您提供三个GitHub链接，这些链接在您完成本章时非常有用。该**浏览**链接将打开GitHub的仓库中微博的地方，为改变你正在阅读中添加，而不包括在以后的章节介绍的任何变化的章节。该**邮编**链接是一个下载链接，一个压缩文件在内的整个应用程序直至并包括一章中的变化。该**DIFF**链接将打开的章节中，您将要读中所做的所有更改的图形视图。

*本章的GitHub链接是：Browse，Zip，Diff。*

## 安装Python

如果您的计算机上没有安装Python，请立即安装。如果您的操作系统没有为您提供Python包，则可以从[Python官方网站](http://python.org/download/)下载安装程序。如果您使用Microsoft Windows以及WSL或Cygwin，请注意您不会使用Windows的本机版本的Python，而是需要从Ubuntu（如果您使用WSL）或Cygwin获取的Unix友好版本。

为了确保您的Python安装正常运行，您可以打开一个终端窗口并键入`python3`，或者如果这不起作用，只需`python`。以下是您应该看到的内容：

```
$ python3
Python 3.5.2 (default, Nov 17 2016, 17:05:23)
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> _
```

Python解释器现在正在交互式提示符处等待，您可以在其中输入Python语句。在以后的章节中，您将了解此交互式提示对哪些内容有用。但是现在，您已经确认在您的系统上安装了Python。要退出交互式提示，您可以键入`exit()`并按Enter键。在Linux和Mac OS X版本的Python上，您也可以通过按Ctrl-D退出解释器。在Windows上，退出快捷方式是Ctrl-Z，然后是Enter。

## 安装烧瓶

下一步是安装Flask，但在我进入之前，我想告诉你与安装Python *软件包*相关的最佳实践。

在Python中，Flask等软件包可以在公共存储库中找到，任何人都可以从中下载并安装它们。官方Python包存储库名为[PyPI](https://pypi.python.org/pypi)，它代表Python Package Index（有些人也将此存储库称为“奶酪商店”）。从PyPI安装软件包非常简单，因为Python附带了一个名为的工具`pip`（在Python 2.7 `pip`中没有捆绑Python，需要单独安装）。

要在您的计算机上安装软件包，请`pip`按以下方式使用：

```
$ pip install <package-name>
```

有趣的是，这种安装包的方法在大多数情况下都不起作用。如果您的计算机的所有用户全局安装了Python解释器，那么您的常规用户帐户可能无权对其进行修改，因此使上述命令工作的唯一方法是从管理员运行它帐户。但即使没有这种复杂性，请考虑在安装上述软件包时会发生什么。该`pip`工具将从PyPI下载包，然后将其添加到Python安装中。从那时起，您系统上的每个Python脚本都可以访问此包。想象一下，您使用Flask的0.11版本完成了一个Web应用程序，这是您启动时Flask的最新版本，但现在已被0.12版本取代。您现在想要启动第二个应用程序，您希望使用0.12版本，但如果您更换已安装的0.11版本，则可能会破坏旧应用程序。你看到了这个问题吗？如果可以安装Flask 0.11以供旧应用程序使用，并且还为您的新应用程序安装Flask 0.12，那将是理想的选择。

为了解决为不同应用程序维护不同版本的软件包的问题，Python使用了*虚拟环境*的概念。虚拟环境是Python解释器的完整副本。在虚拟环境中安装软件包时，系统范围的Python解释器不受影响，只有副本。因此，为每个应用程序安装任何版本的软件包的完全自由的解决方案是为每个应用程序使用不同的虚拟环境。虚拟环境具有额外的好处，即创建它们的用户拥有它们，因此它们不需要管理员帐户。

让我们从创建项目所在的目录开始。我打算将此目录称为*微博*，因为这是应用程序的名称：

```
$ mkdir microblog
$ cd microblog
```

如果您使用的是Python 3版本，则其中包含虚拟环境支持，因此您只需创建一个：

```
$ python3 -m venv venv
```

使用此命令，我要求Python运行该`venv`程序包，该程序包将创建一个名为的虚拟环境`venv`。`venv`命令中的第一个是Python虚拟环境包的名称，第二个是我将用于此特定环境的虚拟环境名称。如果您发现这一点令人困惑，可以使用`venv`要分配给虚拟环境的其他名称替换第二个。一般情况下，我使用`venv`项目目录中的名称创建我的虚拟环境，因此每当我`cd`进入项目时，我都会找到相应的虚拟环境。

请注意，在某些操作系统中，您可能需要使用`python`而不是`python3`在上面的命令中。一些安装`python`用于Python 2.x版本和`python3`3.x版本，而其他安装用于`python`3.x版本。

命令完成后，您将拥有一个名为*venv*的目录，其中存储了虚拟环境文件。

如果您使用的是早于3.4的任何版本的Python（包括2.7版本），则本机不支持虚拟环境。对于那些版本的Python，您需要先下载并安装名为[virtualenv](https://virtualenv.pypa.io/)的第三方工具，然后才能创建虚拟环境。安装virtualenv后，您可以使用以下命令创建虚拟环境：

```
$ virtualenv venv
```

无论您使用何种方法创建它，都应该创建虚拟环境。现在，您必须告诉系统您要使用它，并通过*激活*它来实现。要激活全新的虚拟环境，请使用以下命令：

```
$ source venv/bin/activate
(venv) $ _
```

如果您使用的是Microsoft Windows命令提示符窗口，则激活命令略有不同：

```
$ venv\Scripts\activate
(venv) $ _
```

激活虚拟环境时，会修改终端会话的配置，以便存储在其中的Python解释器是您键入时调用的解释器`python`。此外，修改终端提示以包括激活的虚拟环境的名称。对终端会话所做的更改对于该会话都是临时和私有的，因此在关闭终端窗口时它们不会保留。如果您同时打开多个终端窗口，则可以在每个窗口上激活不同的虚拟环境。

现在您已创建并激活了虚拟环境，您最终可以在其中安装Flask：

```
(venv) $ pip install flask
```

如果要确认您的虚拟环境，现在已经安装了瓶，就可以启动Python解释器和*进口*瓶到它：

```
>>> import flask
>>> _
```

如果此声明没有给您任何错误，您可以祝贺自己，因为Flask已安装并可以使用。

## 一个“Hello，World”Flask应用程序

如果你去[Flask网站](http://flask.pocoo.org/)，你会看到一个非常简单的示例应用程序，它只有五行代码。我不会重复那个简单的例子，而是会向您展示一个更精细的例子，它将为您提供一个良好的基础结构来编写更大的应用程序。

该应用程序将存在于一个*包中*。在Python中，包含*__init__.py*文件的子目录被视为包，可以导入。导入包时，*__ init__.py*执行并定义包暴露给外部世界的符号。

让我们创建一个名为的包`app`，它将托管应用程序。确保您在*微博*目录中，然后运行以下命令：

```
(venv) $ mkdir app
```

该*__init__.py*的`app`包是要包含以下代码：

*app / __ init__.py：Flask*应用程序实例

```
from flask import Flask

app = Flask(__name__)

from app import routes
```

上面的脚本只是将应用程序对象创建为`Flask`从flask包导入的类的实例。`__name__`传递给`Flask`类的变量是Python预定义变量，该变量设置为使用它的模块的名称。当需要加载模板文件等相关资源时，Flask使用此处传递的模块的位置作为起点，我将在[第2章中介绍](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-ii-templates)。出于所有实际目的，传递`__name__`几乎总是以正确的方式配置Flask。然后，应用程序导入`routes`尚不存在的模块。

一开始可能看起来令人困惑的一个方面是有两个名为实体的实体`app`。该`app`包由*app*目录和*__init__.py*脚本定义，并在`from app import routes`语句中引用。该`app`变量`Flask`在*__init__.py*脚本中定义为类的实例，这使其成为`app`包的成员。

另一个特点是`routes`模块是在底部而不是在脚本顶部导入的，因为它始终是完成的。底部导入是*循环*导入的变通方法，这是Flask应用程序的常见问题。您将看到`routes`模块需要导入`app`此脚本中定义的变量，因此将一个倒数导入放在底部可以避免由这两个文件之间的相互引用引起的错误。

那么`routes`模块中的内容是什么？路由是应用程序实现的不同URL。在Flask中，应用程序路由的处理程序被编写为Python函数，称为*视图函数*。视图函数映射到一个或多个路由URL，以便Flask知道客户端请求给定URL时要执行的逻辑。

这是您的第一个视图函数，您需要在名为*app / routes.py*的新模块中编写它：

*app / routes.py*：主页路由

```
from app import app

@app.route('/')
@app.route('/index')
def index():
    return "Hello, World!"
```

这个视图函数实际上非常简单，只是将问候语作为字符串返回。`@app.route`函数上方的两个奇怪的行是*装饰器*，这是Python语言的一个独特功能。装饰器修改后面的函数。与装饰器的常见模式是使用它们将函数注册为某些事件的回调。在这种情况下，`@app.route`装饰器在作为参数给出的URL和函数之间创建关联。在这个例子中有两个装饰器，它们关联URL `/`和`/index`这个功能。这意味着当Web浏览器请求这两个URL中的任何一个时，Flask将调用此函数并将其返回值作为响应传递回浏览器。如果这还没有完全合理，那么当你运行这个应用程序时它会有一点点。

要完成应用程序，您需要在顶层定义一个定义Flask应用程序实例的Python脚本。我们将此脚本称为*microblog.py*，并将其定义为导入应用程序实例的单行：

*microblog.py*：主要应用模块

```
from app import app
```

还记得两个`app`实体吗？在这里你可以在同一个句子中看到两者。Flask应用程序实例被调用`app`并且是`app`包的成员。该`from app import app`语句导入`app`作为`app`包成员的变量。如果您发现这一点令人困惑，您可以将包或变量重命名为其他内容。

为了确保您正确地执行所有操作，下面您可以看到目前为止的项目结构图：

```
microblog/
  venv/
  app/
    __init__.py
    routes.py
  microblog.py
```

信不信由你，这个应用程序的第一个版本现已完成！但是，在运行之前，需要通过设置`FLASK_APP`环境变量告诉Flask如何导入它：

```
(venv) $ export FLASK_APP=microblog.py
```

如果您使用的是Microsoft Windows，请使用`set`而不是`export`上面的命令。

你准备好被吹走吗？您可以使用以下命令运行第一个Web应用程序：

```
(venv) $ flask run
 * Serving Flask app "microblog"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

服务器初始化后，它将等待客户端连接。输出`flask run`指示服务器在IP地址127.0.0.1上运行，该地址始终是您自己计算机的地址。这个地址很常见，也有一个你以前见过的简单名称：*localhost*。网络服务器侦听特定端口号上的连接。部署在生产Web服务器上的应用程序通常侦听端口443，如果它们不实施加密，有时会侦听80，但访问这些端口需要管理权限。由于此应用程序在开发环境中运行，Flask使用免费提供的端口5000.现在打开Web浏览器并在地址字段中输入以下URL：

```
    http://localhost:5000/
```

或者，您可以使用此其他URL：

```
    http://localhost:5000/index
```

您是否看到应用程序路由映射正在进行中？第一个URL映射到`/`，而第二个映射到`/index`。两个路由都与应用程序中唯一的视图函数相关联，因此它们生成相同的输出，即函数返回的字符串。如果您输入任何其他URL，您将收到错误，因为应用程序只能识别这两个URL。

![你好，世界！](https://blog.miguelgrinberg.com/static/images/mega-tutorial/ch01-hello-world.png)

完成服务器播放后，只需按Ctrl-C即可将其停止。

恭喜，您已经完成了成为Web开发人员的第一个重要步骤！

在结束本章之前，我想再提一件事。由于环境变量不会在终端会话中被记住，因此在`FLASK_APP`打开新的终端窗口时，您可能总是需要设置环境变量。从版本1.0开始，Flask允许您注册在运行`flask`命令时要自动导入的环境变量。要使用此选项，您必须安装*python-dotenv*包：

```
(venv) $ pip install python-dotenv
```

然后，您可以在项目的顶级目录中的*.flaskenv*文件中编写环境变量名称和值：

*.flaskenv*：flask命令的环境变量

```
FLASK_APP=microblog.py
```

这样做是可选的。如果您希望手动设置环境变量，那就完全没问题，只要您始终记得这样做。