---
title: Notes on the use of Staging Environments on Heroku
category: Devops
---

The setup and use of a separate staging environment is covered (almost)
adequately by [Heroku's documentation on the matter](https://devcenter.heroku.com/articles/fork-app).

And yet I must be missing something, as forking my app didn't do what I
expected.

In forking a running production app to create a staging clone environment what I
expected to have afterwards was:

- A Heroku app called `appname-staging`, running the same version of the
  application
- A newly-created hobby-dev Postgres instance containing the production data
- That the staging app should be targeting the new staging database, i.e.
  `DATABASE_URL` for `appname-staging` should be set to the URL of the
newly-created database.

The staging app appeared to be working exactly as expected. There was a new
database which contained a copy of the production database, the staging app was
running the right version of the code at the appropriate URL, and it appeared to
be a perfect clone of production.

Unbelievably (for me at least), the `DATABASE_URL` of the new staging app was
set to point to the production database, including credentials. Staging was
masquerading as a perfect clone of production because it was in fact pointing to
the production database.

Here is the relevant portion of the Heroku documentation:

> It is recommended to make sure if you have an expected Heroku Postgres setup
> with your target app. Please run heroku pg:info and/or heroku config command
> to make sure that everything has copied as you expected. If the copied
> database is not being the primary database (DATABASE_URL), use heroku
> pg:promote as described by the Heroku Postgres documentation to make it a
> primary database.

It does not inspire confidence that this important caveat is not highlighted
more clearly, and the concepts involved not explained in more detail. In what
case would we ever want to fork an app, and have it pointing at the live
production database?

Another potentially important caveat, while we are *forking* the app, the
database will be *copied*, i.e. the database will not be *forked* using the
provided *fork* feature. This lack of clarity has likely tripped up enough
people for them to warrant adding another warning to the documentation page.

I have not found these issues discussed much (or at all) online, so I can only
assume that I am one of the few who has fallen foul of this. Still, if you are
experimenting with staging environments on Heroku then I would recommend
checking and double-checking the config of the new app (as they do indeed
suggest). Heroku reduces many complex hosting-related concerns to simple
commands and it's easy to get used to that, but it seems that not all of these
commands receive the same polish and attention.

## Resolving the issue

### The Situation

You have run Heroku's `fork` command and now have a clone of your production
app, but its `DATABASE_URL` is still pointing to that of production.

First, take a fresh backup of production before continuing:

`heroku pg:backups capture -a production-app-name`
``curl `heroku pg:backups public-url -a production-app-name` -o `date +5Y5m5d`.pgbackup``

With a backup safely stored, we look at the configuration of each application:

```
% heroku config -a production-app-name | grep URL
DATABASE_URL:               postgres://user_a:password@ec2-host1.eu-west-1.compute.amazonaws.com:5432/db_identifier1
HEROKU_POSTGRESQL_ROSE_URL: postgres://user_a:password@ec2-host1.eu-west-1.compute.amazonaws.com:5432/db_identifier1
% heroku config -a staging-app-name | grep URL
DATABASE_URL:               postgres://user_a:password@ec2-host1.eu-west-1.compute.amazonaws.com:5432/db_identifier1
HEROKU_POSTGRESQL_ROSE_URL: postgres://user_b:password@ec2-host2.eu-west-1.compute.amazonaws.com:5432/db_identifier2
```

Note that the staging app has a new `ROSE` database URL, and yet its
`DATABASE_URL` is still pointing directly at production.

In the remaining configuration items, new credentials have been automatically
set for New Relic, Papertrail and other add-ons, but for some reason not for the
all-important `DATABASE-URL`.

### The Solution

As per [Heroku's docs on the
matter](https://devcenter.heroku.com/articles/heroku-postgresql#establish-primary-db),
we need to *promote* the newly-created staging database to be the primary
database for the app:

```
% heroku pg:promote HEROKU_POSTGRESQL_ROSE_URL -a staging-app-name
Ensuring an alternate alias for existing DATABASE... done, not needed
Promoting postgresql-addon-id to DATABASE_URL on staging-app-name... done
```

We now confirm that the production configuration has not changed, and the
staging configuration now has `DATABASE_URL` pointing at the new staging
database, and **not** our production database:

```
% heroku config -a production-app-name | grep URL
DATABASE_URL:               postgres://user_a:password@ec2-host1.eu-west-1.compute.amazonaws.com:5432/db_identifier1
HEROKU_POSTGRESQL_ROSE_URL: postgres://user_a:password@ec2-host1.eu-west-1.compute.amazonaws.com:5432/db_identifier1
% heroku config -a staging-app-name | grep URL
DATABASE_URL:               postgres://user_b:password@ec2-host2.eu-west-1.compute.amazonaws.com:5432/db_identifier2
HEROKU_POSTGRESQL_ROSE_URL: postgres://user_b:password@ec2-host2.eu-west-1.compute.amazonaws.com:5432/db_identifier2
```
