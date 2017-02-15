---
layout: page
title: Archery Engine
---

# About

This repository contains build scripts for the _Archery Engine_. The _Engine_ is deployed to
[http://archery.ntr1x.com](http://archery.ntr1x.com), you can see it in action there.
This page describes how to build and install it. Here you can read more about the whole project:
[https://ntr1x.github.io/](https://ntr1x.github.io/).

# Installation

Let's assume that you have a $PROJECT_DIR folder and want to build _Engine_ there.

``` bash
cd $PROJECT_DIR
```

_Engine_ consists of several modules. You have to clone all
necessary modules before you build Archery.

``` bash

git clone git@github.com:ntr1x/ntr1x-archery.git
git clone git@github.com:ntr1x/ntr1x-archery-core.git
git clone git@github.com:ntr1x/ntr1x-archery-shell.git
git clone git@github.com:ntr1x/ntr1x-archery-landing.git
git clone git@github.com:ntr1x/ntr1x-archery-widgets.git
git clone git@github.com:ntr1x/ntr1x-archery-widgets-academy.git
```

Last dependency is optional, it contains some extra site-specific widgets.
This dependency is good entry point for those who code.

Then install dependencies for modules:

``` bash
cd $PROJECT_DIR/ntr1x-archery-core && npm i
cd $PROJECT_DIR/ntr1x-archery-landing && npm i
cd $PROJECT_DIR/ntr1x-archery-shell && npm i
cd $PROJECT_DIR/ntr1x-archery-widgets && npm i
cd $PROJECT_DIR/ntr1x-archery-widgets-academy && npm i
```

Finally build the app:

``` bash
cd $PROJECT_DIR/ntr1x-archery && npm i
```

The app will be built to the `$PROJECT_DIR/ntr1x-archery/`

# Launching

There are two launch entry points, one for the cloud editor
installations and another for the portal viewers. You have
to launch both.

``` bash
pm2 start $PROJECT_DIR/ntr1x-archery/www/bin/server
pm2 start $PROJECT_DIR/ntr1x-archery/www/bin/viewer
```

The Server will be launched at
[http://localhost:3000/](http://localhost:3000/).
The Viewer ill be launched at
[http://localhost:3001/](http://localhost:3001/).

> The Server link will be accessible without additional configuration.
> But the Viewer will fail. The Viewer app uses domain names to determine
> target portals, so you will not be able to use Viewer until you
> configure domain proxy.

# Configuration

The configuration is located in `$PROJECT_DIR/ntr1x-archery/config` folder
You can change there the default server endpoint and application ports.

With the endpoint provided you will see all the data published at [http://archery.ntr1x.com](http://archery.ntr1x.com).
You can change the Storage endpoint when you launch your own Storage.
