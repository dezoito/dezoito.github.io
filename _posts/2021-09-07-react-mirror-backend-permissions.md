---
layout: post
comments: true
title: React and Typescript - How to mirror backend permissions on the frontend app
excerpt_separator: <!--more-->
---

In this article we explore a way to replicate a backend's authorization system on a React built frontend, so that developers may restrict access to features/pages/controls on the basis of group membership and the logged in user's assigned permissions.

Although this example is based on a Django app, it's easily appliable to any backend framework that can return a `user` object by REST or GraphQL API calls.

<!--more-->

## Pre-requisites

- A working backend RESTFul or GraphQL application that implements user authorization and permissions ([See Django's doc for a reference](https://docs.djangoproject.com/en/3.2/topics/auth/default/#topic-authorization)).

- A React web application as the frontend for the above system.

- The React application has to have some global state management solution (Redux, Zustand, MobX or Context API, for example);

## The Problem

Frameworks like Django make it easy to assign permissions to users/groups, and let only authorized users - the ones thatnhave such permissions or are in those groups - perform certain actions or even see certain content.

Some trivial, generic examples:

- Only people in the "Moderators" or "Admin" groups can delete posts or ban users in discussion boards

- Only users in the "Data Scientists" group with the "Create Experiment" permission can create new Machine Learning experiments.

- Only Data Scientists with the permission "Run Any experiment" and members of the "Admin" group can list and run all experiments.

As you can see, we want to implement some granularity where we can filter users by **group membership**, **assigned permissions**, or a combination of **both**.

In terms of React, we want to use this as a way to deny access to **routes**, **pages** or even **sinple components**.

## User Representation

My backend application `/login` endpoint returns the following representation of the current user upon a successfull authentication attempt (simplified for clarity):

```js
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

This user is a member of two groups: `Classifiers` and `Testers`, and is assigned five different permissions (either directly or inherited from group memberships).

> Quick note:
>
> In django, we use a function called `get_all_permissions()` to generate the list of permissions assigned directly to the user and to the groups they belong to.

The object above is stored in global state and is easily accessible to any React component in the tree (I really enjoy using [Zustand](https://github.com/pmndrs/zustand) for state management).



## React Code

Let's assume we have a menu component, and that we want to:

-  show the "List Tests" option only to members of the "Testers" group;

-  show the "Create Experiments" option only to users that have the "ml.can_create_experiments" permission;

Here's what the `Menu` code would look like:

```jsx

import ProtectedContent from "../path/ProtectedContent";

.... 
      <Menu
        id="main-menu"
        someMenuProp={true}
        ...
      >
        <MenuItem onClick={handleClose} component={Link} to={"/datasets"}>
          List Datasets
        </MenuItem>

        <ProtectedContent
          perms={["ml.can_create_experiments"]}
        >
          <MenuItem onClick={handleClose} component={Link} to={"/createexp"}>
            Create Experiments
            </MenuItem>
        </ProtectedContent>

        <ProtectedContent
          groups={["Testers"]}
        >
          <MenuItem
            onClick={handleClose} component={Link} to={"/tests"}
          >
            List Tests
          </MenuItem>
        </ProtectedContent>
      </Menu>
```


And below is the implementation of `<ProtectedContent/>`

```jsx
import { forwardRef, useMemo } from 'react';
import useAuth from '../hooks/useAuth';

interface Props {
  children?: React.ReactNode | React.ReactNodeArray;
  alt?: React.ReactNode | React.ReactNodeArray;
  groups?: string[];
  perms?: string[];
}

const ProtectedContent = forwardRef((props: Props, ref): JSX.Element => {
  const { children, groups, perms, alt } = props;
  const { isAuthenticated, user } = useAuth();

  // groups the user is a member of
  const userGroups = useMemo(() => {
    if (user?.groups?.length) {
      return user.groups.map((g: any) => g.name);
    }
    return [];
  }, [user]);

  // all user permissions
  const permsSet = useMemo(() => {
    return user?.user_permissions || [];
  }, [user]);

  let showContent = false;

  // Block rendering content if the user is not authenticated
  // Might be redundant but that's OK here
  if (!isAuthenticated) {
    return <p>User is not authenticated!</p>;
  }

  // If the user is an admin/superuser, grant access to everything
  if (user?.is_superuser) {
    showContent = true;
  } else {
    // Show  content if the user is a member of any groups passed as props
    showContent = userGroups.some((group: string) => groups?.includes(group));

    // Show content if the user has any of the permissions passed as props
    if (!showContent) {
      showContent = permsSet.some((perm: string) => perms?.includes(perm));
    }
  }

  return showContent ? <>{children}</> : <>{alt}</>;
});

export default ProtectedContent;

```
--- EXPLAIN NON OBVIOUS CODE BLOCKS:
- forwardRef
- useAuth
- prop types


## Advanced Usage

- use in routes
- use in pages
- only admins
- array of groups
- array of permissions
- "alt content"

## Future Improvements

## References
- https://www.carlrippon.com/react-children-with-typescript/
- https://reactrouter.com/web/example/auth-workflow
- https://gitlab.com/saurabhshah231/reactjs-myapp/
- https://stackoverflow.com/a/53111155
