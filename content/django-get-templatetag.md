Title: GET-preserving Django template tag
Date: 2014-12-10 21:17
Category: Web Development
Tags: django, template tags
Summary: Creating a template tag that preserves GET parameters

### The Problem ###

Giving users a view into a long list of items is a common function. A popular
way to manage that view is to use Django's built in pagination feature.  This
feature allows for easy creation of links to different pages of the items via
the defined GET parameter.

This works well as long as no other GET parameters need to be used in parallel.
It starts to break down a bit as sorting, filtering, page size, and other
functions are added.  This post will annotate and explain an elegant fix found
at
[https://djangosnippets.org/snippets/1627/](https://djangosnippets.org/snippets/1627/)

### A Little More In Depth ###

To be clear, the above problem is not insurmountable. Assuming a Django
template with a context containing:

* a
  [page](https://docs.djangoproject.com/en/dev/topics/pagination/#page-objects)
  object named ```page_obj``` (paired with the GET parameter of
  ```page```)
* a filter constraint as ```filter``` (paired with the GET parameter of
  ```filter```)
* a sort constraint as ```sort``` (paired with the GET parameter of
  ```sort```)

then:

    :::django
    <a href="?page={{ page_obj.next_page_number }}&filter={{ filter }}&sort={{ sort }}">
        next
    </a>

would work. However, it requires all of the GET parameters to be known,
as well as their values. It also would need to be updated any time the possible
GET parameters change. This is a lot of extra work for something that is
tangential to the pagination function.

### The Setup ###

The Django feature with which code repetition will be reduced is called a
[template
tag](https://docs.djangoproject.com/en/dev/howto/custom-template-tags/). The
first thing to do when creating a custom template tag is to decide which app to
put it in. If the tag is only needed for a single app, then it can be put
inside that app's directory. It can also be placed in a dedicated app. The only
requirement in order to use the tag is that the app it is defined in is
included in the ```INSTALLED_APPS``` config tuple.

Create a directory named ```templatetags``` inside of the chosen app, create
the ```__init__.py``` file, and then start a new file named
```foo_extras.py```, where ```foo``` is the app name. Start the template tag
off with:

    :::python
    from django import template

    register = template.Library()

This sets up the module to implement a Django template tag.

### A Helping Hand ###

The next step to take is to add a decorator that eases much of the boilerplate
involved with a tag. It is short and sweet:

    :::python
    def easy_tag(func):
        def inner(parser, token):
            try:
                return func(*token.split_contents())
            except TypeError:
                raise template.TemplateSyntaxError(
                    'Bad arguments for tag "%s"' % token.split_contents()[0])
        inner.__name__ = func.__name__
       	inner.__doc__ = func.__doc__
        return inner

The ```easy_tag``` decorator wraps the real logic of the tag with the necessary
parsing of the tag parameters. The ```inner``` function has the signature
required of a template tag: the two arguments ```parser``` and ```token```.

```parser``` will not be used in this tag, so it can be ignored. ```token``` is
the tag in its entirety, as it is called in the template. The inner function
tries to call the ```split_contents``` method of the token, which splits the
text of the entire tag into space delimited pieces. Note that
```split_contents``` is used, which takes care to keep quoted strings as a
single item (as opposed to ```split``` which does not).

For example, the if the template were to call a test tag:

    :::django
    {% test foo "bar baz" foo %}

And a template tag was defined as:

    :::python
    @register.tag()
    @easy_tag
    def test(tag_name, arg1, arg2, arg3):
        ...

Then the ```test``` function would be wrapped in the easy_tag call. The
easy_tag decorator would handle splitting the args into the pieces

    :::python
    ['test', 'foo', '"bar baz"', 'foo']

and passing that as the arg list into ```test```. In this case, tag_name would
be assigned the value ```test```, since that is the first item in the list.
```arg1``` would be assigned ```foo```, ```arg2``` would be assigned
```"bar baz"```, and ```arg3``` would be assigned ```foo```.  In addition to
the splitting the tag, the decorator handles catching any errors encountered
while splitting the tag arguments and throwing the correct
```TemplateSyntaxError``` exception. It creates a descriptive exception message
by adding in the tag name, which is always the first item of the list returned
by ```split_contents```. It also sets the name and documentation string of the
function being wrapped to the new function created by the decorator in order to
keep everything consistent.

### Parsing the Tag Arguments ###

The next step in creating the tag is to begin implementing the
```template.Node``` subclass. This will be the meat of the custom tag. A Node
object is what must be be returned when registering a template tag via
```register.tag```. The Node object requires two methods: ```__init__``` and
```render```. The first step will be to create the ```__init__``` method.

    :::python
    class AppendGetNode(template.Node):
        def __init__(self, args):
            self.dict_pairs = {}
            for pair in dict.split(','):
                pair = pair.split('=')
                self.dict_pairs[pair[0]] = template.Variable(pair[1])

The ```__init__``` method should handle compiling the tag argument into a
usable data structure. The method will assume that argument is a
comma-delimited list of key value pairs, and that each key value pair will be
separated by an equals sign. It also assumes that each value is a template
variable, and must be resolved. It uses each key and resolved value to
construct a dictionary of data, saving it as the instance variable
```dict_pairs```. These will be used as the first step when creating the new
request.

### The Render Method ###

The ```render``` method is the other required method of the ```Node``` instance.
It will handle the final creation of the string that will be returned as the
output of the tag.

    :::python

    def render(self, context):
        get = context['request'].GET.copy()

        for key in self.dict_pairs:
            get[key] = self.dict_pairs[key].resolve(context)

        path = context['request'].META['PATH_INFO']

        if len(get):
            path += "?%s" % "&".join(
                ["%s=%s" % (key, value) for (key, value) in get.items() if value])

        return path

The first thing it does is create a copy of the current GET
[QueryDict](https://docs.djangoproject.com/en/dev/ref/request-response/#querydict-objects).
A copy is made because the GET object itself is immutable. Using this as a
starting point, the key/value pairs passed in via the tag are added (making
sure to overwrite any of the current GET parameters). The path is retrieved
from the request headers, and then the new GET ```QueryDict``` is checked.  If
it has any items, they are appended to the path to return the new request.

### Final Touches ###

The final step to implement the template tag is to create a function that
the decorators can be applied to. This will be the entry point of the tag.

    :::python
    @register.tag()
    @easy_tag
    def append_to_get(tag_name, args):
        return AppendGetNode(dict)

A function named ```append_to_get``` is created, which takes in the name of the
tag as ```tag_name```, and the single argument of the tag (the comma-delimited
key-value pairs) as ```args```. This function is first decorated by the
```easy_tag``` decorator. As described previously, this decorator:

* accepts a function as an argument
* returns a function that:
    * has the arguments expected of a template tag (taking in the ```parser```
      and ```token```)
    * returns the result of the function it decorates (```append_to_get```)
      after it has been passed the ```split_contents``` results

That function is then decorated *again* with the standard Django
```register.tag``` decorator.  The ```append_to_get``` method itself merely
creates the ```template.Node``` subclass.

### The Final Product ###

Putting everything together, this is what the final product looks like:

    :::python
    from django import template

    register = template.Library()

    """
    Decorator to facilitate template tag creation
    """
    def easy_tag(func):
        """Split the tag into pieces and call the wrapped function with the
	list that is returned."""
        def inner(parser, token):
            try:
                return func(*token.split_contents())
            except TypeError:
                raise template.TemplateSyntaxError(
                    'Bad arguments for tag "%s"' % token.split_contents()[0])
        inner.__name__ = func.__name__
        inner.__doc__ = func.__doc__
        return inner


    class AppendGetNode(template.Node):
        def __init__(self, args):
            self.dict_pairs = {}
            for pair in args.split(','):
                pair = pair.split('=')
                self.dict_pairs[pair[0]] = template.Variable(pair[1])

        def render(self, context):
            get = context['request'].GET.copy()

            for key in self.dict_pairs:
                get[key] = self.dict_pairs[key].resolve(context)

            path = context['request'].META['PATH_INFO']

            if len(get):
                path += "?%s" % "&".join(
                    ["%s=%s" % (key, value) for (key, value) in get.items() if value])

            return path

    @register.tag()
    @easy_tag
    def append_to_get(tag_name, args):
        return AppendGetNode(dict)

### New and Improved ###

With all of the above in place, the repetitive and brittle template from above:

    :::django
    <a href="?page={{ page_obj.next_page_number }}&filter={{ filter }}&sort={{ sort }}">
        next
    </a>

can now be replaced with:

    :::django
    {% load foo_extras %}
    <a href="{% append_to_get page=page_obj.next_page_number %}">
        next
    </a>

None of the other GET parameters need to be worried about. The template also
already handles any new GET parameters that are added, as it will submit the
existing request's parameters without any extra instruction!.
