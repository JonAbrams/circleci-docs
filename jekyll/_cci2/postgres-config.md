---
layout: classic-docs
title: "Configuring Databases"
short-title: "Configuring Databases"
description: "Example of Configuring PostgreSQL"
categories: [configuring-jobs]
order: 35
---

*[Build Environments]({{ site.baseurl }}/2.0/build/) > Configuring Databases*

This document describes database configuration and defaults using PostgreSQL and Rails in the following sections:

* TOC
{:toc}

## Database Environment Configuration Overview
In CircleCI 2.0 you must declare your database configuration explicitly because multiple pre-built or custom images may be in use. For example, Rails will try to use a database URL in the following order:

1.	DATABASE_URL environment variable, if set
2.	The test section configuration for the appropriate environment in config.yml (usually `test` for your test suite)

The default user, port, test database for PostgreSQL 9.6 are as follows:

`postgres://ubuntu:@127.0.0.1:5432/circle_test`

**Note:** For version 9.5, the default port is 5433 instead of 5432. To specify a different port, change the `$DATABASE_URL` and all invocations of `psql`.

The following example demonstrates this order by combining the `environment` setting with the image and by also including the `environment` configuration in the shell command to enable the database connection:

```
version: 2
jobs:
  build:
    working_directory: ~/appName
    docker:
      - image: ruby:2.3.1
        environment:
        - PG_HOST=localhost
        - PG_USER=ubuntu
        - RAILS_ENV=test
        - RACK_ENV=test
      - image: postgres:9.5
        environment:
        - POSTGRES_USER=ubuntu
        - POSTGRES_DB=db_name
    steps:
      - checkout
      - run:
          name: Install System Dependencies
          command: apt-get update -qq && apt-get install -y build-essential postgresql libpq-dev nodejs rake
      - run:
          name: Install Ruby Dependencies
          command: bundle install
      - run: 
          name: Set up DB
          command: |
            bundle exec rake db:create db:schema:load --trace
            bundle exec rake db:migrate
        environment:
          DATABASE_URL: "postgres://ubuntu@localhost:5432/db_name"
```

## Using Binaries
To use `pg_dump`, `pg_restore` and similar utilities requires some extra configuration to ensure that `pg_dump` invocations will also use the correct version. Add the following to your config.yml file to enable `pg_*` or equivalent database utilities:

```
     steps:
    # Add the Postgres 9.6 binaries to the path.
       - run: echo ‘/usr/lib/postgresql/9.6/bin/:$PATH’ >> $BASH_ENV
```

## Configuring Passwordless Login
The default PostgreSQL configuration is set up to always ask the user for a password, unless using `sudo`. However, the default user has no password, so you essentially cannot log in without using `sudo`.

To work around this, consider overwriting the existing user with a new user and a password and create your own database, for example:

    steps:
      - run:
    # Add a password to the `ubuntu` user, since Postgres is configured to
    # always ask for a password without sudo, and fails without one.
        command: sudo -u postgres psql -p 5433 -c "create user ubuntu with password ‘ubuntu’;”
        command: sudo -u postgres psql -p 5433 -c "alter user ubuntu with superuser;"

    # Create a new test database.
        command: sudo -u postgres psql -p 5433 -c "create database test;"

## Using Dockerize

Using multiple Docker containers for your jobs may cause race conditions if the service in a container does not start  before the job tries to use it. For example, your PostgreSQL container might be running, but might not be ready to accept connections. Work around this problem by using `dockerize` to wait for dependencies.
Following is an example of how to do this in your CircleCI `config.yml` file:

```
version: 2.0
jobs:
  build:
    working_directory: /your/workdir
    docker:
      - image: your/image_for_primary_container
      - image: postgres:9.6.2-alpine
        environment:
          POSTGRES_USER: your_postgres_user
          POSTGRES_DB: your_postgres_test
    steps:
      - checkout
      - run:
          name: install dockerize
          command: wget https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz && tar -C /usr/local/bin -xzvf dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz && rm dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz
          environment:
            DOCKERIZE_VERSION: v0.3.0
      - run:
          name: Wait for db
          command: dockerize -wait tcp://localhost:5432 -timeout 1m
```

It is possible to apply the same principle for the following databases:

- MySQL:

`dockerize -wait tcp://localhost:3306 -timeout 1m`

- Redis:

`dockerize -wait tcp://localhost:6379 -timeout 1m`

Redis also has a CLI available:

`sudo apt-get install redis-tools ; while ! redis-cli ping 2>/dev/null ; do sleep 1 ; done`

- Other services such as web servers:

`dockerize -wait http://localhost:80 -timeout 1m`


## PostgreSQL CircleCI Configuration Example
Following is a simple example `.circleci/config.yml` file configured for Ruby and Postgres.

{% raw %}
```
version: 2
jobs:
  build:
    working_directory: ~/myapp
    docker:
      - image: circleci/ruby:2.3-node
        environment:
          PGHOST: 127.0.0.1
          PGUSER: myapp-test
          RAILS_ENV: test
      - image: circleci/postgres:9.5-alpine
        environment:
          POSTGRES_USER: myapp
          POSTGRES_DB: myapp-test
          POSTGRES_PASSWORD: ""
    steps:
      - checkout

      - restore_cache:
          name: Restore bundle cache
          keys:
            - myapp-bundle-{{ checksum "Gemfile.lock" }}
            - myapp-bundle-

      - restore_cache:
          name: Restore yarn cache
          keys:
            - myapp-yarn-{{ checksum "yarn.lock" }}
            - myapp-yarn-

      - run:
          name: Bundle Install
          command: bin/bundle check --path vendor/bundle ||  bin/bundle install --path vendor/bundle --jobs 4 --retry 3

      - run:
          name: Yarn Install
          command: yarn install

      - save_cache:
          name: Store bundle cache
          key: myapp-bundle-{{ checksum "Gemfile.lock" }}
          paths:
            - vendor/bundle

      - save_cache:
          name: Store yarn cache
          key: myapp-yarn-{{ checksum "yarn.lock" }}
          paths:
            - ~/.yarn-cache

      - run:
          name: Rubocop
          command: bin/rubocop --rails

      - run:
          name: Wait for DB
          command: dockerize -wait tcp://localhost:5432 -timeout 1m

      - run:
          name: Database setup
          command: bin/rails db:schema:load --trace

      - run:
          name: Run tests
          command: bin/rails test
```
{% endraw %}

## Example CircleCI Configuration for a Rails App With structure.sql

If you are migrating a Rails app configured with a `structure.sql` file make
sure that `psql` is installed in your PATH and has the proper permissions, as
follows, because the circleci/ruby:2.4.1-node image does not have psql installed
by default and uses `pg` gem for database access.

{% raw %}
```
version: 2
jobs:
  build:
    working_directory: ~/circleci-demo-ruby-rails
    docker:
      - image: circleci/ruby:2.4.1-node
        environment:
          RAILS_ENV: test
          PGHOST: 127.0.0.1
          PGUSER: root
      - image: circleci/postgres:9.6.2-alpine
        environment:
          POSTGRES_USER: root
          POSTGRES_DB: circle-test_test
    steps:
      - checkout

      # Restore bundle cache
      - restore_cache:
          keys:
            - rails-demo-{{ checksum "Gemfile.lock" }}
            - rails-demo-

      # Bundle install dependencies
      - run:
          name: Install dependencies
          command: bundle check --path=vendor/bundle || bundle install --path=vendor/bundle --jobs 4 --retry 3

      - run: sudo apt install postgresql-client

      # Store bundle cache
      - save_cache:
          key: rails-demo-{{ checksum "Gemfile.lock" }}
          paths:
            - vendor/bundle

      - run:
          name: Database Setup
          command: |
            bundle exec rake db:create
            bundle exec rake db:structure:load

      - run:
          name: Parallel RSpec
          command: bin/rails test

      # Save artifacts
      - store_test_results:
          path: /tmp/test-results
```
{% endraw %}

An alternative is to build your own image by extending the current image,
installing the needed packages, committing, and pushing it to Docker Hub or the
registry of your choosing.

## Optimizing Postgres Images
The default `circleci/postgres` Docker image uses regular persistent storage on disk.
Using `tmpfs` may make tests run faster and may use fewer resources. To use a variant
leveraging `tmpfs` storage, just append `-ram` to the `circleci/postgres` tag (i.e., 
`circleci/postgres:9.5-alpine-ram`). 

PostGIS is also available and can be combined with the previous example:
`circleci/postgres:9.5-alpine-postgis-ram`