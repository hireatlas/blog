---
slug: how-to-use-multiple-databases-with-laravel-vapor
#issue: how-to-use-multiple-databases-with-laravel-vapor
title: How to use multiple databases with Laravel Vapor
subtitle: Add a second, third, or more databases to your Laravel Vapor project.
description: Learn how to add multiple databases to your Laravel Vapor project.
date: 2024-02-13
category: guide
authors:
  - mitchell-davis
published: false
---

We love [Laravel Vapor](https://vapor.laravel.com/), and we use it on [most of the projects](/tech-stack) we work on
here at Atlas.

One question we often see from the community and from clients is how do you manage databases, and whether it's possible
to connect to multiple databases from your Laravel Vapor project.

While Vapor makes it [very easy](https://docs.vapor.build/resources/databases.html) to create new MySQL and Postgres
databases under your AWS account, and add users, proxies, and alarms, you can only configure your project to use a
single database **natively** via Vapor's manifest files.

To get around this, you can manually set some new environment variables for any non-primary databases.

```dotenv
SECOND_DB_HOST=second-database.abc123.ap-southeast-2.rds.amazonaws.com
SECOND_DB_PORT=3306
SECOND_DB_DATABASE=schema_for_second_database
SECOND_DB_USERNAME=username_for_second_database
SECOND_DB_PASSWORD=password_for_second_database

THIRD_DB_HOST=third-database.abc123.ap-southeast-2.rds.amazonaws.com
THIRD_DB_PORT=3306
THIRD_DB_DATABASE=schema_for_third_database
THIRD_DB_USERNAME=username_for_third_database
THIRD_DB_PASSWORD=password_for_third_database

# Continue this for all of your other non-primary databases
```
