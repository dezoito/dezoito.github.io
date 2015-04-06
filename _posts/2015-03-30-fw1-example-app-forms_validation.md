---
layout: post
title: FW/1 Example Application - Forms and Validation
excerpt_separator: <!--more-->
---

This is the second in a series of articles on how to build a
[ColdFusion and FW/1 Example Application](https://dezoito.github.io/2015/03/26/fw1-example-app-released/),
and it discusses the topic of creating forms and validating submitted data
in an efficient and [DRY](http://programmer.97things.oreilly.com/wiki/index.php/Don%27t_Repeat_Yourself) (Don't Repeat Yourself) way.


 -----
## FW/1 - Patterns for Forms and Validation

Unlike [Django](https://www.djangoproject.com/), the [FW/1 framework](http://framework-one.github.io/) doesn't come
with a "native" way of writing forms, or performing data validation before commiting
changes to the database, so I wrote my own after researching and asking for guidance
on the Framework-One Group.

The proposed pattern used in this project has these goals:

- Use a **single view** to display Create, Update and Delete forms
- Perform validation at "entity level" (meaning that the validation rules are
written within the entity definition component, keeping things in a single file)
- When a validation error occurs, reload the form, with the data last typed by
the user, and inline error messages next to the form fields.
- Perform CSRF validation

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/clipping_form_validation.png?raw=true)
<small>View of a failed form submission</small>



-----

## Form Controller Code

The Clipping Controller is written in `/home/controllers/clipping.cfc`, with the `form()`
method being responsible for:

1. Handling form submissions and performing validation
2. Loading existing data (in case we are updating an article)
3. Loading validation errors, if any
4. Rendering the "form" view

{% highlight js %}
function form (struct rc){

    // Checks if the form is being displayed after a failed validation
    // (i.e. if this function was called from the save() method in this controller)
    // If NOT, instantiate a clipping entity to fill the form fields
    // If so, just display the last data used when filling the forms
    if(!structKeyExists(rc, "Clipping")){

        param name="rc.clipping_id" default="0";

        if(isValid("integer",rc.clipping_id) && val(rc.clipping_id)) {
            rc.Clipping = variables.clippingService.getClipping(rc.clipping_id);
            // if a valid instance was not returned, return error.
            if(IsNull(rc.Clipping)) {
                framework.frameworkTrace( "ORM query returned no Objects.");
                framework.redirect("main");
            }
        } else {
            // if we don't have a valid id,
            // initialize object with the needed defaults
            rc.Clipping = entityNew("clipping");
        }
    }
    // will render clipping.form view from here...
}
{% endhighlight %}

The `if(!structKeyExists(rc, "Clipping"))` code checks to see if we previously saved
an instance of a Clipping on the `Request Context` struct. If we did, it means that we
attempted to save a form sdata to that instance, but some field(s) failed to be validated.

In this case, we can skip instantiating the object and go straight into displaying the form.

-----

## Form View Code
We use a single file to display the form, written in `/home/views/clipping/form.cfm`.

Instead of displaying the whole thing, I'll just highlight the important parts:

Using FW/1 syntax to set the `action` attribute - form data will be handled by
the `save()` method in the `clipping` controller.

{% highlight cfm %}
<form action="#buildURL('clipping.save')#"
    method="post"
    role="form"
    class="form-horizontal"
    id="f_clipping">

    <input name="csrftoken" type="hidden" value="#request.csrfToken#">

    ........
{% endhighlight %}

Noticed that we also set a hidden field with the `request.csrfToken` we
defined in the `application.cfc`.

This value will be checked to avoid CSRF attacks.


Below, we check the RC struct for validation errors and display them in a
dismissable alert box.
{% highlight cfm %}
<!---    display alert if there were errors     --->
<cfif structKeyExists(rc, "stErrors") and (structCount(rc.stErrors) gt 0)>
    <div class="alert alert-danger">
        <a href="#" class="close" data-dismiss="alert">&times;</a>
            <b>Your article could not be posted!</b><br/>
            Please fix the errors below:
    </div>
</cfif>

{% endhighlight %}


This is how the form fields are setup.

They'll be filled with existing data
(or whatever data the user attempted to use before submission).

{% highlight cfm %}
    <div class="form-group">
        <label for="clipping_titulo" class="control-label col-sm-2">Title <span class="required">*</span></label>
        <div class="col-sm-9">
            <input type="text" name="clipping_titulo" id="clipping_titulo"
                value="#HTMLEditFormat(rc.Clipping.getClipping_titulo())#" size="100" class="form-control">
                <!---    display errors?    --->
                #view("helpers/_field_error", {field="clipping_titulo"})#
        </div>
    </div>

{% endhighlight %}


We also invoke the `helpers/_field_error`, passing the fieldname,
and it will display the appropriate error message if needed:


{% highlight cfm %}
<!--- /home/views/clipping/helpers/_field_error.cfm --->
<cfif isDefined("rc.stErrors.#local.field#")>
    <cfoutput><p class="alert alert-danger">#rc.stErrors[local.field]#</p></cfoutput>
</cfif>
{% endhighlight %}

-----

## Saving Form Data
After a form is submitted, the request follows this route:

<pre>
  form
    |
request data (RC struct)
    |
clipping controller
    |
clipping service
    |
clipping bean
    |
clipping service
    |
clipping controller
    |
   view
</pre>

**Clipping controller**: `save()`:

{% highlight js %}
function save( struct rc ) {
    framework.frameworkTrace( "<b>Save Method on Clipping Controller</b>");

    // abort execution in case of CRSF attack (use UDF defined in lib.functions.cfc)
    application.UDFs.abortOnCSRFAttack( rc );

    // save (insert or update) this object
    // using the clippingService
    rc.Clipping = variables.clippingService.save(rc);

    // passed validation?
    if(rc.Clipping.validate().isValid){
        // since there's no clipping.save view, we have to redirect somewhere
        // (in this case, to the main list)
        framework.redirect("main.default");
    } else {
        // Invalid data!
        // copy errors to struct in RC and display form again
        rc.stErrors = rc.Clipping.validate().stErrors
        framework.redirect("clipping.form", "all");
    }
}
{% endhighlight %}
The controller verifies the CSRF token (aborts on failure) then invokes the `clippingService`,
attempting to save the form's data.

It will return to the main page if succssfull, or reload the form if validation fails.

**ClippingService**: `save()`:

{% highlight js %}
public any function save(struct rc) {
    transaction {

        //  Insert or Update?
        if(val(arguments.rc.clipping_id)){
            var c = entityLoadByPk("Clipping", arguments.rc.clipping_id);
        } else {
            var c = entityNew("clipping");
        }

        // populate clipping component
        c.setClipping_titulo(arguments.rc.clipping_titulo);
        c.setClipping_texto(arguments.rc.clipping_texto);
        c.setClipping_link(arguments.rc.clipping_link);
        c.setClipping_fonte(arguments.rc.clipping_fonte);
        c.setPublished(arguments.rc.Published);

        // cleans and formats fields so they can be validated/saved
        c.clean();

        // commit changes IF data is valid
        if (c.validate().isValid) {
            entitySave(c);
            transactionCommit();
        } else {
            // since the data was invalid, don't save and
            // rollback any pending transactions
            // (we are checking for validation errors in the controller)
            transactionRollback();
        }
    }
    return c;
}
{% endhighlight %}

In the code above, we instantiate and populate a Clipping object
using the submitted data and then run two methods:

- `c.clean()` - Sanitizes strings and dates so they can be safely inserted
or Updated in the DB. It uses the function library saved in the application scope
(see `Application.cfc`).
- `c.validate()` - Checks the cleaned data against a set of rules.

`Validate()` returns an `isValid` boolean, indicating whether data can be saved
or not, and a 'stErrors' struct, with keys/values representing fields that failed
validation and their fail error messages.

These methods are defined in the Clipping bean:`/home/models/beans/clipping.cfc`.

**Clipping Bean**:

{% highlight js %}
component persistent="true" table="tbl_clipping" accessors="true" {

    property name="clipping_id" generator="native" ormtype="integer" fieldtype="id";
    property name="clipping_titulo" ormtype="string" length="255" notnull="true";
    .......

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

    public function validate() {
        stValidation = {};
        stErrors = {};

        if(!len(trim(variables.clipping_titulo))) {
            structInsert(stErrors,"clipping_titulo","You must include a title for your clipping.");
        }

        ....

        if(!len(trim(variables.published)) || !isValid("eurodate", trim(variables.published))) {
            structInsert(stErrors,"published","You must specify a valid publishing date.");
        }

        stValidation.isValid = !val(structCount(stErrors)); // true if no errors
        stValidation.stErrors = stErrors;
        return stValidation;
    }
}
{% endhighlight %}

Having a `clean()` and `validate()` defined in the model is <del>stolen</del>borrowed from the
Django framework and makes it easier to keep rules consistent across different
services and controllers.

For more detailed information on this project, follow the other articles in this series:

 - [Project Structure](/2015/03/29/fw1-example-app-project-structure/)
 - Forms and Validation Patterns
 - Use of User Defined Function Libraries
 - Accessing an External Service
 - BDD and Integration Tests.

For the full source code, please visit the **[fw1-clipping](https://github.com/dezoito/fw1-clipping)**
github project page.

