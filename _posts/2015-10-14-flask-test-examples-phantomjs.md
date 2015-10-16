---
layout: post
comments: true
title: Flask - Test Examples and Structure - Unit Tests and PhantomJS
excerpt_separator: <!--more-->
---

This article is similar to the [Ready to use Structure for Django Tests](/2015/09/21/how-to-test-django-applications_pt1/) and the main goals are:

 - get your tests up and running quickly
 - provide a starting point for more complex tests

We'll use my [Flask App to generate summaries](https://github.com/dezoito/flask_Summarizer/) as the test target (feel free to clone it).


---
## Structure
The proposed structure allows developers to:

 - Organize and group different test types (unit, integration, functional tests, etc...).

 - Run tests individually, or by group, or all of them.

 - Define logic that is common to all tests (e.g. client login/logout, utility functions)

 - Get the elapsed running time for individual tests

So starting with a basic Flask App structure, we can use a `/tests` folder to store different tests and utilities:

```sh
/flask_Summarizer
                ├── app
                │   ├── static
                │   │   ├── css
                │   │   ├── fonts
                │   │   └── js
                │   │ 
                │   └── templates
                ├── summarize
                ├── textrank
                └── tests
                    ├── __init__.py
                    ├── tests_functional.py
                    ├── tests_phantomJS.py
                    ├── tests_unit.py
                    └── utils.py
```

---
## Different Test Types

For the purpose of this article, here's my rule of the thumb for different tests types - from fastest to slowest:

**- Unit Tests:**

Those are usually the fastest running tests and you should use them for functions that you wrote from scratch and don't involve output to be rendered, such as calculations or maybe some custom model or view methods.


**- Functional or Integration Tests:**

These are tests that go through the Flask framework resources to assert if views and form submissions are working - They will probably go through configurations, database calls, extensions and libraries and thus take longer to run.

**- Selenium Tests:**

A more complex form of the Integration tests, these use [Selenium](https://selenium-python.readthedocs.org/) and [phantomJS](http://phantomjs.org/) to simulate a browser session.

Since they are **much** slower than everything else, they are used only to test more complex user behavior (e.g. user clicks something that triggers an ajax response).

In this example, tests are grouped according to their category, so you can run only what you need at a moment.

---
## Code Examples and Explanations

First, let's start by describing a possible `utils.py` file.
This should **group logic that is used across test suites**, so we can keep our tests DRY:

```python
# -*- coding: utf-8 -*-
import json
import time


def print_test_time_elapsed(method):
    def run(*args, **kw):
        ts = time.time()
        print('\n\ttesting function %r' % method.__name__)
        method(*args, **kw)
        te = time.time()
        print('\t[OK] in %r %2.2f sec' % (method.__name__, te - ts))

    return run

# test if a string is valid JSON
def other_function():
    ...
```

We will touch on the `print_test_time_elapsed()` later. The important part for now is that we use this to store logic that is common to most of our tests -such as setting up test databases or performing some task as a logged user.

### Unit Test Examples
`/tests/tests_unit.py`:

```python
# -*- coding: utf-8 -*-
#!flask/bin/python

import os
import unittest
import sys
sys.path.append('..')
import sample_strings
from app.views import make_summary

class TestCase(unittest.TestCase):
    def setUp(self):
        # load sample strings
        self.small_str = sample_strings.small_text
        self.medium_str = sample_strings.medium_text
        self.large_str = sample_strings.large_text

    def tearDown(self):
        pass

    def test_summarize_on_view_using_summarize_algo(self):
        """
        Tests summaries using the simplest algorithm
        """
        assert len(self.small_str) >= len(make_summary(self.small_str))
        assert len(self.medium_str) > len(make_summary(self.medium_str))
        assert len(self.large_str) > len(make_summary(self.large_str))

    def test_summarize_on_view_using_text_rank_algo(self):
        """
        Tests summaries using the textRank algorithm (takes longer)
        """
        assert len(self.small_str) >= len(make_summary(self.small_str, "textrank"))
        assert len(self.medium_str) > len(make_summary(self.medium_str, "textrank"))
        assert len(self.large_str) > len(make_summary(self.large_str, "textrank"))


if __name__ == '__main__':
    unittest.main()

```

The flask_Summarizer app has a `make_summary()` that does not output anything, but merely returns a summary of a text, using one of two algorithms:

 - Summarize (default)
 - TextRank

I don't want to tests the methods in these libraries (there's a rule that says "you should only test your own code"), so I'm just going to test if my function
is using them correctly and not messing up anything along the way.

Here's a brief "what's going on":

`setUp()` - This function runs before each test, and in this case it creates instance variables with texts that we will use.

`test_summarize_on_view_using_summarize_algo()` - The name is pretty self-explanatory: invokes the `make_summary()` method and sees if it generates summaries with the default algo, for strings with different lengths.

`test_summarize_on_view_using_text_rank_algo()` - Same as above, but we specify the use of the textRank algo.

`tearDown()` - Runs after a test is done

Running this test should result in this output:

```sh
Ran 2 tests in 19.377s

OK
```
(TextRank is slow and this laptop is even slower!)

### Functional/Integration Test Examples
Make sure application is running

`/tests/tests_functional.py`:

```python

import os
import unittest
import tempfile
import sys
sys.path.append('..')
import urllib  # cant use urllib2 in python3 :P
import config
import sample_strings
from flask import Flask
from flask.ext.testing import TestCase
from app import app


class StartingTestCase(TestCase):
    def setUp(self):
        self.client = app.test_client()
        config.WTF_CSRF_ENABLED = False

        # load sample strings
        self.small_str = sample_strings.small_text
        self.medium_str = sample_strings.medium_text
        self.large_str = sample_strings.large_text

    def tearDown(self):
        pass

    def create_app(self):
        """
        This is a requirement for Flask-Testing
        """
        app = Flask(__name__)
        app.config['TESTING'] = True
        self.baseURL = "http://localhost:5000"
        return app

    # --------------------------------------------------------------------------
    # Simple tests to make sure server is UP
    # (does NOT use LiveServer)
    # --------------------------------------------------------------------------
    def test_real_server_is_up_and_running(self):
        response = urllib.request.urlopen(self.baseURL)
        self.assertEqual(response.code, 200)
        # returned source code is stored in
        # response.read()

    # --------------------------------------------------------------------------
    # Testing Views with GET
    # --------------------------------------------------------------------------
    def test_view_form_resumo_get(self):
        rv = self.client.get('/')
        # print(rv.data)
        assert rv.status_code == 200
        assert 'Please enter your text:' in str(rv.data)

    # --------------------------------------------------------------------------
    # Testing Views with POST
    # --------------------------------------------------------------------------
    def test_view_form_resumo_post(self):
        post_data = {'texto': self.small_str}
        rv = self.client.post('/', data=post_data, follow_redirects=True)
        assert rv.status_code == 200
        assert 'Todos os direitos reservados' in str(rv.data)


    def test_view_form_resumo_post_with_textrank(self):
        post_data = {'texto': self.small_str, 'algorithm': 'textrank'}
        rv = self.client.post('/', data=post_data, follow_redirects=True)
        assert rv.status_code == 200
        assert 'Todos os direitos reservados' in str(rv.data)


    def test_ajax_resumo_post(self):
        post_data = {'texto': self.small_str}
        rv = self.client.post('/ajax_resumo',
                              data=post_data,
                              follow_redirects=True)
        assert rv.status_code == 200
        # the ajax view returns nothing but the string
        assert b'Todos os direitos reservados' == rv.data


    def test_ajax_resumo_post_with_textrank(self):
        post_data = {'texto': self.small_str, 'algorithm': 'textrank'}
        rv = self.client.post('/ajax_resumo',
                              data=post_data,
                              follow_redirects=True)
        assert rv.status_code == 200
        assert b'Todos os direitos reservados' == rv.data


if __name__ == '__main__':
    unittest.main()
```


### Selenium and phantomJS Test Examples
Make sure application is running

`/tests/tests_phantomJS.py`:

```python
import unittest
import sys
import time
sys.path.append('..')
import config
import sample_strings
from flask import Flask
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from app import app
from utils import print_test_time_elapsed


class StartingTestCase(unittest.TestCase):
    def setUp(self):
        self.driver = webdriver.PhantomJS()
        self.baseURL = "http://localhost:5000/"

    def tearDown(self):
        self.driver.quit

    # --------------------------------------------------------------------------
    # Simple tests to make sure server is UP
    # --------------------------------------------------------------------------
    @print_test_time_elapsed
    def test_home(self):
        self.driver.get(self.baseURL)
        assert "Summarizer App" == self.driver.title

    @print_test_time_elapsed
    def test_home_envio_form(self):
        self.driver.get(self.baseURL)
        self.driver.find_element_by_id("texto").send_keys("Resuma isso!")
        self.driver.find_element_by_id("btnSubmit").click()
        resumo = self.driver.find_element_by_id("txt_resumo").text
        assert "Resuma isso!" in resumo

    @print_test_time_elapsed
    def test_sample_text(self):
        self.driver.get(self.baseURL)
        self.driver.find_element_by_link_text("Sample 1").click()
        # WebDriverWait(self.driver, 3).until(
        #     EC.text_to_be_present_in_element((By.ID, "texto"), "a"))
        self.driver.find_element_by_id("btnSubmit").click()
        self.assertIn("Um suéter azul.",
                self.driver.find_element_by_id("txt_resumo").text)


if __name__ == '__main__':
    unittest.main()

```


### Running Tests

From the `tests/` directory, run:
```sh
    python -m unittest discover
```

You can also run individual test suites:
```sh
    python -m unittest tests_unit
    python -m unittest tests_functional
```

---
## References

[Miguel Grinberg's Mega Flask Tutorial](http://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world)
The 'De Facto' Flask Tutorial.

[Flask Testing](https://pythonhosted.org/Flask-Testing/)
A basic introduction to tests using the `Flask-Testing` extension.

[Python Web Applications With Flask - Part III ](https://realpython.com/blog/python/python-web-applications-with-flask-part-iii/)
A comprehensive in-depth guide to build Flask apps and how to perform more complex tests.

[Getting Started with UI Automated tests using (Selenium + Python)](http://engineering.aweber.com/getting-started-with-ui-automated-tests-using-selenium-python/)
An introduction on automated testing with Python and Selenium

[Selenium with Python](https://selenium-python.readthedocs.org/)
Unofficial, but nonetheless excellent documentation.

