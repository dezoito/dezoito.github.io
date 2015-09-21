---
layout: post
title: How to Structure Testing in Django Applications - with Examples
excerpt_separator: <!--more-->
---

Creating an efficient testing structure can be overwhelming for a Django beginner, as authors often show different and contradictory approaches.

In this article, I suggest a logical and **simple** test structure, with these goals:

 - Group Unit Tests, Request Tests, Django Client Tests, and Functional Tests so you can:
    - Run all tests in sequence, or just the ones you need, grouped or individually

 - Define logic that is common to all tests (e.g. client login/logout, utility functions)

 - Initialize your test database programmatically.

 - Get the ellapsed running time for individual tests (so you can easily see what's taking too long!)



This methodology is aimed at getting your tests up and running quickly, so think of it as a starting point.
Since there are better ways to do everything, I encourage you to do your own research once you find its limitations.

### Folder Structure

Starting with a basic Django App structure, we can use a `/test` folder to store different tests and utilities

```
/project_root
    |
    +- /my_application
            |
            +- /my_application
                |
                +- forms.py
                +- models.p
                +- etc....
            |
            +- /static
            |
            +- /templates
            |
            +- /admin
            |
            +- manage.py
            |
            +- /tests
                |
                +- /functional
                |
                +- /unit
                |
                +- __init__.py
                +- testing_utilities.py

```

### Different Test Types

For the purpose of this article, here's my rule of the thumb for different tests types - from fastest to slowest:

**- Unit Tests:**

Those are usually the fastest running tests. Write them for functions that you wrote from scratch and don't involve output to be rendered, such as calculations or maybe custom model or view methods.

**- Request Tests:**

These are used to test Django views, and replicate a user request.
If you only want to check if a view works with a `GET` request and there's no need for authentication, you can use "RequestFactory" as it's faster than using Django's client.


**- Django Client Tests:**

Client tests go through Settings, URL configs and middleware, so they take longer than the simpler resquest tests (technically speaking, they are integration tests).

I use those to test authenticated views and form submissions - by setting up some form data and using self.client.post().

**- Functional Tests (or Full Integration tests)**

These are tests that use [Selenium](https://selenium-python.readthedocs.org/) to replicate a browser session.
Since they are **much** slower than everything else, they are used only to test more complex user behavior (e.g. user clicks something that triggers an ajax response).

Following the KISS principle, I'm keeping the Functional tests in their own folder, and everything else in the `/unit` folder, but of course you separate things even further as your app grows.

### Code Examples and Explanations

First, let's start by describing a possible `testing_utilities.py` file.
This should group logic that is used across test suites, so we can keep our tests DRY:

```python
# -*- coding: utf-8 -*-
import json
import time
from django.contrib.auth.models import User
from my_application.models import Category, Thing

def populate_test_db():
    """
    Adds records to an empty test database
    """
    cat = Category.objects.create(cat_nome='Widgets')
    cat_inactive = Category.objects.create(cat_nome='Inactive Category',
                                            cat_active=False)
    thing1 = Thing.objects.create(categoria=cat,
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

# more common funcionality below

```

The `populate_test_db()` function will run before each test suite and (re)create the records needed to perform our tests. For more complex applications, you should definitelly look into usi
ng fixtures, [Factory Boy](https://factoryboy.readthedocs.org/en/latest/) or [Mocking](http://www.mattjmorrison.com/2011/09/mocking-django.html) (or [not](http://hernantz.github.io/mock-yourself-not-your-tests.html)).

The other functions simply perform actions that can be called by the different tests.

#### Request Test Examples

`\unit\test_requests.py`

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
        # request = self.factory.get('/category/1')
        # OR:
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


