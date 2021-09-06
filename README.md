## `rucksac/skeleton` :skull:

An Odoo project skeleton and workflow for local development.

---

- [What is this?](#what-is-this)
- [Requirements](#requirements)
- [Quickstart](#quickstart)
- [Getting started guide](#getting-started-guide)
    - [1. Clone the project](#1-clone-the-project)
    - [2. Install `gitman` dependencies](#2-install-gitman-dependencies)
    - [3. Install python packages](#3-install-python-packages)
    - [4. Setup the database](#4-setup-the-database)
    - [5. Add `odoo.conf` and update](#5-add-odoo.conf-and-update)
    - [6. Run the thing](#6-run-the-thing)
- [Day to day workflow](#day-to-day-workflow)
- [Multiple version workflow](#multiple-version-workflow)
- [A note about `source_paths`](#a-note-about-source_paths)

### What is this?

This is a project skeleton and workflow that I use to manage and build multiple local development environments day to day. The skeleton is meant to be a starting place for a brand new project.

---

### Requirements

Here are the tools I use.

- [`gitman`](https://github.com/jacebrowning/gitman): Git repo dependencies. Used for odoo core code, enterprise code, and 3rd party addons repos.
- [`pdm`](https://github.com/pdm-project/pdm): Used for python package management and to avoid virtual environments.
- [`docker-compose`](https://docs.docker.com/compose/install/): Used for local database development.

Technically none of these are hard requirements, but these are the tools I like to work with. I honestly don't think the skeleton provides much value without them.

I **highly recommend caching the Odoo core code** first before doing anything with this. Building a mirror of Odoo takes a long time (20-30 min), but only has to be done once per machine. Also if you don't do it yourself manually, then `gitman` will automatically do this for you. That means the very first project you setup with the skeleton with take a long time to build. This can be confusing, so it's best to just pre build the mirror with the expectation that it's going to take a bit. Once the mirror is built though, all future `git clone`'s of the odoo repository only take a few seconds:

```sh
$ git clone --mirror https://github.com/odoo/odoo ~/.gitcache/odoo.reference
```

---

### Quickstart

If you have already used this skeleton and process before, here are a quick set of steps as a reminder to get a project going. If you **don't have any experience with this skeleton** I'd recommend going through the [Requirements](#requirements) and then the [Getting started guide](#getting-started-guide) sections.

```sh
# clone...
$ git clone https://github.com/rucksac/skeleton newproject
$ cd myproject

# dependencies...
$ vim gitman.yml  # add gitman dependencies
$ gitman update
$ pdm init -n
$ pdm import vendor/odoo/requirements.txt
$ vim pyproject.toml  # clean up duplicate dependencies, and set python required
$ pdm update

# bring up database...
$ cp .env.sample .env
$ docker-compose up -d

# make odoo.conf...
$ cp samples/conf/odoo14.conf odoo.conf

# run...
$ pdm run vendor/odoo/odoo-bin -c odoo.conf

```

---

### Getting started guide

#### 1. Clone the project

```sh
$ git clone https://github.com/rucksac/skeleton myproject
$ cd myproject
```

#### 2. Install `gitman` dependencies

Open up the `gitman.yml` and add in any 3rd party odoo module dependencies. This will include the odoo source code, the enterprise addons (if needed), and then any open source git repos. Here's a sample of what that will look like:

```sh
$ vim gitman.yml
```

```sh
location: vendor
sources:
  - repo: https://github.com/odoo/odoo
    name: odoo
    rev: 14.0
  - repo: https://github.com/OCA/project
    name: OCA--project
    rev: e64f8b5785ae25ef14e2e7084c37be0b4e89ba1e
    sparse_paths:
      - project_key
      - project_list
      - project_tag
```

Each `source` has a few common options that are helpful:

- `repo`: The url of the git repository.
- `name`: A unique name for the source. For addons, I like the pattern `{org}--{repo}`, for example if I was using `https://github.com/OCA/project` addons, I would name it `OCA--project`.

Then download the addons:

```sh
$ gitman update
```

This will populate the `vendor` folder

```
vendor/
├── OCA--project
│   ├── project_key
│   ├── project_list
│   ├── project_tag
└── odoo
    ├── CONTRIBUTING.md
    ├── COPYRIGHT
    ├── LICENSE
    ├── MANIFEST.in
    ├── README.md
    ├── SECURITY.md
    ├── addons
    ├── debian
    ├── doc
    ├── odoo
    ├── odoo-bin
    ├── requirements.txt
    ├── setup
    ├── setup.cfg
    └── setup.py
```

#### 3. Install python packages

Initialize the `pdm` environment and import requirements:

```sh
$ pdm init -n
$ pdm import vendor/odoo/requirements.txt
```

Then we do need to clean up the `pyproject.toml` file a bit. There are a couple of updates to do:

3.a. Update the python version required. For example, for 14.0 I use `>=3.8`.

```toml
requires-python = ">3.8"
```

3.b. Remove duplicate dependencies.

There's an [open issue](https://github.com/pdm-project/pdm/issues/46) on the `pdm` project about the ability to add duplicate dependencies. Odoo does use multiple dependencies in its `requirements.txt` for different versions. We'll need to go through and pick the proper option for each duplicate version.


For example, in a 14.0 instance using python 3.8 today, I would remove the lines commented below so that there is a single line for each package:

```toml
dependencies = [
    "freezegun==0.3.15; python_version >= \"3.8\"",
    # remove...
    # "freezegun==0.3.11; python_version < \"3.8\"",

    "gevent==20.9.0; python_version >= \"3.8\"",
    # remove...
    # "gevent==1.1.2; sys_platform != \"win32\" and python_version < \"3.7\"",
    # "gevent==1.5.0; python_version == \"3.7\"",

    "greenlet==0.4.17; python_version > \"3.7\"",
    # remove...
    # "gevent==1.4.0; sys_platform == \"win32\" and python_version < \"3.7\"",
    # "greenlet==0.4.10; python_version < \"3.7\"",
    # "greenlet==0.4.15; python_version == \"3.7\"",

    # ...
]
```

Then we can run the update:

```sh
$ pdm update
```

#### 4. Setup the database

I use `docker-compose` to bring up a quick postgres database for local development projects:

```sh
$ cp .env.sample .env
$ docker-compose up -d
```

#### 5. Add `odoo.conf` and update

I have some sample `odoo.conf` file per version that can be dropped in to get going:

```sh
$ cp samples/conf/odoo14.conf odoo.conf
```

Then you'll want to take a look to see if you need to make any configuration updates. You may need to set a master password, add any addon dependencies to your `addons_path`, change ports, update db connection details, etc.

```sh
$ vim odoo.conf
```

#### 6. Run the thing

```sh
$ pdm run vendor/odoo/odoo-bin -c odoo.conf

Running at localhost:8069 by default...
```

---

### Day to day workflow

I'm jumping in and out of projects throughout the day, so here are some common commands I'm using:

**Stopping the current project.**

If you are running the project via the `odoo-bin` script, then I simply need to `ctrl-c` to kill the Odoo process. Otherwise if you are using service scripts or a tool like surpervisor, you'll need to stop the service.

Then stop the docker containers with `docker-compose stop`. If you don't stop the containers, you could run into port conflict errors when starting up a new project. If you ever need to get around this, just change port mapping in your `docker-compose.yml`.

**Moving to the next project.**

I will move over into my next project:

```sh
$ pwd
~/work/projects/a

$ cd ~/work/projects/b
```

I will usually check the branch, and double check the dependencies are up to date:

```sh
$ gitman update
$ pdm update
```

Bring up the database:

```sh
$ docker-compose up -d
```

Run the instance:

```sh
$ pdm run vendor/odoo/odoo-bin -c odoo.conf
```

And then get to work.

---

### Multiple version workflow

Handling projects that support multiple versions (12.0, 13.0, 14.0, etc.) can be a little bit tricky when it comes to git branching. For example, when you switch from branch 12 to branch 13, then you need to make sure that your dependencies are up to date. This means deleting `vendor/` dependencies, or the `pdm` dependencies and re-running everything each time to switch major version branches.

That's possible, but personally I have not enjoyed that workflow.

So I take advantage of the fact that I have plenty of storage on my computer and I create projects for each major version.

That means my work directory looks something like the following where both the `craftstore_erp/v12` and `craftstore_erp/v13` folders are clones of the same repository, but the `v12` directory is only for the `12.0` branch and feature branches off of `12.0`, and the `v13` directory is only for the `13.0` branch and feature branches off of `13.0`.

```
work/
└── clients
    └── craftstore_erp
        ├── v12
        │   ├── LICENSE
        │   ├── README.md
        │   ├── __pypackages__
        │   ├── apps
        │   ├── docker-compose.yml
        │   ├── docs
        │   ├── gitman.yml
        │   ├── logs
        │   ├── odoo.conf
        │   ├── pdm.lock
        │   ├── pyproject.toml
        │   ├── samples
        │   └── vendor
        └── v13
            ├── LICENSE
            ├── README.md
            ├── __pypackages__
            ├── apps
            ├── docker-compose.yml
            ├── docs
            ├── gitman.yml
            ├── logs
            ├── odoo.conf
            ├── pdm.lock
            ├── pyproject.toml
            ├── samples
            └── vendor
```

---

### A note about `source_paths`

Gitman allows you to extract specific files from a dependency. This is great for Odoo projects because you can depend on an open source repo of addons like an OCA project and list the specific addons that your project needs vs downloading every addon.

But there is a bug where running `gitman install` or `gitman update` doesn't bring in new modules after changing `source_paths`. I reported this as [an issue here](https://github.com/jacebrowning/gitman/issues/265).

As a workaround, delete the folder dependency in `vendor/` and then re-run `gitman install`.
