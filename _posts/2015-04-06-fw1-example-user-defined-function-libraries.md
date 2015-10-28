---
layout: post
comments: false
title: FW/1 Example Application - User Defined Function Libraries
excerpt_separator: <!--more-->
---

This is the third in a series of articles on how to build a
[ColdFusion and FW/1 Example Application](https://dezoito.github.io/2015/03/26/fw1-example-app-released/),
and it discusses how you can structure your User Defined Functions in libraries
that can be **easily** accessed in any part of your project, whether it's a view,
controller, service or model.


### Structuring and Accessing Function Libraries.

A UDF library is a like a toolbox for the developer, and while a few of these tools
(functions) maybe used rarely, over time, many of them will endup being used in
every application (sometimes in several pieces of code).

As an example, I use this function to "clean" strings before **every** database
Insert or Update:

{% highlight js %}
        /**
         *  Prepares a string to be inserted (or updated on a DB):
         *  - removes extra espaces
         *  - removes unnecessary single quotes
         *  - keeps numeral characters consistent
         */
        function prepara_string(texto){
            return this.remove_aspas_simples_extras(this.normaliza_numeral(this.remove_espaco_extra(trim(texto))));
        }
        this.prepara_string = prepara_string;

        // more functions implemented below
{% endhighlight %}

As you can probably guess, the `prepara_string()` function depends on a series
of other UDFs...those are all on the same `/lib/functions.cfc` file and I'm
leaving them out for brevity.

The important thing to consider is that I need to use this function before saving
data to a database, and that might happen on and ORM or SQL call, on a service,
and even on a model.

So I createad my function library as a CFC component, and saved an instance in the
`application scope`:

`/lib/functions.cfc`
{% highlight cfm %}

<cfcomponent cacheUse="read-only" output="false">

    <cfscript>
        /**
         * prevents CSRF attacks by checking for valid CSRF Tokens
         */
        function abortOnCSRFAttack( struct rc ){
            if(!structKeyExists(rc, "csrfToken") || (!CSRFVerifyToken(rc.csrfToken))){
                abort showerror="Invalid CSRF Token...aborting execution.";
            }
        }
        this.abortOnCSRFAttack = abortOnCSRFAttack;

        /**
         * stripHTML description: removes HTML tags from a string
         * (from http://www.cflib.org/udf/stripHTML)
         */
        string function stripHTML(str) output="false" {
            // return REReplaceNoCase(arguments.str,"<[^>]*>","","ALL");

            .....
            .....
    </cfscript>
</cfcomponent>
{% endhighlight %}

 Here's how this is saved to the application scope:

 `Application.cfc`

{% highlight js %}
    // ------------------------ CALLED WHEN APPLICATION STARTS ------------------------ //
    function setupApplication() {

        .....

        // include UDF functions
        // the functions inside the CFC cann be referred by application.UDFs.functionName()
        application.UDFs = createObject("component", "lib.functions");

        .....
        .....
    }
{% endhighlight %}

Now, I don't want to get into a debate over tight vs loose coupling - using that
library from the application scope is certainly "tight"; I merelly wanted an easy
and convenient way to make these functions accessible throughout the application,
especially where data is saved to the DB.

Below, we use our clipping bean as an example:

`home/model/beans/clipping.cfc`

{% highlight js %}
component persistent="true" table="tbl_clipping" accessors="true" {

    property name="clipping_id" generator="native" ormtype="integer" fieldtype="id";
    property name="clipping_titulo" ormtype="string" length="255" notnull="true";
    property name="clipping_texto" ormtype="text" notnull="true";
    property name="clipping_link".......
    .....
    .....

    public function clean(){
        UDFs = application.UDFs
        this.setClipping_titulo(UDFs.prepara_string(UDFs.stripHTML(variables.clipping_titulo)));
        this.setClipping_texto(UDFs.safetext(variables.clipping_texto, true));
        this.setClipping_link(UDFs.prepara_string(UDFs.stripHTML(variables.clipping_link)));
        this.setClipping_fonte(UDFs.prepara_string(UDFs.stripHTML(variables.clipping_fonte)));

        // try to format only if the user submitted a valid eurodate
        if(isValid("eurodate", variables.Published)){
            this.setPublished(dateformat(variables.Published, "dd/mm/yyyy")); // handle eurodates
        }
    }
}

{% endhighlight %}

Borrowing yet again from the [Django Framework](https://www.djangoproject.com),
the `clean()` function is applied to fields before an object is saved.

In the example, it uses the `prepara_strings()` and `savetext()` UDFs to format
text entered in form fields, so they become "safe" to go into our database.

**Note**:

If you want to use a less tightly coupled approach, you could create an instance
of the UDF library only where and when it's needed:

{% highlight js %}
component persistent="true" table="tbl_clipping" accessors="true" {

    property name="clipping_id" generator="native" ormtype="integer" fieldtype="id";
    property name="clipping_titulo" ormtype="string" length="255" notnull="true";
    ......

    // Instantiate UDF library component
    UDFs = createObject("component", "lib.functions");

    // use it to format strings:
    this.setClipping_texto(UDFs.safetext(variables.clipping_texto, true))
}

{% endhighlight %}

 ----


For more detailed information on this project, follow the other articles in this series:

 - [Project Structure](/2015/03/29/fw1-example-app-project-structure/)
 - [Forms and Validation Patterns](/2015/03/30/fw1-example-app-forms_validation/)
 - Use of User Defined Function Libraries
 - [Interaction with a Remote Service](/2015/04/07/fw1-example-accessing-external-service/)
 - [BDD and Integration Tests](/2015/04/18/fw1-example-bdd-integration-testing/)

For the full source code, please visit the **[fw1-clipping](https://github.com/dezoito/fw1-clipping)**
github project page.

