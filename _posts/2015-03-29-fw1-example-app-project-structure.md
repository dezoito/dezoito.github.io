---
layout: post
title: FW/1 Example Application - Project Structure
excerpt_separator: <!--more-->
---

This article talks about the topic of project structure in a
[ColdFusion and FW/1 Example Application](https://dezoito.github.io/2015/03/26/fw1-example-app-released/)
and is the first one in the following series:

 - Project Structure
 - [Forms and Validation Patterns](/2015/03/30/fw1-example-app-forms_validation/)
 - Use of User Defined Function Libraries
 - Accessing an External Service
 - BDD and Integration Tests.

It assumes you are not new to CFML, so don't take this as a detailed guide or
tutorial, but more of a 'getting started' reference on using this MVC framework
to create Object-Oriented ColdFusion apps.

 -----

## FW/1 Project Structure

Reading pre-requisites: [Developing Applications with FW/1](https://github.com/framework-one/fw1/wiki/Developing-Applications-Manual).

The Clipping app was designed with
[Subsystems](https://github.com/framework-one/fw1/wiki/Using-Subsystems) enabled.
Using those is completely optional, but I decided to use them so that, in the future,
it will be easier to add functionality, by incorporating third party apps.

The basic project structure:

```
/clipping
    |
    +- common
        |
        +- layouts
    |
    +- customtags
    +- home
        |
        +- controllers
        +- model
            |
            +- beans
            +- services
        |
        +- views
            |
            +- clipping
            +- helpers
            +- main
    |
    +- lib
    +- setup
    +- static
    +- tests

/testbox
/CFSelenium
```

With the `/clipping` folder being the project's root, we have:

`/common` - This folder holds the layout files that will be used in **every** subsystem.

`/customtags` - This is where CFML tags are stored. I added a mapping in Application.cfc pointing to this folder.

`/home` - The directory that contains the clipping application's code -  A.K.A. the `default subsystem` -
with its own controllers, models and views.

`/lib` - Stores User Defined Functions and Libraries, that will be available to all subsystems.

`/setup` - Stores SQL scripts used to prepare the databases
(those files are **not** used by the application, so feel free to delete this).

`/static` - Keeps all static files, separated in /js, /css, /images and /ckeditor subfolders.

`/tests` - This is where we keep and run 'unit' and integration tests (more on that later.)

**Important:**

The `/testbox` and `/CFSelenium` folders are int the webserver's root for simplicity.
You should not use this configuration in a web-accessible environment.

-----

## FW/1 Application.cfc file

Here's the interesting bits of code:

{% highlight js %}
// ------------------------ APPLICATION SETTINGS ------------------------ //
this.name = "clipping_app";
this.sessionManagement = true;
this.sessionTimeout = createTimeSpan(0,2,0,0);
this.dataSource = "dtb_clipping";
this.test_datasource = "dtb_clipping_test";
this.ormEnabled = true;
this.ormsettings = {
    dbcreate="update",
    dialect="MySQL",
    eventhandling="False",
    eventhandler="root.home.model.beans.eventhandler",
    logsql="true",
    flushAtRequestEnd = "false"
};
{% endhighlight %}

Seems pretty straight forward, right? Notice that we also set a `'test_datasource'` variable
(that should point to a mirror database), to be used exclusively when running tests.
This is **completely optional**, and I'll write about some pros and cons in the "running tests" article.


{% highlight js %}
    // mappings and other settings
    this.mappings["/root"] = getDirectoryFromPath(getCurrentTemplatePath());
    this.customTagPaths = this.mappings["/root"] & "customtags"
    this.triggerDataMember = true // so we can access properties directly (no need for getters and setters)
{% endhighlight %}

The second mapping tells our application server to look for custom tags in this
app's specific folder (I like this approach since it makes source control and distribution easier)

Framework settings below:

{% highlight js %}
// ------------------------ FW/1 SETTINGS ------------------------ //
variables.framework = {
    reloadApplicationOnEveryRequest = true, //use only in dev
    trace = false,
    // places where you don't want to load the framework
    unhandledPaths = '/clipping/tests/',
        .....
        .....
    // enable the use of subsystems
    usingSubsystems = true,
    defaultSubsystem = 'home',
    siteWideLayoutSubsystem = 'common',

    // changes for FW/1 3.0
    diLocations = "./home/model/"
}
{% endhighlight %}

`unhandled paths` is a list of subdirectories where we don't want to use FW/1.
In this case it's only the `tests` folder, but I can see this being used when
integrating with legacy apps, for example.

`diLocations`, according to the docs, is "<em>the set of folders that DI/1, AOP/1,
or WireBox will scan for CFCs.</em>" - This parameter was added in version 3.0.
In this case, it's referencing the path to the models in the 'home' subsystem.


Here's a way to define settings according to environment:

{% highlight js %}
// ------------------------ ENVIRONMENT DEFINITIONS ------------------------ //
public function getEnvironment() {
   if ( findNoCase( "localhost", CGI.SERVER_NAME ) ) return "prod";
   if ( findNoCase( "127.0.0.1", CGI.SERVER_NAME ) ) return "dev";
   else return "prod";
}

variables.framework.environments = {
   dev = { reloadApplicationOnEveryRequest = true,  trace = true,},
   prod = { password = "supersecret" }
}
{% endhighlight %}

Setting application scoped variables when the app starts:

{% highlight js %}
function setupApplication() {

    // copy dsn names to application scope
    application.datasource = this.datasource;
    application.recordsPerPage = 12 // pagination setting,
                                    // used in all services and tests

    // include UDF functions
    // functions inside the CFC can be referred by application.UDFs.functionName()
    application.UDFs = createObject("component", "lib.functions");

    // settings used in tests
    application.test_datasource = this.test_datasource;
    application.testsRootMapping = "/clipping/tests/specs";
    application.testsBrowseURL = "http://" & CGI.HTTP_HOST & "/clipping";
}
{% endhighlight %}

Please notice how the line `application.UDFs = createObject("component", "lib.functions");`
saves an instance of the 'functions' library to the application scope,
so those functions can be used anywhere.

Finally:

{% highlight js %}
    function setupRequest() {
        // CSRF Token, unique for each user/request
        request.csrftoken = CSRFGenerateToken();
    }
{% endhighlight %}

This generates a CSRF Token at the start of the every request.
We are going to be using this when forms are submitted, to prevent
[CSRF attacks](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29).

There are other setup methods available to be used in the `Application.cfc`.
They are optional but I left them in the code for future reference.

Please see the **[fw1-clipping](https://github.com/dezoito/fw1-clipping)** github
project page for the full source code.





