Title: ASP.NET MVC 5 Error Pages
Date: 2014-10-28 20:15
Category: Web Development
Tags: aspnet mvc, errors
Summary: Configuring customizable and correct error pages in ASP.NET MVC

### Introduction ###

The default configuration for errors in ASP.NET MVC leaves a lot to be desired.
There is an error attribute that handles redirecting any errors thrown by an
action method, but this constrains you to handling everything in a single view.
It also feels (subjectively) to be more magic, instead of allowing you to
describe everything explicitly. In this post, I  will detail how I set up error
handling in a way that is both flexible and easy to understand.

### Getting Started: Outside of ASP.NET ###

The first step you must do is to make sure you have handled anything that
occurs *outside* of the ASP.NET pipeline. If an error occurs while IIS is
processing a request, but it does not occur inside of the ASP.NET pipeline,
then the configuration specified in the httpErrors tag in the server's
web.config is what describes the error behaviour. Inside your web.config's
```system.webServer``` tag (direct child of the ```configuration``` tag),
configure the ```httpErrors``` tag:

    :::xml
    <httpErrors errorMode="DetailedLocalOnly">
    </httpErrors>

This will configure the behaviour for HTTP errors occuring in IIS to only be
used when the client is not local. This means that while you are developing on
your local machine, you will still receive the default detailed error messages.
If you make a request from a non-local client, however, then the configuration
specified will be used instead.

Now, add the configuration for the errors you want to customize as child tags
of the ```httpErrors``` tag:

    :::xml
    <remove statusCode="404" />
    <error statusCode="404" path="404.html" responseMode="File" />
    <remove statusCode="500" />
    <error statusCode="500" path="500.html" responseMode="File" />

What this does is to remove any existing error handling for both 404 and 500
errors which may have been set up at a higher level configuration file (like
the user-level or machine-level config). After removing any existing
configuration, it adds a fresh entry to the configuration, specifying the html
page and response mode for the given error. The ```responseMode``` attribute
value  of **File** tells the IIS server to rewrite the response with the
contents of the specified file. This is important to do, in order to maintain
the original request URL. It would be confusing to send a request to
```/foo/bar```, encounter a 404 error, and then be redirected to
```/404.html```.  Not only will the user see a URL they did not request, but
the original URL will be lost.

The ```404.html``` and ```500.html``` files should be saved in your project
root, not in your ```Views``` directory. IIS may have trouble finding them if
you save them elsewhere.

The final configuration should look something like this:

    :::xml
    <httpErrors errorMode="DetailedLocalOnly">
        <remove statusCode="404" />
        <error statusCode="404" path="404.html" responseMode="File" />
        <remove statusCode="500" />
        <error statusCode="500" path="500.html" responseMode="File" />
    </httpErrors>

What you must note here, is that the error is handled **outside** of ASP.NET.
This means you do not have access to any view engine for templating, nor do you
have any session data (authentication, authorization, cookies, etc.). This
error page must be a simple html page. Ideally, the client will never see this
page. However, you want to have something that will mesh well with the theme of
your site in the rare chance that a client does end up seeing it. If you do not
set this up, then the default error page will be rendered.

### Clearing Defaults: Remove the Error Attribute ###

ASP.NET projects come pre-configured with default error handling via the
```HandleErrorAttribute``` and the ```Error.cshtml``` view. Remove the addition
of a new ```HandleErrorAttribute``` instance to the global filter collection in
your ```RegisterGlobalFilters``` method of ```FilterConfig.cs```. This will
prevent any exceptions thrown from your controller actions from being handled
by the ```HandleErrorAttribute```. You can now remove ```Error.cshtml``` from
your Views/Shared folder as well.

### ASP.NET Error Configuration ###

Back in the web.config of your project, you must enable custom errors in the
ASP.NET pipeline:

    :::xml
    <customErrors mode="RemoteOnly"></customErrors>

This little snippet is similar to the IIS configuration. It tells ASP.NET that
custom errors are enabled, but should only be used when the client is not
local. The ```customErrors``` element also allows you to configure individual
error pages, similarly to the ```httpErrors``` tag above. However, we will be
using a much more flexible mechanism.

### Configuring Error Behaviour ###

The next step is to configure the steps ASP.NET should take when an
```HTTPException``` is thrown. To begin, open up your project's
```Global.asax.cs``` file, and add a stub ```Application_Error``` method:

    :::csharp
    protected void Application_Error(object sender, EventArgs e)
    {

    }

This method is called any time an exception is thrown by an action in one of
your controllers, and will be the primary mechanism by which we control what
the client sees when an error is encountered.

The first step we must take in this method is to check if custom errors are
enabled.

    :::csharp
    var app = (MvcApplication)sender;
    var context = app.Context;

    if (!context.IsCustomErrorEnabled)
        return;

If they are not enabled, we do not want to take any further action. The error
will continue and eventually render the familiar detailed error page. If custom
errors are enabled, we want to get the error, and then clear the context's
response.

    :::csharp
    var ex = app.Server.GetLastError();
    context.Response.Clear();
    context.ClearError();

This takes any existing response (namely, the default detailed error page), and
clears it from the context. There is now a clean slate to work with, and we can
get to the routing:

    :::csharp
    var httpException = ex as HttpException;
    var routeData = new RouteData();
    routeData.Values["controller"] = "Error";
    routeData.Values["exception"] = ex;
    routeData.Values["action"] = "InternalServerError";
    if (httpException != null)
    {
        switch (httpException.GetHttpCode())
        {
            case 403:
                routeData.Values["action"] = "Forbidden";
                break;
            case 404:
                routeData.Values["action"] = "NotFound";
                break;
            case 500:
                routeData.Values["action"] = "InternalServerError";
                break;
        }
    }
    IController controller = new ErrorController();
    controller.Execute(new RequestContext(new HttpContextWrapper(context), routeData));

First, the exception is cast into an HttpException, and saved for later.  Then,
the default routing behaviour is set. I use an ```Error``` controller, with
several actions defined therein. The default action is to use the 500 error,
which I map to the ```InternalServerError``` action. I also add the exception
to the route data in case I want to use it in the controller action.

Then the exception is checked. If it is not an ```HttpException```, then the
default action of ```InternalServerError``` is executed. If the exception
**is** an ```HttpException```, then the the HTTP code can be conditionally
tested against. I have specified a total of three actions in my controller, but
you can add however many you like.

In the end, your ```Application_Error``` method should look something like
 this:

    :::csharp
    protected void Application_Error(object sender, EventArgs e)
    {
        var app = (MvcApplication)sender;
        var context = app.Context;

        if (!context.IsCustomErrorEnabled)
            return;

        var ex = app.Server.GetLastError();
        context.Response.Clear();
        context.ClearError();

        var httpException = ex as HttpException;
        var routeData = new RouteData();
        routeData.Values["controller"] = "Error";
        routeData.Values["exception"] = ex;
        routeData.Values["action"] = "InternalServerError";
        if (httpException != null)
        {
            switch (httpException.GetHttpCode())
            {
                case 403:
                    routeData.Values["action"] = "Forbidden";
                    break;
                case 404:
                    routeData.Values["action"] = "NotFound";
                    break;
                case 500:
                    routeData.Values["action"] = "InternalServerError";
                    break;
            }
        }
        IController controller = new ErrorController();
        controller.Execute(new RequestContext(new HttpContextWrapper(context), routeData));
    }

If you want additional routing data to be sent to the action, then you have all
the power of C# at your disposal. For instance, you could subclass
```HttpException``` to have a new 404 exception class that tracks the type of
entity that was not found. A customized 404 page for each entity in your
application would be an easy step from here.

### Your Custom Error Templates ###
Your error templates can now be made. If you used my routing configuration
above, then create a new controller named ```Error```, and add action methods
according to the actions defined in your ```Application_Error``` method. Add
view templates that will be rendered according to your action methods, and you
should be good to go. The best part about this setup is that there is no magic!
The controller, actions, and templates are all exactly the same as any other in
your application. You do not have to worry about any special cases.

### How to Handle Errors In Your Code ###
Now that everything is configured, whenever you receive a request for an action
with an ID that is not found, you can
 ```throw new HttpException(404, "Foo entity not found.");```, and everything
will be handled for you. If a client tries to reach an action they should not
have access to, you can ```throw new HttpException(403, "Forbidden");```. Any
other error that is thrown will be handled as an internal server error, and the
server will render the appropriate response.

### Adding a Catch-All Not Found Route ###
The previously mentioned way to handle 404 errors will work in any case where
the controller exists. However, what if the client requests a URL that does not
match any of your controllers? In this case, ASP.NET sees no route that matches
in its routing table, so it would let the request percolate back up to IIS.
You will probably not have anything set up to handle the request in IIS, so
this is where your previously configured IIS error pages would normally come
in. However, we would rather avoid that, if possible.  This is easily fixed by
adding a catch-all route to the routing table in your RouteConfig.cs. After all
existing ```MapRoute``` calls, add:

    :::csharp
    routes.MapRoute(
        name: "404",
        url: "{*url}",
        defaults: new { controller = "Error", action = "CatchAllAction" }
    );

This will define a catch-all route that is only matched if no other route you
defined matches the request. Note that the action is set to
```CatchAllAction```. This could just as easily use the existing NotFound
action, if you do not care to distinguish between an object not being found
and a route not being recognized.

### Carry On! ###

You now have an easily understood and flexible error handling process. Carry on
with your web application without worrying about how you are going to handle
errors down the road.
