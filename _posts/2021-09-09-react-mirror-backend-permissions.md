---
layout: post
comments: true
title: Mirror backend permissions on a React frontend
excerpt_separator: <!--more-->
---

In this article we explore a way to replicate a backend's authorization system on a React SPA, so that developers can restrict access to features, pages or components based on group memberships or user permissions.

Although this example is based on a Django backend, it's easily applicable to any backend framework that can return a `user` object by REST or GraphQL API calls.

<br/>

> IMPORTANT
> 
>This is not an end all security solution, as any content or data sent to the browser might be accessible to a knowledgeable or malicious user.
>
> Permission checks **have** to be enforced at the backend.
>
> The goal is to prevent users from seeing content that is not useful to them, like buttons or menu options that would just throw an "UNAUTHORIZED" error at the server.

<!--more-->

<br/>

## Pre-requisites

- A working RESTFul or GraphQL backend application that implements user authorization and permissions ([See Django's doc for a reference](https://docs.djangoproject.com/en/3.2/topics/auth/default/#topic-authorization)).

- A React web application as the frontend for the above system.

- The React application has to have a global state management solution implemented (Redux, Zustand, MobX or Context API, for example);

<br/>

## The Problem

Frameworks like Django make it easy to assign permissions to users and groups, and let only **authorized** users - the ones that have such permissions or are in those groups - perform actions or even see specific content.

Some trivial examples of authorization rules:

- Only people in the "Moderators" or "Admin" groups can delete posts or ban users in discussion boards.

- Only users in the "Data Scientists" group with the "Create Experiment" permission can create new Machine Learning experiments.

- Only users with the permission "Run Any experiment" and members of the "Admin" group can list and run all experiments.

We want to implement some granularity where we can authorize users by **group membership**, **assigned permissions**, or a combination of **both**.

In terms of React, we want to use this as a way to deny access to **routes**, **pages** or even **simple components**.

<br/>

## Protecting Content

Let's assume we have a menu component, and that we want to:

- show the "List Tests" option only to members of the "Testers" group;

- show the "Create Experiments" option only to users that have the "ml.can_create_experiments" permission;

We could use a `<ProtectedContent/>` component to wrap `MenuItems`, and test if the user has a permission or is a member of a group before adding an option to the menu:

```jsx

import ProtectedContent from "../path/ProtectedContent";
...

<Menu
  id="main-menu"
  someMenuProp={true}
  ...
>

  {/* Checks if current user can create experiments */}
  <ProtectedContent perms={["ml.can_create_experiments"]}>
    <MenuItem component={Link} to={"/createexp"}>
      Create Experiments
    </MenuItem>
  </ProtectedContent>

  {/* Checks if current user is a member of "Testers" group */}
  <ProtectedContent groups={["Testers"]}>
    <MenuItem component={Link} to={"/tests"}>
      List Tests
    </MenuItem>
  </ProtectedContent>

  {/* All logged in users can see this */}
  <MenuItem component={Link} to={"/datasets"}>
    List Datasets
  </MenuItem>
</Menu>
```

> Note that this is just an example. Unauthorized users won't see the menu options but may still be able to go directly to their URLs if they knew them in advance (see Advanced Usage for ways to address that).

<br/>

## User Representation

Before we can implement `<ProtectedContent/>`, we need to understand how the `user` object is structured.

The backend application's `/login` endpoint returns the following representation of the current user upon a successful authentication (simplified for clarity):

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
> In django, we use a function called `get_all_permissions()` to generate the list of permissions assigned directly to the user or inherited through group memberships.

The object above is stored in global state and is accessible to any React component in the tree (I really enjoy using [Zustand](https://github.com/pmndrs/zustand) for state management).

<br/>

## Component Implementation

Here is a possible implementation of `<ProtectedContent/>`:

```jsx
import { forwardRef, useMemo } from "react";
import useAuth from "../hooks/useAuth";

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
  if (!isAuthenticated) {
    return <p>User is not authenticated!</p>;
  }

  // If the user is an admin/superuser, grant access to everything
  if (user?.is_superuser) {
    showContent = true;
  } else {
    // Show content if the user is a member of any groups passed as props
    showContent = userGroups.some((group: string) => groups?.includes(group));

    // Show content if the user has any of the permissions passed as props
    if (!showContent) {
      showContent = permsSet.some((perm: string) => perms?.includes(perm));
    }
  }

  // Either render the protected content or an alterative for unauthorized users
  return showContent ? <>{children}</> : <>{alt}</>;
});

export default ProtectedContent;
```

<br/>

Let's break that down and go over the less obvious parts.

First we define a typescript interface to describe the `props` the component can accept:

```ts
interface Props {
  children?: React.ReactNode | React.ReactNodeArray;
  alt?: React.ReactNode | React.ReactNodeArray;
  groups?: string[];
  perms?: string[];
}
```

- `children` is the content that we want to display if the user is authorized.
- `alt` is the **optional** content we display if the user is not authorized

Both are either a single instance or an array of React nodes (for example, another component or tag).

- `groups` is an **optional** array of group names - the user will be authorized if they are a member of at least one of specified groups.

- `perms` is an **optional** array of permission names - the user will be authorized if they have of at least one of specified groups.

<br/>

We need to wrap our component with `forwardRef` so higher level components can access `refs` from the components we are wrapping (in our example, the `<Menu/>` component accesses refs from `<MenuItem/>`).

```ts
const ProtectedContent = forwardRef((props: Props, ref): JSX.Element => { .... }

```

For a more detailed explanationplease read [Forwarding React Refs with TypeScript](https://www.carlrippon.com/react-forwardref-typescript/).

<br/>

Next we get the user data and authentication status:

```ts
const { isAuthenticated, user } = useAuth();
```

`useAuth()` is a custom hook that returns the current user authentication status and the `user` object as described above.

<br/>

The rest of the code consists of checking if the user:

- is an admin
- is member of a group (if `groups` was passed as a prop)
- has a permission (if `perms` was passed as a prop)

if they pass one of these checks the content is rendered, otherwise we will render the contents of the `alt` prop (if it was passed), or nothing at all.

> If the use is an "admin" (`is_superuser === true`), they are automatically authorized.

<br/>

## Advanced Usage

Here are some recipes for protecting content in different ways:

Protecting routes:

```jsx
<Switch>
  <ProtectedContent groups={["Members"]}>
    <Route path={"/members-only"} exact>
      <MembersClub />
    </Route>
  </ProtectedContent>
</Switch>
```

Protecting content in a "page":

```jsx
<div>
 <p>Anyone can read this</p>

  <ProtectedContent groups={["Members"]}>
    <div className="special-message">
      <p>... but only members can see the special message
    </div>
  </ProtectedContent>
</div>
```

Protecting content in a "page", but display alternative content to unauthorized users:

```jsx
<div>
 <p>Anyone can read this</p>

  {/* Displays a link if the user is not a member */}
  <ProtectedContent
    groups={["Members"]}
    alt={<Link to={"/join-us"}>Become a member!</Link>}>

      <div className="special-message">
        <p>... but only members can see the special message
      </div>
  </ProtectedContent>
</div>
```

Admins only:

```jsx
<div>
  <p>Anyone can read this</p>

  {/* No props needed! */}
  <ProtectedContent>
    <button className="dangerous-action">
      Click here to perform admin action
    </button>
  </ProtectedContent>
</div>
```

Allow multiple groups:

```jsx
<ProtectedContent groups={["Members", "Staff"]}>
  <div>Today's specials!</div>
</ProtectedContent>
```

Allow different permissions:

```jsx
  <ProtectedContent
    perms={["can_manage_whisky", "can_see_all_whisky"]}>
    <div className="whisky-list">
      <li> Laphroaig Lore </li>
      <li> Laphroaig PX Cask </li>
      <li> Hibiki Japanese Harmony </li>
      <li> Jameson Caskmates </li>
      <li> Caol Ila Distillers Edition </li>
      <li> ... </li>
    </div>
  </ProtectedContent>

```

Combine Groups and permissions:

```jsx

  {/* Allow either "staff" members OR any users with "can_see_all_whisky" */}
  <ProtectedContent
    groups={["Staff"]}>
    perms={["can_see_all_whisky"]}
    >
    <div className="whisky-list">
      <li> Laphroaig Lore </li>
      <li> Laphroaig PX Cask </li>
      <li> Hibiki Japanese Harmony </li>
      <li> Jameson Caskmates </li>
      <li> Caol Ila Distillers Edition </li>
      <li> ... </li>
    </div>
  </ProtectedContent>

```

<br/>

## Future Improvements
Some possible improvements for this component include:

- Redirecting unauthenticated users to a `login` page.

- Adding an optional `rule` prop, so we can add other custom checks (for example, display a **Delete** button only if a user is the creator of an object):

```jsx
  <ProtectedContent
    rule={user.id===thing.creator.id}
  >
    <button>
      Delete thing!
    </button>
  </ProtectedContent>

```

Do you see any different ways this component could be improved? Let me know in the comments!

<br/>

## References

- [Forwarding React Refs with TypeScript](https://www.carlrippon.com/react-forwardref-typescript/)
- [React Children with TypeScript](https://www.carlrippon.com/react-children-with-typescript/)
- [React Router's Auth workflow](https://reactrouter.com/web/example/auth-workflow)
- [SO answer on "Implement react-router PrivateRoute in Typescript project"](https://stackoverflow.com/a/53111155)
