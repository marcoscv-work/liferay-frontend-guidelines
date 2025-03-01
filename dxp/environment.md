# Environment

Unlike many other, smaller frontend projects, [Liferay DXP](https://github.com/liferay/liferay-portal) does not have a simplistic notion of running in a "production" or "development" environment: instead, it has a large number of individual properties [used at run-time](https://github.com/liferay/liferay-portal/blob/master/portal-impl/src/portal.properties) and [build-time](https://github.com/liferay/liferay-portal/blob/master/build.properties) that control specific aspects of its behavior, and these can be used in many combinations.

This document will cover:

1. Run-time settings that are useful for development-like environments.
2. Run-time settings that are useful for production-like environments.
3. Build-time settings and what `NODE_ENV` means in the context of Liferay DXP.

## Run-time settings for "development"-like enviroments

For the most part, Liferay DXP ships with defaults that make it behave in a reasonable way "out of the box" in production-like environments. This means that for development purposes, some adjustment is required. Assuming a typical development set up in which you have a clone of the [liferay-portal repo](https://github.com/liferay/liferay-portal) alongside a "bundles" directory from which you run Tomcat, you can add the following settings to your `bundles/portal-ext.properties` file.

For the latest recommended developer settings, see the [portal-developer.properties file](https://github.com/liferay/liferay-portal/blob/master/portal-impl/src/portal-developer.properties).

### Turning off caching

```
com.liferay.portal.servlet.filters.alloy.CSSFilter=false
com.liferay.portal.servlet.filters.cache.CacheFilter=false
com.liferay.portal.servlet.filters.etag.ETagFilter=false
com.liferay.portal.servlet.filters.header.HeaderFilter=false
combo.check.timestamp=true
javascript.fast.load=false
layout.template.cache.enabled=false
theme.css.fast.load.check.request.parameter=true
theme.css.fast.load=false
theme.images.fast.load.check.request.parameter=true
theme.images.fast.load=false
```

### Turning off minification

```
javascript.log.enabled=false
minifier.enabled=false
```

### Enabling telnet access to the Gogo shell

You can access the [Gogo shell](https://portal.liferay.dev/docs/7-2/customization/-/knowledge_base/c/using-the-felix-gogo-shell) from the web interface (Control Panel &raquo; Configuration &raquo; Gogo Shell), even in production-like environments. In development contexts, however, it is useful to enable local access via telnet:

```
module.framework.properties.osgi.console=localhost:11311
```

### Assorted quality-of-life improvements

```
# Don't pop up a browser window at launch.
browser.launcher.url=

# Enable theme previews.
com.liferay.portal.servlet.filters.themepreview.ThemePreviewFilter=true

# Automatically update database schema when needed.
schema.module.build.auto.upgrade=true

# Delay the expiry of login sessions.
session.timeout=100000
session.timeout.auto.extend=true
```

### Allow [IPv6 from localhost](https://dev.liferay.com/en/discover/deployment/-/knowledge_base/7-0/choosing-ipv4-or-ipv6)

```
tunnel.servlet.hosts.allowed=0:0:0:0:0:0:0:1
```

## Run-time properties for "production"-like enviroments

As mentioned above, the default settings are usually suitable for "production"-like environments, so an empty `portal-ext.properties` file is a good approximation for developing and testing functionality in a production environment.

### Turning on GZIP

This one is off by default because of its CPU utilization, but you may want to activate it for testing GZIP-related functionality:

```
com.liferay.portal.servlet.filters.gzip.GZipFilter=true
gzip.compression.level=1
```

## Build-time properties and the role of `NODE_ENV` in Liferay DXP

### `NODE_ENV=production`

Similar to how our runtime settings have "production"-like defaults, in the absence of an explicit `NODE_ENV`, our build tooling will behave suitably for a production environment.

### `NODE_ENV=test`

Jest tests will _run_ in the "test" environment by default.

Note, however, that our CI infrastructure performs a standard build before running any tests, and that build effectively uses a `NODE_ENV` of "production".

### `NODE_ENV=development`

`NODE_ENV=development` will force our build tooling to produce development-friendly versions of artifacts.

### Setting `NODE_ENV` temporarily

There are three main ways to invoke frontend tooling such as [liferay-npm-bundler](https://github.com/liferay/liferay-js-toolkit/tree/master/packages/liferay-npm-bundler) and [liferay-npm-scripts](https://github.com/liferay/liferay-npm-tools/tree/master/packages/liferay-npm-scripts) in the context of Liferay DXP:

-   Using Yarn (eg. `yarn build`, `yarn checkFormat` etc); or:
-   Via Gradle (eg. `gradlew deploy -a`); or:
-   Ant (eg. `ant all`).

> **Note:** We recommend installing the [gradle-launcher package](https://www.npmjs.com/package/gradle-launcher) which provides a global `gradlew` executable that automatically locates and runs the nearest local `gradlew` script.

When invoking `yarn` directly, you can set the `NODE_ENV` in the traditional way; that is, with a command like:

```sh
env NODE_ENV=development yarn build
```

When using `ant` or `gradlew`, the `nodejs.node.env` property defined in [the `build.properties` file](https://github.com/liferay/liferay-portal/blob/master/build.properties) at the top of the [liferay-portal repo](https://github.com/liferay/liferay-portal) controls the value of `NODE_ENV` passed to the tools. It defaults to empty, which is equivalent to "production".

To override the value in a Gradle run:

```sh
# Use `-P` to override `nodejs.node.env` from build.properties:
gradlew clean deploy -a -Pnodejs.node.env=development

# If `nodejs.node.env` is not set in build.properties, a simple `env NODE_ENV` will work too:
env NODE_ENV=development gradlew clean deploy -a
```

To override the value in an Ant run:

```sh
# Use `-D` to override `nodejs.node.env` from build.properties:
ant all -Dnodejs.node.env=development

# If `nodejs.node.env` is not set in build.properties, a simple `env NODE_ENV` will work too:
env NODE_ENV=development ant all
```

> **Warning:** Using `-D` in this way will cause `ant` to cache a copy of the override in your `.gradle/gradle.properties` file, which means it _will_ take effect on following `gradlew` runs.

### Setting `NODE_ENV` persistently

If you want `NODE_ENV=development` to apply persistently when developing inside liferay-portal you have several options:

-   For `ant` and `gradlew`:
    -   Create a `build.$USER.properties` file in liferay-portal repo root containing `nodejs.node.env=development`.
-   For `yarn` and other commands run in the shell:
    -   Run `export NODE_ENV=development` at the start of your shell session; or:
    -   Add that `export` to your `~/.bash_profile`/`~/.zshrc` etc.

In practice, actually deploying changes requires the use of Gradle, so we recommend setting `nodejs.node.env` in your `build.$USER.properties` file.

> **Note:** `build.*.properties` files are ignored by Git, which means that they won't survive a `git clean`. Either keep a copy of your customizations in a safe place, or pass the `-e build.$USER.properties` flag when you run `git clean`.

#### Use case example: making a development build of React

Building a development version of React enhances the debugging experience in [the React Dev Tools](https://github.com/facebook/react-devtools). To do this, deploy a development build from inside [the frontend-js-react-web module](https://github.com/liferay/liferay-portal/tree/master/modules/apps/frontend-js/frontend-js-react-web):

```sh
# Override nodejs.node.env just once:
gradlew clean deploy -a -Pnodejs.node.env=development

# Or, if you have a persistent override set up in your
# build.$USER.properties file, as described above:
gradlew clean deploy -a
```

### Other useful `build.properties`

```
# Speed up builds by skipping JSP precompilation.
jsp.precompile=off
```
