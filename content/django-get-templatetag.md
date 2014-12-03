Title: GET-preserving Django template tag
Date: 2014-12-02 19:45
Category: Django
Tags: django, template tags
Summary: Creating a template tag that preserves GET parameters
Status: draft

### The Problem ###

Giving users a view into a long list of items is a common function. A very
popular way to manage that view is to use Django's built in pagination feature.
This feature lets you create anchors to the page while adding the page
GET parameter.

This works well as long as you don't need to run any other similar logic in
parallel, but it starts to break down a little as you add things like sorting,
filtering, adjusting page size, and other functions that are best submitted as
GET parameters. This post will describe the fix that I found at
<a href="https://djangosnippets.org/snippets/1627/">
https://djangosnippets.org/snippets/1627/</a>

### A Little More In Depth ###

To be clear, the above problem isn't insurmountable. For example, if you have a
Django template that assumes a context with:

* a <a href="https://docs.djangoproject.com/en/dev/topics/pagination/#page-objects">page</a>
object named ```page_obj``` (paired with the ```GET``` param of ```page```)
* the filter constraint as ```filter``` (paired with the ```GET``` param of
```filter```)
* the sort constraint as ```sort``` (paired with the ```GET``` param of
```sort```)

then you could do something like:

    :::django
    <a href="?page={{ page_obj.next_page_number }}&filter={{ filter }}&sort={{ sort }}">next</a>

This would work, however it requires you to not only know what all of the
potential ```GET``` params are for the paginated view, but also what their
values are (you must pass the values in via the template context). This is a
lot of extra work for something you don't even care about, other than needing
everything but the ```page``` paramter to remain the same as the current
request.

### The Setup ###

The way we will reduce the work required for this repetitive task is to use a
Django
<a href="https://docs.djangoproject.com/en/dev/howto/custom-template-tags/">template
tag</a>. The first thing you must do is choose which Django app the template
tag lives inside of. If you only need to use this tag inside of a single app,
then you can put it inside of that app's directory. Otherwise, feel free to
make a dedicated app for shareable code. The only requirement to use this
template tag is that the app it is defined in is included in the
```INSTALLED_APPS``` config tuple.

Create a directory named ```templatetags``` inside of your chosen app, create
the ```__init__.py``` file, and then start a new file named
```foo_extras.py```, where ```foo``` is your app name. Start the template tag
off with:

    :::python
    from django import template

    register = template.Library()

This readies you to create a custom template tag via Django's template module.

### Digging Deeper ###


