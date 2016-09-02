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
