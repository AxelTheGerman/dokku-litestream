# dokku-litestream

[Litestream](https://litestream.io) plugin for [Dokku](https://dokku.com) that allows easy replication and restoration of SQLite!

*Note this plugin is pre-release*

## Getting Started

There are only a few steps to get started. This guide assumes you have a Dokku instance up and running and a SQLite based application to deploy.

```
sudo dokku plugin:install https://github.com/axelthegerman/dokku-litestream litestream
```

The following guide is based on a Ruby on Rails application but should apply to most other languages and web frameworks as well. Find the sample application source code at [AxelTheGerman/dokku-litestream-example-rails](https://github.com/AxelTheGerman/dokku-litestream-example-rails).

### Step 0: Why SQLite

First of all, a few quick thoughts on using SQLite... SQLite is a very capable database for most web applications. It allows us to keep our stack simple by avoiding client/server based database systems. Additionally by running in-process the latency can be much lower than other DBMS.

Still not sure? Maybe read [this](https://fly.io/blog/all-in-on-sqlite-litestream/#you-should-take-this-option-more-seriously), [this](https://antonz.org/sqlite-is-not-a-toy-database/), [this](https://unixsheikh.com/articles/sqlite-the-only-database-you-will-ever-need-in-most-cases.html) or [this](https://blog.wesleyac.com/posts/consider-sqlite).

### Step 1: SQLite up your Application

In order to run a SQLite based application on Dokku we most likely need to add the SQLite package to the application's buildpack. A buildpack is what builds the application environment. Alternatively applications can be deployed via Dockerfile - in that case make sure your docker image provides SQLite.

Create a `.buildpacks` file at the application root with the following content:

```
heroku-community/apt
heroku/ruby
```

The [apt buildpack](https://elements.heroku.com/buildpacks/heroku/heroku-buildpack-apt) allows us to install additional system packages for our application.

Create a `Aptfile` file at the application root with the following content:

```
libsqlite3-0
libsqlite3-dev
sqlite3
```

By the nature of SQLite, which is stored on local disk and runs in-process, we will run into problems when deploying the application as is to Dokku. The reason is that every deploy will create a new Docker image with a new filesystem and a new database. Also e.g. database migrations are running in a different process or container compared to the application and will not actually migrate your application container's database.

Configure persistent storage for you Dokku application:

*On your Dokku host:*
```
dokku storage:ensure-directory my-sqlite-application my-sqlite-application--db
dokku storage:mount my-sqlite-application /var/lib/dokku/data/storage/my-sqlite-application--db:/app/db/litestream
```

**Important: Do not use the /app/db directory directly as it contains other important database files that will otherwise be erased.**

Now this directory needs to be the location for your application's SQLite database, e.g. in Rails update the `config/database.yml` as so:

```
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: db/development.sqlite3

test:
  <<: *default
  database: db/test.sqlite3

production:
  <<: *default
  database: db/litestream/production.sqlite3
```

*Note: Use the path from the previous step (without the /app prefix)*

Now deploy your application and it should be fully functioning and persist the database across deployments and application restarts.

### Step 2: Litestream for Database Replication

The nature of Dokku being a single host system can cause massive problems for your data reliability and integrity needs. Even if the persistent storage locations are backed up, some SQLite database transactions might be lost when anything happens to the Dokku host.

That's where [Litestream](https://litestream.io) comes in as it automatically replicates SQLite databases to a remote replica - this would usually be AWS S3 or any compatible service.

Install the dokku-litestream plugin on your Dokku host:

```
sudo dokku plugin:install https://github.com/axelthegerman/dokku-litestream litestream
```

Authorize dokku-litestream to access your remote storage service:

```
dokku litestream:auth
=====> Authorizing litestream for replica location
Enter your ACCESS_KEY_ID:    <paste your access key ID here>
Enter you SECRET_ACCESS_KEY: <paste your secret access key here>
       Saving credentials
       Credentials saved
```

Create a litestream replication for you application database:

```
dokku litestream:create my-sqlite-application /var/lib/dokku/data/storage/my-sqlite-application--db production.sqlite3 s3://my-backup-bucket/my-sqlite-application
```

The parameters for the above command are:

1. Application name - `my-sqlite-application`
2. Local storage location (the storage directory that is mounted to your application) - `/var/lib/dokku/data/storage/my-sqlite-application--db`
3. The database filename for your application - `production.sqlite3`
4. The replica storage URL - `s3://my-backup-bucket/my-sqlite-application` (It is recommended to use your application name as top level backup directory. This allows for storing multiple replicas in a single bucket.)

Start the replication:

```
dokku litestream:start my-sqlite-application
```

Check the litestream replication logs:

```
dokku litestream:logs my-sqlite-application
```

### Step 3: Profit

Alright the previous setup is a bit involving but it's definitely worth it!

Now go ahead, build great things and see how far SQLite scales with your application (I bet further than you'd think).

## Restoring a replica

Sooner or later you will need to restore your replica - try it out once or twice so your confident to know how to when it counts!

Most likely the replica will be restored into a new copy of your application, e.g. when moving/ restoring the Dokku host server. But a replica can be restored anywhere on the local Dokku host file system.

Assuming the following:

- local target directory is `/var/lib/dokku/data/storage/copy-my-sqlite-application--db`
- database name for the application is `production.sqlite3`
- replica is stored at `s3://my-backup-bucket/my-sqlite-application`

Run the following command:

```
dokku litestream:restore /var/lib/dokku/data/storage/copy-my-sqlite-application--db production.sqlite3 s3://my-backup-bucket/my-sqlite-application
```

If you get the following error: `cannot restore, output path already exists: /data/production.sqlite3` your target folder already has a database with the same name and litestream will not overwrite it. Please delete the existing file(s), rename the database file or use a different target directory.
