# 19 Docker容器上的部署

在[第17章中，](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux)您了解了传统部署，您必须在其中处理服务器配置的每个方面。然后在[第18章中，](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xviii-deployment-on-heroku)当我向您介绍Heroku时，我将您带到了另一个极端，Heroku是一项完全控制配置和部署任务的服务，使您可以完全专注于您的应用程序。在本章中，您将学习基于*容器*的第三个应用程序部署策略，尤其是[Docker](https://www.docker.com/)容器平台。就您需要的部署工作量而言，第三个选项位于其他两个选项之间。

容器构建在轻量级虚拟化技术之上，允许应用程序及其依赖关系和配置完全隔离，但无需使用虚拟机等完整的虚拟化解决方案，因为虚拟机需要更多资源，有时需要与主机相比，性能显着下降。配置为容器主机的系统可以执行许多容器，所有容器共享主机的内核并直接访问主机的硬件。这与虚拟机形成对比，虚拟机必须模拟完整的系统，包括CPU，磁盘，其他硬件，内核等。

尽管必须共享内核，但容器中的隔离级别相当高。容器具有自己的文件系统，并且可以基于与容器主机使用的操作系统不同的操作系统。例如，您可以在Fedora主机上运行基于Ubuntu Linux的容器，反之亦然。虽然容器是Linux操作系统的原生技术，但由于虚拟化，还可以在Windows和Mac OS X主机上运行Linux容器。这允许您在开发系统上测试部署，并且如果您愿意，还可以在开发工作流程中包含容器。

*本章的GitHub链接是：Browse，Zip，Diff。*

## 安装Docker CE

虽然Docker不是唯一的容器平台，但它是迄今为止最受欢迎的，所以这将是我的选择。Docker有两个版本，一个免费社区版（CE）和一个基于订阅的企业版（EE）。出于本教程的目的，Docker CE非常适合。

要使用Docker CE，首先必须在系统上安装它。[Docker网站上](https://www.docker.com/community-edition)有Windows，Mac OS X和几个Linux发行版的安装程序。如果您正在使用Microsoft Windows系统，请务必注意Docker CE需要Hyper-V。如有必要，安装程序将为您启用此功能，但请记住，启用Hyper-V可防止其他虚拟化技术（如VirtualBox）正常工作。

在系统上安装Docker CE后，您可以通过在终端窗口或命令提示符下键入以下命令来验证安装是否成功：

```
$ docker version
Client:
 Version:      17.09.0-ce
 API version:  1.32
 Go version:   go1.8.3
 Git commit:   afdb6d4
 Built:        Tue Sep 26 22:40:09 2017
 OS/Arch:      darwin/amd64

Server:
 Version:      17.09.0-ce
 API version:  1.32 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   afdb6d4
 Built:        Tue Sep 26 22:45:38 2017
 OS/Arch:      linux/amd64
 Experimental: true
```

## 构建容器图像

为Microblog创建容器的第一步是为它构建一个*图像*。容器图像是用于创建容器的模板。它包含容器文件系统的完整表示，以及与网络，启动选项等相关的各种设置。

为应用程序创建容器映像的最基本方法是为要使用的基本操作系统启动容器（Ubuntu，Fedora等），连接到在其中运行的bash shell进程，然后手动安装应用程序，可能遵循我在[第17章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux)介绍的传统部署指南。安装完所有内容后，您可以拍摄容器的快照，然后成为图像。`docker`命令支持这种类型的工作流，但我不打算讨论它，因为每次需要生成新映像时都必须手动安装应用程序。

更好的方法是通过脚本生成容器图像。创建脚本化容器映像的命令是`docker build`。此命令从名为*Dockerfile*的文件中读取并执行构建指令，我将需要创建该文件。Dockerfile基本上是一种安装程序脚本，它执行安装步骤以部署应用程序，以及一些特定于容器的设置。

这是*Microblog*的基本*Dockerfile*：

*Dockerfile*：Microblog的Dockerfile。

```
FROM python:3.6-alpine

RUN adduser -D microblog

WORKDIR /home/microblog

COPY requirements.txt requirements.txt
RUN python -m venv venv
RUN venv/bin/pip install -r requirements.txt
RUN venv/bin/pip install gunicorn

COPY app app
COPY migrations migrations
COPY microblog.py config.py boot.sh ./
RUN chmod +x boot.sh

ENV FLASK_APP microblog.py

RUN chown -R microblog:microblog ./
USER microblog

EXPOSE 5000
ENTRYPOINT ["./boot.sh"]
```

Dockerfile中的每一行都是一个命令。该`FROM`命令指定将在其上构建新映像的基本容器映像。我们的想法是，您从现有图像开始，添加或更改某些内容，最终得到派生图像。图像由名称和标记引用，用冒号分隔。标记用作版本控制机制，允许容器图像提供多个变体。我选择的图像的名称是`python`，这是Python的官方Docker镜像。此映像的标记允许您指定解释器版本和基本操作系统。该`3.6-alpine`tag选择安装在Alpine Linux上的Python 3.6解释器。由于其体积小，通常使用Alpine Linux发行版而不是像Ubuntu这样的更流行的发行版。您可以在[Python映像存储库中](https://hub.docker.com/r/library/python/tags/)查看Python映像可用的标记。

该`RUN`命令在容器的上下文中执行任意命令。这与您在shell提示符下键入命令类似。该`adduser -D microblog`命令创建一个名为的新用户`microblog`。大多数容器映像都是`root`默认用户，但以root身份运行应用程序并不是一个好习惯，因此我创建了自己的用户。

该`WORKDIR`命令设置将安装应用程序的默认目录。当我在`microblog`上面创建用户时，创建了一个主目录，所以现在我将该目录设为默认目录。新的默认目录将应用于Dockerfile中的任何剩余命令，以及稍后执行容器时。

该`COPY`命令将文件从您的机器传输到容器文件系统。此命令采用两个或多个参数，即源文件或目标文件或目录。源文件必须相对于Dockerfile所在的目录。目标可以是绝对路径，也可以是相对于上一个`WORKDIR`命令中设置的目录的路径。在第一个`COPY`命令中，我将*requirements.txt*文件复制到`microblog`容器文件系统中的用户主目录。

现在我在容器中有了*requirements.txt*文件，我可以使用该`RUN`命令创建一个虚拟环境。首先我创建它，然后我在其中安装所有要求。因为需求文件只包含通用依赖项，所以我然后显式安装*gunicorn*，我将把它用作Web服务器。或者，我可以在我的*requirements.txt*文件中添加gunicorn 。

以下三个`COPY`命令通过复制*应用程序*包，*迁移*目录和数据库迁移以及顶级目录中的*microblog.py*和*config.py*脚本，在容器中安装应用程序。我也在复制一个新文件*boot.sh*，我将在下面讨论。

该`RUN chmod`命令确保将此新的*boot.sh*文件正确设置为可执行文件。如果您在基于Unix的文件系统中并且源文件已标记为可执行文件，则复制的文件也将设置可执行位。我添加了一个显式集，因为在Windows上设置可执行位更难。如果你正在使用Mac OS X或Linux，你可能不需要这个声明，但无论如何都没有它。

该`ENV`命令在容器内设置环境变量。我需要设置`FLASK_APP`，这是使用`flask`命令所必需的。

以下`RUN chown`命令将存储在*/ home / microblog中*的所有目录和文件的所有者设置为新`microblog`用户。即使我在Dockerfile顶部附近创建了这个用户，所有命令的默认用户仍然存在`root`，因此所有这些文件都需要切换到`microblog`用户，以便该用户可以在启动容器时使用它们。

`USER`下一行中的命令使此新`microblog`用户成为任何后续指令的默认值，也适用于容器启动时的默认值。

The `EXPOSE` command configures the port that this container will be using for its server. This is necessary so that Docker can configure the network in the container appropriately. I've chosen the standard Flask port 5000, but this can be any port.

Finally, the `ENTRYPOINT` command defines the default command that should be executed when the container is started. This is the command that will start the application web server. To keep things well organized, I decided to create a separate script for this, and this is the *boot.sh* file that I copied to the container earlier. Here are the contents of this script:

*boot.sh*: Docker container start-up script.

```
#!/bin/sh
source venv/bin/activate
flask db upgrade
flask translate compile
exec gunicorn -b :5000 --access-logfile - --error-logfile - microblog:app
```

这是一个相当标准的启动脚本，非常类似于在部署如何[第17](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux)和[第18章](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xviii-deployment-on-heroku)中开始。我激活虚拟环境，通过迁移框架升级数据库，编译语言翻译，最后用gunicorn运行服务器。

注意在`exec`gunicorn命令之前的那个。在shell脚本中，`exec`触发运行脚本的进程将替换为给定的命令，而不是将其作为新进程启动。这很重要，因为Docker将容器的生命周期与在其上运行的第一个进程相关联。在这种情况下，启动过程不是容器的主要过程，您需要确保主进程取代第一个进程，以确保Docker不会提前终止容器。

Docker的一个有趣的方面是容器写入`stdout`或`stderr`将被捕获并存储为容器日志的任何内容。出于这个原因，`--access-logfile`和`--error-logfile`都配置了a `-`，它将日志发送到标准输出，以便它们由Docker存储为日志。

创建Dockerfile后，我现在可以构建一个容器图像：

```
$ docker build -t microblog:latest .
```

`-t`我给该`docker build`命令的参数设置了新容器图像的名称和标记。在`.`表示基本目录所在的容器是要建。这是*Dockerfile*所在的目录。构建过程将评估*Dockerfile*中的所有命令并创建映像，该映像将存储在您自己的计算机上。

您可以使用以下`docker images`命令获取本地图像的列表：

```
$ docker images
REPOSITORY    TAG          IMAGE ID        CREATED              SIZE
microblog     latest       54a47d0c27cf    About a minute ago   216MB
python        3.6-alpine   a6beab4fa70b    3 months ago         88.7MB
```

此列表将包含您的新图像以及构建它的基本图像。每次对应用程序进行更改时，都可以通过再次运行build命令来更新容器映像。

## 启动容器

使用已创建的映像，您现在可以运行应用程序的容器版本。这是通过`docker run`命令完成的，该命令通常需要大量参数。我将首先向您展示一个基本示例：

```
$ docker run --name microblog -d -p 8000:5000 --rm microblog:latest
021da2e1e0d390320248abf97dfbbe7b27c70fefed113d5a41bb67a68522e91c
```

该`--name`选项提供新容器的名称。该`-d`选项告诉Docker在后台运行容器。如果`-d`容器不作为前台应用程序运行，则阻止命令提示符。该`-p`选项将容器端口映射到主机端口。第一个端口是主机上的端口，右边的端口是容器内的端口。上面的示例在主机的端口8000上的容器中公开了端口5000，因此您将在8000上访问该应用程序，即使容器内部使用5000也是如此。`--rm`选项将在终止后删除容器。虽然这不是必需的，但通常不再需要完成或中断的容器，因此可以自动删除它们。最后一个参数是容器映像名称和用于容器的标记。运行上述命令后，您可以访问*http：// localhost：8000上*的应用程序。

输出`docker run`是分配给新容器的ID。这是一个长十六进制字符串，只要您需要在后续命令中引用容器，就可以使用它。实际上，只有前几个字符是必需的，足以使ID唯一。

如果要查看正在运行的容器，可以使用以下`docker ps`命令：

```
$ docker ps
CONTAINER ID  IMAGE             COMMAND      PORTS                   NAMES
021da2e1e0d3  microblog:latest  "./boot.sh"  0.0.0.0:8000->5000/tcp  microblog
```

您甚至可以看到`docker ps`命令缩短了容器ID。如果您现在想要停止容器，可以使用`docker stop`：

```
$ docker stop 021da2e1e0d3
021da2e1e0d3
```

如果您还记得，应用程序配置中有许多选项来源于环境变量。例如，Flask密钥，数据库URL和电子邮件服务器选项都是从环境变量导入的。在`docker run`上面的示例中，我并不担心这些，因此所有这些配置选项都将使用默认值。

在更现实的示例中，您将在容器内设置这些环境变量。您在上一节中看到*Dockerfile*中的`ENV`命令设置了环境变量，对于将变为静态的变量，它是一个方便的选项。但是，对于依赖于安装的变量，将它们作为构建过程的一部分是不方便的，因为您希望拥有一个相当可移植的容器映像。如果您想将您的应用程序作为容器映像提供给另一个人，您可能希望该人员能够按原样使用它，而不必使用不同的变量重建它。

因此构建时环境变量可能很有用，但是还需要具有可以通过`docker run`命令设置的运行时环境变量，对于这些变量，`-e`可以使用该选项。以下示例设置密钥并通过Gmail帐户发送电子邮件：

```
$ docker run --name microblog -d -p 8000:5000 --rm -e SECRET_KEY=my-secret-key \
    -e MAIL_SERVER=smtp.googlemail.com -e MAIL_PORT=587 -e MAIL_USE_TLS=true \
    -e MAIL_USERNAME=<your-gmail-username> -e MAIL_PASSWORD=<your-gmail-password> \
    microblog:latest
```

`docker run`由于具有许多环境变量定义，命令行非常长并不罕见。

## 使用第三方“集装箱化”服务

Microblog的容器版本看起来不错，但我还没有真正考虑存储。实际上，由于我没有设置`DATABASE_URL`环境变量，因此应用程序使用默认的SQLite数据库，该数据库由磁盘上的文件支持。当您停止并删除容器时，您认为该SQLite文件会发生什么？该文件将消失！

容器中的文件系统是*短暂的*，这意味着当容器消失时它会消失。您可以将数据写入文件系统，如果容器需要读取数据，数据将会存在，但如果出于任何原因需要回收容器并将其替换为新容器，则应用程序保存的任何数据磁盘将永远丢失。

容器应用程序的一个好的设计策略是使应用程序容器*无状态*。如果您的容器具有应用程序代码且没有数据，您可以将其丢弃并将其替换为新的容器而没有任何问题，容器变为真正的一次性容器，这在简化升级部署方面非常有用。

但是，当然，这意味着数据必须放在应用程序容器之外的某个位置。这就是梦幻般的Docker生态系统发挥作用的地方。Docker Container Registry包含各种容器映像。我已经告诉过你关于Python容器图像的信息，我将其用作我的Microblog容器的基本图像。除此之外，Docker还为Docker注册表中的许多其他语言，数据库和其他服务维护图像，如果这还不够，注册表还允许公司为其产品发布容器图像，还有像您或我这样的普通用户发布自己的图像。这意味着安装第三方服务的努力减少到在注册表中查找适当的图像，并使用`docker run`具有适当参数的命令启动它。

所以我现在要做的是创建两个额外的容器，一个用于MySQL数据库，另一个用于Elasticsearch服务，然后我将创建启动Microblog容器的命令行甚至更长的选项使其能够访问这两个新容器。

### 添加MySQL容器

与许多其他产品和服务一样，MySQL在Docker注册表中提供了公共容器映像。就像我自己的Microblog容器一样，MySQL依赖于需要传递给的环境变量`docker run`。这些配置密码，数据库名称等。虽然注册表中有许多MySQL映像，但我决定使用由MySQL团队正式维护的映像。您可以在其注册表页面中找到有关MySQL容器映像的详细信息：*https*：*//hub.docker.com/r/mysql/mysql-server/*。

如果您还记得[第17章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux)设置MySQL的繁琐过程，那么当您看到部署MySQL是多么容易时，您将会欣赏Docker。这是`docker run`启动MySQL服务器的命令：

```
$ docker run --name mysql -d -e MYSQL_RANDOM_ROOT_PASSWORD=yes \
    -e MYSQL_DATABASE=microblog -e MYSQL_USER=microblog \
    -e MYSQL_PASSWORD=<database-password> \
    mysql/mysql-server:5.7
```

这就对了！在安装了Docker的任何计算机上，您可以运行上面的命令，您将获得一个完全安装的MySQL服务器，其中包含一个随机生成的root密码，一个名为的全新数据库`microblog`，以及一个配置为full的同名用户访问数据库的权限。请注意，您需要输入正确的密码作为`MYSQL_PASSWORD`环境变量的值。

现在在应用程序方面，我需要添加一个MySQL客户端软件包，就像我在Ubuntu上的传统部署一样。我将`pymysql`再次使用，我可以添加到*Dockerfile*：

*Dockerfile*：将pymysql添加到Dockerfile。

```
# ...
RUN venv/bin/pip install gunicorn pymysql
# ...
```

*每次*对应用程序或*Dockerfile进行更改时*，都需要重建容器映像：

```
$ docker build -t microblog:latest .
```

现在我可以再次启动微博，但这次有一个数据库容器的链接，以便两者都可以通过网络进行通信：

```
$ docker run --name microblog -d -p 8000:5000 --rm -e SECRET_KEY=my-secret-key \
    -e MAIL_SERVER=smtp.googlemail.com -e MAIL_PORT=587 -e MAIL_USE_TLS=true \
    -e MAIL_USERNAME=<your-gmail-username> -e MAIL_PASSWORD=<your-gmail-password> \
    --link mysql:dbserver \
    -e DATABASE_URL=mysql+pymysql://microblog:<database-password>@dbserver/microblog \
    microblog:latest
```

该`--link`选项告诉Docker使另一个容器可以访问。该参数包含两个以冒号分隔的名称。第一部分是要链接的容器的名称或ID，在本例中`mysql`是我在上面创建的名称。第二部分定义了一个主机名，可以在此容器中使用该主机名来引用链接的主机名。这里我`dbserver`用作代表数据库服务器的通用名称。

通过建立两个容器之间的链接，我可以设置`DATABASE_URL`环境变量，以便指向SQLAlchemy在另一个容器中使用MySQL数据库。数据库URL将`dbserver`用作数据库主机名，`microblog`数据库名称和用户以及启动MySQL时选择的密码。

我在试验MySQL容器时注意到的一件事是，这个容器需要几秒钟才能完全运行并准备好接受数据库连接。如果启动MySQL容器然后立即启动应用程序容器，当*boot.sh*脚本尝试运行时`flask db migrate`，由于数据库未准备好接受连接，它可能会失败。为了使我的解决方案更健壮，我决定在*boot.sh中*添加一个重试循环：

*boot.sh*：重试数据库连接。

```
#!/bin/sh
source venv/bin/activate
while true; do
    flask db upgrade
    if [[ "$?" == "0" ]]; then
        break
    fi
    echo Upgrade command failed, retrying in 5 secs...
    sleep 5
done
flask translate compile
exec gunicorn -b :5000 --access-logfile - --error-logfile - microblog:app
```

此循环检查`flask db upgrade`命令的退出代码，如果它不为零，则表示出现错误，因此等待五秒钟然后重试。

### 添加Elasticsearch容器

[Docker](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html)的[Elasticsearch文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html)显示了如何将服务作为单节点进行开发，以及作为双节点生产就绪部署。现在我将使用单节点选项并使用“oss”图像，该图像只有开源引擎。使用以下命令启动容器：

```
$ docker run --name elasticsearch -d -p 9200:9200 -p 9300:9300 --rm \
    -e "discovery.type=single-node" \
    docker.elastic.co/elasticsearch/elasticsearch-oss:6.1.1
```

这个`docker run`命令与我用于Microblog和MySQL的命令有很多相似之处，但是有一些有趣的区别。首先，有两个`-p`选项，这意味着此容器将侦听两个端口而不是一个端口。端口9200和9300都映射到主机中的相同端口。

另一个区别在于用于引用容器图像的语法。对于我在本地构建的图像，语法是`<name>:<tag>`。MySQL容器使用稍微更完整的语法格式`<account>/<name>:<tag>`，适用于引用Docker注册表上的容器映像。我正在使用的Elasticsearch图像遵循模式`<registry>/<account>/<name>:<tag>`，其中包括注册表的地址作为第一个组件。此语法用于未托管在Docker注册表中的图像。在这种情况下，Elasticsearch在*docker.elastic.co上*运行自己的容器注册表服务，而不是使用Docker维护的主注册表。

现在我已启动并运行Elasticsearch服务，我可以修改我的Microblog容器的启动命令，以创建指向它的链接并设置Elasticsearch服务URL：

```
$ docker run --name microblog -d -p 8000:5000 --rm -e SECRET_KEY=my-secret-key \
    -e MAIL_SERVER=smtp.googlemail.com -e MAIL_PORT=587 -e MAIL_USE_TLS=true \
    -e MAIL_USERNAME=<your-gmail-username> -e MAIL_PASSWORD=<your-gmail-password> \
    --link mysql:dbserver \
    -e DATABASE_URL=mysql+pymysql://microblog:<database-password>@dbserver/microblog \
    --link elasticsearch:elasticsearch \
    -e ELASTICSEARCH_URL=http://elasticsearch:9200 \
    microblog:latest
```

在运行此命令之前，请记住如果仍然运行它，请停止以前的Microblog容器。在命令的适当位置设置数据库和Elasticsearch服务的正确密码时也要小心。

现在您应该可以访问*http：// localhost：8000*并使用搜索功能。如果遇到任何错误，可以通过查看容器日志对其进行故障排除。您很可能希望查看Microblog容器的日志，其中将显示任何Python堆栈跟踪：

```
$ docker logs microblog
```

## Docker容器注册表

所以现在我在Docker上运行完整的应用程序，使用三个容器，其中两个来自公开的第三方图像。如果您想将自己的容器图像提供给其他人，那么您必须*将*它们*推*送到Docker注册表，任何人都可以从中获取图像。

要访问Docker注册表，您需要访问*https://hub.docker.com*并为自己创建一个帐户。确保选择您喜欢的用户名，因为这将用于您发布的所有图像。

要从命令行访问您的帐户，您需要使用以下`docker login`命令登录：

```
$ docker login
```

如果您一直按照我的说明操作，那么您现在可以`microblog:latest`在计算机上本地存储一个名为“ 存储” 的图像。为了能够将此图像推送到Docker注册表，需要将其重命名为包含该帐户，例如来自MySQL的图像。这是通过以下`docker tag`命令完成的：

```
$ docker tag microblog:latest <your-docker-registry-account>/microblog:latest
```

如果您再次列出图像，`docker images`则不会看到微博的两个条目，原始的一个带有`microblog:latest`名称，另一个带有您的帐户名称。这些实际上是同一图像的两个别名。

要将映像发布到Docker注册表，请使用以下`docker push`命令：

```
$ docker push <your-docker-registry-account>/microblog:latest
```

现在您的图像是公开的，您可以记录如何安装它，并以与MySQL和其他人相同的方式从Docker注册表运行。

## 部署容器化应用程序

让您的应用程序在Docker容器中运行的最好的事情之一是，一旦您在本地测试了容器，就可以将它们带到任何提供Docker支持的平台。例如，您可以使用我在[第17章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux)推荐的来自Digital Ocean，Linode或Amazon Lightsail 的相同服务器。即使是这些提供商提供的最便宜的产品也足以使用少量容器运行Docker。

在[亚马逊集装箱服务（ECS），](https://aws.amazon.com/ecs/)使您能够创建容器主机群集在其上运行您的容器，在一个完全集成的AWS环境下使用私营集装箱注册表的能力，以支持缩放和负载均衡，加上选项你的容器图像。

最后，像[Kubernetes](https://kubernetes.io/)这样的容器编排平台提供了更高级别的自动化和便利性，允许您以YAML格式在简单文本文件中描述多容器部署，具有负载平衡，扩展，安全的秘密管理和滚动升级和回滚。