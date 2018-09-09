# 14 Ajax

在本文中，我将脱离服务器端开发的“安全区域”，并开发具有同样重要的服务器和客户端组件的功能。您是否看到过某些网站在用户生成的内容旁边显示的“翻译”链接？这些链接触发不是用户母语的内容的实时自动翻译。翻译的内容通常插入原始版本之下。Google将其显示为外语搜索结果。Facebook是为帖子做的。Twitter是为推文做的。今天我将向您展示如何将相同的功能添加到微博！

*本章的GitHub链接是：Browse，Zip，Diff。*

## 服务器端与客户端

在我到目前为止遵循的传统服务器端模型中，有一个客户端（用户命令的Web浏览器）向应用程序服务器发出HTTP请求。请求可以简单地询问HTML页面，例如当您单击“配置文件”链接时，或者它可以触发操作，例如在您编辑配置文件信息后单击“提交”按钮时。在这两种类型的请求中，服务器通过直接或通过发出重定向向客户端发送新网页来完成请求。然后客户端用新的页面替换当前页面。只要用户停留在应用程序的网站上，此循环就会重复。在此模型中，服务器完成所有工作，而客户端只显示网页并接受用户输入。

有一种不同的模型，客户端扮演更积极的角色。在此模型中，客户端向服务器发出请求，服务器响应网页，但与前一种情况不同，并非所有页面数据都是HTML，页面上还有代码部分，通常用Javascript编写。一旦客户端收到页面，它就会显示HTML部分，然后执行代码。从那时起，您就拥有了一个活跃的客户端，可以独立完成工作而不需要与服务器接触。在严格的客户端应用程序中，整个应用程序通过初始页面请求下载到客户端，然后应用程序完全在客户端上运行，只接触服务器以检索或存储数据并对其外观进行动态更改而且只有网页。这种类型的应用程序被调用[单页应用程序](http://en.wikipedia.org/wiki/Single-page_application)或SPA。

大多数应用程序是两种模型之间的混合，并结合了两者的技术。我的Microblog应用程序主要是服务器端应用程序，但今天我将添加一些客户端操作。要对用户帖子进行实时翻译，客户端浏览器将向服务器发送*异步请求*，服务器将响应该请求而不会导致页面刷新。然后，客户端将动态地将翻译插入当前页面。这种技术称为[Ajax](http://en.wikipedia.org/wiki/Ajax_(programming))，它是异步JavaScript和XML的缩写（即使现在XML经常被JSON替换）。

## 实时翻译工作流程

由于Flask-Babel，该应用程序对外语有很好的支持，这可以支持尽可能多的语言，因为我可以找到翻译器。但当然，缺少一个元素。用户将使用他们自己的语言撰写博客文章，因此用户很可能会遇到使用未知语言编写的帖子。自动翻译的质量并不总是很好，但在大多数情况下，如果您想要的是对另一种语言的文本意味着什么的基本概念，那就足够了。

这是作为Ajax服务实现的理想功能。考虑索引或探索页面可能会显示几个帖子，其中一些可能是外语。如果我使用传统的服务器端技术实现翻译，请求翻译将导致原始页面被新页面替换。事实上，要求翻译许多显示的博客帖子中的一个并不足以要求完整页面更新，如果将翻译后的文本动态插入到原始文本下方，则此功能可以更好地工作，同时保留其余部分页面未触动过。

实现实时自动翻译需要几个步骤。首先，我需要一种方法来识别要翻译的文本的源语言。我还需要知道每个用户的首选语言，因为我只想为其他语言编写的帖子显示“翻译”链接。当提供翻译链接并且用户点击它时，我将需要将Ajax请求发送到服务器，并且服务器将联系第三方翻译API。一旦服务器发回带有翻译文本的响应，客户端javascript代码将动态地将此文本插入页面。正如你可以肯定的那样，这里有一些非平凡的问题。我将逐一看一下这些。

## 语言识别

第一个问题是确定一篇文章所写的语言。这不是一门精确的科学，因为并不总是能够明确地检测出一种语言，但在大多数情况下，自动检测工作得相当好。在Python中，有一个很好的语言检测库叫做`guess_language`。这个包的原始版本相当陈旧，从未移植到Python 3，因此我将安装支持Python 2和3的派生版本：

```
(venv) $ pip install guess_language-spirit
```

计划是将每个博客帖子提供给此包，以尝试确定语言。由于进行此分析有点耗时，因此我不希望每次将帖子呈现到页面时都重复此工作。我要做的是在提交帖子时设置帖子的源语言。然后，检测到的语言将存储在posts表中。

第一步是向模型添加一个`language`字段`Post`：

*app / models.py*：将检测到的语言添加到Post模型。

```
class Post(db.Model):
    # ...
    language = db.Column(db.String(5))
```

您还记得，每次对数据库模型进行更改时，都需要发出数据库迁移：

```
(venv) $ flask db migrate -m "add language to posts"
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added column 'post.language'
  Generating migrations/versions/2b017edaa91f_add_language_to_posts.py ... done
```

然后需要将迁移应用于数据库：

```
(venv) $ flask db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Upgrade ae346256b650 -> 2b017edaa91f, add language to posts
```

我现在可以在提交帖子时检测并存储语言：

*app / routes.py*：保存新帖子的语言。

```
from guess_language import guess_language

@app.route('/', methods=['GET', 'POST'])
@app.route('/index', methods=['GET', 'POST'])
@login_required
def index():
    form = PostForm()
    if form.validate_on_submit():
        language = guess_language(form.post.data)
        if language == 'UNKNOWN' or len(language) > 5:
            language = ''
        post = Post(body=form.post.data, author=current_user,
                    language=language)
        # ...
```

通过此更改，每次提交帖子时，我都会通过该`guess_language`函数运行文本以尝试确定语言。如果语言以未知方式返回或者我得到意外长的结果，我会安全地播放它并将空字符串保存到数据库中。我将采用这样的约定，即任何将语言设置为空字符串的帖子都假定为具有未知语言。

## 显示“翻译”链接

第二步很简单。我现在要做的是在任何不属于当前用户活动语言的帖子旁边添加“翻译”链接。

*app / templates / _post.html*：为帖子添加翻译链接。

```
                {% if post.language and post.language != g.locale %}
                <br><br>
                <a href="#">{{ _('Translate') }}</a>
                {% endif %}
```

我在`_post.html`子模板中执行此操作，以便此功能显示在显示博客帖子的任何页面上。翻译链接仅出现在检测到语言的帖子上，并且此语言与使用Flask-Babel `localeselector`装饰器修饰的功能选择的语言不匹配。回忆一下[第13章](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xiii-i18n-and-l10n)，所选语言环境存储为`g.locale`。链接的文本需要以Flask-Babel可以翻译的方式添加，所以我`_()`在定义它时使用了该函数。

请注意，我还没有与此链接关联的操作。首先，我想弄清楚如何进行实际的翻译。

## 使用第三方翻译服务

两个主要的翻译服务是[Google Cloud Translation API](https://developers.google.com/translate/)和[Microsoft Translator Text API](https://www.microsoft.com/en-us/translator/business/)。两者都是付费服务，但微软提供的入门级选项适用于免费的少量翻译。Google过去提供免费翻译服务，但今天即使是最低的服务等级也是如此。因为我希望能够在不产生费用的情况下试验翻译，所以我将实施Microsoft解决方案。

在使用Microsoft Translator API之前，您需要获得[Azure的](https://azure.com/) Microsoft 帐户。您可以选择免费套餐，同时在注册过程中会要求您提供信用卡号码，当您保持该级别的服务时，不会向您的信用卡收费。

获得Azure帐户后，转到Azure门户并单击左上角的“新建”按钮，然后键入或选择“转换器文本API”。单击“创建”按钮后，您将看到一个表单，您可以在其中定义将添加到您帐户的新翻译资源。您可以在下面看到我如何填写表格：

![Azure转换器](https://blog.miguelgrinberg.com/static/images/mega-tutorial/ch15-azure-translator.png)

再次单击“创建”按钮后，翻译器API资源将添加到您的帐户中。如果您想要几秒钟，您将在顶部栏中收到部署翻译器资源的通知。单击通知中的“转到资源”按钮，然后单击左侧栏上的“密钥”选项。您现在将看到两个键，标记为“键1”和“键2”。将其中一个键复制到剪贴板，然后将其输入到终端中的环境变量中（如果您使用的是Microsoft Windows，则替换`export`为`set`）：

```
(venv) $ export MS_TRANSLATOR_KEY=<paste-your-key-here>
```

此密钥用于通过转换服务进行身份验证，因此需要将其添加到应用程序配置中：

*config.py*：将Microsoft Translator API密钥添加到配置中。

```
class Config(object):
    # ...
    MS_TRANSLATOR_KEY = os.environ.get('MS_TRANSLATOR_KEY')
```

与配置值一样，我更喜欢将它们安装在环境变量中，并从那里将它们导入Flask配置中。这对于能够访问第三方服务的敏感信息（如密钥或密码）尤为重要。您绝对不希望在代码中明确地编写它们。

Microsoft Translator API是一种接受HTTP请求的Web服务。Python中有一些HTTP客户端，但最流行和最简单易用的是`requests`包。所以让我们将其安装到虚拟环境中：

```
(venv) $ pip install requests
```

您可以在下面看到我使用Microsoft Translator API编写的用于翻译文本的功能。我正在推出一个新的*app / translate.py*模块：

*app / translate.py*：文本翻译功能。

```
import json
import requests
from flask_babel import _
from app import app

def translate(text, source_language, dest_language):
    if 'MS_TRANSLATOR_KEY' not in app.config or \
            not app.config['MS_TRANSLATOR_KEY']:
        return _('Error: the translation service is not configured.')
    auth = {'Ocp-Apim-Subscription-Key': app.config['MS_TRANSLATOR_KEY']}
    r = requests.get('https://api.microsofttranslator.com/v2/Ajax.svc'
                     '/Translate?text={}&from={}&to={}'.format(
                         text, source_language, dest_language),
                     headers=auth)
    if r.status_code != 200:
        return _('Error: the translation service failed.')
    return json.loads(r.content.decode('utf-8-sig'))
```

该函数将文本转换为源语言和目标语言代码作为参数，并返回带有翻译文本的字符串。首先检查配置中是否存在转换服务的密钥，如果不存在，则返回错误。错误也是一个字符串，因此从外部看，它看起来就像翻译的文本。这确保了在出现错误的情况下，用户将看到有意义的错误消息。

包中的`get()`方法`requests`将带有`GET`方法的HTTP请求发送到作为第一个参数给出的URL。我正在使用*/v2/Ajax.svc/Translate* URL，它是翻译服务的端点，它将翻译作为JSON有效负载返回。需要将文本，源和目标语言作为URL中的查询字符串参数给出`text`，`from`并`to`分别命名。要对服务进行身份验证，我需要传递我添加到配置中的密钥。需要在具有名称的自定义HTTP标头中提供此密钥`Ocp-Apim-Subscription-Key`。我创建了`auth`这个头字典，然后把它传递给`requests`在`headers`争论。

该`requests.get()`方法返回一个响应对象，该对象包含服务提供的所有详细信息。我首先需要检查状态代码是否为200，这是成功请求的代码。如果我得到任何其他代码，我知道有一个错误，所以在这种情况下我返回一个错误字符串。如果状态代码是200，那么响应的主体有一个带有转换的JSON编码字符串，所以我需要做的就是使用`json.loads()`Python标准库中的函数将JSON解码为我可以使用的Python字符串。`content`响应对象的属性包含响应的原始主体作为bytes对象，该对象被转换为UTF-8字符串并发送到`json.loads()`。

下面你可以看到我使用新`translate()`函数的Python控制台会话：

```
>>> from app.translate import translate
>>> translate('Hi, how are you today?', 'en', 'es')  # English to Spanish
'Hola, ¿cómo estás hoy?'
>>> translate('Hi, how are you today?', 'en', 'de')  # English to German
'Are Hallo, how you heute?'
>>> translate('Hi, how are you today?', 'en', 'it')  # English to Italian
'Ciao, come stai oggi?'
>>> translate('Hi, how are you today?', 'en', 'fr')  # English to French
"Salut, comment allez-vous aujourd'hui ?"
```

很酷，对吗？现在是时候将此功能与应用程序集成。

## 来自服务器的Ajax

我将从实现服务器端部分开始。当用户单击帖子下方显示的翻译链接时，将向服务器发出异步HTTP请求。我将在下一个会话中向您展示如何执行此操作，因此现在我将专注于实现服务器对此请求的处理。

异步（或Ajax）请求类似于我在应用程序中创建的路由和视图函数，唯一的区别是它不是返回HTML或重定向，而是返回格式为[XML](http://en.wikipedia.org/wiki/XML)或更常见的[JSON的数据](http://en.wikipedia.org/wiki/JSON)。您可以在下面看到翻译视图功能，该功能调用Microsoft Translator API，然后以JSON格式返回翻译后的文本：

*app / routes.py*：文本翻译视图功能。

```
from flask import jsonify
from app.translate import translate

@app.route('/translate', methods=['POST'])
@login_required
def translate_text():
    return jsonify({'text': translate(request.form['text'],
                                      request.form['source_language'],
                                      request.form['dest_language'])})
```

如您所见，这很简单。我将此路由实现为`POST`请求。关于什么时候使用`GET`或`POST`（或者你还没有看过的其他请求方法），确实没有绝对的规则。由于客户端将发送数据，我决定使用`POST`请求，因为这类似于提交表单数据的请求。该`request.form`属性是Flask公开的字典，其中包含提交中包含的所有数据。当我使用Web表单时，我不需要查看，`request.form`因为Flask-WTF为我做了所有工作，但在这种情况下，实际上没有Web表单，所以我必须直接访问数据。

所以我在这个函数中做的是调用`translate()`上一节中的函数直接从请求提交的数据中传递三个参数。结果被合并到密钥下的单键字典中，`text`字典作为参数传递给Flask的`jsonify()`函数，该函数将字典转换为JSON格式的有效负载。返回值`jsonify()`是将要发送回客户端的HTTP响应。

例如，如果客户端想要将字符串转换`Hello, World!`为西班牙语，则此请求的响应将具有以下有效负载：

```
{ "text": "Hola, Mundo!" }
```

## 来自客户端的Ajax

现在服务器能够通过*/ translate* URL 提供翻译，当用户点击我上面添加的“翻译”链接，将文本传递给翻译以及源语言和目标语言时，我需要调用此URL。如果您不熟悉在浏览器中使用JavaScript，这将是一个很好的学习体验。

在浏览器中使用JavaScript时，当前显示的页面在内部表示为文档对象模型或仅作为DOM。这是一个层次结构，它引用页面中存在的所有元素。在此上下文中运行的JavaScript代码可以更改DOM以触发页面中的更改。

我们首先讨论我在浏览器中运行的JavaScript代码如何获取我需要发送到服务器中运行的translate函数的三个参数。要获取文本，我需要在DOM中找到包含博客帖子主体的节点并读取其内容。为了便于识别包含博客帖子的DOM节点，我将为它们附加一个唯一的ID。如果您查看*_post.html*模板，则呈现帖子正文的行只会读取`{{ post.body }}`。我要做的是将这个内容包装在一个`<span>`元素中。这不会在视觉上改变任何东西，但它给了我一个可以插入标识符的地方：

*app / templates / _post.html*：为每篇博文添加ID。

```
                <span id="post{{ post.id }}">{{ post.body }}</span>
```

这将为每个博客帖子分配一个唯一的标识符，格式为`post1`，`post2`等等，其中数字与每个帖子的数据库标识符匹配。现在每篇博文都有一个唯一的标识符，给定一个ID值，我可以使用jQuery来定位该`<span>`帖子的元素并提取其中的文本。例如，如果我想获取ID为123的帖子的文本，那就是我要做的：

```
$('#post123').text()
```

这里的`$`标志是jQuery库提供的函数的名称。这个库由Bootstrap使用，因此Flask-Bootstrap已经包含它。它`#`是jQuery使用的“selector”语法的一部分，这意味着接下来是元素的ID。

我还希望有一个地方，一旦我从服务器收到翻译文本，我将插入它。我将要做的是将“翻译”链接替换为翻译文本，因此我还需要为该节点提供唯一标识符：

*app / templates / _post.html*：在翻译链接中添加ID。

```
                <span id="translation{{ post.id }}">
                    <a href="#">{{ _('Translate') }}</a>
                </span>
```

所以现在对于给定的帖子ID，我有一个`post<ID>`博客帖子的节点，以及一个相应的`translation<ID>`节点，我需要用翻译后的文本替换翻译文本。

下一步是编写一个可以完成所有翻译工作的函数。此函数将使用输入和输出DOM节点以及源语言和目标语言，使用所需的三个参数向服务器发出异步请求，最后在服务器响应后将Translate链接替换为已翻译的文本。这听起来像很多工作，但实现起来相当简单：

*app / templates / base.html*：客户端翻译功能。

```
{% block scripts %}
    ...
    <script>
        function translate(sourceElem, destElem, sourceLang, destLang) {
            $(destElem).html('<img src="{{ url_for('static', filename='loading.gif') }}">');
            $.post('/translate', {
                text: $(sourceElem).text(),
                source_language: sourceLang,
                dest_language: destLang
            }).done(function(response) {
                $(destElem).text(response['text'])
            }).fail(function() {
                $(destElem).text("{{ _('Error: Could not contact server.') }}");
            });
        }
    </script>
{% endblock %}
```

前两个参数是post和translate链接节点的唯一ID。最后两个参数是源语言和目标语言代码。

该函数以一个很好的触摸开始：它添加一个*微调器*替换Translate链接，以便用户知道翻译正在进行中。这是通过jQuery完成的，使用该`$(destElem).html()`函数将基于链接的新HTML内容替换定义翻译链接的原始HTML `<img>`。对于微调器，我将使用我添加到*app / static / loading.gif*目录的小动画GIF，Flask为静态文件保留。要生成引用此图像的URL，我正在使用该`url_for()`函数，传递特殊路径名称`static`并将图像的文件名作为参数。您可以在本章的[下载包中](https://github.com/miguelgrinberg/microblog/tree/v0.14)找到*loading.gif*图像。

所以现在我有一个很好的微调器取代了Translate链接，因此用户知道等待翻译出现。下一步是将`POST`请求发送到我在上一节中定义的*/ translate* URL。为此，我还将使用jQuery，在这种情况下是`$.post()`函数。此函数以类似于浏览器提交Web表单的格式向服务器提交数据，这很方便，因为这允许Flask将此数据合并到`request.form`字典中。参数`$.post()`是两个，首先是发送请求的URL，然后是服务器期望的三个数据项的字典（或者在JavaScript中调用的对象）。

您可能知道JavaScript可以使用回调函数或更高级的回调形式（称为*promises）*。我现在要做的是在请求完成并且浏览器收到响应后指示我想要做什么。在JavaScript中没有等待某事的东西，一切都是*异步的*。我需要做的是提供一个回调函数，浏览器在收到响应时会调用它。而且作为一种使所有内容尽可能健壮的方法，我想指出在出现错误的情况下该怎么做，这样才能成为处理错误的第二个回调函数。有几种方法可以指定这些回调，但是对于这种情况，使用promises会使代码相当清晰。语法如下：

```
$.post(<url>, <data>).done(function(response) {
    // success callback
}).fail(function() {
    // error callback
})
```

promise语法允许您基本上将回调“链接”到调用的返回值`$.post()`。在成功回调中，我需要做的就是`$(destElem).text()`使用翻译后的文本进行调用，该文本位于`text`密钥下的字典中。在出现错误的情况下，我也这样做，但我显示的文本是一般错误消息，我确保将其作为可翻译文本输入基本模板。

所以现在唯一剩下的就是`translate()`用户点击翻译链接时用正确的参数触发函数。还有一些方法可以做到这一点，我要做的只是`href`在链接的属性中嵌入对函数的调用：

*app / templates / _post.html*：翻译链接处理程序。

```
                <span id="translation{{ post.id }}">
                    <a href="javascript:translate(
                                '#post{{ post.id }}',
                                '#translation{{ post.id }}',
                                '{{ post.language }}',
                                '{{ g.locale }}');">{{ _('Translate') }}</a>
                </span>
```

如果`href`链接的元素带有前缀，则链接的元素可以接受任何JavaScript代码`javascript:`，因此这是调用转换函数的便捷方式。因为当客户端请求页面时，将在服务器中呈现此链接，所以我可以使用`{{ }}`表达式生成函数的四个参数。每个帖子都有自己的翻译链接，其唯一生成的参数。在`#`你看到一个前缀`post<ID>`和`translation<ID>`元素表明，接下来是一个元素ID。

现在，实时翻译功能已经完成！如果您在环境中设置了有效的Microsoft Translator API密钥，则现在应该能够触发转换。假设您将浏览器设置为更喜欢英语，则需要使用其他语言撰写帖子以查看“翻译”链接。您可以在下面看到一个示例：

![翻译](https://blog.miguelgrinberg.com/static/images/mega-tutorial/ch15-translate.png)

在本章中，我介绍了一些需要翻译成应用程序支持的所有语言的新文本，因此有必要更新翻译目录：

```
(venv) $ flask translate update
```

对于您自己的项目，您需要编辑每个语言存储库中的*messages.po*文件以包含这些新测试的翻译，但我已经在本章的下载包或GitHub存储库中创建了西班牙语翻译。

要发布新翻译，需要编译它们：

```
(venv) $ flask translate compile
```