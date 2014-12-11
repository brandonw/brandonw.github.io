Title: ASP.NET MVC 5 Error Pages
Date: 2014-10-28 20:15
Category: Web Development
Tags: aspnet mvc, errors
Summary: Configuring customizable and correct error pages in ASP.NET MVC

### Introduction ###

The default configuration for errors in ASP.NET MVC leaves a lot to be desired.
There is an error attribute that handles redirecting any errors thrown by an
action method, but this constrains everything to be handled in a single view.
It also feels (subjectively) to be more magic, instead of allowing everything
to be described explicitly. In this post, a flexible and easily understood
error handling mechanism will be detailed.

### Getting Started: Outside of ASP.NET ###

The first step is to make sure everything *outside* of the ASP.NET pipeline is
handled. If an error occurs while IIS is processing a request, but it does not
occur inside of the ASP.NET pipeline, then the configuration specified in the
httpErrors tag in the server's web.config is what describes the error
behaviour. Inside your web.config's ```system.webServer``` tag (direct child of
the ```configuration``` tag), configure the ```httpErrors``` tag:

    :::xml
    <httpErrors errorMode="DetailedLocalOnly">
    </httpErrors>

This will configure the behaviour for HTTP errors occuring in IIS to only be
used when the client is not local. This means that development on a local
machine will receive the default detailed error messages.  If a request is made
from a non-local client, then the configuration specified will be used instead.

Add the configuration for the errors to customize as child tags of the
```httpErrors``` tag:

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

The ```404.html``` and ```500.html``` files should be saved in the project
root, not in the ```Views``` directory. IIS may have trouble finding them if
they are saved elsewhere.

The final configuration should look something like this:

    :::xml
    <httpErrors errorMode="DetailedLocalOnly">
        <remove statusCode="404" />
        <error statusCode="404" path="404.html" responseMode="File" />
        <remove statusCode="500" />
        <error statusCode="500" path="500.html" responseMode="File" />
    </httpErrors>

What must be noted here, is that the error is handled **outside** of ASP.NET.
This means the view engine and session data will not be accessible. This error
page must be a simple html page. Ideally, the client will never see this page.
However, in the event it is seen, it should have a consistent theme with the
rest of the site.

### Clearing Defaults: Remove the Error Attribute ###

ASP.NET projects come pre-configured with default error handling via the
```HandleErrorAttribute``` and the ```Error.cshtml``` view. Remove the addition
of a new ```HandleErrorAttribute``` instance to the global filter collection in
the ```RegisterGlobalFilters``` method of ```FilterConfig.cs```. This will
prevent any exceptions thrown in controller actions from being handled by the
```HandleErrorAttribute```. The ```Error.cshtml``` can now be removed from the
Views/Shared folder as well.

### ASP.NET Error Configuration ###

Back in the web.config, custom errors must be enabled in the ASP.NET pipeline:

    :::xml
    <customErrors mode="RemoteOnly"></customErrors>

This little snippet is similar to the IIS configuration. It tells ASP.NET that
custom errors are enabled, but should only be used when the client is not
local. The ```customErrors``` element also allows for individual error page
configuration, similar to the ```httpErrors``` tag above. However, that method
will be skipped in favor of a more flexible approach.

### Configuring Error Behaviour ###

The next step is to configure what ASP.NET should do when an
```HTTPException``` is thrown. To begin, add an ```Application_Error``` stub to
```Global.asax.cs```:

    :::csharp
    protected void Application_Error(object sender, EventArgs e)
    {

    }

This method is called any time an uncaught exception trickles up to the edge of
ASP.NET.  It will be the primary mechanism by which the error response is
controlled.

The first step is to check if custom errors are enabled:

    :::csharp
    var app = (MvcApplication)sender;
    var context = app.Context;

    if (!context.IsCustomErrorEnabled)
        return;

If they are not enabled, no further action is required. The error will continue
and eventually render the familiar detailed error page. If custom errors are
enabled, the error must be retrieved, and then the context response cleared.

    :::csharp
    var ex = app.Server.GetLastError();
    context.Response.Clear();
    context.ClearError();

This takes any existing response (namely, the default detailed error page), and
clears it from the context. There is now a clean slate to work with, and routing
can be handled:

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

First, the exception is cast into an HttpException, and saved for later. The
default routing behaviour is then set. Here, an ```Error``` controller is used,
with several actions defined therein. The default action is to use the 500
error, which is mapped to the ```InternalServerError``` action. The exception
is also added to the route data, in case it is needed in the controller action.

The exception must now be checked. If it is not an ```HttpException```, then the
default action of ```InternalServerError``` is executed. If the exception
**is** an ```HttpException```, then the the HTTP code can be conditionally
tested against. Three actions are specified in this controller, but any number
can be added.

The ```Application_Error``` method should look something like this:

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

If additional routing data needs to be sent to the action, then all the power
of C# is available. A subclass of ```HttpException``` could be made in order to
customize 404 behaviour by tracking the entity type that was not found.  A
customized 404 page for each entity in the application would be an easy step
from here.

### Custom Error Templates ###
Error templates can now be made. Assuming the configuration above, create a new
controller named ```Error```, and add action methods according to the actions
defined in ```Application_Error```. Add view templates that will be rendered
according to the required action methods, and everything will be ready. The
best part about this setup is that there is no magic!  The controller, actions,
and templates are all exactly the same as any other in the application. Nothing
is special-cased.

### How to Handle Errors In Code ###
Now that everything is configured, whenever a request is received for an action
with an ID that is not found, you can:

    :::csharp
    throw new HttpException(404, "Foo entity not found.");

and everything will be handled. If a client tries to reach an unauthorized
action:

    :::csharp
    throw new HttpException(403, "Forbidden");

Any other error that is thrown will be handled as an internal server error, and
the server will render the appropriate response.

### Adding a Catch-All Not Found Route ###
The previously mentioned way to handle 404 errors will work in any case where
the controller exists. However, what if the client requests a URL that does not
match any controllers? In this case, ASP.NET sees no route that matches in its
routing table, so it lets the request percolate up to IIS. This is where the
previously configured IIS error pages would normally come in.  However, that
can be avoided. Add a catch-all route to the routing table after all existing
```MapRoute``` calls in ```RouteConfig.cs```:

    :::csharp
    routes.MapRoute(
        name: "404",
        url: "{*url}",
        defaults: new { controller = "Error", action = "CatchAllAction" }
    );

This will define a catch-all route that is only matched if no other route
defined matches the request. Note that the action is set to
```CatchAllAction```. This could just as easily use the existing NotFound
action, if there is no need to distinguish between an object not being found
and a route not being recognized.

### Carry On! ###

Use this new error routing process to handle routes in a flexible and obvious
manner, and don't worry about how it will be resolved down the road!.
