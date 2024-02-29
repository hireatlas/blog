---
slug: how-to-use-multiple-databases-with-laravel-vapor-extended
#issue: how-to-use-multiple-databases-with-laravel-vapor-extended
title: How to use multiple databases with Laravel Vapor
subtitle: Add a second, third, or more databases to your Laravel Vapor project.
description: Learn how to add multiple databases to your Laravel Vapor project.
date: 2024-02-12
category: guide
authors:
  - mitchell-davis
published: false
---

We love [Laravel Vapor](https://vapor.laravel.com/), and we use it on [most of the projects](/tech-stack) we work on
here at Atlas.

One question we often see from the community and from clients is how do you manage databases, and whether it's possible
to connect to multiple databases from your Laravel Vapor project.

Vapor makes it [very easy](https://docs.vapor.build/resources/databases.html) to create new MySQL and Postgres databases
under your AWS account, and add users, proxies, and alarms. This article will guide you through the process of setting
up one, two, or more databases and connecting to them from Vapor.

{% tldr %}
You can only configure your project to use a single database natively via Vapor's manifest files.

To get around this, create a new database via Vapor, and then manually set some new environment variables to match the
second database.

```dotenv
SECOND_DB_HOST=second-database.c1fnkwpphvul.ap-southeast-2.rds.amazonaws.com
SECOND_DB_PORT=3306
SECOND_DB_DATABASE=vapor
SECOND_DB_USERNAME=vapor
SECOND_DB_PASSWORD=oW8VAxaYx8zQyzPgdlfUVrpACvkIesbq9pOAF4OW
```

{% /tldr %}

## Creating a project with Laravel Vapor

To get started, let's create a new project in the Vapor console. The onboarding flow for setting up a new project is
nothing short of incredible, and once you connect your GitHub account, Vapor will:

- Create a new repo for you.
- Install Laravel and the Laravel Vapor CLI.
- Configure the GitHub Actions workflow for automatic deployments.
- Add your repository secrets.

![Creating the GitHub repository via Laravel Vapor](/assets/articles/how-to-use-multiple-databases-with-laravel-vapor/new-project.png)
![Tracking the progress of the deployment](/assets/articles/how-to-use-multiple-databases-with-laravel-vapor/deploying-new-project.png)

Once the GitHub Actions workflow finishes, your project will be deployed to a vanity URL which you can find on the Vapor
console, at the top of the page near your project name:

![The vanity URL](/assets/articles/how-to-use-multiple-databases-with-laravel-vapor/vanity-url.png)

Following that link takes us to a fresh Laravel application:

![So fresh and so clean](/assets/articles/how-to-use-multiple-databases-with-laravel-vapor/fresh-homepage.png)

Now head to your GitHub account, and you'll find your newly created repository. Clone this to your local machine, so you
can start working on the project.

Let's take a look at the `vapor.yml` manifest file in the root of your project, and notice that it **does not** contain
any references to databases.

```yaml
id: 57633
name: multiple-databases
environments:
  production:
    memory: 1024
    cli-memory: 512
    runtime: 'php-8.2:al2'
    build:
      - 'COMPOSER_MIRROR_PATH_REPOS=1 composer install --no-dev'
      - 'php artisan event:cache'
      # - 'npm ci && npm run build && rm -rf node_modules'
```

To understand what's happening on Lambda, where your Vapor project is ultimately being hosted, login to the AWS console
and head to the Lambda console in the region that you deployed your application in.

The functions there represent the different entry points into your application:

- The function ending in `-cli` is where all of your Artisan commands will be invoked via Vapor, if you run them using
  the Vapor console or CLI.
- The function ending in `-queue` is where all of your queued jobs will be invoked whenever a job is queued via SQS.
- The function without a suffix is where all of your HTTP requests will be invoked.

Click into any of these functions, and then click on the Configuration tab, and then the Environment variables tab.

These environment variables should be familiar from any time you've looked at your `.env` file. Again, note that there
are no `DB_*` environment variables.

![Where Vapor configures your environment variables](/assets/articles/how-to-use-multiple-databases-with-laravel-vapor/initial-environment-variables.png)

## Adding your first database to the Laravel Vapor project

It's time to add a new database from the Vapor console. Head to the [Databases](https://vapor.laravel.com/app/databases)
page, and click the Create Database button.

If it doesn't exist, first head to the [Networks](https://vapor.laravel.com/app/networks) page, and create your network
in the region you wish to deploy your project to, and then return and create the database.

Here I have named the database `first-database`, and set it to use a very small instance size since this is just for
demonstration.

I have also made the database public for demonstration purposes, but I would normally **recommend using a private
database** and paying the extra ~$32 / month for the NAT gateway to keep your network private.

![Creating the first database](/assets/articles/how-to-use-multiple-databases-with-laravel-vapor/creating-the-first-database.png)

Once submitted, Vapor will give you a set of master credentials, which you should save securely now.

![The first database master credentials](/assets/articles/how-to-use-multiple-databases-with-laravel-vapor/first-database-master-credentials.png)

After a few minutes the database will become available, and because it is public, you'll be able to connect to it from
your local machine without using a jumpbox.

In your local `.env`, set the following environment variables using the details from the Vapor console and your master
credentials.

```dotenv
DB_CONNECTION=mysql
DB_HOST=first-database.c1fnkwpphvul.ap-southeast-2.rds.amazonaws.com
DB_PORT=3306
DB_DATABASE=vapor
DB_USERNAME=vapor
DB_PASSWORD=oW8VAxaYx8zQyzPgdlfUVrpACvkIesbq9pOAF4OW
```

To keep this guide as focused as possible, we are just going to add a single model called `PageView`, and then have
multiple instances of it in each database. You can generate the model and database migration using this command:

```shell
php artisan make:model -m PageView
```

You can leave the model and migration as-is, as we don't need to look at anything other than the `COUNT(*)` of each
table for this demonstration.

Run the migration using this command:

```shell
php artisan migrate
```

We are going to interact with the application exclusively using simple HTTP routes for demonstration purposes, so inside
our `routes/web.php`, remove the default `welcome` route and instead replace it with the following.

```php

```
