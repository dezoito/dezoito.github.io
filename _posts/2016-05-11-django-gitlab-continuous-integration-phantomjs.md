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

```yaml
image: python:3.5

all_tests:
  script:
   - sh ./sh_scripts/install.sh
   - python3 manage.py test -k
  when: on_success
  only:
    - dev
```


Going line by line, here's what we are doing:

`image: python:3.5` - Tells the runner which Docker Image to use. 

> I had to choose `python:3.5` because the `django:1.8.3` image was failing when installing some dependencies (took me a while to figure that out too). Django is in the project's list of requirements anyway, so it's nice to be able to define which version of Python I want.

`all_tests:` - This is just an arbitrary definition of a stage (I copied it from one of Gitlab's examples).

`  script:` - Here I'm grouping all the actions that the runner is to perform in this stage, as follows:

`   - sh ./sh_scripts/install.sh` - This is the Shell Script that provisions the Docker machine, installing dependencies, running pip to install requirements and setting up PhantomJS - the headless browser we use to run integration tests (see examples ahead).

`   - python3 manage.py test -k` - **If** the previous step is successful, this will simply use Django's test runner to run all tests defined withing the project (the "-k" switch tells Django to keep the test database across tests, speeding things up a bit.)

`  when: on_success` - Tells the runner that the actions should only run when the previous one succeeded.

`only:` and `- dev` - Tells the runner to run this stage only if the push was made to the "dev" branch.

The reason for that is just a personal choice for the moment, but I left it in the article as it shows how you can perform different actions depending on what branch code is being pushed too - you can also ommit it completely if you want things to run regardless of branch.

### Going for a run

Assuming you add a similar  `.gitlab-ci.yml` to your project and pushed the commited changes to "dev", you should be able to go to GitLab's Dashboard for your project, click on the "Builds" tab and see a new "pending" or "running" build on your list.

Click on the build, and when the status changes to "running", you should see a console-like display showing what's going on:

![GitLab's Build Interface](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/gitlab-build1.png?raw=true)

I think it's a good idea to follow it closely during the first runs, so you can see any problems and error messages that may pop up along the way.

> Tip: I like to add some attention starved "echo" statements before each section of my install script, to make the process easier to follow (again, see the examples ahead).

#### Failed?

![GitLab's Build Failed](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/gitlab-build-fail.png?raw=true)

Sometimes your build will fail not because there's anything wrong with your scripts or tests, **but because the docker machine wasn't able to download and install some dependency** - pay attention to the job execution and rerun the build if you think that was the case.

I've had builds that would fail three or four times in a row, then "automagically" work a few minutes later - this can be frustrating when you are just trying to get started.

After some troubleshooting and bearing no inaccessible dependency hosts, you should see something like this: 

![GitLab's Build Passed](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/gitlab-build-passed.png?raw=true)

> Tip: I added a `pip freeze` statement at the end of my install script, so I could see which python dependencies were actually installed and where the build failed.







