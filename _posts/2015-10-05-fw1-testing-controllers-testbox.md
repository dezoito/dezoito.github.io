---
layout: post
comments: false
title: ColdFusion - Testing FW/1 Controllers with testBox
excerpt_separator: <!--more-->
---

Some frameworks allow you to test your controllers directly - usually a faster approach to using Selenium and headless browsers. 

I decided to try that for a new FW/1 app, but since I could only find discussions on how to do that, and not a lot of code, I decided to try that on my own, using [TestBox](http://wiki.coldbox.org/wiki/TestBox.cfm). 

This picks up where my [FW/1 Example Application Articles](/2015/04/18/fw1-example-bdd-integration-testing/) left... I suggest you read that in case any of the code seems confusing.

### A Test Spec for controllers

**[Test_7_Controllers.cfc](https://github.com/dezoito/fw1-clipping/blob/master/tests/specs/Test_7_Controllers.cfc)**:

```cfc
/**
 * Tests the Application's controllers
 *
 */
component extends="testbox.system.BaseSpec"{

    // executes before all suites
    function beforeAll(){
        // reinitialise ORM for the application
        ORMReload();

        // invoke controller object
        mainController = createObject("component", "root.home.controllers.main");

        // injects the FW/1 framework into the tested controller
        mainController.framework = createObject("component", "framework.one");

        // binds service to main controller
        mainController.clippingService = createObject("component", "root.home.model.services.clippingService");

        // Mock request context struct
        rc = structNew();

        //creates fake data for tests
        ..........
        }
    }

    ...

```


In TestBox `beforeAll()` runs once, before any test, so this is where we invoke the `main` controller and inject its dependencies for this test case:

 - `framework` - An instance for FW/1
 - `clippingService` - The service that is queried by this controller

We also create a mock `rc` struct that will be used to pass data to and from the tested controller and insert some test data on the test database.

```cfc


    function run( testResults, testBox ){

        describe("Main Controller: ", function(){

            beforeEach( function( currentSpec ){
                // don't let rc data persist through tests
                structClear(rc);
            });

            it("The DEFAULT method returns an ORM object with an array of articles", function(){
                // execute default method
                mainController.default( rc );
                expect( structKeyExists(rc, "qry_clipping")).toBeTrue();
                expect( rc.qry_clipping.count).toBe( 20 ); // total records
                expect( arrayLen(rc.qry_clipping.data) ).toBe( application.recordsPerPage ); //records per page
            });

        });

    }

}

```

As you can probably guess, the `beforeEach()` closure runs before every test in its `describe` block, and in this case it resets the `rc` struct so request data does not persist among tests (we only have one test here, but in my real app this was hapenning).

The `it()` block details how to execute the `default` method in the main controller and how to test that the response is accurate.

There you go! Now we can stick to using Selenium only when and where it's really needed.
---

The source code for this project is hosted on the **[fw1-clipping](https://github.com/dezoito/fw1-clipping)** github page.
