---
layout: post
comments: true
title: Ready to use Structure for Django Tests + Examples (Pt. 3)
excerpt_separator: <!--more-->
---

This article builds up on the [Structure for Django Tests](/2015/09/21/how-to-test-django-applications_pt1/) and shows how you can easily display individual running times for each test in a group or suite - and you can use in Flask tests too! - check out my [Flask App to generate summaries](https://github.com/dezoito/flask_Summarizer/).

## The Problem
By default, Django will only display the execution time for the entire series of tests ran, for example:

```
$ python manage.py test tests.functional.test_search

Creating test database for alias default...
....
----------------------------------------------------------------------
Ran 4 tests in 34.365s

OK
```

That's not a lot of information here... What if one of these 29 tests was misbehaving and taking too long? Wouldn't it be better if we could

 - Tell the sequence in which tests are being ran?
 - Know the time each one of these tests took to execute?```

Something like this would be nice:

```
$ python manage.py test tests.functional.test_search

Creating test database for alias 'default'...

    testing function 'setUp'
    [OK] in 'setUp' 9.12 sec

    testing function 'test_ajax_search_thing'
    [OK] in 'test_ajax_search_thing' 3.94 sec
.
    testing function 'setUp'
    [OK] in 'setUp' 10.81 sec

    testing function 'test_ajax_search_model'
    [OK] in 'test_ajax_search_model' 9.55 sec
.
----------------------------------------------------------------------
Ran 2 tests in 34.270s

OK
```
Now it's much easier to understand what's going on: `setUp()` - which runs before every test as expected, is taking between 9-10s!

Time to change dev notebook!

## Tracking Time with a Python Decorator
Sometimes you find solutions in places you were **not** looking for, and in this case I was researching scraping projects when I came upon Lucas Ou-Young's excellent resource on [Article scraping & curation](https://github.com/codelucas/newspaper), where he uses a decorator to produce the detailed output above.

Let's modify the code on the [previous post](/2015/09/21/how-to-test-django-applications_pt1/) to do that:

Open `tests/testing_utilities` and add the decorator code:

```python

# add this to existing code:
def print_test_time_elapsed(method):
    """
    Utility method for print verbalizing test suite, prints out
    time taken for test and functions name, and status
    """
    def run(*args, **kw):
        ts = time.time()
        print('\n\ttesting function %r' % method.__name__)
        method(*args, **kw)
        te = time.time()
        print('\t[OK] in %r %2.2f sec' % (method.__name__, te - ts))

    return run
```

That is so slick!

Now all we have to do is import `print_test_time_elapsed` into our test suites and use it as a decorator:


For example : `tests/unit/test_requests.py`

```python
from django.test import TestCase
from django.test.client import RequestFactory
from my_application.views import home, ajax_search
from ..testing_utilities import populate_test_db, print_test_time_elapsed

class RequestTests(TestCase):

    @print_test_time_elapsed
    def setUp(self):
        # Every test needs access to the request factory.
        self.factory = RequestFactory()
        # Add records to test DB
        populate_test_db()

    @print_test_time_elapsed
    def test_home_view_without_client(self):
        request = self.factory.get('/')
        response = home(request)
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Some text that should be in the HOME view")

    @print_test_time_elapsed
    def test_category_view(self):
        request = self.factory.get(reverse('category',
                                           kwargs={'cat_id': 1}))
        response = category(request, 1)
        self.assertEqual(response.status_code, 200)


    # .......

```


---
## References
[Django's Official Tutorial](https://docs.djangoproject.com/en/1.8/intro/tutorial05/) and [Django's Testing Tools Docs](https://docs.djangoproject.com/en/1.8/topics/testing/tools/) - Comprehensive resources, but they made more sense to me after I understood the different test types.

[The Most Efficient Django Test](http://blog.doismellburning.co.uk/the-most-efficient-django-test/) - If you could only write one single test for your Django App, this would be it.

[Marina Mele's Django Tutorial](http://www.marinamele.com/taskbuster-django-tutorial/create-home-page-with-tdd-staticfiles-templates-settings#tdd-tests) - Not just a great Django tutorial, but also a good introduction on using `Selenium Webdriver` and `LiveServerTestCase` for functional tests.

[Toast Drive's Guide to Testing in Django #2](http://toastdriven.com/blog/2011/apr/17/guide-to-testing-in-django-2/) - The reference used to testing POST requests.

[Newspaper3k: Article scraping & curation](https://github.com/codelucas/newspaper) - Great reference on scraping that also has an interesting approach on how to measure individual test's running times.

