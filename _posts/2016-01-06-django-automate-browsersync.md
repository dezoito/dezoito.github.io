---
layout: post
comments: true
title: Django - Develop faster with BrowserSync
excerpt_separator: <!--more-->
---

This is probably the simplest/fastest way to integrate [BrowserSync](https://www.browsersync.io/) with your Django development workflow, allowing a few cool thinks to happen automatically:

- Browser reloads whenever you edit your code (no more CTRL+R or F5);

- Sync your view across multiple browsers and devices, so you can see how changes are displayed in different platforms/window sizes;

- Interactions on one device are also synced, any scrolling or typing in one window is simultaneously reflected in the other ones;

<!--more-->

This can easily be used in Flask (or any other framework really), by changing a `proxy` variable appropriately.


### Pre-requisites

First you need to have `NodeJS` and `npm ` and `gulp` running on your dev machine, then follow the [BrowserSync](https://www.browsersync.io/docs/gulp/) instructions for installation.

### Configuration

Save this file as `gulpfile.js` in your Django project's root:

```js
var gulp        = require('gulp');
var browserSync = require('browser-sync');
var reload      = browserSync.reload;

gulp.task('default', function() {
    browserSync.init({
        notify: false,
        proxy: "localhost:8000"
    });
    gulp.watch(['./**/*.{scss,css,html,py,js}'], reload);
});
```

Then on your terminal:

```sh
$ cd /path/to/project_root
$ gulp
```

That's it!

If everything goes according to plan, this will open a browser window on port 3000, proxying your django project's URL, and listening to changes on files with the following extensions `scss,css,html,py,js`.

The original project's URL is defined in `proxy: "localhost:8000"`, so this is where you make changes if your Django test server is using a different port, or if you are developing in Flask (or any other framework, really).


