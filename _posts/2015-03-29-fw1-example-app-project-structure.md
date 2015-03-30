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


The Clipping app was designed with
**[Subsystems](https://github.com/framework-one/fw1/wiki/Using-Subsystems)** enabled.
Using those is completely optional, but I decided to use them so that, in the future,
it will be eaqsier to add functionality, by incorporating third party apps.

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

| Folder | Description |
| --- | --- |

**`/common`** - This folder holds the layout files that will be used in **every** subsystem.

**`/customtags`** - This is where CFML tags are stored. I added a mapping in Application.cfc pointing to this folder.

**`/home`** - The directory that contains **the clipping application**, which is also the **default subsystem**
