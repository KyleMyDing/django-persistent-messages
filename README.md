
Django Persistent Messages
==========================

这是一个持久消息\通知的Django App,基于 Django  [messages framework](http://docs.djangoproject.com/en/dev/ref/contrib/messages/) (`django.contrib.messages`).

此程序提供持久化消息的支持，超越浏览器会话，并将显示为“sticky”标记发送给用户，直到主动标记为已读。用户读取后，消息仍会列在每个用户的邮件收件箱中。`django.contrib.messages`确保创建的消息传送给用户，此程序确保主动推送到用户端。

* 对于已认证的用户，消息存储在数据库中。消息是临时的，如上所述持久化。
* 对于匿名用户，使用默认的基于cookie /session的方式，不需要用于存储消息的数据库访问，并且持久化消息是*不可能*的。
* 有一个统一的API用于向这两种类型的用户显示消息，也就是说，可以使用与Django的消息传递框架相同的代码来添加和显示消息，前提是用户是验证通过的。
* 消息可以在屏幕上显示或作为电子邮件通知发送给个别用户。


安装
------------

本文假设你已熟练使用Python和Django.

1.  clone 源码 :

        $ git clone git://github.com/KyleMyDing/django-persistent-messages.git

2. 确保 `persistent_messages` 在你的 `PYTHONPATH`中.
3. 添加 `persistent_messages` 到你的Django `INSTALLED_APPS` settings.py中.

        INSTALLED_APPS = (
            ...
            'persistent_messages',
        )

4. 确保 Django 的 `MessageMiddleware` 在你的 `MIDDLEWARE_CLASSES` 设置中 (默认设置):

        MIDDLEWARE_CLASSES = (
            ...
            'django.contrib.messages.middleware.MessageMiddleware',
        )

5. 添加 persistent_messages  到你的URL 设置中. 例如，为了使消息在 `http://domain.com/messages/`中使用, 添加到 `urls.py`中

        urlpatterns = patterns('',
            (r'^messages/', include('persistent_messages.urls')),
            ...
        )

6. 在你的settings.py中,设置消息 [后端存储](http://docs.djangoproject.com/en/dev/ref/contrib/messages/#message-storage-backends) 到 `persistent_messages.storage.PersistentMessageStorage`:

        MESSAGE_STORAGE = 'persistent_messages.storage.PersistentMessageStorage'

7. 同步数据库 

      $ manage.py syncdb

8. 如果想要使用模板,添加 `templates` 文件夹到你的 `TEMPLATE_DIRS` 设置中:

        TEMPLATE_DIRS = (
            ...
            'path/to/persistent_messages/templates')
        )



## 在视图和模板中使用消息

### 消息级别

Django的消息框架提供了许多[消息级别](http://docs.djangoproject.com/en/dev/ref/contrib/messages/#message-levels)，用于各种目的，例如成功消息，警告等。此程序提供具有相同名称的常量，区别在于具有这些级别的消息将持久化：

    import persistent_messages
    # persistent message levels:
    persistent_messages.INFO 
    persistent_messages.SUCCESS 
    persistent_messages.WARNING
    persistent_messages.ERROR
    
    from django.contrib import messages
    # temporary message levels:
    messages.INFO 
    messages.SUCCESS 
    messages.WARNING
    messages.ERROR

### 添加消息

由于该程序作为Django [消息框架](http://docs.djangoproject.com/en/dev/ref/contrib/messages/)的[存储后端](http://docs.djangoproject.com/en/dev/ref/contrib/messages/#message-storage-backends)实现，仍然可以使用常规Django API添加消息：

    from django.contrib import messages
    messages.add_message(request, messages.INFO, 'Hello world.')

这是兼容的，相当于使用由以下API提供的API `persistent_messages`：

    import persistent_messages
    from django.contrib import messages
    persistent_messages.add_message(request, messages.INFO, 'Hello world.')

为了添加持久消息，使用上面列出的消息级别：

    messages.add_message(request, persistent_messages.WARNING, 'You are going to see this message until you mark it as read.')

相当于:

    persistent_messages.add_message(request, persistent_messages.WARNING, 'You are going to see this message until you mark it as read.')

注意，这仅适用于登录的用户，因此可能会确保当前用户不是匿名的`request.user.is_authenticated()`。为匿名用户添加持久消息会引发一个`NotImplementedError`。

使用`persistent_messages.add_message`，。当使用电子邮件通知功能时，也可以添加主题行的消息。以下消息将显示在屏幕上，并发送到与当前用户相关联的电子邮件地址(也可以扩展为发送短信)：

    persistent_messages.add_message(request, persistent_messages.INFO, 'Message body', subject='Please read me', email=True)

`User`如果消息发送给除了当前认证的用户之外的用户，那么也可以将此函数传递给对象。用户Sally将在下次登录时看到此消息：

    from django.contrib.auth.models import User
    sally = User.objects.get(username='Sally')
    persistent_messages.add_message(request, persistent_messages.SUCCESS, 'Hi Sally, here is a message to you.', subject='Success message', user=sally)

### 显示消息

消息可以按照[Django手册中的描述显示](http://docs.djangoproject.com/en/dev/ref/contrib/messages/#displaying-messages)。但是，可能想要包括用于关闭每个消息的链接标记（将其标记为已读）。在模板中，使用以下内容：

    {% if messages %}
    <ul class="messages">
        {% for message in messages %}
        <li{% if message.tags %} class="{{ message.tags }}"{% endif %}>
            {% if message.subject %}<strong>{{ message.subject }}</strong><br />{% endif %}
            {{ message.message }}<br />
            {% if message.is_persistent %}<a href="{% url message_mark_read message.pk %}">close</a>{% endif %}
        </li>
        {% endfor %}
    </ul>
    {% endif %}

您也可以使用模板。用下面的代码替换了上面的代码。它允许用户删除消息，并将它们标记为使用Ajax请求读取，前提是您的HTML页面包含JQuery：

    {% include "persistent_messages/message/includes/messages.jquery.html" %}
