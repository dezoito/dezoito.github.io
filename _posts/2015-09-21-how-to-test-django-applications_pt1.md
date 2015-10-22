---
layout: post
comments: true
title: Ready to use Structure for Django Tests + Examples (Pt. 1)
excerpt_separator: <!--more-->
---

This article proposes a flexible and efficient test structure, 
so readers don't have to go through the many Django testing resources available 
online before getting started.

This structure allows developers to:

 - Organize and group different test types (unit, integration, functional tests, etc...).

 - Run tests individually, or by group, or all of them.

 - Define logic that is common to all tests (e.g. client login/logout, utility functions)

 - Populate your test database programmatically.

 - Get the [elapsed running time for individual tests](/2015/09/26/how-to-test-django-applications_pt2/) [see part 2].

This should get your tests up and running quickly, so think of it as a **starting point** (I encourage you to do your own research once you find its limitations).

---
##Folder Structure

Starting with a basic Django App structure, we can use a `/tests` folder to store different tests and utilities:

```
    project_root
    │
    ├── my_application
    │   ├── my_application
    │   │   ├── settings
    │   │   ├── forms.py
    │   │   ├── models.py
    │   │   └── etc....
    │   ├── media
    │   ├── static
    │   ├── templates
    │   ├── manage.py
    │   └── tests 
    │       ├── functional
    │       ├── unit
    │       ├── __init__.py
    │       └── testing_utilities.py
    └── requirements

```

---
## Different Test Types

For the purpose of this article, here's my rule of the thumb for different tests types - from fastest to slowest:

**- Unit Tests:**

Those are usually the fastest running tests and you should use them for functions that you wrote from scratch and don't involve output to be rendered, such as calculations or maybe some custom model or view methods.

**- Request Tests:**

These are used to test Django views, by simulating a user request.
If you only want to check if a view works with a `GET` request and there's no need for authentication, you can use `django.test.RequestFactory` as it's faster than using Django's client.

**- Django Client Tests:**

Client tests go through Settings, URL configs and middleware, so they take longer than the simpler request tests (technically speaking, they are integration tests).

I use those to test authenticated views and form submissions - by setting up some form data and using self.client.post().

**- Functional Tests:**

These are tests that use [Selenium](https://selenium-python.readthedocs.org/) to simulate a browser session.

Since they are **much** slower than everything else, they are used only to test more complex user behavior (e.g. user clicks something that triggers an ajax response).

Following the KISS principle, I'm keeping the Functional tests in their own folder, and everything else in the `/unit` folder, but of course you separate things even further as your app grows.

---
## Code Examples and Explanations

First, let's start by describing a possible `testing_utilities.py` file.
This should group logic that is used across test suites, so we can keep our tests DRY:

```python
import json
import time
from django.contrib.auth.models import User
from my_application.models import Category, Thing

def populate_test_db():
    """
    Adds records to an empty test database
    """
    cat = Category.objects.create(cat_name='Widgets')
    cat_inactive = Category.objects.create(cat_name='Inactive Category',
                                            cat_active=False)
    thing1 = Thing.objects.create(category=cat,
                                thing_desc="Test Thing",
                                thing_model="XYZ1234",
                                thing_brand="Brand X")

    User.objects.create_user(
        username='admin',
        email='admin@test.com',
        password='secret666')


def login_client_user(self):
    self.client.login(username='admin', password='secret666')
    return self

def logout_client_user(self):
    self.client.logout()
    return self

def is_json(myjson):
    """
    tests if a string is valid JSON
    """
    try:
        json_object = json.loads(myjson)
    except ValueError, e:
        return False
    return True

# more common functionality below

```

The `populate_test_db()` can be called by the test suites and (re)create the records needed to perform our tests. For more complex applications, you should definitely look into using fixtures, [Factory Boy](https://factoryboy.readthedocs.org/en/latest/) or [Mocking](http://www.mattjmorrison.com/2011/09/mocking-django.html) (or [not mocking](http://hernantz.github.io/mock-yourself-not-your-tests.html)).

The other functions can also be called when needed (some tests need the client logged in, for example).

### Request Test Examples

`/unit/test_requests.py`

```python
from django.test import TestCase
from django.test.client import RequestFactory
from my_application.views import home, ajax_search
from ..testing_utilities import populate_test_db

class RequestTests(TestCase):

    def setUp(self):
        # Every test needs access to the request factory.
        self.factory = RequestFactory()
        # Add records to test DB
        populate_test_db()

    def test_home_view_without_client(self):
        request = self.factory.get('/')
        response = home(request)
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Some text that should be in the HOME view")


    def test_category_view(self):
        request = self.factory.get(reverse('category',
                                           kwargs={'cat_id': 1}))
        response = category(request, 1)
        self.assertEqual(response.status_code, 200)


    def test_ajax_search(self):
        request = self.factory.get('/ajax_search?q=a',
                                   HTTP_X_REQUESTED_WITH='XMLHttpRequest')
        response = ajax_search(request)
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Thing: ASD1234") 

```

After importing the needed dependencies, `class RequestTests(TestCase)` is used to group
a test suite and its different methods:

`setup()` - This is called before every test and in this case we use it to create an instance
of RequestFactory and to populate our temporary test database.

`test_home_view_without_client()` - This is the most basic test, sending a request to the view mapped to `'/'` and asserting that the response is returned as expected.

`test_category_view()` -A slightly more complicated test, where we invoke a view by its name (instead of URL mapping) and pass it parameters (using **kwargs).

`test_ajax_search()` - Tests sending a request to a view where an AJAX call is expected.


See the Official Django Reference for more details on [django.test.RequestFactory](https://docs.djangoproject.com/en/1.8/topics/testing/advanced/#django.test.RequestFactory).


### Django's Test Client Examples

`/unit/test_views.py`

```python
from django.core.urlresolvers import reverse
from django.test import TestCase, Client
from django.utils.http import urlencode
from ..testing_utilities import populate_test_db, login_client_user, logout_client_user
from my_application.models import Category

class ViewTests(TestCase):
    def setUp(self):
        self.client = Client(enforce_csrf_checks=True)
        populate_test_db()

    def test_home_view(self):
        response = self.client.get(reverse('home'))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Some text that should be in the HOME view")        
        # Asserts that ONLY active categories are displayed in the home view
        self.assertQuerysetEqual(
            # cat_list is a querySet appended to the context dict.
            response.context['cat_list'],
            [
                repr(r) for r in Category.objects.filter(cat_active=True)
            ])

    def test_category_view(self):
        response = self.client.get(reverse('category',
                                           kwargs={'cat_id': 1}))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, u"Widgets")


    def test_form_new_thing(self):
        # Authenticates User
        login_client_user(self)
        response = self.client.get('/category/1/new_thing/')
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, u"Add a Thing")
```

Again, after handling the imports, we group our tests in the `ViewTests` class:

`setup()` - Creates an instance of the test Client and populates our temporary test database.
You can set `enforce_csrf_checks` to `False` if you want..I am just being extra-paranoid.

`test_home_view()` - Tests that the `Home` view loads correctly and also that a list of categories shows only those that are marked as **active**.

`test_category_view()` - Shows how to pass parameters to a view and test its response.

`test_form_new_thing()` - Shows how to test a Django view that requires an authenticated user.


### Testing POST Form Submissions

I use those to test how the application behaves **after** the user submits a form, 
and not so much to make sure it's displayed correctly.

`/unit/test_post.py`

```python
from django.test import TestCase, Client
from django.core.urlresolvers import reverse
from django.utils.http import urlencode
from ..testing_utilities import populate_test_db, login_client_user

class FormTests(TestCase):

    def setUp(self):
        self.client = Client(enforce_csrf_checks=False)
        populate_test_db()
        # Log user for all tests
        login_client_user(self)

        # define some form fields/values
        self.thing_post_data = {
            'category': '1',
            'thing_desc': 'Name of the thing',
            'thing_model': 'ABC5555',
            'thing_brand': 'brand Y',
            'thing_quantity': '2'
        }


        def test_include_thing_fail_validation(self):
            """
            Tests if Field Validations messages are displayed
            """
            form_addr = reverse('form_new_thing', kwargs={'cat_id': 1})
            post_data = {}  # form does not send any data!
            response = self.client.post(form_addr, post_data)
            self.assertEqual(response.status_code, 200)
            self.assertFormError(response, 'form', 'thing_desc',
                                 u'Fill in the field Description')
            self.assertFormError(response, 'form', 'thing_model',
                                 u'Fill in the field Model')
            self.assertFormError(response, 'form', 'thing_brand',
                                 u'Fill in the field Brand')


        def test_include_thing_ok(self):
            """
            Tests the response when the form is correctly filled
            """
            form_addr = reverse('form_new_thing', kwargs={'cat_id': 1})
            # Use follow=true since there will be a redirect after processing
            response = self.client.post(form_addr,
                                        self.thing_post_data,
                                        follow=True)
            self.assertEqual(response.status_code, 200)
            self.assertContains(response, u"Thing included successfully!")


```

In the above example, `setUp()` does a few things:

1- Creates an instance of the test Client

2- Populates the test database

3- Logs in before every test (assume users have to be authenticated to submit forms)

4- Defines a dict with the POST data (mirroring what would be sent by filling in the actual form fields.)

`test_include_thing_fail_validation()` - Tests a submission that should **FAIL** (in this case, because the were unfilled form fields)

`test_include_thing_ok()`- Tests what happens when all the fields are correctly filled.


### Functional Tests using Selenium
`/functional/test_post.py`

```python
from selenium.webdriver.firefox import webdriver
from selenium.webdriver.common.keys import Keys
from django.core.urlresolvers import reverse
from django.contrib.staticfiles.testing import StaticLiveServerTestCase
from django.utils import formats
from ..testing_utilities import populate_test_db


class FunctionalTest(StaticLiveServerTestCase):
    def setUp(self):
        self.selenium = webdriver.WebDriver()
        self.selenium.implicitly_wait(3)
        populate_test_db()

    def tearDown(self):
        self.selenium.quit()

    # Auxiliary function to add view subdir to URL
    def _get_full_url(self, namespace):
        return self.live_server_url + reverse(namespace)

    def test_home_title(self):
        """
        Tests that Home is loading properly
        """
        self.selenium.get(self._get_full_url("home"))
        self.assertIn(u'Title that you expect', self.selenium.title)

    def test_ajax_search_thing(self):
        self.selenium.get(self._get_full_url("home"))
        search_input = self.selenium.find_element_by_name("search_input")
        # testing search for thing
        search_input.send_keys('XYZ1234')
        tab_things = self.selenium.find_element_by_id("tab_things")
        self.assertTrue(tab_things)
        self.assertIn('XYZ1234', tab_things.text)

```

There's a couple of important things happening in the imports:

 - We add selenium's webdriver and its `keys` package
 - We import `StaticLiveServerTestCase` which frees us from having to have a running instance of the application.

Again, we create a class - `FunctionalTest` - that groups our tests and helper methods:

`setup()` - Starts a running instance of selenium's webdriver and populates the Test Database.

`tearDown()` - After tests are run, this stops the running webdriver.

`test_home_title()` - Opens the `home` view in a browser and tests the response.

`test_ajax_search_thing()` - "Types" text in the Search Box and asserts that the
 application finds the expected result.

---
## Running Tests

First, CD into your App's root folder (the one where you can find `manage.py`)

To run ALL tests:

`python manage.py test tests`

If you are using Django 1.8+, you can keep a test database across tests, speeding things up a bit:

`python manage.py test tests -k`

Running "unit" tests only:

`python manage.py test tests.unit [-k]`

Running functional tests only:

`python manage.py test tests.functional [-k]`

Testing POST submissions only:

`python manage.py test tests.unit.test_post [-k]`

Running a SINGLE test (notice that we specify the `FormTests` class before the test we want):

`python manage.py test tests.unit.test_post.FormTests.test_include_thing_ok [-k]`

Be sure to read [Part 2](/2015/09/26/how-to-test-django-applications_pt2/) to see how to get individual test times.

---
## References
[Django's Official Tutorial](https://docs.djangoproject.com/en/1.8/intro/tutorial05/) and [Django's Testing Tools Docs](https://docs.djangoproject.com/en/1.8/topics/testing/tools/) - Comprehensive resources, but they made more sense to me after I understood the different test types.


[The Most Efficient Django Test](http://blog.doismellburning.co.uk/the-most-efficient-django-test/) - If you could only write one single test for your Django App, this would be it.

[Marina Mele's Django Tutorial](http://www.marinamele.com/taskbuster-django-tutorial/create-home-page-with-tdd-staticfiles-templates-settings#tdd-tests) - Not just a great Django tutorial, but also a good introduction on using `Selenium Webdriver` and `LiveServerTestCase` for functional tests.

[Toast Drive's Guide to Testing in Django #2](http://toastdriven.com/blog/2011/apr/17/guide-to-testing-in-django-2/) - The reference used to testing POST requests.

[Newspaper3k: Article scraping & curation](https://github.com/codelucas/newspaper) - Great reference on scraping that also has an interesting approach on how to measure individual test's running times (Part 2).

