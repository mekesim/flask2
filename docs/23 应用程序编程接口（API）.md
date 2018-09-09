# 23 应用程序编程接口（API）

到目前为止，我为此应用程序构建的所有功能都适用于一种特定类型的客户端：Web浏览器。但是其他类型的客户呢？例如，如果我想构建Android或iOS应用程序，我有两种主要方式可以解决它。最简单的解决方案是构建一个简单的应用程序，其中只有一个Web视图组件可以填充Microblog网站所在的整个屏幕，但这比在设备的Web浏览器中打开应用程序几乎没什么好处。一个更好的解决方案（虽然更加费力）将是构建本机应用程序，但该应用程序如何与仅返回HTML页面的服务器交互？

这是应用程序编程接口（或API）可以提供帮助的问题区域。API是HTTP路由的集合，设计为应用程序的低级入口点。API不允许定义返回HTML以供Web浏览器使用的路由和视图函数，而是允许客户端直接使用应用程序的*资源*，从而决定如何将信息完全呈现给客户端。例如，Microblog中的API可以向客户端提供用户和博客帖子信息，并且还可以允许用户编辑现有博客帖子，但仅限于数据级别，而不将此逻辑与HTML混合。

如果您研究当前在应用程序中定义的所有路径，您会注意到有一些路径符合我上面使用的API的定义。你找到了吗？我在谈论返回JSON的少数路由，例如[第14章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xiv-ajax)定义的*/ translate*路由。这是一种采用文本，源语言和目标语言的路由，所有这些语言都以JSON格式在请求中提供。对此请求的响应是该文本的翻译，也是JSON格式。服务器仅返回所请求的信息，使客户端有责任向用户显示此信息。`POST`

虽然应用程序中的JSON路由具有API“感觉”，但它们旨在支持在浏览器中运行的Web应用程序。考虑一下，如果智能手机应用程序想要使用这些路由，则无法使用这些路由，因为它们需要登录用户，并且登录只能通过HTML表单进行。在本章中，我将展示如何构建不依赖于Web浏览器的API，并且不会假设哪种客户端连接到它们。

*本章的GitHub链接是：Browse，Zip，Diff。*

## REST作为API设计的基础

有些人可能强烈不同意上面的声明*/ translate*和其他JSON路由是API路由。其他人可能同意，他们认为这是一个设计糟糕的API的免责声明。那么设计良好的API的特征是什么？为什么不是该类别中的JSON路由？

您可能听说过[REST API](https://en.wikipedia.org/wiki/Representational_state_transfer)这个术语。REST代表Representational State Transfer，是Roy Fielding博士在他的[博士论文中](http://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)提出的一种架构。在他的工作中，Fielding博士以相当抽象和通用的方式介绍了REST的六个定义特征。

除了菲尔丁博士的论文之外，还没有关于REST的权威规范，这留下了很多细节需要读者解释。关于给定API是否符合REST的主题通常是REST“纯粹主义者”之间激烈争论的根源，他们认为REST API必须遵循所有六个特征并以非常特定的方式进行，而不是REST“实用主义者” “，他将菲尔丁博士在论文中提出的想法作为指导或建议。菲尔丁博士本人支持纯粹主义阵营，并在博客文章和在线评论中为他的愿景提供了一些额外的见解。

目前实施的绝大多数API都遵循“实用”的REST实现。这包括来自“大玩家”的大多数API，例如Facebook，GitHub，Twitter等。很少有公共API被一致认为是纯REST，因为大多数API都错过了纯粹主义者认为必须的某些实施细节 - 富人。尽管Fielding博士和其他REST纯粹主义者对于什么是或不是REST API有严格的观点，但在实际意义上，在软件行业中通常会引用REST。

为了让您了解REST论文中的内容，以下部分描述了Fielding博士列举的六个原则。

### 客户端服务器

客户端 - 服务器原则相当简单，因为它只是说明在REST API中应明确区分客户端和服务器的角色。实际上，这意味着客户端和服务器位于通过传输进行通信的单独进程中，在大多数情况下，这些进程是TCP网络上的HTTP协议。

### 分层系统

分层系统原则表明，当客户端需要与服务器通信时，它可能最终连接到中间设备而不是实际服务器。我们的想法是，对于客户端，如果没有直接连接到服务器，它发送请求的方式绝对没有区别，事实上，它甚至可能不知道它是否连接到目标服务器。同样，该原则指出服务器可以直接从中间设备而不是客户端接收客户端请求，因此它绝不能假设连接的另一端是客户端。

这是REST的一个重要特性，因为能够添加中间节点允许应用程序架构师设计大型复杂网络，这些网络能够通过使用负载平衡器，缓存，代理服务器等来满足大量请求。

### 高速缓存

此原则通过明确指示服务器或中介允许缓存对经常接收的请求的响应以提高系统性能来扩展分层系统。您可能熟悉的缓存实现：所有Web浏览器中的缓存。Web浏览器缓存层通常用于避免不得不一遍又一遍地请求相同的文件，例如图像。

出于API的目的，目标服务器需要通过使用*缓存控件*来指示响应是否可以在中介返回到客户端时被缓存。请注意，出于安全原因，部署到生产的API必须使用加密，缓存通常不在中间节点中完成，除非此节点*终止* SSL连接，或执行解密和重新加密。

### 代码随需应变

这是一个可选要求，指出服务器可以在响应客户端时提供可执行代码。因为这个原则要求服务器和客户端之间就客户端能够运行什么样的可执行代码达成协议，所以很少在API中使用。您可能认为服务器可以返回用于执行Web浏览器客户端的JavaScript代码，但REST并非专门针对Web浏览器客户端。例如，如果客户端是iOS或Android设备，则执行JavaScript可能会引入复杂性。

### 无状态

无国籍原则是REST纯粹主义者和实用主义者之间争论最多的两个中心之一。它声明REST API不应保存每次给定客户端发送请求时要调用的任何客户端状态。这意味着，在Web开发中，当用户浏览应用程序页面时，没有一种常见的“记住”用户的机制。在无状态API中，每个请求都需要包含服务器识别和验证客户端以及执行请求所需的信息。它还意味着服务器无法在数据库或其他形式的存储中存储与客户端连接相关的任何数据。

如果您想知道为什么REST需要无状态服务器，主要原因是无状态服务器非常容易扩展，您需要做的就是在负载均衡器后面运行多个服务器实例。如果服务器存储客户端状态变得更复杂，因为您必须弄清楚多个服务器如何访问和更新该状态，或者确保给定客户端始终由同一服务器处理，这通常称为*粘性会话*。

如果再考虑本章介绍中讨论的*/ translate*路由，你会发现它不能被认为是*RESTful*，因为与该路由相关的视图函数依赖于`@login_required`Flask-Login 的装饰器，而装饰器又存储登录状态Flask用户会话中的用户。

### 统一界面

最后，最重要，最有争议和最模糊记录的REST原则是统一界面。Fielding博士列举了REST统一接口的四个不同方面：唯一资源标识符，资源表示，自描述消息和超媒体。

通过为每个资源分配唯一的URL来实现唯一的资源标识符。例如，与给定用户关联的URL可以是*/ api / users / <user-id>*，其中*<user-id>*是在数据库表的主键中分配给用户的标识符。大多数API都可以很好地实现这一点。

资源表示的使用意味着当服务器和客户端交换有关资源的信息时，它们必须使用商定的格式。对于大多数现代API，JSON格式用于构建资源表示。API可以选择支持多种资源表示格式，在这种情况下，HTTP协议中的*内容协商*选项是客户端和服务器可以就两者都喜欢的格式达成一致的机制。

自描述消息意味着客户端和服务器之间交换的请求和响应必须包含另一方所需的所有信息。作为典型示例，HTTP请求方法用于指示客户端希望服务器执行的操作。一个`GET`请求指示客户端想要检索关于资源的信息，一个`POST`请求表示客户想创建一个新的资源，`PUT`或者`PATCH`请求定义修改现有资源，并`DELETE`指示要删除资源的请求。目标资源表示为请求URL，HTTP标头中提供了附加信息，URL的查询字符串部分或请求正文。

超媒体要求是集合中最具争议性的，并且很少有API实现，而那些实现它的API很少以满足REST纯粹主义者的方式实现。由于应用程序中的资源都是相互关联的，因此该要求要求这些关系包含在资源表示中，以便客户端可以通过遍历关系来发现新资源，这与您在Web应用程序中发现新页面的方式非常相似通过单击从一个页面到下一个页面的链接。这个想法是客户可以在没有任何关于其中资源的任何知识的情况下输入API，并且仅通过遵循超媒体链接来了解它们。使该要求的实现复杂化的一个方面是与HTML和XML不同，[JSON-API](http://jsonapi.org/)，[HAL](http://stateless.co/hal_specification.html)，[JSON-LD](https://json-ld.org/)或类似的。

## 实现API蓝图

为了让您了解开发API所涉及的内容，我将在Microblog中添加一个。这不是一个完整的API，我将实现与用户相关的所有功能，将其他资源（如博客文章）的实现留给读者作为练习。

为了保持组织有序，并遵循我在[第15章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xv-a-better-application-structure)描述的结构，我将创建一个包含所有API路由的新蓝图。那么让我们首先创建这个蓝图所在的目录：

```
(venv) $ mkdir app/api
```

蓝图的*__init__.py*文件创建蓝图对象，类似于应用程序中的其他蓝图：

*app / api / __ init__.py：API*蓝图构造函数。

```
from flask import Blueprint

bp = Blueprint('api', __name__)

from app.api import users, errors, tokens
```

您可能还记得，有时需要将导入移到底部以避免循环依赖性错误。这就是为什么在创建蓝图后导入*app / api / users.py*，*app / api / errors.py*和*app / api / tokens.py*模块（我还要编写）的原因。

API的内容将存储在*app / api / users.py*模块中。下表总结了我要实现的路由：

| HTTP方法 | 资源网址                  | 笔记                   |
| -------- | ------------------------- | ---------------------- |
| `GET`    | */ API /用户/ <ID>*       | 返回用户。             |
| `GET`    | */ API /用户*             | 返回所有用户的集合。   |
| `GET`    | */ API /用户/ <ID> /跟踪* | 返回此用户的关注者。   |
| `GET`    | */ API /用户/ <ID> /随后* | 返回此用户关注的用户。 |
| `POST`   | */ API /用户*             | 注册一个新的用户帐户。 |
| `PUT`    | */ API /用户/ <ID>*       | 修改用户。             |

现在我将创建一个包含所有这些路径的占位符的骨架模块：

*app / api / users.py*：用户API资源占位符。

```
from app.api import bp

@bp.route('/users/<int:id>', methods=['GET'])
def get_user(id):
    pass

@bp.route('/users', methods=['GET'])
def get_users():
    pass

@bp.route('/users/<int:id>/followers', methods=['GET'])
def get_followers(id):
    pass

@bp.route('/users/<int:id>/followed', methods=['GET'])
def get_followed(id):
    pass

@bp.route('/users', methods=['POST'])
def create_user():
    pass

@bp.route('/users/<int:id>', methods=['PUT'])
def update_user(id):
    pass
```

该*应用程序/ API / errors.py*模块去定义与错误响应处理一些辅助功能。但是现在，我还将使用一个占位符，我将在稍后填写：

*app / api / errors.py*：错误处理占位符。

```
def bad_request():
    pass
```

的*应用程序/ API / tokens.py*是在认证子系统将被定义的模块。这将为不是Web浏览器的客户端提供另一种登录方式。目前，我还要为此模块编写占位符：

*app / api / tokens.py*：令牌处理占位符。

```
def get_token():
    pass

def revoke_token():
    pass
```

新的API蓝图需要在应用程序工厂函数中注册：

*app / __ init__.py*：在应用程序中注册API蓝图。

```
# ...

def create_app(config_class=Config):
    app = Flask(__name__)

    # ...

    from app.api import bp as api_bp
    app.register_blueprint(api_bp, url_prefix='/api')

    # ...
```

## 将用户表示为JSON对象

实现API时要考虑的第一个方面是确定其资源的表示形式。我将实现一个与用户一起工作的API，因此我需要决定用户资源的表示。经过一番头脑风暴，我想出了以下JSON表示：

```
{
    "id": 123,
    "username": "susan",
    "password": "my-password",
    "email": "susan@example.com",
    "last_seen": "2017-10-20T15:04:27Z",
    "about_me": "Hello, my name is Susan!",
    "post_count": 7,
    "follower_count": 35,
    "followed_count": 21,
    "_links": {
        "self": "/api/users/123",
        "followers": "/api/users/123/followers",
        "followed": "/api/users/123/followed",
        "avatar": "https://www.gravatar.com/avatar/..."
    }
}
```

许多字段直接来自用户数据库模型。该`password`字段的特殊之处在于它仅在注册新用户时使用。正如您在[第5章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-v-user-logins)记得的那样，用户密码不存储在数据库中，只有哈希值，因此永远不会返回密码。该`email`字段也是专门处理的，因为我不想公开用户的电子邮件地址。`email`只有当用户请求他们自己的条目时才会返回该字段，但是当他们从其他用户检索条目时不会返回该字段。的`post_count`，`follower_count`和`followed_count`字段是“虚拟”字段，它们不作为数据库中的字段存在，但为方便起见提供给客户端。这是一个很好的示例，它演示了资源表示不需要与服务器中实际资源的定义方式相匹配。

请注意`_links`实现超媒体要求的部分。定义的链接包括指向当前资源的链接，跟随此用户的用户列表，用户所遵循的用户列表，以及最终指向用户头像图像的链接。将来，如果我决定向此API添加帖子，则此处还应包含指向用户帖子列表的链接。

关于JSON格式的一个好处是它总是将表示转换为Python字典或列表。`json`Python标准库中的包负责将Python数据结构转换为JSON和从JSON转换。因此，为了生成这些表示，我将向`User`被调用的模型添加一个方法，该方法`to_dict()`返回一个Python字典：

*app / models.py*：表示的用户模型。

```
from flask import url_for
# ...

class User(UserMixin, db.Model):
    # ...

    def to_dict(self, include_email=False):
        data = {
            'id': self.id,
            'username': self.username,
            'last_seen': self.last_seen.isoformat() + 'Z',
            'about_me': self.about_me,
            'post_count': self.posts.count(),
            'follower_count': self.followers.count(),
            'followed_count': self.followed.count(),
            '_links': {
                'self': url_for('api.get_user', id=self.id),
                'followers': url_for('api.get_followers', id=self.id),
                'followed': url_for('api.get_followed', id=self.id),
                'avatar': self.avatar(128)
            }
        }
        if include_email:
            data['email'] = self.email
        return data
```

这个方法应该大部分是不言自明的，只需生成并返回我用户表示的词典。正如我上面提到的，该`email`字段需要特殊处理，因为我只想在用户请求他们自己的数据时包含电子邮件。所以我正在使用该`include_email`标志来确定该字段是否包含在表示中。

请注意该`last_seen`字段是如何生成的。对于日期和时间字段，我将使用[ISO 8601](https://en.wikipedia.org/wiki/ISO_8601)格式，Python `datetime`可以通过该`isoformat()`方法生成。但是因为我使用的`datetime`是UTC的天真物体，但没有记录状态的时区，我需要`Z`在最后添加，这是ISO 8601的UTC时区代码。

最后，看看我是如何实现超媒体链接的。对于指向我用于`url_for()`生成URL 的三个链接（当前指向我在*app / api / users.py中*定义的占位符视图函数）。头像链接是特殊的，因为它是应用程序外部的Gravatar URL。对于这个链接，我使用与`avatar()`用于在网页中呈现化身的方法相同的方法。

该`to_dict()`方法将用户对象转换为Python表示，然后将其转换为JSON。我还需要查看相反的方向，客户端在请求中传递用户表示，服务器需要解析它并将其转换为`User`对象。以下是`from_dict()`实现从Python字典到模型的转换的方法：

*app / models.py*：表示用户模型。

```
class User(UserMixin, db.Model):
    # ...

    def from_dict(self, data, new_user=False):
        for field in ['username', 'email', 'about_me']:
            if field in data:
                setattr(self, field, data[field])
        if new_user and 'password' in data:
            self.set_password(data['password'])
```

在这种情况下，我决定使用一个循环导入任何客户端可以设置字段，它是的`username`，`email`和`about_me`。对于每个字段，我检查参数中是否有值`data`，如果有，我使用Python `setattr()`在对象的相应属性中设置新值。

该`password`字段被视为特殊情况，因为它不是对象中的字段。所述`new_user`参数确定这是否是一个新的用户注册，这意味着`password`被包括。要在用户模型中设置密码，我调用`set_password()`方法，该方法创建密码哈希。

## 代表用户的集合

除了使用单个资源表示之外，此API还需要一组用户的表示。例如，这将是客户端请求用户列表或关注者时使用的格式。以下是一组用户的表示：

```
{
    "items": [
        { ... user resource ... },
        { ... user resource ... },
        ...
    ],
    "_meta": {
        "page": 1,
        "per_page": 10,
        "total_pages": 20,
        "total_items": 195
    },
    "_links": {
        "self": "http://localhost:5000/api/users?page=1",
        "next": "http://localhost:5000/api/users?page=2",
        "prev": null
    }
}
```

在此表示中，`items`是用户资源列表，每个用户资源都按照上一节中的描述进行定义。该`_meta`部分包括客户端在向用户呈现分页控件时可能会发现有用的集合的元数据。该`_links`部分定义了相关链接，包括指向集合本身的链接，以及上一页和下一页链接，以帮助客户端对列表进行分页。

由于分页逻辑，生成用户集合的表示很棘手，但是逻辑对于我将来可能想要添加到此API的其他资源是常见的，所以我将在通用的方式，然后我可以应用于其他模型。回到[第16章](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvi-full-text-search)我遇到了与全文搜索索引类似的情况，这是我想要一般性地实现的另一个功能，以便它可以应用于任何模型。我使用的解决方案是实现一个`SearchableMixin`类，任何需要全文索引的模型都可以继承。我将使用相同的想法，所以这里有一个我命名的新mixin类`PaginatedAPIMixin`：

*app / models.py：Paginated*表示mixin类。

```
class PaginatedAPIMixin(object):
    @staticmethod
    def to_collection_dict(query, page, per_page, endpoint, **kwargs):
        resources = query.paginate(page, per_page, False)
        data = {
            'items': [item.to_dict() for item in resources.items],
            '_meta': {
                'page': page,
                'per_page': per_page,
                'total_pages': resources.pages,
                'total_items': resources.total
            },
            '_links': {
                'self': url_for(endpoint, page=page, per_page=per_page,
                                **kwargs),
                'next': url_for(endpoint, page=page + 1, per_page=per_page,
                                **kwargs) if resources.has_next else None,
                'prev': url_for(endpoint, page=page - 1, per_page=per_page,
                                **kwargs) if resources.has_prev else None
            }
        }
        return data
```

该`to_collection_dict()`方法产生与所述用户集合表示，包括一个字典`items`，`_meta`和`_links`部分。您可能需要仔细检查该方法以了解其工作原理。前三个参数是Flask-SQLAlchemy查询对象，页码和页面大小。这些是确定将返回的项目的参数。该实现使用`paginate()`查询对象的方法来获取页面值的项目，就像我在索引中的帖子，浏览和Web应用程序的配置文件页面一样。

复杂的部分是生成链接，其中包括自引用以及指向下一页和上一页的链接。我想让这个函数变得通用，所以我不能`url_for('api.get_users', id=id, page=page)`用来生成自我链接。参数`url_for()`将依赖于特定的资源集合，因此我将依赖于调用者在`endpoint`参数中传递需要发送的视图函数`url_for()`。并且由于许多路由都有参数，我还需要捕获其他关键字参数`kwargs`，并将其传递给`url_for()`它们。该`page`和`per_page`查询字符串参数明确给出，因为所有的API路线这些控制分页。

这个mixin类需要`User`作为父类添加到模型中：

*app / models.py*：将PaginatedAPIMixin添加到用户模型。

```
class User(PaginatedAPIMixin, UserMixin, db.Model):
    # ...
```

在收集的情况下，我不需要反向，因为我不会有任何需要客户端发送用户列表的路由。

## 错误处理

我在[第7章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-vii-error-handling)定义的错误页面仅适用于使用Web浏览器与应用程序交互的用户。当API需要返回错误时，它需要是“机器友好”类型的错误，客户端应用程序可以轻松解释。因此，我以相同的方式为JSON中的API资源定义了表示，现在我将决定API错误消息的表示。这是我将要使用的基本结构：

```
{
    "error": "short error description",
    "message": "error message (optional)"
}
```

除了错误的有效负载之外，我还将使用HTTP协议中的状态代码来指示错误的一般类。为了帮助我生成这些错误响应，我将`error_response()`在*app / api / errors.py中*编写该函数：

*app / api / errors.py*：错误响应。

```
from flask import jsonify
from werkzeug.http import HTTP_STATUS_CODES

def error_response(status_code, message=None):
    payload = {'error': HTTP_STATUS_CODES.get(status_code, 'Unknown error')}
    if message:
        payload['message'] = message
    response = jsonify(payload)
    response.status_code = status_code
    return response
```

此函数使用`HTTP_STATUS_CODES`来自Werkzeug（Flask的核心依赖项）的便捷字典，为每个HTTP状态代码提供简短的描述性名称。我`error`在我的错误表示中使用这些名称作为字段，因此我只需要担心数字状态代码和可选的长描述。该`jsonify()`函数返回一个Flask `Response`对象，其默认状态代码为200，因此在创建响应后，我将状态代码设置为错误的正确代码。

API将返回的最常见错误将是代码400，这是“错误请求”的错误。这是客户端发送包含无效数据的请求时使用的错误。为了更容易生成这个错误，我将为它添加一个专用函数，只需要长描述性消息作为参数。这是`bad_request()`我之前添加的占位符：

*app / api / errors.py*：请求响应错误。

```
# ...

def bad_request(message):
    return error_response(400, message)
```

## 用户资源端点

我需要使用JSON表示用户的支持现已完成，因此我已准备好开始编写API端点。

### 检索用户

让我们从检索单个用户的请求开始，由下式给出`id`：

*app / api / users.py*：返回一个用户。

```
from flask import jsonify
from app.models import User

@bp.route('/users/<int:id>', methods=['GET'])
def get_user(id):
    return jsonify(User.query.get_or_404(id).to_dict())
```

视图函数接收`id`请求的用户作为URL中的动态参数。该`get_or_404()`查询对象的方法是非常有用的变体`get()`，你以前见过方法，也返回给定的对象`id`，但是，如果它存在，而不是返回`None`时`id`不存在，这将中止请求，并返回404错误给客户。的优点`get_or_404()`超过`get()`在于，它消除了需要检查查询的结果，从而简化了在视图功能的逻辑。

`to_dict()`我添加的方法`User`用于生成具有所选用户的资源表示的字典，然后Flask的`jsonify()`函数将该字典转换为JSON格式以返回到客户端。

如果要查看第一个API路由的工作原理，请启动服务器，然后在浏览器的地址栏中键入以下URL：

```
http://localhost:5000/api/users/1
```

这应该显示第一个以JSON格式呈现的用户。还尝试使用大`id`值，以查看`get_or_404()`SQLAlchemy查询对象的方法如何触发404错误（稍后我将向您展示如何扩展错误处理，以便这些错误也以JSON格式返回）。

为了测试这条新路线，我将安装[HTTPie](https://httpie.org/)，这是一个用Python编写的命令行HTTP客户端，可以轻松发送API请求：

```
(venv) $ pip install httpie
```

现在我可以请求关于与用户信息`id`的`1`使用下面的命令（这可能是你自己）：

```
(venv) $ http GET http://localhost:5000/api/users/1
HTTP/1.0 200 OK
Content-Length: 457
Content-Type: application/json
Date: Mon, 27 Nov 2017 20:19:01 GMT
Server: Werkzeug/0.12.2 Python/3.6.3

{
    "_links": {
        "avatar": "https://www.gravatar.com/avatar/993c...2724?d=identicon&s=128",
        "followed": "/api/users/1/followed",
        "followers": "/api/users/1/followers",
        "self": "/api/users/1"
    },
    "about_me": "Hello! I'm the author of the Flask Mega-Tutorial.",
    "followed_count": 0,
    "follower_count": 1,
    "id": 1,
    "last_seen": "2017-11-26T07:40:52.942865Z",
    "post_count": 10,
    "username": "miguel"
}
```

### 检索用户的集合

要返回所有用户的集合，我现在可以依赖以下`to_collection_dict()`方法`PaginatedAPIMixin`：

*app / api / users.py*：返回所有用户的集合。

```
from flask import request

@bp.route('/users', methods=['GET'])
def get_users():
    page = request.args.get('page', 1, type=int)
    per_page = min(request.args.get('per_page', 10, type=int), 100)
    data = User.to_collection_dict(User.query, page, per_page, 'api.get_users')
    return jsonify(data)
```

对于此实现，我首先从请求的查询字符串中提取`page`，`per_page`如果未定义，则分别使用默认值1和10。该`per_page`有帽它在100给客户端的控制要求真正大的页面是不是一个好主意，因为这可能会导致服务器性能问题的其他逻辑。然后将`page`和`per_page`参数`to_collection_query()`与查询一起传递给该方法，在这种情况下`User.query`，查询只是返回所有用户的最通用查询。最后一个参数是`api.get_users`，这是表示中使用的三个链接所需的端点名称。

要使用HTTPie测试此端点，请使用以下命令：

```
(venv) $ http GET http://localhost:5000/api/users
```

接下来的两个端点是返回跟随者并跟随用户的端点。这些与上面的相似：

*app / api / users.py*：返回关注者并关注用户。

```
@bp.route('/users/<int:id>/followers', methods=['GET'])
def get_followers(id):
    user = User.query.get_or_404(id)
    page = request.args.get('page', 1, type=int)
    per_page = min(request.args.get('per_page', 10, type=int), 100)
    data = User.to_collection_dict(user.followers, page, per_page,
                                   'api.get_followers', id=id)
    return jsonify(data)

@bp.route('/users/<int:id>/followed', methods=['GET'])
def get_followed(id):
    user = User.query.get_or_404(id)
    page = request.args.get('page', 1, type=int)
    per_page = min(request.args.get('per_page', 10, type=int), 100)
    data = User.to_collection_dict(user.followed, page, per_page,
                                   'api.get_followed', id=id)
    return jsonify(data)
```

由于这两个路由特定于用户，因此它们具有`id`动态参数。将`id`用于从数据库中获取的用户，然后提供`user.followers`和`user.followed`关系基础的查询`to_collection_dict()`，所以希望现在你可以看到为什么花费额外的时间一点点，并且在一般的方式设计这种方法是值得的。最后两个参数`to_collection_dict()`是端点名称，以及该`id`方法将作为额外关键字参数的方法`kwargs`，然后`url_for()`在生成表示的链接部分时将其传递给它。

与前面的示例类似，您可以使用HTTPie执行这两个路由，如下所示：

```
(venv) $ http GET http://localhost:5000/api/users/1/followers
(venv) $ http GET http://localhost:5000/api/users/1/followed
```

我应该注意，由于超媒体，您不需要记住这些URL，因为它们包含在`_links`用户表示的部分中。

### 注册新用户

`POST`对*/ users*路由的请求将用于注册新用户帐户。您可以在下面看到此路线的实施：

*app / api / users.py*：注册一个新用户。

```
from flask import url_for
from app import db
from app.api.errors import bad_request

@bp.route('/users', methods=['POST'])
def create_user():
    data = request.get_json() or {}
    if 'username' not in data or 'email' not in data or 'password' not in data:
        return bad_request('must include username, email and password fields')
    if User.query.filter_by(username=data['username']).first():
        return bad_request('please use a different username')
    if User.query.filter_by(email=data['email']).first():
        return bad_request('please use a different email address')
    user = User()
    user.from_dict(data, new_user=True)
    db.session.add(user)
    db.session.commit()
    response = jsonify(user.to_dict())
    response.status_code = 201
    response.headers['Location'] = url_for('api.get_user', id=user.id)
    return response
```

此请求将接受来自客户端的JSON格式的用户表示，在请求正文中提供。Flask提供了`request.get_json()`从请求中提取JSON并将其作为Python结构返回的方法。`None`如果在请求中找不到JSON数据，则此方法返回，因此我可以确保始终使用表达式获取字典`request.get_json() or {}`。

在我可以使用数据之前，我需要确保我已经掌握了所有信息，因此我首先检查是否包含了三个必填字段。这些是`username`，`email`和`password`。如果缺少任何一个，那么我使用*app / api / errors.py*模块中的`bad_request()`帮助函数将错误返回给客户端。除了该检查之外，我还需要确保其他用户尚未使用和字段，因此我尝试通过提供的用户名和电子邮件从数据库加载用户，如果其中任何一个返回有效用户，我也将错误返回给客户端。`username``email`

一旦我通过了数据验证，我就可以轻松创建用户对象并将其添加到数据库中。要创建用户，我依赖于模型中的`from_dict()`方法`User`。该`new_user`参数被设置为`True`，使得其还接受`password`其通常不是用户表示的部分字段。

我为此请求返回的响应将是新用户的表示，因此`to_dict()`生成该有效负载。`POST`创建资源的请求的状态代码应为201，即创建新实体时使用的代码。此外，HTTP协议要求201响应包括`Location`设置为新资源的URL的标头。

您可以在下面看到如何通过HTTPie从命令行注册新用户：

```
(venv) $ http POST http://localhost:5000/api/users username=alice password=dog \
    email=alice@example.com "about_me=Hello, my name is Alice!"
```

### 编辑用户

我将在API中使用的最后一个端点是修改现有用户的端点：

*app / api / users.py*：修改用户。

```
@bp.route('/users/<int:id>', methods=['PUT'])
def update_user(id):
    user = User.query.get_or_404(id)
    data = request.get_json() or {}
    if 'username' in data and data['username'] != user.username and \
            User.query.filter_by(username=data['username']).first():
        return bad_request('please use a different username')
    if 'email' in data and data['email'] != user.email and \
            User.query.filter_by(email=data['email']).first():
        return bad_request('please use a different email address')
    user.from_dict(data, new_user=False)
    db.session.commit()
    return jsonify(user.to_dict())
```

对于此请求，我收到一个用户`id`作为URL的动态部分，因此我可以加载指定的用户并返回404错误（如果找不到）。就像新用户的情况一样，我需要验证客户端提供的字段`username`和`email`字段在我可以使用之前不会与其他用户发生冲突，但在这种情况下，验证有点棘手。首先，这些字段在此请求中是可选的，因此我需要检查字段是否存在。第二个复杂因素是客户端可能提供相同的值，因此在检查用户名或电子邮件是否被采用之前，我需要确保它们与当前的不同。如果这些验证检查中的任何一个失败，那么我将像以前一样将400错误返回给客户端。

验证数据后，我可以使用模型的`from_dict()`方法`User`导入客户端提供的所有数据，然后将更改提交到数据库。此请求的响应将更新的用户表示返回给用户，并使用默认的200状态代码。

以下是`about_me`使用HTTPie 编辑字段的示例请求：

```
(venv) $ http PUT http://localhost:5000/api/users/2 "about_me=Hi, I am Miguel"
```

## API身份验证

我在上一节中添加的API端点目前对任何客户端开放。显然，他们只需要注册用户可用，为此，我需要添加*身份验证*和*授权*，或者简称“AuthN”和“AuthZ”。这个想法是客户端发送的请求提供某种标识，以便服务器知道客户端代表什么用户，并且可以验证该用户是否允许所请求的操作。

保护这些API端点最明显的方法是使用`@login_required`Flask-Login中的装饰器，但这种方法存在一些问题。当装饰器检测到未经过身份验证的用户时，它会将用户重定向到HTML登录页面。在API中没有HTML或登录页面的概念，如果客户端发送具有无效或缺少凭证的请求，则服务器必须拒绝返回401状态代码的请求。服务器不能假设API客户端是Web浏览器，或者它可以处理重定向，或者它可以呈现和处理HTML登录表单。当API客户端收到401状态代码时，它知道它需要向用户询问凭据，但它是如何做到的，这实际上不是服务器的业务。

### 用户模型中的标记

对于API身份验证需求，我将使用*令牌*身份验证方案。当客户端想要开始与API交互时，它需要请求临时令牌，使用用户名和密码进行身份验证。然后，只要令牌有效，客户端就可以发送传递令牌作为身份验证的API请求。令牌过期后，需要请求新令牌。为了支持用户令牌，我将扩展`User`模型：

*app / models.py*：支持用户令牌。

```
import base64
from datetime import datetime, timedelta
import os

class User(UserMixin, PaginatedAPIMixin, db.Model):
    # ...
    token = db.Column(db.String(32), index=True, unique=True)
    token_expiration = db.Column(db.DateTime)

    # ...

    def get_token(self, expires_in=3600):
        now = datetime.utcnow()
        if self.token and self.token_expiration > now + timedelta(seconds=60):
            return self.token
        self.token = base64.b64encode(os.urandom(24)).decode('utf-8')
        self.token_expiration = now + timedelta(seconds=expires_in)
        db.session.add(self)
        return self.token

    def revoke_token(self):
        self.token_expiration = datetime.utcnow() - timedelta(seconds=1)

    @staticmethod
    def check_token(token):
        user = User.query.filter_by(token=token).first()
        if user is None or user.token_expiration < datetime.utcnow():
            return None
        return user
```

通过此更改，我`token`将向用户模型添加一个属性，因为我将需要通过它搜索数据库，所以我将其设置为唯一并编制索引。我还补充说`token_expiration`，它有令牌到期的日期和时间。这使得令牌在很长一段时间内不会保持有效，这可能成为安全风险。

我创建了三种方法来处理这些令牌。该`get_token()`方法为用户返回一个令牌。令牌生成为在base64中编码的随机字符串，以便所有字符都在可读范围内。在创建新令牌之前，此方法检查当前分配的令牌是否在到期前至少还剩一分钟，并且在这种情况下返回现有令牌。

使用令牌时，最好立即采取策略撤销令牌，而不是仅仅依赖于到期日期。这是一种经常被忽视的安全性最佳实践。该`revoke_token()`方法使得当前分配给用户的令牌无效，只需将到期日期设置为当前时间之前的一秒即可。

该`check_token()`方法是一种静态方法，它将令牌作为输入，并将此令牌所属的用户作为响应返回。如果令牌无效或已过期，则该方法返回`None`。

因为我对数据库进行了更改，所以我需要生成一个新的数据库迁移，然后用它来升级数据库：

```
(venv) $ flask db migrate -m "user tokens"
(venv) $ flask db upgrade
```

### 令牌请求

编写API时，必须考虑到客户端并不总是连接到Web应用程序的Web浏览器。当智能手机应用程序等独立客户端甚至基于浏览器的单页应用程序可以访问后端服务时，API的真正威力就来了。当这些专业客户端需要访问API服务时，它们首先请求令牌，该令牌与传统Web应用程序中的登录表单相对应。

为了在使用令牌认证时简化客户端和服务器之间的交互，我将使用名为[Flask-HTTPAuth](https://flask-httpauth.readthedocs.io/)的Flask扩展。使用pip安装Flask-HTTPAuth：

```
(venv) $ pip install flask-httpauth
```

Flask-HTTPAuth支持一些不同的身份验证机制，所有API都很友好。首先，我将使用[HTTP基本身份验证](https://en.wikipedia.org/wiki/Basic_access_authentication)，其中客户端在标准的[授权](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization) HTTP标头中发送用户凭据。要与Flask-HTTPAuth集成，应用程序需要提供两个功能：一个定义用于检查用户提供的用户名和密码的逻辑，另一个在认证失败的情况下返回错误响应。这些函数通过装饰器在Flask-HTTPAuth中注册，然后在认证流程中根据需要由扩展自动调用。你可以看到实现：

*app / api / auth.py*：基本身份验证支持。

```
from flask import g
from flask_httpauth import HTTPBasicAuth
from app.models import User
from app.api.errors import error_response

basic_auth = HTTPBasicAuth()

@basic_auth.verify_password
def verify_password(username, password):
    user = User.query.filter_by(username=username).first()
    if user is None:
        return False
    g.current_user = user
    return user.check_password(password)

@basic_auth.error_handler
def basic_auth_error():
    return error_response(401)
```

`HTTPBasicAuth`Flask-HTTPAuth中的类是实现基本身份验证流的类。这两个必需的函数分别通过`verify_password`和`error_handler`装饰器配置。

验证功能接收客户端提供的用户名和密码，`True`如果凭证有效或`False`不存在，则返回。要检查密码，我依赖于类的`check_password()`方法，`User`在Web应用程序的身份验证期间，Flask-Login也使用该方法。我保存了经过身份验证的用户`g.current_user`，以便我可以从API视图函数中访问它。

错误处理程序函数只返回由*app / api / errors.py中*的`error_response()`函数生成的401错误。401错误在HTTP标准中定义为“未授权”错误。HTTP客户端知道，当他们收到此错误时，他们发送的请求需要使用有效凭据重新发送。

现在我已经实现了基本的身份验证支持，因此我可以添加客户端在需要令牌时将调用的令牌检索路由：

*app / api / tokens.py*：生成用户令牌。

```
from flask import jsonify, g
from app import db
from app.api import bp
from app.api.auth import basic_auth

@bp.route('/tokens', methods=['POST'])
@basic_auth.login_required
def get_token():
    token = g.current_user.get_token()
    db.session.commit()
    return jsonify({'token': token})
```

此视图函数使用实例中的`@basic_auth.login_required`装饰器进行修饰`HTTPBasicAuth`，这将指示Flask-HTTPAuth验证身份验证（通过上面定义的验证函数），并且只允许函数在提供的凭据有效时运行。此视图函数的实现依赖于`get_token()`用户模型生成令牌的方法。生成令牌后会发出数据库提交，以确保将令牌及其过期写回数据库。

如果您尝试向令牌API路由发送POST请求，则会发生以下情况：

```
(venv) $ http POST http://localhost:5000/api/tokens
HTTP/1.0 401 UNAUTHORIZED
Content-Length: 30
Content-Type: application/json
Date: Mon, 27 Nov 2017 20:01:00 GMT
Server: Werkzeug/0.12.2 Python/3.6.3
WWW-Authenticate: Basic realm="Authentication Required"

{
    "error": "Unauthorized"
}
```

HTTP响应包括401状态代码，以及我在`basic_auth_error()`函数中定义的错误有效负载。这是相同的请求，这次包括基本身份验证凭据：

```
(venv) $ http --auth <username>:<password> POST http://localhost:5000/api/tokens
HTTP/1.0 200 OK
Content-Length: 50
Content-Type: application/json
Date: Mon, 27 Nov 2017 20:01:22 GMT
Server: Werkzeug/0.12.2 Python/3.6.3

{
    "token": "pC1Nu9wwyNt8VCj1trWilFdFI276AcbS"
}
```

现在状态代码是200，这是成功请求的代码，并且有效负载包括为用户新生成的令牌。请注意，发送此请求时，您需要`<username>:<password>`使用自己的凭据进行替换。用户名和密码需要以冒号作为分隔符提供。

### 使用令牌保护API路由

客户端现在可以请求令牌与API端点一起使用，因此剩下的就是向这些端点添加令牌验证。这是Flask-HTTPAuth也可以为我处理的事情。我需要根据类创建第二个身份验证实例`HTTPTokenAuth`，并提供令牌验证回调：

*app / api / auth.py*：令牌认证支持。

```
# ...
from flask_httpauth import HTTPTokenAuth

# ...
token_auth = HTTPTokenAuth()

# ...

@token_auth.verify_token
def verify_token(token):
    g.current_user = User.check_token(token) if token else None
    return g.current_user is not None

@token_auth.error_handler
def token_auth_error():
    return error_response(401)
```

使用令牌身份验证时，Flask-HTTPAuth使用`verify_token`修饰函数，但除此之外，令牌身份验证的工作方式与基本身份验证相同。我的令牌验证功能用于`User.check_token()`定位拥有所提供令牌的用户。该函数还通过将当前用户设置为来处理丢失令牌的情况`None`。的`True`或`False`返回值确定是否烧瓶HTTPAuth允许运行或不视图函数。

要使用令牌保护API路由，`@token_auth.login_required`需要添加装饰器：

*app / api / users.py*：使用令牌身份验证保护用户路由。

```
from app.api.auth import token_auth

@bp.route('/users/<int:id>', methods=['GET'])
@token_auth.login_required
def get_user(id):
    # ...

@bp.route('/users', methods=['GET'])
@token_auth.login_required
def get_users():
    # ...

@bp.route('/users/<int:id>/followers', methods=['GET'])
@token_auth.login_required
def get_followers(id):
    # ...

@bp.route('/users/<int:id>/followed', methods=['GET'])
@token_auth.login_required
def get_followed(id):
    # ...

@bp.route('/users', methods=['POST'])
def create_user():
    # ...

@bp.route('/users/<int:id>', methods=['PUT'])
@token_auth.login_required
def update_user(id):
    # ...
```

请注意，装饰器被添加到所有API视图函数中`create_user()`，但显然不能接受验证，因为需要首先创建将请求令牌的用户。

如果您向任何这些端点发送请求，如前所示，您将收到401错误响应。要获得访问权限，您需要添加`Authorization`标头，并将您从请求中收到的令牌添加到*/ api / tokens*。Flask-HTTPAuth期望令牌作为“承载”令牌发送，HTTPie不直接支持该令牌。对于使用用户名和密码的基本身份验证，HTTPie提供了一个`--auth`选项，但是对于令牌，需要明确提供标头。以下是发送承载令牌的语法：

```
(venv) $ http GET http://localhost:5000/api/users/1 \
    "Authorization:Bearer pC1Nu9wwyNt8VCj1trWilFdFI276AcbS"
```

### 撤销令牌

我要实现的最后一个令牌相关功能是令牌撤销，您可以在下面看到：

*app / api / tokens.py*：撤销令牌。

```
from app.api.auth import token_auth

@bp.route('/tokens', methods=['DELETE'])
@token_auth.login_required
def revoke_token():
    g.current_user.revoke_token()
    db.session.commit()
    return '', 204
```

客户端可以向*/令牌* URL 发送`DELETE`请求以使*令牌*无效。此路由的身份验证是基于令牌的，实际上`Authorization`标头中发送的令牌是被撤销的令牌。撤销本身使用类中的辅助方法`User`，它重置令牌上的失效日期。提交数据库会话，以便将此更改写入数据库。此请求的响应没有正文，因此我可以返回一个空字符串。return语句中的第二个值将响应的状态代码设置为204，这是用于没有响应主体的成功请求的代码。

以下是从HTTPie发送的示例令牌撤销请求：

```
(venv) $ http DELETE http://localhost:5000/api/tokens \
    Authorization:"Bearer pC1Nu9wwyNt8VCj1trWilFdFI276AcbS"
```

## API友好错误消息

你还记得当我要求你从浏览器发送带有无效用户URL的API请求时，本章前面发生了什么？服务器返回404错误，但此错误被格式化为标准404 HTML错误页面。API可能需要返回的许多错误都可以使用API蓝图中的JSON版本覆盖，但Flask仍然会处理一些错误，这些错误仍然通过全局为应用程序注册的错误处理程序，并且这些错误继续返回HTML 。

HTTP协议支持一种机制，通过该机制，客户端和服务器可以就响应的最佳格式达成一致，称为*内容协商*。客户端需要发送`Accept`带有请求的标头，指示格式首选项。然后，服务器查看列表，并使用客户端提供的列表中支持的最佳格式进行响应。

我想要做的是修改全局应用程序错误处理程序，以便它们使用内容协商根据客户端首选项以HTML或JSON进行回复。这可以使用`request.accept_mimetypes`Flask中的对象完成：

*app / errors / handlers.py*：错误响应的内容协商。

```
from flask import render_template, request
from app import db
from app.errors import bp
from app.api.errors import error_response as api_error_response

def wants_json_response():
    return request.accept_mimetypes['application/json'] >= \
        request.accept_mimetypes['text/html']

@bp.app_errorhandler(404)
def not_found_error(error):
    if wants_json_response():
        return api_error_response(404)
    return render_template('errors/404.html'), 404

@bp.app_errorhandler(500)
def internal_error(error):
    db.session.rollback()
    if wants_json_response():
        return api_error_response(500)
    return render_template('errors/500.html'), 500
```

该`wants_json_response()`辅助函数比较由所述客户机在其的优选格式列表选择用于JSON或HTML的偏好。如果JSON的速率高于HTML，那么我将返回JSON响应。否则，我将根据模板返回原始HTML响应。对于JSON响应，我`error_response`将从API蓝图中导入辅助函数，但是在这里我将重命名它`api_error_response()`以便清楚它的作用和来源。