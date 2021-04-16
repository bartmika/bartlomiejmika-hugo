---
title: "Example of using JavaScript AJAX XmlHttpRequest with Django JsonResponse view"
date: 2021-01-01T00:00:00-00:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "Python"
- "Django"
---

Forget [Axios](https://github.com/axios/axios) or any other third-party JavaScript library pertaining to API calling, the purpose of this article is to explain how to utilize the basic ``XmlHttpRequest`` with your Django project?

<!--more-->

1. Begin by setting up your ``Django`` project:

    {{< highlight bash "linenos=false">}}
    mkdir ajax_django_example
    cd ajax_django_example/
    virtualenv env
    source env/bin/activate
    python --version
    pip install django
    django-admin startproject example
    cd example
    python manage.py startapp hello
    python manage.py runserver
    {{</ highlight >}}

2. Populate your ``views.py`` file:

    {{< highlight python "linenos=false">}}
    # hello/views.py
    from django.http import HttpResponse
    from django.shortcuts import render
    from django.http import JsonResponse


    def index(request):
        return render(request, 'hello/index.html', {

        })

    def get_version(request):
        return JsonResponse({'version': '0.1.0-beta'})

    def post_add(request):
        a = request.POST.get("a")
        b = request.POST.get("b")

        a = float(a)
        b = float(b)
        c = a + b
        return JsonResponse({'result': c})

    # Would you like to know more?
    # https://docs.djangoproject.com/en/2.2/ref/request-response/#jsonresponse-objects
    {{</ highlight >}}

3. Set the ``urls.py`` file:

    {{< highlight python "linenos=false">}}
    # hello/urls.py
    from django.urls import path

    from . import views

    urlpatterns = [
        path('', views.index, name='index'),
        path('version', views.get_version, name='version'),
        path('add', views.post_add, name='add'),
    ]
    {{</ highlight >}}

4. Set the project's ``settings.py`` file:

    {{< highlight python "linenos=false">}}
    # example/settings.py

    # ...

    INSTALLED_APPS = [
        'hello.apps.HelloConfig',
        'django.contrib.admin',
        'django.contrib.auth',
        'django.contrib.contenttypes',
        'django.contrib.sessions',
        'django.contrib.messages',
        'django.contrib.staticfiles',
    ]

    MIDDLEWARE = [
        'django.middleware.security.SecurityMiddleware',
        'django.contrib.sessions.middleware.SessionMiddleware',
        'django.middleware.common.CommonMiddleware',
        # 'django.middleware.csrf.CsrfViewMiddleware',
        'django.contrib.auth.middleware.AuthenticationMiddleware',
        'django.contrib.messages.middleware.MessageMiddleware',
        'django.middleware.clickjacking.XFrameOptionsMiddleware',
    ]

    # ...
    {{</ highlight >}}

5. Set the project's ``urls.py`` file:

    {{< highlight python "linenos=false">}}
    # example/urls.py
    """example URL Configuration

    The `urlpatterns` list routes URLs to views. For more information please see:
    https://docs.djangoproject.com/en/2.2/topics/http/urls/
    Examples:
    Function views
        1. Add an import:  from my_app import views
        2. Add a URL to urlpatterns:  path('', views.home, name='home')
    Class-based views
        1. Add an import:  from other_app.views import Home
        2. Add a URL to urlpatterns:  path('', Home.as_view(), name='home')
    Including another URLconf
        1. Import the include() function: from django.urls import include, path
        2. Add a URL to urlpatterns:  path('blog/', include('blog.urls'))
    """
    from django.contrib import admin
    from django.urls import path, include

    urlpatterns = [
        path('admin/', admin.site.urls),
        path('', include('hello.urls')),
    ]
    {{</ highlight >}}

6. Next step is the JavaScript portion.

    {{< highlight javascript "linenos=false">}}
    // hello/templates/hello/index.js
    function getVersion() {
        var xhttp = new XMLHttpRequest();
        xhttp.onreadystatechange = function() {
            if (this.readyState == 4 && this.status == 200) {
                document.getElementById('demoGet').innerHTML = this.responseText;
            }
        };
        xhttp.open('GET', 'version', true);
        xhttp.send();
    }

    function postAdd() {
        var xhttp = new XMLHttpRequest();
        xhttp.onreadystatechange = function() {
            if (this.readyState == 4 && this.status == 200) {
                document.getElementById('demoPost').innerHTML = this.responseText;
            }
        };
        xhttp.open('POST', 'add', true);
        xhttp.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
        xhttp.send("a=1&b=2");
    }

    // Would you like to know more?
    // https://www.w3schools.com/js/js_ajax_intro.asp
    {{</ highlight >}}

7. Continue

    {{< highlight html "linenos=false">}}
    <!-- hello/templates/hello/index.html -->
    <!DOCTYPE html>
    <html>
        <body>
            <h1>Using XMLHttpRequest Object and Django JsonResponse</h1>
            <h2>GET Example</h2>

            <p id="demoGet">Let AJAX change this text.</p>

            <button type="button" onclick="getVersion()">Run GET</button>

            <h2>POST Example</h2>

            <p id="demoPost">Let AJAX change this text.</p>

            <button type="button" onclick="postAdd()">Run POST</button>


            <script>
                {% include "hello/index.js" %}
            </script>

        </body>
    </html>
    {{</ highlight >}}

8. That's it! Test it out the code and you should see it working.
