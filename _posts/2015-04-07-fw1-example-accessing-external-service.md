---
layout: post
title: FW/1 Example Application - Accessing an External Service
excerpt_separator: <!--more-->
---

On this fourth article on how to build a
[ColdFusion and FW/1 Example Application](https://dezoito.github.io/2015/03/26/fw1-example-app-released/),
I'll present a way to access an external service that generate a succint summary
for a selected article.

The external service, in this case, is [flask-Summarizer](https://github.com/dezoito/flask-Summarizer)
- an API written in Python/Flask, that receives a `POST` request with some
paragraphs of text, and returns a string with a summarized version of that.

### FW/1 - Accessing an External Service

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/view_summary.png?raw=true)

{% highlight js %}

{% endhighlight %}



 ----


For more detailed information on this project, follow the other articles in this series:

 - [Project Structure](/2015/03/29/fw1-example-app-project-structure/)
 - [Forms and Validation Patterns](/2015/03/30/fw1-example-app-forms_validation/)
 - [Use of User Defined Function Libraries](/2015/04/06/fw1-example-user-defined-function-libraries/)
 - Accessing an External Service
 - BDD and Integration Tests.

For the full source code, please visit the **[fw1-clipping](https://github.com/dezoito/fw1-clipping)**
github project page.

