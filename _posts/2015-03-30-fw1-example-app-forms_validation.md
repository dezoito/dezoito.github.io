---
layout: post
title: FW/1 Example Application - Forms and Validation
---

This is the second in a series of articles on how to build a
[ColdFusion and FW/1 Example Application](https://dezoito.github.io/2015/03/26/fw1-example-app-released/).

These articles follow this sequence:

 - [Project Structure](/2015/03/29/fw1-example-app-project-structure/)
 - Forms and Validation Patterns
 - Use of User Defined Function Libraries
 - Accessing an External Service
 - BDD and Integration Tests.

 -----

## FW/1 - Patterns for Forms and Validation

Unlike [Django](https://www.djangoproject.com/), the FW/1 framework doesn't come
with a "native" way of writing forms, or performing data validation before commiting
changes to the database, so I wrote my own after researching and asking for opinions
on the Framework-One Group.

The proposed pattern used in this project has these goals:

 - Use the save view to display Create, Update and delete forms
 - Perform validation at "entity level" (meaning that the validation rules are
 written within the entity definition component, keeping things in a single file)
 - When a validation error occurs, reload the form, with the data last typed by
 the user, and inline error messages next to the form fields.

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







