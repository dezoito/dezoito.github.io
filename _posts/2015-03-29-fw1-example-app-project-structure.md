---
layout: post
title: FW/1 Example Application - Project Structure
---

This is the first in a series of articles on how to build a
[ColdFusion and FW/1 Example Application](https://dezoito.github.io/2015/03/26/fw1-example-app-released/).

I'll assume you are not new to CFML, so don't take this as a detailed guide or
tutorial, but more of a starting point (I'll be a little more detailed in the article on tests).

Here's the list of topics:

 - Project Structure
 - Forms and Validation Patterns
 - Use of User Defined Function Libraries
 - Accessing an External Service
 - BDD and Integration Tests.

# FW/1 Project Structure

Reading pre-requisites: [Developing Applications with FW/1](https://github.com/framework-one/fw1/wiki/Developing-Applications-Manual).

The Clipping app was designed with
**[Subsystems](https://github.com/framework-one/fw1/wiki/Using-Subsystems)** enabled.
Using those is completely optional, but I decided to use them so that, in the future,
it will be easier to add functionality, by incorporating third party apps.

So here's the basic project structure:

```
/clipping
    |
    +- common
        |
        +- layouts
    |
    +- customtags
    +- home
        |
        +- controllers
        +- model
            |
            +- beans
            +- services
        |
        +- views
            |
            +- clipping
            +- helpers
            +- main
    |
    +- lib
    +- setup
    +- static
    +- tests

/testbox
/CFSelenium
```

With the `/clipping` folder being the project's root, we have:

**`/common`** - This folder holds the layout files that will be used in **every** subsystem.

**`/customtags`** - This is where CFML tags are stored. I added a mapping in Application.cfc pointing to this folder.

**`/home`** - The directory that contains **the clipping application** - which is also the **default subsystem** -
containing it's own controllers, models and views.

**`/lib`** - Stores User Defined Functions and Libraries, that will be available to all subsystems.

**`/setup`** - Stores SQL scripts used to prepare the databases
(those files are **not** used by the application, so feel free to delete this).

**`/static`** - Keeps all static files, separated in /js, /css, /images and /ckeditor subfolders.

**`/tests`** - This is where we keep and run 'unit' and integration tests (more on that later.)

Important:
The **`/testbox`** and **`/CFSelenium`** folders are int the webserver's root for simplicity.
You should not use this configuration in a web-accessible environment.




