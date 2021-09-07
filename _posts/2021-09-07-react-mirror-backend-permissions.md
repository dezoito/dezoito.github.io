---
layout: post
comments: true
title: React and Typescript - How to mirror backend permissions on the frontend app
excerpt_separator: <!--more-->
---

In this article we explore a way to replicate a backend's authorization system on a React built frontend, so that developers may restrict access to features/pages/controls on the basis of group membership and the logged in user's assigned permissions.

Although this example is based on a Django app, it's easily appliable to any backend framework that can return a `user` object by REST or GraphQL API calls.

<!--more-->

### Pre-requisites

- A working backend RESTFul or GraphQL application that implements user authorization and permissions ([See Django's doc for a reference](https://docs.djangoproject.com/en/3.2/topics/auth/default/#topic-authorization)).

- A React web application as the frontend for the above system.

### The Problem

Frameworks like Django make it easy to assign permissions to users/groups, and let only authorized users - the ones thatnhave such permissions or are in those groups - perform certain actions or even see certain content.

Some trivial, generic examples:

- Only people in the "Moderators" or "Admin" groups can delete posts or ban users in discussion boards

- Only users in the "Data Scientists" group with the "Create Experiment" permission can create new Machine Learning experiments.

- Only Data Scientists with the permission "Run Any experiment" and members of the "Admin" group can list and run all experiments.

As you can see, we want to implement some granularity where we can filter users by **group membership**, **assigned permissions**, or a combination of **both**.

In terms of React, we want to use this as a way to deny access to **routes**, **pages** or even **sinple components**.

### User Representation

My backend application `/login` endpoint returns the following representation of the current user upon a successfull authentication attempt (simplified for clarity):

```json
user: {
    id: 1233,
    is_superuser: false,
    username: "SDali",
    email: "sdali@somwehere.com",
    is_staff: true,
    is_active: true,
    groups: [
        {
            id: 3,
            name: "Classifiers",
        },
        {
            id: 4,
            name: "Testers",
        },
    ],
    user_permissions: [
        "ml.can_create_experiments",
        "ml.can_run_experiments"
        "ml.can_delete_experiments"
        "core.can_run_classifiers"
        "core.can_create_classifiers"        
        ],
},
```

The object above is stored in global state and is easily accessible to any React in the component tree (I strongly suggest using [Zustand](https://github.com/pmndrs/zustand) for state management).

- explain groups and user permissions

### React Code

### Improvements

### References
