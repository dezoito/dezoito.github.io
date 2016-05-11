---
layout: post
comments: true
title: Django and GitLab - Running Continous Integration and tests with your FREE account
excerpt_separator: <!--more-->
---

This article atempts to explain how to setup your Django project so you can leverage [GitLab.com's](https://www.gitlab.com) **free** Continuous Integration features - they are available on their hosted environment, for their free account tier, on top of unlimited private repositories!

The TLDR version is:

You push your changes to a remote "dev" branch and GitLab will:

- Read a very simple config file.
- Build your environment on their cloud.
- Run your tests - including PhantomJS Integration tests - on this environment.
- Mark the build as "successful" if all tests pass.

See the "References" section if you need more complicated options, because this is intended to be pretty basic.
<!--more-->

### Pre-requisites

1. You have to have an account on [GitLab.com](https://www.gitlab.com).

2. Your Django project should have scripts describing how to set it up and install its dependencies (I used Pydanny's excellent [Django Cookiecutter](https://github.com/pydanny/cookiecutter-django) project template, but I'll go into a little more detail ahead).

3. It also needs to have some automated unit or integration tests (check my [Ready to use Structure for Django Tests + Examples](/2015/09/21/how-to-test-django-applications_pt1/) entry for some ideas).

### Configuration
First thing we need to do is to tell the GitLab Runner what it should do whenever you push code to your repository.

This is done in a file called `.gitlab-ci.yml` that has to be saved in your project's root directory:

{% highlight yaml linenos %}
image: python:3.5

all_tests:
  script:
   - sh ./sh_scripts/install.sh
   - python3 manage.py test -k
  when: on_success
  only:
    - dev
{% endhighlight %}


Going line by line, here's what we are doing
