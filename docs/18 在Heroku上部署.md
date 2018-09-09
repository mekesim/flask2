# 18 在Heroku上部署

在上一篇文章中，我向您展示了托管Python应用程序的“传统”方式，并向您提供了两个基于Linux的服务器部署的实际示例。如果您不习惯管理Linux系统，您可能认为需要投入到任务中的工作量很大，而且肯定必须有一种更简单的方法。

在本章中，我将向您展示一种完全不同的方法，在该方法中，您依靠第三方*云*托管提供商来执行大多数管理任务，从而使您可以将更多时间花在处理应用程序上。

许多云托管提供商都提供可运行应用程序的托管平台。为了在这些平台上部署应用程序，您需要提供的只是实际应用程序，因为硬件，操作系统，脚本语言解释器，数据库等都由服务管理。这种类型的服务称为[平台即服务](https://en.wikipedia.org/wiki/Platform_as_a_service)或PaaS。

听起来好得令人难以置信，对吧？

我将着眼于将Microblog部署到[Heroku](http://heroku.com/)，这是一种流行的云托管服务，对Python应用程序也非常友好。我选择Heroku不仅因为它很受欢迎，而且因为它有一个免费的服务水平，可以让你跟着我做一个完整的部署而不花任何钱。

*本章的GitHub链接是：Browse，Zip，Diff。*

## 在Heroku上托管

Heroku是第一个作为服务提供商的平台之一。它最初是作为基于Ruby的应用程序的托管选项，但后来发展到支持许多其他语言，如Java，Node.js，当然还有Python。

将Web应用程序部署到Heroku是通过`git`版本控制工具完成的，因此您必须将应用程序放在git存储库中。Heroku 在应用程序的根目录中查找名为*Procfile*的文件，以获取有关如何启动应用程序的说明。对于Python项目，Heroku还需要一个*requirements.txt*文件，其中列出了需要安装的所有模块依赖项。在通过git将应用程序上传到Heroku的服务器之后，您基本上已经完成了，只需等待几秒钟直到应用程序在线。这真的很简单。

Heroku提供的不同服务层允许您选择为应用程序获得多少计算能力和时间，因此随着用户群的增长，您将需要购买更多的计算单元，Heroku称之为“dynos”。

准备好尝试Heroku了吗？让我们开始吧！

## 创建Heroku帐户

在部署到Heroku之前，您需要拥有一个帐户。请访问[heroku.com](https://id.heroku.com/signup)并创建一个免费帐户。拥有帐户并登录Heroku后，您将可以访问仪表板，其中列出了所有应用程序。

## 安装Heroku CLI

Heroku提供了一个命令行工具，用于与他们的服务进行交互，称为[Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli)，可用于Windows，Mac OS X和Linux。该文档包含所有支持平台的安装说明。如果您计划部署应用程序以测试服务，请继续将其安装在您的系统上。

安装CLI后，您应该做的第一件事是登录您的Heroku帐户：

```
$ heroku login
```

Heroku CLI将要求您输入您的电子邮件地址和帐户密码。您的身份验证状态将在后续命令中记住。

## 设置Git

该`git`工具是将应用程序部署到Heroku的核心，因此如果您还没有它，则必须将其安装在您的系统上。如果您的操作系统没有可用的软件包，则可以访问[git站点](https://git-scm.com/)下载安装程序。

为什么使用`git`你的项目是有道理的有很多原因。如果您计划部署到Heroku，那么还有一个，因为要部署到Heroku，您的应用程序必须位于`git`存储库中。如果要对Microblog进行测试部署，可以从GitHub克隆应用程序：

```
$ git clone https://github.com/miguelgrinberg/microblog
$ cd microblog
$ git checkout v0.18
```

该`git checkout`命令选择在其历史记录中与本章对应的点处具有应用程序的特定提交。

如果您更喜欢使用自己的代码而不是我自己的代码，您可以`git`通过`git init .`在顶级目录上运行将您自己的项目转换为存储库（注意后面的句点`init`，它告诉git您要在当前目录中创建存储库）。

## 创建Heroku应用程序

要使用Heroku注册新应用程序，请使用`apps:create`应用程序根目录中的命令，将应用程序名称作为唯一参数传递：

```
$ heroku apps:create flask-microblog
Creating flask-microblog... done
http://flask-microblog.herokuapp.com/ | https://git.heroku.com/flask-microblog.git
```

Heroku要求应用程序具有唯一的名称。`flask-microblog`我之前使用的名称不可用，因为我正在使用它，因此您需要为部署选择不同的名称。

此命令的输出将包括Heroku分配给应用程序的URL，以及它的git存储库。您的本地git存储库将配置一个名为的额外*远程数据库*`heroku`。您可以使用以下`git remote`命令验证它是否存在：

```
$ git remote -v
heroku  https://git.heroku.com/flask-microblog.git (fetch)
heroku  https://git.heroku.com/flask-microblog.git (push)
```

根据您创建git存储库的方式，上述命令的输出还可能包含另一个被调用的远程`origin`。

## 短暂的文件系统

Heroku平台与其他部署平台的不同之处在于它具有在虚拟化平台上运行的*临时*文件系统。那是什么意思？这意味着Heroku可以随时将服务器运行的虚拟服务器重置为干净状态。您不能假设您保存到文件系统的任何数据都会持久存在，事实上，Heroku会经常回收服务器。

在这些条件下工作会为我的应用程序引入一些问题，它使用了一些文件：

- 默认的SQLite数据库引擎将数据写入磁盘文件
- 应用程序的日志也会写入文件系统
- 编译的语言翻译存储库也写入本地文件

以下部分将介绍这三个方面。

## 使用Heroku Postgres数据库

为了解决第一个问题，我将切换到另一个数据库引擎。在[第17章中，](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux)您看到我使用MySQL数据库为Ubuntu部署添加了健壮性。Heroku有一个基于Postgres数据库的自己的数据库产品，所以我将切换到那个以避免基于文件的SQLite。

Heroku应用程序的数据库配置了相同的Heroku CLI。在这种情况下，我将在免费套餐上创建一个数据库：

```
$ heroku addons:add heroku-postgresql:hobby-dev
Creating heroku-postgresql:hobby-dev on flask-microblog... free
Database has been created and is available
 ! This database is empty. If upgrading, you can transfer
 ! data from another database with pg:copy
Created postgresql-parallel-56076 as DATABASE_URL
Use heroku addons:docs heroku-postgresql to view documentation
```

新创建的数据库的URL存储在`DATABASE_URL`应用程序运行时可用的环境变量中。这非常方便，因为应用程序已在该变量中查找数据库URL。

## 登录到stdout

Heroku希望应用程序直接登录`stdout`。当您使用该`heroku logs`命令时，将保存并返回应用程序打印到标准输出的任何内容。所以我要添加一个配置变量，指示我是否需要登录`stdout`或者像我一直在做的文件。以下是配置的更改：

*config.py*：登录到stdout的选项。

```
class Config(object):
    # ...
    LOG_TO_STDOUT = os.environ.get('LOG_TO_STDOUT')
```

然后在应用程序工厂函数中，我可以检查此配置以了解如何配置应用程序的记录器：

*app / __ init__.py*：登录到stdout或文件。

```
def create_app(config_class=Config):
    # ...
    if not app.debug and not app.testing:
        # ...

        if app.config['LOG_TO_STDOUT']:
            stream_handler = logging.StreamHandler()
            stream_handler.setLevel(logging.INFO)
            app.logger.addHandler(stream_handler)
        else:
            if not os.path.exists('logs'):
                os.mkdir('logs')
            file_handler = RotatingFileHandler('logs/microblog.log',
                                               maxBytes=10240, backupCount=10)
            file_handler.setFormatter(logging.Formatter(
                '%(asctime)s %(levelname)s: %(message)s '
                '[in %(pathname)s:%(lineno)d]'))
            file_handler.setLevel(logging.INFO)
            app.logger.addHandler(file_handler)

        app.logger.setLevel(logging.INFO)
        app.logger.info('Microblog startup')

    return app
```

所以现在我需要`LOG_TO_STDOUT`在应用程序在Heroku中运行时设置环境变量，而不是在其他配置中。Heroku CLI使这很容易，因为它提供了一个选项来设置在运行时使用的环境变量：

```
$ heroku config:set LOG_TO_STDOUT=1
Setting LOG_TO_STDOUT and restarting flask-microblog... done, v4
LOG_TO_STDOUT: 1
```

## 编译翻译

依赖于本地文件的微博的第三个方面是编译语言翻译文件。确保这些文件永远不会从短暂文件系统中消失的更直接的选项是将编译的语言文件添加到git存储库，以便它们在部署到Heroku后成为应用程序初始状态的一部分。

在我看来，更优雅的选择是`flask translate compile`在Heroku的启动命令中包含该命令，以便在服务器重新启动时，这些文件将再次编译。我将使用此选项，因为我知道我的启动过程无论如何都需要多个命令，因为我还需要运行数据库迁移。所以现在，我将把这个问题放在一边，稍后当我编写*Procfile*时会重新访问它。

## Elasticsearch Hosting

Elasticsearch是可以添加到Heroku项目的众多服务之一，但与Postgres不同，这不是Heroku提供的服务，而是与Heroku合作提供附加组件的第三方。在我撰写本文时，有三个不同的集成Elasticsearch服务提供商。

在配置Elasticsearch之前，请注意Heroku要求您的帐户在安装任何第三方加载项之前将信用卡存档，即使您保留在其免费层中也是如此。如果您不想向Heroku提供信用卡，请跳过本节。您仍然可以部署应用程序，但搜索功能不起作用。

在作为附加组件提供的Elasticsearch选项中，我决定尝试[SearchBox](https://elements.heroku.com/addons/searchbox)，它带有免费的入门计划。要将SearchBox添加到您的帐户，您必须在登录Heroku时运行以下命令：

```
$ heroku addons:create searchbox:starter
```

此命令将部署Elasticsearch服务，并将服务的连接URL保留在`SEARCHBOX_URL`与应用程序关联的环境变量中。再一次请记住，除非您将信用卡添加到Heroku帐户，否则此命令将失败。

如果您从[第16章](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvi-full-text-search)回忆起来，我的应用程序会在`ELASTICSEARCH_URL`变量中查找Elasticsearch连接URL ，因此我需要添加此变量并将其设置为SearchBox分配的连接URL：

```
$ heroku config:get SEARCHBOX_URL
<your-elasticsearch-url>
$ heroku config:set ELASTICSEARCH_URL=<your-elasticsearch-url>
```

这里我首先要求Heroku打印出值`SEARCHBOX_URL`，然后我添加了一个新的环境变量，其名称`ELASTICSEARCH_URL`设置为相同的值。

## 需求更新

Heroku希望依赖项位于*requirements.txt*文件中，就像我在[第15章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xv-a-better-application-structure)定义的那样。但是对于在Heroku上运行的应用程序，我需要为此文件添加两个新的依赖项。

Heroku不提供自己的Web服务器。相反，它希望应用程序在环境变量中给出的端口号上启动自己的Web服务器`$PORT`。由于Flask开发Web服务器不够强大，无法用于生产，我将再次使用[gunicorn](http://gunicorn.org/)，即Heroku推荐的Python应用程序服务器。

该应用程序还将连接到Postgres数据库，因此SQLAlchemy需要`psycopg2`安装该程序包。

双方`gunicorn`并`psycopg2`需要添加到*requirements.txt*文件。

## Procfile

Heroku需要知道如何执行应用程序，并且它在应用程序的根目录中使用名为*Procfile*的文件。此文件的格式很简单，每行包括进程名称，冒号，然后是启动进程的命令。在Heroku上运行的最常见类型的应用程序是Web应用程序，对于此类应用程序，应该是进程名称`web`。下面你可以看到微博的*Procfile*：

*Procfile*：Heroku Procfile。

```
web: flask db upgrade; flask translate compile; gunicorn microblog:app
```

在这里，我定义了命令以按顺序启动Web应用程序作为三个命令。首先我运行数据库迁移升级，然后编译语言翻译，最后我启动服务器。

因为前两个子命令都是基于`flask`命令的，所以我需要添加`FLASK_APP`环境变量：

```
$ heroku config:set FLASK_APP=microblog.py
Setting FLASK_APP and restarting flask-microblog... done, v4
FLASK_APP: microblog.py
```

该应用程序还依赖于其他环境变量，例如配置电子邮件服务器或实时翻译令牌的变量。那些需要添加添加`heroku config:set`命令。

该`gunicorn`命令比我用于Ubuntu部署的命令更简单，因为该服务器与Heroku环境有很好的集成。例如，`$PORT`默认情况下，环境变量受到尊重，而不是使用`-w`选项来设置工作器数量，heroku建议添加一个名为的变量`WEB_CONCURRENCY`，该变量`gunicorn`在`-w`未提供时使用，使您可以灵活地控制工作者数量而无需修改Procfile。

## 部署应用程序

所有准备步骤都已完成，因此现在是时候运行部署了。要将应用程序上载到Heroku的服务器以进行部署，请使用该`git push`命令。这类似于将本地git存储库中的更改推送到GitHub或其他远程git服务器的方式。

现在我已经到了最有趣的部分，我将应用程序推送到我们的Heroku主机帐户。这实际上非常简单，我只需要将`git`应用程序推送到Heroku git存储库的主分支。关于如何执行此操作有几种变体，具体取决于您创建git存储库的方式。如果您正在使用我的`v0.18`代码，那么您需要基于此标记创建分支，并将其作为远程主分支推送，如下所示：

```
$ git checkout -b deploy
$ git push heroku deploy:master
```

相反，如果您正在使用自己的存储库，那么您的代码已经在`master`分支中，因此您首先需要确保提交更改：

```
$ git commit -a -m "heroku deployment changes"
```

然后，您可以运行以下命令来开始部署：

```
$ git push heroku master
```

无论你如何推动分支，你都应该看到Heroku的以下输出：

```
$ git push heroku deploy:master
Counting objects: 247, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (238/238), done.
Writing objects: 100% (247/247), 53.26 KiB | 3.80 MiB/s, done.
Total 247 (delta 136), reused 3 (delta 0)
remote: Compressing source files... done.
remote: Building source:
remote:
remote: -----> Python app detected
remote: -----> Installing python-3.6.2
remote: -----> Installing pip
remote: -----> Installing requirements with pip
...
remote:
remote: -----> Discovering process types
remote:        Procfile declares types -> web
remote:
remote: -----> Compressing...
remote:        Done: 57M
remote: -----> Launching...
remote:        Released v5
remote:        https://flask-microblog.herokuapp.com/ deployed to Heroku
remote:
remote: Verifying deploy... done.
To https://git.heroku.com/flask-microblog.git
 * [new branch]      deploy -> master
```

`heroku`我们在`git push`命令中使用的标签是在创建应用程序时由Heroku CLI自动添加的远程。该`deploy:master`参数意味着我将代码从`deploy`分支引用的本地存储库推送到`master`Heroku存储库的分支。当您使用自己的项目时，您可能会使用命令`git push heroku master`推送您的本地`master`分支。由于这个项目的结构方式，我正在推动一个不是的分支`master`，但是Heroku方面的目标分支总是需要成为`master`Heroku接受部署的唯一分支。

就是这样，应用程序现在应该部署在创建应用程序的命令输出中给出的URL。在我的例子中，URL是*https://flask-microblog.herokuapp.com*，因此我需要键入以访问应用程序。

如果要查看正在运行的应用程序的日志条目，请使用该`heroku logs`命令。如果由于任何原因应用程序无法启动，这可能很有用。如果有任何错误，那些将在日志中。

## 部署应用程序更新

要部署新版本的应用程序，只需`git push`使用新代码运行新命令即可。这将重复部署过程，使旧部署脱机，然后将其替换为新代码。Procfile中的命令将作为新部署的一部分再次运行，因此在此过程中将更新任何新的数据库迁移或转换。