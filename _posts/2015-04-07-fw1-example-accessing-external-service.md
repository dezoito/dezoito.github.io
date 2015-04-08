---
layout: post
title: FW/1 Example Application - Interaction with an External Service
excerpt_separator: <!--more-->
---

On this fourth article on how to build a
[ColdFusion and FW/1 Example Application](https://dezoito.github.io/2015/03/26/fw1-example-app-released/),
I'll present a way to access an external service that generate a succint summary
for a selected article.

The external service, in this case, is [flask-Summarizer](https://github.com/dezoito/flask-Summarizer)
- an API written in Python/Flask, that receives a `POST` request with some
paragraphs of text, and returns a string with a summarized version of that.

### FW/1 - Interaction with an External Service
To make this series a little more interesting, I used an ajax call to load the
summary into a modal window, whenever the user clicks on the "View Summary" link:

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/view_summary.png?raw=true)

The link above is constructed with the following code:

{% highlight cfm %}
<a class="summaryLink"
    href="javascript: ajaxViewSummary('#buildURL('clipping.summary')#',#Clipping.getClipping_Id()#);">View Summary (Ajax)</a>
{% endhighlight %}

Yes, I understand that there's a lot going on, but bear with me:

`ajaxViewSummary()` is our custom javascript function that:

1. Passes the ID of a clipping article to the "summary" controller
2. Receives the output of that controller (either a summarized text or an error message)
3. Loads a beautiful modal window with the output above.

`#buildURL('clipping.summary')#` is a FW/1 framework function that generates the
 URL to a "section / item" pair. In this case, it calls the `summary()` method
 in the `clipping` controller.

 `#Clipping.getClipping_Id()#` outputs the id of the current clipping object
 (in more complicated terms, it invokes the "getter" function for the clipping object's id).

 All these steps render an HTML output similar to:

{% highlight html %}
<a class="summaryLink"
href="javascript: ajaxViewSummary('/clipping/index.cfm?action=clipping.summary',49);">View Summary (Ajax)</a>
{% endhighlight %}

For your reference, here's the  function in `/static/js/clipping.js`:

{% highlight js %}
function ajaxViewSummary(url, clipping_id){
  // if no id was passed, set it to zero
  clipping_id = typeof clipping_id !== 'undefined' ? clipping_id : 0;

  // load modal window
  $.get( url + '&clipping_id=' + clipping_id, function( data ) {
    $( ".modal-body" ).html( data );
    $( ".modal-title" ).html( "Article Summary" );
    $('#myModal').modal({show:true});
  });
}
{% endhighlight %}

Modal windows are provided by [Bootstrap CSS](http://getbootstrap.com/javascript/#modals),
so I won't detail that.

 ----

### Summary Controller

I apologize for the complicated javascript+cfml interaction above
(it could be more elegant), but now that it's over, let's move on to
what happens in the clipping controller - `/home/controllers/clipping.cfc`

{% highlight js %}
component accessors="true" {

    /**
     * This controller needs the services bellow
     * (they are found in <subsystem>/models/services)
     */
    property clippingService;
    property summaryService;

    /**
     * init FW variables and methods so that they are available to this controller
     */
    property framework;

    /**
     * Uses a webservice to summarize the Article's text
     * It retuns only TEXT and does not use a layout
     */
    function summary( struct rc ) {
        framework.frameworkTrace( "Summary Method on Clipping Controller");
        rc.Clipping = variables.clippingService.getClipping(rc.clipping_id);
        rc.Summary = variables.summaryService.getSummary(rc.Clipping.getClipping_texto());

        // comment out the three lines below
        // to render the clipping.summary view instead
        // (useful for debugging)
        var contentType = 'text';
        setting showdebugoutput='false';
        framework.renderData( contentType, rc.Summary );
    }
{% endhighlight %}

Here's a detailed breakdown:

 - `function summary( struct rc ) ` receives `rc`, the FW/1's "Request Context" struct
 that contains whatever URL or FORM parameters have been passed by the referrer;


 - `rc.Clipping = variables.clippingService.getClipping(rc.clipping_id);`
 loads an instance of the Clipping entity we selected;

 - `rc.Summary = variables.summaryService.getSummary(rc.Clipping.getClipping_texto());`
 invokes the summaryService's `getSummary()` function, and passes it the selected
 article's full text. The generated summary string is stored in `rc.Summary`

 - `framework.renderData( contentType, rc.Summary );` renders the summary string
  (`rc.Summary `) as a "plain text" content, without any HTML layout.

### Summary Service

 The summary service is written in `/home/models/services/summaryService.cfc` and
 is actually quite simple:

{% highlight js %}
component {

    public function getSummary(string clipping_texto){

        cfhttp(url='http://localhost:5000/ajax_resumo' method='post' result='st_summary'){
            cfhttpparam (type="formfield" name = "texto" value = clipping_texto);
        }

        .....

        if (len(st_summary.errordetail) > 0) {
            return "There was an error trying to use the summary service :'(";
        } else {
            return st_summary.filecontent;
        }
    }
}
{% endhighlight %}




 ----


For more detailed information on this project, follow the other articles in this series:

 - [Project Structure](/2015/03/29/fw1-example-app-project-structure/)
 - [Forms and Validation Patterns](/2015/03/30/fw1-example-app-forms_validation/)
 - [Use of User Defined Function Libraries](/2015/04/06/fw1-example-user-defined-function-libraries/)
 - Interaction with an External Service
 - BDD and Integration Tests.

For the full source code, please visit the **[fw1-clipping](https://github.com/dezoito/fw1-clipping)**
github project page.

