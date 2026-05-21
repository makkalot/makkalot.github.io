---
title: "Enlivepy: A Different Approach to Html Templating in Python"
date: 2014-10-22
categories: ["Python"]
tags: ["django", "template", "python"]
slug: "enlivepy-html-transformation"
---

## The Problem About Html Templating

Don't worry - this blog entry won't consist of me rambling about yet another templating engine that will change the world. Instead, I'll be focusing on a practical aspect - the problems the current HTML templating libraries have and what can be possibly done about these issues.

Django and all other MVC/MVP frameworks do a great job at separating the Controller logic from the View logic. This way you can have all your complex operations in Controller and send back the data that needs to be shown or rendered to the end user. On the other end of this process you have a templating library - it takes Python objects and renders them.

Now, some of these templating libraries argue that having lots of logic in the representation layer is a bad thing (Django templating for example). Others, like Jinja2, are more lax and allow you to do lots of interesting things.

In both cases what we are actually doing is mixing data with logic. And while this might not seem like a problem with small applications, when you start operating on a greater scale, issues start to appear (a lot). Let me provide a few examples:

- Front end developers (CSS/HTML developers) start having difficulties when editing html pages when something does not appear the way it should be appearing. They have to learn the templating engine your framework uses.

- Sometimes HTML/CSS developers break the logic in html pages by replacing something by mistake.

- Backend developers are not the best at editing both HTML and CSS (well, in most cases, at least!). I have had this happen to me quite a few times too. I receive the HTML from a designer team, after that I put it into the templating engine...and it has a radically changed appearance. When I compare the way it looks on the server and the original things are just looking completely different.

- If HTML/CSS developers need to play or change something on the system, what they need is a working copy of the whole system. You need to give them ssh access to the staging server or something similar.

## A possible solution to the problem

Recently I was playing around with different programming languages like Clojure, Go and Scala. I've found this Clojure library called [enlive](https://github.com/cgrand/enlive) that approaches html templating in a slightly different manner. The general idea is you have the html of your site as static entity and you do some transformations on this html by generating a new html to be served on the server. The mentioned transformations are basic functions that take some nodes of html elements and return back new changed elements back.

This solves some of the painful problems I mentioned before hand. What's different now is that:

- We have a static html that does not include any logic in it. The designer team can edit it any time without breaking the logic. They can also have a fully working version of the representation without the need of having a running server around.

- The backend team is happy - they don't have to edit the html and combine it with a certain framework just to see it being radically different than the one they expected to have.

- The backend team can use all of the programming language tricks without being afraid of polluting the representation with too many logic.

I liked this idea and created a clone of this library for Python programming language. It is called [enlivepy](https://github.com/makkalot/enlivepy). The purpose of my current blogpost is to show you a very simple template with a navigation bar. This will illustrate the concept in a pretty easy and educational manner and it will show you how it's actually put into practice.

## Getting Started

For the purpose of this blog entry I've created an example Django application. The mentioned application replaces the default Django templating with enlivepy html transformation library (I will explain more on this topic in another topic). To start with the application:

```bash
git clone https://github.com/makkalot/django-enlivepy-example.git
cd django-enlivepy-example/
virtualenv venv
source venv/bin/activate
pip install -r requirements.txt
cd example/
python manage.py syncdb
python manage.py runserver
```

If you don't want to wait to the end of the entry, you can check the expected result at `http://127.0.0.1:8000/index/`.

# A Basic Html Static Page

The first step will involve starting with a static html page that looks like [this](https://github.com/makkalot/django-enlivepy-example/blob/master/example/templates/logonav/index.html). You can open it in your browser to see that it is a working page without any logic in it. To put it brief, it is something like this (only the parts we're interested are shown):

```html
<body>

    <!-- Page Content -->
    <div class="container">
        <div class="row">
            <div class="col-lg-12">
                <h1>Logo Nav by Start Bootstrap</h1>
                <p>Note: You may need to adjust some CSS based on your needs.</p>
            </div>
        </div>
    </div>
    <!-- /.container -->

    <!-- jQuery Version 1.11.0 -->
    <script src="static/js/jquery-1.11.0.js"></script>

    <!-- Bootstrap Core JavaScript -->
    <script src="static/js/bootstrap.min.js"></script>

</body>
```

The first task is to put some dynamic content inside the "container" which now has some static text. However, before we get there, we need to replace the static paths with those that will work on Django's `STATIC_URL` paths. For that purpose lets create a mixin:

```python
class BaseMediaUrlTemplateMixin(object):

    def transform(self, nodes, *args, **kwargs):

        if not kwargs.get("STATIC_URL"):
            raise Exception("STATIC_URL not set")

        media_url = kwargs.get("STATIC_URL")

        at(nodes,
           "script", partial(self.trans_path, media_url, "script", "src"),
           "link",   partial(self.trans_path, media_url, "link", "href"))


    def trans_path(self, media_url, tag_name, attr_name, node):

        path = node.get(attr_name)
        if not path:
            return

        path = path.strip()
        if path.startswith("http") or not path:
            return

        fpath = ""
        if path.startswith(media_url[1:]):
            fpath = "".join(["/", path])
        else:
            fpath = os.path.join(media_url, path)

        attr_fn = set_attr(**{attr_name:fpath})
        return transform(node, tag_name, attr_fn)
```

The most important part of the code above is found in the `transform` method. This method takes the selected nodes so far (in this case - the whole page), some arguments which can be passed via `view` code (`render_to_response`), and selects the all `script` and `link` elements. Afterwards, it transforms their paths with paths that are compatible with Django's `STATIC_URL` variable. Therefore the transformed page's elements will look like this:

```html
....
<script src="/static/js/jquery-1.11.0.js"></script>
<script src="/static/js/bootstrap.min.js"></script>
....
```

# The Snippets

Enlive has something called a "Snippet" - a concept for reusable html code blocks. The idea here is to have some sort of transformation code that will select a proper part of the html page and then process it. Snippets come in many forms and have to be reusable. So for example a sidebar, navigation, forms, footers and others all count as a snippet. To illustrate this, I'll create a snippet for the navigation. The static version of navigation looks like [this](https://github.com/makkalot/django-enlivepy-example/blob/master/example/templates/logonav/nav.html):

```html
<body>
    <!-- Navigation -->
    <nav class="navbar navbar-inverse navbar-fixed-top" role="navigation">
        <div class="container">
            <!-- Brand and toggle get grouped for better mobile display -->
            <div class="navbar-header">
                <button type="button" class="navbar-toggle" data-toggle="collapse" data-target="#bs-example-navbar-collapse-1">
                    <span class="sr-only">Toggle navigation</span>
                    <span class="icon-bar"></span>
                    <span class="icon-bar"></span>
                    <span class="icon-bar"></span>
                </button>
                <a class="navbar-brand" href="#">
                    <img src="http://placehold.it/150x50&text=Logo" alt="">
                </a>
            </div>
            <!-- Collect the nav links, forms, and other content for toggling -->
            <div class="collapse navbar-collapse" id="bs-example-navbar-collapse-1">
                <ul class="nav navbar-nav">
                    <li>
                        <a href="#">About</a>
                    </li>
                </ul>
            </div>
            <!-- /.navbar-collapse -->
        </div>
        <!-- /.container -->
    </nav>

    <!-- jQuery Version 1.11.0 -->
    <script src="static/js/jquery-1.11.0.js"></script>

    <!-- Bootstrap Core JavaScript -->
    <script src="static/js/bootstrap.min.js"></script>

</body>
```

As you can see, there is only one link named "About". The purpose of the snippet is to get only the "nav" element, transform its contents with our dynamic navigation links and return only that portion back. Let's create this snippet like this:

```python
class TodoNavSnippet(DjangoSnippet):

    template = "logonav/nav.html"
    selection = "nav.navbar"

    menu_items = ("Home", "About", "Services", "Contact")

    def transform(self, nodes, *args, **kwargs):
        navs_fn = clone_for(self.menu_items,
                            "li > a", lambda i: content(i),
                            "li > a", lambda i: set_attr(href="/"+i.lower()+"/"))
        at(nodes, "ul.nav > li", navs_fn)
        return nodes
```

The snippet above has a certain property called `selection`. What it does is selecting only the nodes (with CSS selection) that constitute an interest for us. In my case, it selects `nav.navbar`. In its transformation method it uses the **clone_for** transformer function, which duplicates the given node. Therefore, we're able to clone and produce a list for the **menu_items** attribute specified.

It is useful to note that `template` property finds in the same way Django templating system finds its files. What this means is that you can structure your html static pages as you would do in normal Django applications. Here's the generated code for the snippet:

```html
<nav class="navbar navbar-inverse navbar-fixed-top" role="navigation"><div class="container">
        <!-- Brand and toggle get grouped for better mobile display -->
        <div class="navbar-header">
            <button type="button" class="navbar-toggle" data-toggle="collapse" data-target="#bs-example-navbar-collapse-1">
                <span class="sr-only">Toggle navigation</span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
            </button>
            <a class="navbar-brand" href="#">
                <img src="http://placehold.it/150x50&amp;text=Logo" alt=""></a>
        </div>
        <!-- Collect the nav links, forms, and other content for toggling -->
        <div class="collapse navbar-collapse" id="bs-example-navbar-collapse-1">
            <ul class="nav navbar-nav"><li>
                    <a href="/home/">Home</a>
                </li>
            <li>
                    <a href="/about/">About</a>
                </li>
            <li>
                    <a href="/services/">Services</a>
                </li>
            <li>
                    <a href="/contact/">Contact</a>
                </li>

            </ul></div>
        <!-- /.navbar-collapse -->
    </div>
    <!-- /.container -->
</nav>
```

# The Template Concept

The next step is to replace the static content with dynamic content in the index page and add the transformer navigation node to it. For this purpose I'm going to use enlive's "template" concept. It is very similar to a snippet, but instead of returning a portion of the page, it will return the whole page back. In other words, the template is the part that combines the snippets together, similar to `base.html` in Django.

Once again, let me demonstrate:

```python
class TodoIndex(BaseMediaUrlTemplateMixin, DjangoTemplate):

    template = "logonav/index.html"

    def transform(self, nodes, *args, **kwargs):
        super(TodoIndex, self).transform(nodes, *args, **kwargs)
        nav_snip = TodoNavSnippet()
        navbar = nav_snip(*args, **kwargs)

        at(nodes,
           "body", prepend(navbar),
           "div.col-lg-12 > h1", content("Index Todo List"),
           "div.col-lg-12 > p", None)
```

As you can see, what it does first is to call the `BaseMediaUrlTemplateMixin` class's transform method which takes care of replacing the script and link tags as a start. Next, I prepend the already created `NavigationSnippet` to the "body" element. To finish all off, I replace the static content with the dynamic content.

In the example above I used the **at** function - it accepts multi selectors with multi transformer functions (you can check documentation for more information).

The transformed content will now look like this (navigation stripped):

```html
<div class="container">
    <div class="row">
        <div class="col-lg-12">
            <h1>Index Todo List</h1>
        </div>
    </div>
</div>
```

# Using Enlivepy in Django's Views

You can use enlivepy's templates in Django's views as you'd use the Django's default templates. In order to be able to do that, you have to first add the enlivepy's template loader in settings:

```python
TEMPLATE_LOADERS = (
    'enlivepy.django.loader.EnlivepyLoader',
)
```

Then you have to put your templates and snippets in a file called `enlivetmpl.py` in apps you want to have enlivepy enabled. Afterwards, you have to call enlivepy's registry function **autodiscover** (it's really the same pattern as Django admin) and put it in a place you know will be loaded as fast as possible (i.e `__init__.py`):

```python
import enlivepy.django as enlivetmpl

enlivetmpl.autodiscover()
```

Then at the end of your `enlivetmpl.py` file you have to register the templates you will be calling your views like:

```python
register("todo_index", TodoIndex())
```

I want to bring to your attention the fact that you can name your templates however you wish.

And finally, in your views you can call your templates as you'd do in normal applications:

```python
class TodoIndexView(TemplateView):

    template_name = "todo_index"
```

## Summary

Enlivepy is still a pretty new and fresh concept I've been working on. Without any doubt there is a lot room of improvement. Examples that come off the top of my head include performance optimization or adding some missing parts from the original implementation.

I've started using it on my personal projects and it has been both effective and fun. However, it is definitely not ready for mainstream production usage. Hopefully after some more tinkering around with it, it maybe a very good solution to many of the problems found in traditional template-based programming.

So, in this post I did my best to explain how Enlivepy works and what current issues it aims to solve. I hope it's been an interesting and educational read! I'd also love to hear your ways of tackling the problems you encounter with the current templating systems. How do you do it and is it effective?

Let me know in the comments below!
