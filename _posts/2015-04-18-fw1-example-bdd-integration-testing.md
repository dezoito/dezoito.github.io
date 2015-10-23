---
layout: post
comments: false
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

Please note that one of these settings is a "tests only" datasource, pointing to
a clone of the default database. I'm still evaluating the pros and cons of this
approach and, honestly, it's not really something you need to replicate,
but I left it this way for the sake of learning.

`ìndex.cfm` - This is just a copy of the "test browser" that comes with TestBox.
It lets you see and choose the tests available, or run all of them.

`/specs` - Each test is a CFC file stored in this folder. I gave them descriptive
names like `Test_1_ExampleSpec.cfc` because I wanted them displayed in order (and
first tests to be the simplest), but in truth you can name them however you like
(I still suggest you be descriptive).

Accessing `http://your-server/clipping/tests` should display a test browser page:

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/test_browser.png?raw=true)

We start with `Test_1_ExampleSpec.cfc`, a most basic test, just to show the standard structure
and progress to `Test_6_Integration_Selenium.cfc` , in which we use CFSelenium to perform full Integration tests.

In a nutshell, Selenium starts a browser instance on your server and follows a "test script",
allowing TestBox to record what went as expected or not, and displaying the results:

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/test_results.png?raw=true)

### Writing Tests (this was actually fun!)

The basic anatomy of a test can be seen in `Test_1_ExampleSpec.cfc`

{% highlight js %}
component extends="testbox.system.BaseSpec"{

    // executes before all suites
    function beforeAll(){}

    // executes after all suites
    function afterAll(){}

    // All suites go in here
    function run( testResults, testBox ){

        describe("A suite", function(){
            it("contains a very simple spec", function(){
                expect( true ).toBeTrue();
            });
            ......
        });
    }
}
{% endhighlight %}

TestBox has excellent reference material at [http://wiki.coldbox.org/wiki/TestBox.cfm](http://wiki.coldbox.org/wiki/TestBox.cfm),
but to make things clear, here's what's happening:

1. Our component extends TestBox's "BaseSpec" class, allowing us to use some of its methods

2. Function `beforeAll()` allows us to dictate what will happen **before** the tests
run (similar to a setup() method in other testing frameworks). We could use this,
for example, to insert test data into the database.

3. Function `afterAll()` allows us to dictate what will happen **after** the tests
run (similar to a tearDown() method in other testing frameworks). We could use this,
for example, to remove test data from the database.

4. Function `run()` initiates the testing itself, running one "suite" at a time

5. `Describe()` is how we define a test suite and its scope

6. Finally, `it()` is a single spec in that suite, and it might contain one or
more <em>expectations</em>, that will dictate whether this spec passed, or failed.

The tests included in the [source code](https://github.com/dezoito/fw1-clipping/tree/master/tests/specs)
are very much self-explanatory and provide many examples on how to set up and
accomplish things, from "unit" testing your UDFs, to databases and remote services.

### Integration Tests with Selenium
[CFSelenium](https://github.com/teamcfadvance/CFSelenium) is a CFC wrapper for
Selenium functionality, making it easy to use it inside CFML code.

It allows you to script browser sessions and test the outputs and behaviours
against what's expected. We are using it on top of TestBox, so the syntax is
pretty much the same as the one above, and the results can be reported along the
other tests we've done.

Here's how we set it up in `tests/specs/Test_6_Integration_Selenium.cfc`:

{% highlight js %}
component extends="testbox.system.BaseSpec"{

    // executes before all suites
    function beforeAll(){
        // set url of APP installation
        browserURL = application.testsBrowseURL;
        // set browser to be used for testing
        browserStartCommand = "*googlechrome";

        // create a new instance of CFSelenium
        selenium = createobject("component", "CFSelenium.selenium").init();
        // start Selenium server
        selenium.start(browserUrl, browserStartCommand);
        // set timeout period to be used when waiting for page to load
        timeout = 120000;
        // rebuild current App
        httpService = new http();
        httpService.setUrl(browserURL & "/index.cfm?rebuild=true");
        httpService.send();

        // create some random title string (we will use this to delete the article later)
        str_random_title = createUUID();
        // text to be used in articles
        str_default_text = repeatString("<p>Lorem ipsum dolor sit amet, consectetur adipiscing elit. <br>Integer nec nulla ac justo viverra egestas.</p>", 10);
    }

    // executes after all suites
    function afterAll(){
        selenium.stop();
        selenium.stopServer();
    }
{% endhighlight %}

Function `beforeAll()` creates an instance of Selenium, telling it what browser
to use and what URL to start with (it also sets some variables that will be used
in the tests).

When everything is done `afterAll()` stops the Selenium session and results are
displayed in the report.

The first actual tests makes sure that the starting page is titled "Clippings"
(please note the BDD style language):

{% highlight js %}
describe("Loading home page", function(){

    it("Should load and have the correct title", function(){
        selenium.open(browserUrl);
        selenium.waitForPageToLoad(timeout);
        expect( selenium.getTitle() ).toBe( "Clippings" );
    });
});

{% endhighlight %}

Selenium opens the start URL and waits for the page to load. We then use
`selenium.getTitle()` to extract the page's title so we can compare it to our
expectation.

Simple, right?

Here's a little more complex interaction:

{% highlight js %}
//----------------------------------------------------------------------
// Testing forms
//----------------------------------------------------------------------
describe("Testing the clipping form:", function(){

    it("Clicking -add an article- link should load the form page", function(){
        selenium.open(browserURL);
        selenium.waitForPageToLoad(timeout);
        selenium.click("link=Add an Article");
        selenium.waitForPageToLoad(timeout);
        expect( selenium.isElementPresent("id=f_clipping") ).toBe( true );
        expect( selenium.isElementPresent("id=published") ).toBe( true );
    });

    it("The app must validade form entry (leave article TEXT empty)", function(){
        selenium.open(browserURL & "?action=clipping.form");
        selenium.waitForPageToLoad(timeout);
        selenium.type("id=clipping_titulo", "test");
        selenium.type("id=clipping_texto", "");
        selenium.click("id=btn_save");
        selenium.waitForPageToLoad(timeout);
        // didn't fill all the required fields...should return with error
        expect( selenium.isTextPresent("Your article could not be posted!") ).toBe( true );
    });

{% endhighlight %}

 - First test makes Selenium "click" on a link and sees if a "New Article" form
was loaded.

 - Second one attempts to submit the form without filling all the required fields,
making sure we get the apropriate error message.

For the sake of brevity I won't detail all the other tests performed, but the CFC
contains several different examples:

 - Using CSS and XPATH to find elements and values
 - Using Javascript to fill form elements
 - [Integrating CFML code for even more complex tests](https://github.com/dezoito/fw1-clipping/blob/cc694585f1435a1db1e32c5bad113c52611d3b11/tests/specs/Test_6_Integration_Selenium.cfc#L153)

Again, I have to give credit to Simon Bingham and his excellent [Xindi CMS project](https://github.com/simonbingham/xindi).
I feel incredibly lucky that he made that repository public so I could see how he
dealt with several issues that showed up during development.

 ----

For more detailed information on this project, follow the other articles in this series:

 - [Project Structure](/2015/03/29/fw1-example-app-project-structure/)
 - [Forms and Validation Patterns](/2015/03/30/fw1-example-app-forms_validation/)
 - [Use of User Defined Function Libraries](/2015/04/06/fw1-example-user-defined-function-libraries/)
 - [Interaction with a Remote Service](/2015/04/07/fw1-example-accessing-external-service/)
 - BDD and Integration Tests.

For the full source code, please visit the **[fw1-clipping](https://github.com/dezoito/fw1-clipping)**
github project page.

