# 17 Linux上的部署

在本章中，我将在Microblog应用程序的生命周期中达到一个里程碑，因为我将讨论如何在生产服务器上部署应用程序，以便真实用户可以访问它。

部署的主题很广泛，因此不可能在此处涵盖所有可能的选项。本章致力于探索传统的托管选项，作为主题，我将使用运行Ubuntu的专用Linux服务器，以及广受欢迎的Raspberry Pi小型计算机。我将在后面的章节中介绍其他选项，例如云和容器部署。

*本章的GitHub链接是：Browse，Zip，Diff。*

## 传统的托管

当我提到“传统托管”时，我的意思是应用程序是手动安装的，也可以通过股票服务器上的脚本安装程序安装。该过程涉及安装应用程序，其依赖项和生产规模Web服务器，并配置系统以确保其安全。

在您要部署自己的项目时需要询问的第一个问题是在哪里可以找到服务器。这些天有许多经济托管服务。例如，每月5美元，[Digital Ocean](https://www.digitalocean.com/)，[Linode](https://www.linode.com/)或[Amazon Lightsail](https://amazonlightsail.com/)将租用一台虚拟化Linux服务器来运行您的部署实验（Linode和Digital Ocean为其入门级服务器提供1GB内存，而亚马逊只提供512MB ）。如果您喜欢在不花钱的情况下练习部署，那么[Vagrant](https://www.vagrantup.com/)和[VirtualBox](https://www.virtualbox.org/)是两个结合使用的工具，允许您在自己的计算机上创建类似于付费虚拟服务器的虚拟服务器。

就操作系统选择而言，从技术角度来看，该应用程序可以部署在任何主要操作系统上，包括各种开源Linux和BSD发行版的列表，以及商业OS X和Microsoft Windows（虽然OS X是混合开源/商业选项，因为它基于Darwin，一种开源BSD衍生产品）。

由于OS X和Windows是未优化为桌面操作系统的桌面操作系统，我将把它们作为候选者丢弃。Linux或BSD操作系统之间的选择很大程度上取决于偏好，因此我将选择最受欢迎的两种，即Linux。就Linux发行版而言，我将再次按人气选择并选择Ubuntu。

## 创建Ubuntu服务器

如果您有兴趣与我一起进行此部署，您显然需要一台服务器来处理。我将为您推荐两种购买服务器的选项，一种是付费的，一种是免费的。如果您愿意花一点钱，可以在Digital Ocean，Linode或Amazon Lightsail获得一个帐户，并创建一个Ubuntu 16.04虚拟服务器。您应该使用最小的服务器选项，在我写这篇文章时，所有三个提供商每月花费5美元。成本按照服务器启动的小时数按比例分配，因此如果您创建服务器，使用它几个小时然后删除它，您只需支付几美分。

免费替代方案基于您可以在自己的计算机上运行的虚拟机。要使用此选项，请在计算机上安装[Vagrant](https://www.vagrantup.com/)和[VirtualBox](https://www.virtualbox.org/)，然后创建名为*Vagrantfile*的文件，以使用以下内容描述VM的规范：

*Vagrantfile*：流浪汉配置。

```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
  end
end
```

此文件配置具有1GB RAM的Ubuntu 16.04服务器，您可以从IP地址为192.168.33.10的主机访问该服务器。要创建服务器，请运行以下命令：

```
$ vagrant up
```

请参阅Vagrant [命令行文档](https://www.vagrantup.com/docs/cli/)以了解管理虚拟服务器的其他选项。

## 使用SSH客户端

您的服务器是无头的，因此您不会像在自己的计算机上那样拥有桌面。您将通过SSH客户端连接到您的服务器，并通过命令行处理它。如果您使用的是Linux或Mac OS X，则可能已经安装了[OpenSSH](http://www.openssh.org/)。如果您使用的是Microsoft Windows，则[Cygwin](https://www.cygwin.com/)，[Git](https://git-scm.com/)和[Windows的Linux子系统](https://msdn.microsoft.com/en-us/commandline/wsl/about)提供OpenSSH，因此您可以安装任何这些选项。

如果您使用的是来自第三方提供商的虚拟服务器，则在创建服务器时，您会获得一个IP地址。您可以使用以下命令打开与全新服务器的终端会话：

```
$ ssh root@<server-ip-address>
```

系统将提示您输入密码。根据服务的不同，密码可能是在您创建服务器后自动生成并显示给您的，或者您可能已选择自己选择密码。

如果您使用的是Vagrant VM，则可以使用以下命令打开终端会话：

```
$ vagrant ssh
```

如果您使用的是Windows并拥有Vagrant VM，请注意您需要从可以`ssh`从OpenSSH 调用命令的shell运行上述命令。

## 无密码登录

如果您使用的是Vagrant VM，则可以跳过此部分，因为您的VM已正确配置为使用名为的非root帐户`ubuntu`，而Vagrant不会自动使用密码。

If you are using a virtual server, it is recommended that you create a regular user account to do your deployment work, and configure this account to log you in without using a password, which at first may seem like a bad idea, but you'll see that it is not only more convenient but also more secure.

I'm going to create a user account named `ubuntu` (you can use a different name if you prefer). To create this user account, log in to your server's root account using the `ssh` instructions from the previous section, and then type the following commands to create the user, give it `sudo` powers, and finally switch to it:

```
$ adduser --gecos "" ubuntu
$ usermod -aG sudo ubuntu
$ su ubuntu
```

Now I'm going to configure this new `ubuntu` account to use [public key](http://en.wikipedia.org/wiki/Public-key_cryptography) authentication so that you can log in without having to type a password.

将已打开的终端会话暂时保留在服务器上，然后在本地计算机上启动第二个终端。如果您使用的是Windows，那么这需要是您可以访问该`ssh`命令的终端，因此它可能是一个`bash`或类似的提示，而不是本机Windows终端。在该终端会话中，检查*〜/ .ssh*目录的内容：

```
$ ls ~/.ssh
id_rsa  id_rsa.pub
```

如果目录列表显示如上所述的名为*id_rsa*和*id_rsa.pub的*文件，那么您已经有了一个密钥。如果您没有这两个文件，或者根本没有*〜/ .ssh*目录，则需要通过运行以下命令来创建SSH密钥对，该命令也是OpenSSH工具集的一部分：

```
$ ssh-keygen
```

此应用程序将提示您输入一些内容，我建议您在所有提示中按Enter键接受默认值。如果你知道自己在做什么，想要做什么，你当然可以。

运行此命令后，您应该具有上面列出的两个文件。文件*id_rsa.pub*是您的*公钥*，这是一个文件，您将提供给第三方作为识别您的方式。该*id_rsa*文件是你的*私钥*，不应与任何人共享。

您现在需要将公钥配置为服务器中的*授权主机*。在您自己的计算机上打开的终端上，将您的公钥打印到屏幕上：

```
$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCjw....F8Xv4f/0+7WT miguel@miguelspc
```

这将是一个很长的字符序列，可能跨越多行。您需要将此数据复制到剪贴板，然后切换回远程服务器上的终端，您将在其中发出以下命令来存储公钥：

```
$ echo <paste-your-key-here> >> ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys
```

无密码登录现在应该正常工作。我们的想法是，`ssh`您的计算机上将通过执行需要私钥的加密操作来向服务器标识自己。然后，服务器使用您的公钥验证操作是否有效。

您现在可以退出`ubuntu`会话，然后退出`root`会话，然后尝试使用以下命令直接登录`ubuntu`帐户：

```
$ ssh ubuntu@<server-ip-address>
```

这次你不必输入密码！

## 保护您的服务器

为了最大限度地降低服务器遭受入侵的风险，您可以采取一些步骤，针对关闭攻击者可能获取访问权限的许多可能门。

我要做的第一个改变是通过SSH禁用root登录。您现在可以无密码访问该`ubuntu`帐户，并且您可以通过此帐户运行管理员命令`sudo`，因此实际上不需要公开root帐户。要禁用root登录，您需要编辑服务器上的*/ etc / ssh / sshd_config*文件。你可能有`vi`，并`nano`安装在您的服务器，你可以用它来编辑文件的文本编辑器（如果你不熟悉的任何一个，尝试`nano`在前）。您需要为编辑器添加前缀`sudo`，因为常规用户无法访问SSH配置（即`sudo vi /etc/ssh/sshd_config`）。您需要更改此文件中的一行：

*/ etc / ssh / sshd_config*：禁用root登录。

```
PermitRootLogin no
```

Note that to make this change you need to locate the line that starts with `PermitRootLogin` and change the value, whatever that might be in your server, to `no`.

The next change is in the same file. Now I'm going to disable password logins for all accounts. You have a password-less login set up, so there is no need to allow passwords at all. If you feel nervous about disabling passwords altogether you can skip this change, but for a production server it is a really good idea, since attackers are constantly trying random account names and passwords on all servers hoping to get lucky. To disable password logins, change the following line in */etc/ssh/sshd_config*:

*/etc/ssh/sshd_config*: Disable password logins.

```
PasswordAuthentication no
```

After you are done editing the SSH configuration, the service needs to be restarted for the changes to take effect:

```
$ sudo service ssh restart
```

The third change I'm going to make is to install a *firewall*. This is a software that blocks accesses to the server on any ports that are not explicitly enabled:

```
$ sudo apt-get install -y ufw
$ sudo ufw allow ssh
$ sudo ufw allow http
$ sudo ufw allow 443/tcp
$ sudo ufw --force enable
$ sudo ufw status
```

These commands install [ufw](https://wiki.ubuntu.com/UncomplicatedFirewall), the Uncomplicated Firewall, and configure it to only allow external traffic on port 22 (ssh), 80 (http) and 443 (https). Any other ports will not be allowed.

## Installing Base Dependencies

If you followed my advice and provisioned your server with the Ubuntu 16.04 release, then you have a system that comes with full support for Python 3.5, so this is the release that I'm going to use for the deployment.

The base Python interpreter is probably pre-installed on your server, but there are some extra packages that are likely not, and there are also a few other packages outside of Python that are going to be useful in creating a robust, production-ready deployment. For a database server, I'm going to switch from SQLite to MySQL. The postfix package is a mail transfer agent, that I will use to send out emails. The supervisor tool will monitor the Flask server process and automatically restart it if it ever crashes, or also if the server is rebooted. The nginx server is going to accept all request that come from the outside world, and forward them to the application. Finally, I'm going to use git as my tool of choice to download the application directly from its git repository.

```
$ sudo apt-get -y update
$ sudo apt-get -y install python3 python3-venv python3-dev
$ sudo apt-get -y install mysql-server postfix supervisor nginx git
```

这些安装主要是无人值守的，但是在运行第三个安装语句的某个时刻，系统会提示您选择MySQL服务的root密码，并且还会询问有关安装postfix软件包的几个问题。你可以接受他们的默认答案。

请注意，对于此部署，我选择不安装Elasticsearch。此服务需要大量RAM，因此只有拥有大于2GB RAM的大型服务器才能实现。为了避免服务器内存不足的问题，我将保留搜索功能。如果您有足够大的服务器，可以从[Elasticsearch网站](https://elastic.co/)下载官方.deb软件包，并按照其安装说明将其添加到您的服务器。请注意，Ubuntu 16.04软件包存储库中提供的Elasticsearch软件包太旧而无法使用，您需要6.x或更高版本。

我还应该注意，postfix的默认安装可能不足以在生产环境中发送电子邮件。为了避免垃圾邮件和恶意电子邮件，许多服务器要求发件人服务器通过安全扩展来识别自己，这意味着至少您必须拥有与您的服务器关联的域名。如果您想了解如何完全配置电子邮件服务器以使其通过标准安全测试，请参阅以下数字海洋指南：

- [后缀配置](http://do.co/2FhdIes)
- [添加SPF记录](http://do.co/2Ff8ksk)
- [DKIM安装和配置](http://do.co/2HW2oTD)

## 安装应用程序

现在我将使用`git`从我的GitHub存储库下载Microblog源代码。如果您不熟悉git源代码控制，我建议您[为初学者](http://ryanflorence.com/git-for-beginners/)阅读[git](http://ryanflorence.com/git-for-beginners/)。

要将应用程序下载到服务器，请确保您位于`ubuntu`用户的主目录中，然后运行：

```
$ git clone https://github.com/miguelgrinberg/microblog
$ cd microblog
$ git checkout v0.17
```

这将在您的服务器上安装代码，并将其与本章同步。如果您将本教程的代码版本保存在自己的git存储库中，则可以将存储库URL更改为您的URL，在这种情况下，您可以跳过该`git checkout`命令。

现在我需要创建一个虚拟环境并使用所有包依赖项填充它，我方便地保存到[第15章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xv-a-better-application-structure)的*requirements.txt*文件：

```
$ python3 -m venv venv
$ source venv/bin/activate
(venv) $ pip install -r requirements.txt
```

除了常见的要求*requirements.txt*，我将使用两个包特定于该生产部署，因此它们不包括在需求文件。该`gunicorn`包是Python应用程序的生产Web服务器。该`pymysql`软件包包含MySQL驱动程序，使SQLAlchemy能够使用MySQL数据库：

```
(venv) $ pip install gunicorn pymysql
```

我需要创建一个*.env*文件，其中包含所有必需的环境变量：

*/home/ubuntu/microblog/.env*：环境配置。

```
SECRET_KEY=52cb883e323b48d78a0a36e8e951ba4a
MAIL_SERVER=localhost
MAIL_PORT=25
DATABASE_URL=mysql+pymysql://microblog:<db-password>@localhost:3306/microblog
MS_TRANSLATOR_KEY=<your-translator-key-here>
```

这个*.env*文件大致类似于我在[第15章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xv-a-better-application-structure)展示的示例，但我使用了随机字符串`SECRET_KEY`。为了生成这个随机字符串，我使用了以下命令：

```
python3 -c "import uuid; print(uuid.uuid4().hex)"
```

对于`DATABASE_URL`变量，我定义了一个MySQL URL。我将在下一节中向您展示如何配置数据库。

我需要将`FLASK_APP`环境变量设置为应用程序的入口点以使`flask`命令起作用，但在解析*.env*文件之前需要此变量，因此需要手动设置。为了避免每次都设置它，我将把它添加到帐户的*〜/ .profile*文件的底部`ubuntu`，以便每次登录时自动设置它：

```
$ echo "export FLASK_APP=microblog.py" >> ~/.profile
```

如果您退出并重新登录，现在`FLASK_APP`将为您设置。您可以通过运行确认它已设置`flask --help`。如果帮助消息显示`translate`应用程序添加的命令，则表示您已找到该应用程序。

现在`flask`命令功能正常，我可以编译语言翻译：

```
(venv) $ flask translate compile
```

## 设置MySQL

我在开发过程中使用的sqlite数据库非常适合简单的应用程序，但是当部署一个可能需要一次处理多个请求的完整Web服务器时，最好使用更强大的数据库。出于这个原因，我将建立一个我将调用的MySQL数据库`microblog`。

要管理数据库服务器，我将使用该`mysql`命令，该命令应该已经安装在您的服务器上：

```
$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.7.19-0ubuntu0.16.04.1 (Ubuntu)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

请注意，您需要键入在安装MySQL时选择的MySQL root密码才能访问MySQL命令提示符。

这些是创建新数据库的命令`microblog`，以及具有完全访问权限的同名用户：

```
mysql> create database microblog character set utf8 collate utf8_bin;
mysql> create user 'microblog'@'localhost' identified by '<db-password>';
mysql> grant all privileges on microblog.* to 'microblog'@'localhost';
mysql> flush privileges;
mysql> quit;
```

您需要`<db-password>`使用您选择的密码替换。这将是`microblog`数据库用户的密码，因此最好不要使用为root用户选择的相同密码。对于密码`microblog`的用户需要匹配您在包含密码`DATABASE_URL`的变量*.ENV*文件。

如果数据库配置正确，您现在应该能够运行创建所有表的数据库迁移：

```
(venv) $ flask db upgrade
```

在继续之前，请确保上述命令完成而不会产生任何错误。

## 设置Gunicorn和主管

当您运行服务器时`flask run`，您正在使用Flask附带的Web服务器。此服务器在开发期间非常有用，但它不适合用于生产服务器，因为它没有考虑到性能和稳健性。而不是Flask开发服务器，对于这个部署，我决定使用[gunicorn](http://gunicorn.org/)，它也是一个纯Python Web服务器，但与Flask不同，它是一个强大的生产服务器，很多人都使用它，同时它很容易使用。

要在gunicorn下启动Microblog，您可以使用以下命令：

```
(venv) $ gunicorn -b localhost:8000 -w 4 microblog:app
```

该`-b`选项告诉gunicorn在哪里监听请求，我将其设置为端口8000的内部网络接口。在没有外部访问的情况下运行Python Web应用程序通常是一个好主意，然后有一个非常快速的Web服务器，该服务器经过优化以便提供服务静态文件接受来自客户端的所有请求。此快速Web服务器将直接提供静态文件，并将针对应用程序的任何请求转发到内部服务器。我将在下一节中向您展示如何将nginx设置为面向公众的服务器。

该`-w`选项配置gunicorn将运行的*工人*数量。拥有四个工作程序允许应用程序同时处理多达四个客户端，对于Web应用程序而言通常足以处理大量客户端，因为并非所有客户端都在不断地请求内容。根据服务器的RAM量，您可能需要调整工作器数量，以免内存不足。

该`microblog:app`参数告诉gunicorn如何加载应用程序实例。冒号前面的名称是包含应用程序的模块，冒号后面的名称是此应用程序的名称。

虽然gunicorn设置非常简单，但从命令行运行服务器实际上并不是生产服务器的好解决方案。我想要做的是让服务器在后台运行，并让它在持续监控下，因为如果由于任何原因服务器崩溃并退出，我想确保自动启动新服务器取代它。而且我还想确保如果重新启动计算机，服务器会在启动时自动运行，而无需我自己登录和启动。我将使用上面安装的[supervisor](http://supervisord.org/)包来执行此操作。

Supervisor实用程序使用配置文件来告诉它要监视哪些程序以及在必要时如何重新启动它们。配置文件必须存储在*/etc/supervisor/conf.d中*。这是Microblog的配置文件，我将其称为*microblog.conf*：

*/etc/supervisor/conf.d/microblog.conf*：主管配置。

```
[program:microblog]
command=/home/ubuntu/microblog/venv/bin/gunicorn -b localhost:8000 -w 4 microblog:app
directory=/home/ubuntu/microblog
user=ubuntu
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
```

的`command`，`directory`并`user`设置告诉上司如何运行应用程序。的`autostart`和`autorestart`设定自动重启由于电脑开机，或崩溃。在`stopasgroup`和`killasgroup`选项确保当上司需要停止应用程序重新启动它，它也达到了顶级gunicorn进程的子进程。

编写此配置文件后，必须重新加载管理程序服务才能导入它：

```
$ sudo supervisorctl reload
```

就这样，gunicorn Web服务器应该启动并运行和监控！

## 设置Nginx

由gunicorn提供支持的微博应用服务器现在私有运行端口8000.我现在需要做的是将应用程序暴露给外部世界是为了在端口80和443上启用面向公众的Web服务器，这是我打开的两个端口防火墙来处理应用程序的Web流量。

我希望这是一个安全的部署，因此我将配置端口80以将所有流量转发到将要加密的端口443。所以我将首先创建一个SSL证书。现在我要创建一个*自签名SSL证书*，可以测试所有但不适合实际部署，因为Web浏览器会警告用户证书不是由受信任的证书颁发机构颁发的。为微博创建SSL证书的命令是：

```
$ mkdir certs
$ openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
  -keyout certs/key.pem -out certs/cert.pem
```

该命令将询问您有关您的应用程序和您自己的一些信息。这是将包含在SSL证书中的信息，如果用户请求查看，则会向用户显示这些信息。上面命令的结果将是两个名为*key.pem*和*cert.pem的*文件，我将它放在Microblog根目录的*certs*子目录中。

要使网站由nginx提供服务，您需要为其编写配置文件。在大多数nginx安装中，此文件需要位于*/ etc / nginx / sites-enabled*目录中。Nginx在这个位置安装了一个我不需要的测试站点，所以我将首先删除它：

```
$ sudo rm /etc/nginx/sites-enabled/default
```

下面你可以看到微博的nginx配置文件，它位于*/ etc / nginx / sites-enabled / microblog中*：

*/ etc / nginx / sites-enabled /微博*：Nginx配置。

```
server {
    # listen on port 80 (http)
    listen 80;
    server_name _;
    location / {
        # redirect any requests to the same URL but on https
        return 301 https://$host$request_uri;
    }
}
server {
    # listen on port 443 (https)
    listen 443 ssl;
    server_name _;

    # location of the self-signed SSL certificate
    ssl_certificate /home/ubuntu/microblog/certs/cert.pem;
    ssl_certificate_key /home/ubuntu/microblog/certs/key.pem;

    # write access and error logs to /var/log
    access_log /var/log/microblog_access.log;
    error_log /var/log/microblog_error.log;

    location / {
        # forward application requests to the gunicorn server
        proxy_pass http://localhost:8000;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /static {
        # handle static files directly, without forwarding to the application
        alias /home/ubuntu/microblog/app/static;
        expires 30d;
    }
}
```

nginx配置远非微不足道，但我添加了一些注释，以便至少你知道每个部分的作用。如果您想获得有关特定指令的信息，请参阅[nginx官方文档](https://nginx.org/en/docs/)。

添加此文件后，您需要告诉nginx重新加载配置以激活它：

```
$ sudo service nginx reload
```

现在应该部署应用程序。在Web浏览器中，您可以键入服务器的IP地址（如果使用的是Vagrant VM，则输入192.168.33.10）并将连接到应用程序。由于您使用的是自签名证书，因此您将从Web浏览器收到警告，您必须将其解雇。

在完成部署后，请按照您自己项目的上述说明进行操作，我强烈建议您将自签名证书替换为真实证书，以便浏览器不会警告您的用户您的站点。为此，您首先需要购买域名并将其配置为指向服务器的IP地址。拥有域名后，您可以申请免费的[Let's Encrypt](https://letsencrypt.org/) SSL证书。我在博客上写了一篇关于如何[通过HTTPS运行Flask应用程序](https://blog.miguelgrinberg.com/post/running-your-flask-application-over-https)的详细文章。

## 部署应用程序更新

我想讨论的关于基于Linux的部署的最后一个主题是如何处理应用程序升级。应用程序源代码通过安装在服务器中`git`，因此，只要您希望将应用程序升级到最新版本，就可以运行`git pull`以下载自上次部署以来所做的新提交。

但是，当然，下载新版本的代码不会导致升级。当前正在运行的服务器进程将继续使用旧代码运行，旧代码已经读取并存储在内存中。要触发升级，您必须停止当前服务器并启动一个新服务器，以强制再次读取所有代码。

进行升级通常比重新启动服务器更复杂。您可能需要应用数据库迁移或编译新的语言翻译，因此实际上，执行升级的过程涉及一系列命令：

```
(venv) $ git pull                              # download the new version
(venv) $ sudo supervisorctl stop microblog     # stop the current server
(venv) $ flask db upgrade                      # upgrade the database
(venv) $ flask translate compile               # upgrade the translations
(venv) $ sudo supervisorctl start microblog    # start a new server
```

## Raspberry Pi Hosting

该[树莓派](http://www.raspberrypi.org/)是具有非常低的功耗低成本的革命性的小台Linux计算机，因此它是主机，可以是每天24小时在线而不会占用您的桌面电脑或笔记本电脑家庭为基础的Web服务器的理想设备。有几个Linux发行版在Raspberry Pi上运行。我的选择是[Raspbian](http://www.raspbian.org/)，这是Raspberry Pi Foundation的官方发行版。

为了准备Raspberry Pi，我将安装一个新的Raspbian版本。我将使用2017年9月版的Raspbian Stretch Lite，但是当你读到这篇文章时，可能会出现更新的版本，所以请查看官方[下载页面](https://www.raspberrypi.org/downloads/raspbian/)以获取最新版本。

Raspbian映像需要安装在SD卡上，然后将其插入Raspberry Pi，以便它可以随机启动。[Raspberry Pi网站](https://www.raspberrypi.org/documentation/installation/installing-images/)上提供了将Raspbian映像从Windows，Mac OS X和Linux复制到SD卡的说明。

当您第一次启动Raspberry Pi时，请在连接到键盘和显示器时执行此操作，以便您可以进行设置。至少应启用SSH，以便您可以从计算机登录以更舒适地执行部署任务。

像Ubuntu一样，Raspbian是Debian的衍生产品，因此上面针对Ubuntu Linux的说明大部分都适用于Raspberry Pi。但是，如果您计划在家庭网络上运行小型应用程序而无需外部访问，则可以决定跳过某些步骤。例如，您可能不需要防火墙或无密码登录。你可能想在这么小的计算机上使用SQLite而不是MySQL。您可以选择不使用nginx，只需让gunicorn服务器直接监听来自客户端的请求即可。你可能只想要一个枪手工人。管理程序服务在确保应用程序始终处于运行状态时非常有用，因此我建议您也在Raspberry Pi上使用它。