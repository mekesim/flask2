# 20 一些JavaScript Magic

如今，构建一个不使用至少一点JavaScript的Web应用程序是不可能的。我相信你知道，原因是JavaScript是唯一一种在Web浏览器中本机运行的语言。在[第14章中，](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xiv-ajax)您看到我在Flask模板中添加了一个简单的JavaScript链接，以提供博客文章的实时语言翻译。在本章中，我将深入探讨该主题，并向您展示另一个有用的JavaScript技巧，使应用程序更有趣，更吸引用户。

用户可以相互交互的社交网站的常见用户界面模式是当您将鼠标悬停在用户名称上时，在页面上显示的任何位置显示用户在弹出式面板中的快速摘要。如果你从来没有注意到这一点，请转到Twitter，Facebook，LinkedIn或任何其他主要社交网络，当你看到一个用户名时，只需将鼠标指针放在它上面几秒钟，看看弹出窗口出现。本章将专门为Microblog构建该功能，您可以在其中看到以下预览：

![用户弹出窗口](https://blog.miguelgrinberg.com/static/images/mega-tutorial/ch20-demo.png)

*本章的GitHub链接是：Browse，Zip，Diff。*

## 服务器端支持

在我们深入研究客户端之前，让我们获得一些必要的服务器工作，以支持这些用户弹出窗口。用户弹出窗口的内容将由新路由返回，该路由将是现有用户配置文件路由的简化版本。这是视图功能：

*app / main / routes.py*：用户弹出视图功能。

```
@bp.route('/user/<username>/popup')
@login_required
def user_popup(username):
    user = User.query.filter_by(username=username).first_or_404()
    return render_template('user_popup.html', user=user)
```

此路由将附加到*/ user / <username> / popup* URL，并将只加载请求的用户，然后使用它呈现模板。模板是用于用户配置文件页面的较短版本：

*app / templates / user_popup.html*：用户弹出模板。

```
<table class="table">
    <tr>
        <td width="64" style="border: 0px;"><img src="{{ user.avatar(64) }}"></td>
        <td style="border: 0px;">
            <p>
                <a href="{{ url_for('main.user', username=user.username) }}">
                    {{ user.username }}
                </a>
            </p>
            <small>
                {% if user.about_me %}<p>{{ user.about_me }}</p>{% endif %}
                {% if user.last_seen %}
                <p>{{ _('Last seen on') }}: 
                   {{ moment(user.last_seen).format('lll') }}</p>
                {% endif %}
                <p>{{ _('%(count)d followers', count=user.followers.count()) }},
                   {{ _('%(count)d following', count=user.followed.count()) }}</p>
                {% if user != current_user %}
                    {% if not current_user.is_following(user) %}
                    <a href="{{ url_for('main.follow', username=user.username) }}">
                        {{ _('Follow') }}
                    </a>
                    {% else %}
                    <a href="{{ url_for('main.unfollow', username=user.username) }}">
                        {{ _('Unfollow') }}
                    </a>
                    {% endif %}
                {% endif %}
            </small>
        </td>
    </tr>
</table>
```

当用户将鼠标指针悬停在用户名上时，我将在以下部分中编写的JavaScript代码将调用此路由。作为响应，服务器将返回弹出窗口的HTML内容，然后客户端显示该内容。当用户移开鼠标时，将删除弹出窗口。听起来很简单吧？

如果您想了解弹出窗口的外观，现在可以运行应用程序，转到任何用户的配置文件页面，然后将*/ popup*附加到地址栏中的URL以查看弹出内容的全屏版本。

## Bootstrap Popover组件简介

在[第11章中，](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xi-facelift)我向您介绍了Bootstrap框架，作为创建外观漂亮的网页的便捷方式。到目前为止，我只使用了这个框架的一小部分。Bootstrap捆绑了许多常见的UI元素，所有这些元素都在*https://getbootstrap.com*的Bootstrap文档中有演示和示例。其中一个组件是[Popover](https://getbootstrap.com/docs/3.3/javascript/#popovers)，它在文档中被描述为“用于存放次要信息的内容的小覆盖”。正是我需要的！

大多数引导程序组件都是通过HTML标记定义的，该标记引用了添加漂亮样式的Bootstrap CSS定义。一些最先进的也需要JavaScript。应用程序在网页中包含这些组件的标准方法是在适当的位置添加HTML，然后对于需要脚本支持的组件，调用初始化或激活它的JavaScript函数。popover组件确实需要JavaScript支持。

执行弹出窗口的HTML部分非常简单，您只需要定义将触发弹出窗口的元素。在我的情况下，这将转到每篇博文中显示的可点击用户名。该*应用程序/模板/ _post.html*子模板已经定义了用户名：

```
            <a href="{{ url_for('main.user', username=post.author.username) }}">
                {{ post.author.username }}
            </a>
```

现在根据popover文档，我需要`popover()`在每个链接上调用JavaScript函数，如上面显示的那个链接，这将初始化弹出窗口。初始化调用接受许多配置弹出窗口的选项，包括传递要在弹出窗口中显示的内容的选项，用于触发弹出窗口显示或消失的方法（单击，悬停在元素上等等）。 ），如果内容是纯文本或HTML，还有一些选项可以在文档页面中看到。不幸的是，在阅读完这些信息后，我得到的问题多于答案，因为这个组件看起来并不像我需要的那样工作。以下是我需要解决以实现此功能的问题列表：

- 页面中将有许多用户名链接，每个用户链接显示一个博客帖子。我需要有一种方法在呈现页面后从JavaScript中找到所有这些链接，以便我可以将它们初始化为弹出窗口。
- Bootstrap文档中的popover示例都提供了popover的内容作为`data-content`添加到目标HTML元素的属性，因此当触发悬停事件时，所有Bootstrap需要做的是显示弹出窗口。这对我来说真的很不方便，因为我想对服务器进行Ajax调用以获取内容，并且只有在收到服务器的响应时我才想出现弹出窗口。
- 使用“悬停”模式时，只要将鼠标指针保持在目标元素内，弹出窗口就会保持可见状态。当您移开鼠标时，弹出窗口将消失。这具有丑陋的副作用，如果用户想要将鼠标指针移动到弹出窗口本身，则弹出窗口将消失。我需要找出一种方法来扩展悬停行为以包括弹出窗口，以便用户可以移动到弹出窗口中，例如，单击那里的链接。

在使用基于浏览器的应用程序时，事情变得非常复杂，实际上并不罕见。您必须非常具体地考虑DOM元素如何相互交互，并使它们以一种为用户提供良好体验的方式运行。

## 在页面加载上执行功能

很明显，每个页面加载后我都需要运行一些JavaScript代码。我要运行的函数将搜索页面中用户名的所有链接，并使用Bootstrap中的popover组件配置它们。

jQuery JavaScript库作为Bootstrap的依赖项加载，所以我将利用它。使用jQuery时，您可以通过将页面包装到页面中来注册要在页面加载时运行的函数`$( ... )`。我可以在*app / templates / base.html*模板中添加它，以便在应用程序的每个页面上运行：

*app / templates / base.html*：页面加载后运行功能。

```
...
<script>
    // ...

    $(function() {
        // write start up code here
    });
</script>
```

如您所见，我在[第14章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xiv-ajax)`<script>`定义`translate()`函数的元素中添加了启动函数。

## 使用选择器查找DOM元素

我的第一个问题是创建一个JavaScript函数，找到页面中的所有用户链接。当页面完成加载时，此函数将运行，完成后，将为所有这些函数配置悬停和弹出行为。现在我将集中精力寻找链接。

如果您从[第14章](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xiv-ajax)回忆起，实时翻译中涉及的HTML元素具有唯一ID。例如，ID = 123的帖子`id="post123"`添加了属性。然后使用jQuery，`$('#post123')`在JavaScript中使用该表达式在DOM中定位此元素。该`$()`函数功能非常强大，并且具有相当复杂的查询语言，可以搜索基于[CSS选择器的](https://api.jquery.com/category/selectors/) DOM元素。

我用于转换功能的选择器旨在查找一个具有唯一标识符设置为`id`属性的特定元素。标识元素的另一个选项是使用`class`属性，该属性可以分配给页面中的多个元素。例如，我可以用a标记所有用户链接`class="user_popup"`，然后我可以从JavaScript获取链接列表`$('.user_popup')`（在CSS选择器中，`#`前缀按ID搜索，而`.`前缀按类搜索）。在这种情况下，返回值将是具有该类的所有元素的集合。

## 弹出窗口和DOM

通过使用Bootstrap文档中的popover示例并在浏览器的调试器中检查DOM，我确定Bootstrap将popover组件创建为DOM中目标元素的兄弟。如上所述，这会影响悬停事件的行为，只要用户将鼠标从`<a>`链接移动到弹出窗口本身，就会触发“鼠标移出” 。

我可以用来扩展悬停事件以包含弹出窗口的技巧是使popover成为目标元素的子元素，这样就可以继承悬停事件。查看文档中的popover选项，可以通过在选项中传递父元素来完成`container`。

使popover成为hovered元素的子元素对于按钮或一般`<div>`或`<span>`基础元素很有效，但在我的情况下，popover的目标将是`<a>`显示用户名的可点击链接的元素。使popover成为`<a>`元素的子元素的问题是popover将获取`<a>`父元素的链接行为。最终结果将是这样的：

```
        <a href="..." class="user_popup">
            username
            <div> ... popover elements here ... </div>
        </a>
```

为了避免popover在`<a>`元素中，我将要使用的是另一个技巧。我将把`<a>`元素包装在一个`<span>`元素中，然后将悬停事件和popover与`<span>`。结果结构将是：

```
        <span class="user_popup">
            <a href="...">
                username
            </a>
            <div> ... popover elements here ... </div>
        </span>
```

在`<div>`和`<span>`元素是不可见的，所以他们是伟大的元素，用于帮助组织和构造DOM。该`<div>`元素是*块元素*，有点像HTML文档中的段落，而`<span>`元素是一个*内嵌元素*，这会比作一个字。对于这种情况，我决定使用`<span>`元素，因为`<a>`我正在包装的元素也是一个内联元素。

所以我将继续重构我的*app / templates / _post.html*子模板以包含`<span>`元素：

```
...
                {% set user_link %}
                    <span class="user_popup">
                        <a href="url_for('main.user', username=post.author.username)">
                            {{ post.author.username }}
                        </a>
                    </span>
                {% endset %}
...
```

如果你想知道popover HTML元素在哪里，好消息是我不必担心。当我`popover()`在`<span>`刚刚创建的元素上调用初始化函数时，Bootstrap框架将为我动态插入弹出组件。

## 悬停事件

正如我上面提到的，来自Bootstrap的popover组件使用的悬停行为不够灵活，无法满足我的需求，但是如果你看一下该`trigger`选项的文档，“hover”只是可能的值之一。我的眼睛是“手动”模式，可以通过调用JavaScript来手动显示或删除弹出窗口。这种模式让我可以自己实现悬停逻辑，所以我将使用该选项并实现我的拥有按我需要的方式工作的悬停事件处理程序。

所以我的下一步是将“悬停”事件附加到页面中的所有链接。使用jQuery，可以通过调用将悬停事件附加到任何HTML元素`element.hover(handlerIn, handlerOut)`。如果在元素集合上调用此函数，jQuery会方便地将事件附加到所有元素。这两个参数是两个函数，当用户分别将鼠标指针移入和移出目标元素时调用这两个函数。

*app / templates / base.html*：悬停事件。

```
    $(function() {
        $('.user_popup').hover(
            function(event) {
                // mouse in event handler
                var elem = event.currentTarget;
            },
            function(event) {
                // mouse out event handler
                var elem = event.currentTarget;
            }
        )
    });
```

该`event`参数是事件对象，其中包含有用的信息。在这种情况下，我正在使用。提取作为事件目标的元素`event.currentTarget`。

鼠标进入受影响的元素后，浏览器会立即调度悬停事件。在弹出窗口的情况下，您只想在等待鼠标停留在元素上的一小段时间后才激活，这样当鼠标指针短暂地越过元素但没有留在它上面时，没有弹出窗口闪烁。由于事件不支持延迟，这是我需要自己实现的另一件事。所以我要在“鼠标输入”事件处理程序中添加一秒计时器：

*app / templates / base.html*：悬停延迟。

```
    $(function() {
        var timer = null;
        $('.user_popup').hover(
            function(event) {
                // mouse in event handler
                var elem = event.currentTarget;
                timer = setTimeout(function() {
                    timer = null;
                    // popup logic goes here
                }, 1000);
            },
            function(event) {
                // mouse out event handler
                var elem = event.currentTarget;
                if (timer) {
                    clearTimeout(timer);
                    timer = null;
                }
            }
        )
    });
```

该`setTimeout()`功能在浏览器环境中可用。它需要两个参数，一个函数和一个以毫秒为单位的时间。效果`setTimeout()`是在给定的延迟之后调用该函数。所以我添加了一个现在为空的函数，它将在调度悬停事件后一秒钟调用。由于JavaScript语言中的闭包，此函数可以访问在外部作用域中定义的变量，例如`elem`。

我将计时器对象存储在`timer`我在`hover()`调用之外定义的变量中，以使计时器对象也可以访问“鼠标移出”处理程序。我需要这个的原因再次是为了获得良好的用户体验。如果用户将鼠标指针移动到这些用户链接中的一个并且在移动它之前停留在它上面半秒钟，我不希望该计时器仍然关闭并调用将显示弹出窗口的功能。所以我的鼠标输出事件处理程序检查是否有一个活动的计时器对象，如果有，它取消它。

## Ajax请求

Ajax请求不是一个新主题，因为我在[第14章中](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xiv-ajax)介绍了这个主题，作为实时语言翻译功能的一部分。使用jQuery时，该`$.ajax()`函数向服务器发送异步请求。

我要发送到服务器的请求将包含*/ user / <username> / popup* URL，我在本章开头添加到应用程序中。此请求的响应将包含我需要在弹出窗口中插入的HTML。

我对这个请求的直接问题是要知道`username`我需要在URL中包含什么值。事件处理函数中的鼠标是通用的，它将针对页面中找到的所有用户链接运行，因此该函数需要从其上下文中确定用户名。

的`elem`变量包含从悬停事件，这是在目标元件`<span>`封装了元件`<a>`元件。要提取用户名，我可以从DOM开始导航`<span>`，移动到第一个子`<a>`元素，即元素，然后从中提取文本，这是我需要在URL中使用的用户名。使用jQuery的DOM遍历函数，这实际上很简单：

```
elem.first().text().trim()
```

`first()`应用于DOM节点的函数返回其第一个子节点。该`text()`函数返回节点的文本内容。此功能不会对文本进行任何修剪，因此，例如，如果您有`<a>`一行，则下一行中的文本和`</a>`另一行中`text()`的文本将返回文本周围的所有空格。为了消除所有空格并仅留下文本，我使用`trim()`JavaScript函数。

这就是我需要能够向服务器发出请求的所有信息：

*app / templates / base.html*：XHR请求。

```
    $(function() {
        var timer = null;
        var xhr = null;
        $('.user_popup').hover(
            function(event) {
                // mouse in event handler
                var elem = $(event.currentTarget);
                timer = setTimeout(function() {
                    timer = null;
                    xhr = $.ajax(
                        '/user/' + elem.first().text().trim() + '/popup').done(
                            function(data) {
                                xhr = null
                                // create and display popup here
                            }
                        );
                }, 1000);
            },
            function(event) {
                // mouse out event handler
                var elem = $(event.currentTarget);
                if (timer) {
                    clearTimeout(timer);
                    timer = null;
                }
                else if (xhr) {
                    xhr.abort();
                    xhr = null;
                }
                else {
                    // destroy popup here
                }
            }
        )
    });
```

在这里，我在外部范围中定义了一个新变量`xhr`。这个变量将保存异步请求对象，我从一个调用初始化`$.ajax()`。不幸的是，当直接在JavaScript方面构建URL时，我无法使用`url_for()`Flask，因此在这种情况下，我必须明确地连接URL部分。

该`$.ajax()`调用返回一个promise，这是一个表示异步操作的特殊JavaScript对象。我可以通过添加附加完成回调`.done(function)`，因此一旦请求完成，我的回调函数将被调用。回调函数将接收响应作为参数，您可以看到我`data`在上面的代码中命名。这将是我将要放在popover中的HTML内容。

但在我们进入popover之前，还有一个与为用户提供需要照顾的良好体验相关的细节。回想一下，我在“鼠标移出”事件处理函数中添加了逻辑，如果用户将鼠标指针移出，则取消一秒超时`<span>`。需要将相同的想法应用于异步请求，因此我添加了第二个子句来中止我的`xhr`请求对象（如果存在）。

## 弹出创造与毁灭

最后，我可以使用`data`在Ajax回调函数中传递给我的参数创建我的popover组件：

*app / templates / base.html*：显示弹出窗口。

```
                                function(data) {
                                    xhr = null;
                                    elem.popover({
                                        trigger: 'manual',
                                        html: true,
                                        animation: false,
                                        container: elem,
                                        content: data
                                    }).popover('show');
                                    flask_moment_render_all();
                                }
```

弹出窗口的实际创建非常简单，`popover()`Bootstrap 的功能完成了设置它所需的所有工作。弹出窗口的选项作为参数给出。我已经使用“手动”触发模式，HTML内容，没有淡入淡出动画（使其显示并更快地消失）配置此弹出窗口，并且我已将父设置为`<span>`元素本身，以便悬停行为延伸到通过继承popover。最后，我将`data`参数传递给Ajax回调作为`content`参数。

`popover()`调用的返回是新创建的popover组件，由于一个奇怪的原因，还有另一个`popover()`用于显示它的方法。所以我不得不添加第二个`popover('show')`调用以使弹出窗口显示在页面上。

弹出窗口的内容包括“上次看到”日期，该日期是通过Flask-Moment插件生成的，如[第12章所述](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xii-dates-and-times)。正如扩展所[记录的](https://github.com/miguelgrinberg/Flask-Moment#ajax-support)那样，当通过Ajax添加新的Flask-Moment元素时，`flask_moment_render_all()`需要调用该函数来适当地呈现这些元素。

现在剩下的就是处理鼠标输出事件处理程序上的弹出窗口的删除。如果用户将鼠标移出目标元素，则该处理程序已经具有中止弹出操作的逻辑。如果这些条件都不适用，那么这意味着当前显示了弹出框并且用户正在离开目标区域，因此在这种情况下，`popover('destroy')`对目标元素的调用会进行适当的删除和清理。

*app / templates / base.html*：销毁popover。

```
                function(event) {
                    // mouse out event handler
                    var elem = $(event.currentTarget);
                    if (timer) {
                        clearTimeout(timer);
                        timer = null;
                    }
                    else if (xhr) {
                        xhr.abort();
                        xhr = null;
                    }
                    else {
                        elem.popover('destroy');
                    }
                }
```