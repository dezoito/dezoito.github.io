---
layout: post
title: FW/1 Example Application - Testing your App with BDD and Integration Tests
excerpt_separator: <!--more-->
---

This final article is probably the most interesting (**and important!**) in the series
and details how to implement [Behaviour Driven Development](http://dannorth.net/introducing-bdd/)
tests and [Integration Tests](http://programmers.stackexchange.com/questions/48237/what-is-an-integration-test-exactly)
on the [Example Application using FW/1](https://dezoito.github.io/2015/03/26/fw1-example-app-released/).

The links above provide excellent definitions on these different concepts, so
I strongly suggest the reader to become familiar with them.

BDD, specifically, encompasses much more than just the tests -
learning and implementing these concepts was incredibly rewarding, as it changed
some of my stubborn ways of approaching development!

### Getting Technical (Starting with Dependencies)

This project has two dependencies: [testBox](http://wiki.coldbox.org/wiki/TestBox.cfm)
and [CFSelenium](http://cfselenium.riaforge.org/).

These frameworks, respectively, allow us to implement BDD style tests, and
integration tests using Selenium  - which look so awesome,
like if a ghost was hired to do nothing but QA tests!

Assuming you are using a **development machine** (don't do this in a production
 environment) download and install both of these
tools at your webserver's root:

 - [testBox Download](http://www.coldbox.org/download/testbox)
 - [CFSelenium Download](https://github.com/teamcfadvance/CFSelenium)

When in doubt, check the the [Project Structure](/2015/03/29/fw1-example-app-project-structure/)
article.
You can also choose another location if you want, but then you'll have to create
appropriate mappings.

### Running Tests

Testing files are stored within the `/tests` folder:

`Application.cfc` - inherits settings from the main application, and then sets
some test-specific configs.

`ìndex.cfm` - This is just a copy of the "test browser" that comes with TestBox.
It lets you see and choose the tests available, or run all of them.

`/specs` - Each test is a CFC file stored in this folder. I gave them descriptive
names like `Test_1_ExampleSpec.cfc` because I wanted them displayed in order (and
first tests to be the simplest), but in truth you can name them however you like
(I still suggest you be descriptive).

Accessing `http://your-server/clipping/tests` should display a test browser page:

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/test_browser.png?raw=true)

We start with `Test_1_ExampleSpec.cfc`, a most basic test, just to show the standard strdcture
and progress to Test_6_Integration_Selenium.cfc , which adds the CFSelenium component.

In a nutshell, Selenium starts a browser instance on your server and follows a "test script",
allowing TestBox to record what went as expected or not, and displaying the results:

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/test_results.png?raw=true)

### Writting Tests (this was actually fun!)
 ----

For more detailed information on this project, follow the other articles in this series:

 - [Project Structure](/2015/03/29/fw1-example-app-project-structure/)
 - [Forms and Validation Patterns](/2015/03/30/fw1-example-app-forms_validation/)
 - [Use of User Defined Function Libraries](/2015/04/06/fw1-example-user-defined-function-libraries/)
 - [Interaction with a Remote Service](/2015/04/07/fw1-example-accessing-external-service/)
 - BDD and Integration Tests.

For the full source code, please visit the **[fw1-clipping](https://github.com/dezoito/fw1-clipping)**
github project page.

