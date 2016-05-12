---
layout: post
comments: true
title: ColdFusion FW/1 Example Application Project
excerpt_separator: <!--more-->
---

I just released a [ColdFusion Example Application](https://github.com/dezoito/fw1-clipping),
built using the FW/1 framework in its latest version:

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/clipping_screen.png?raw=true)

It's a simple app to store articles and clippings from on and offline sources,
using a single model/database table, but handling a series of interesting topics:

 - [Project Structure](/2015/03/29/fw1-example-app-project-structure/)
 - [Forms and Validation Patterns](/2015/03/30/fw1-example-app-forms_validation/)
 - [Use of User Defined Function Libraries](/2015/04/06/fw1-example-user-defined-function-libraries/)
 - [Interaction with a Remote Service](/2015/04/07/fw1-example-accessing-external-service/)
 - [BDD and Integration Tests](/2015/04/18/fw1-example-bdd-integration-testing/)

<!--more-->

 The project also has several examples on how to implement
 [BDD style testing](http://wiki.coldbox.org/wiki/TestBox.cfm)
 and [integration tests](http://cfselenium.riaforge.org/).

A few notes before the articles detailing ways to handle these topics:

 - As with most frameworks, there was a bit of a learning curve with
 [FW/1](http://framework-one.github.io/), but I now wish I could port all my CFML
 projects to it! It certainly seems to work **with** the developer and
 I never felt like I had to follow a rigid workflow, or jump through hoops
 to get something working.

 - This project also left me with the desire to kick myself for not implementing
 integration tests sooner...they are fun to write!
It's also very satisfying to see selenium acting like a ghost, clicking links and filling forms
 "magically" for you.

The full source code is hosted on the **[fw1-clipping](https://github.com/dezoito/fw1-clipping)** github
project page.
