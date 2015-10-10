---
title: Robust Integration Testing in Rails 4 with RSpec, Capybara and Selenium
category: Rails
---

## The Problem

Using RSpec and Capybara to test your Rails apps can make writing tests easier
which is good both from a perspective of getting people to actually write tests
but also for our general productivity as developers.

While it has its detractors, I like the terse syntax and ease with which we can
define our own helpers which help set up certain contexts such as in this case
setting up the default account and switching into its subdomain:

```ruby
feature "Onboarding", :js do
  with_default_account do
    within_account_subdomain do
      before {
        Service.create_default_services
      }

      scenario "User can complete onboarding process" do
        visit dashboard_path

        # User is redirected
        expect(page.current_path).to eql(getting_started_path)

        fill_in "Name", with: "New Service XYZ"
        click_button "Create Service"

        # ...
      end
    end
  end
end
```

Those helpers can be used in any test, which eliminates repeated setup code
between test cases. Then from Capybara we get a simple but powerful DSL which
lets us `visit` pages, `fill_in` forms and interact as if we were a user
(`click_button`, `click_link`). This is an attractive mix and it isn't hard to
see why this has caught on.

The testing setup that ships with Rails is also perfectly capable but generally
I prefer this syntax and workflow. One great advantage that the standard testing
setup (using `Test::Unit`) has over RSpec however is **support**--quite simply
when you write your tests using `Test::Unit` and the default Rails helpers you
will be in good company and can expect minimum fuss. As the Rails core team
advance the test harness then you can also benefit from out-of-the-box
improvements like Spring, which will cut app startup time out of the test
running equation.

When you move to RSpec however, and start mixing in things like Capybara,
FactoryGirl (for fixtures) and Selenium then you are entering potentially
uncharted territory, in particular because your app may have something unusual
about it that causes your tests to fail, perhaps mysteriously.

One way to look at the argument between the two approaches is that if you work
with Rails on a contract/consulting basis then having familiarity with both is
going to be advantageous--if you work only on your own projects then the choice
is entirely up to you.

In this post I'm going to step through the several roadblocks that I have
encountered in setting up the above tools such that they all work together
correctly and reliably. In doing so I will attempt to keep the setup as simple
as possible, and minimise the use of blunt instruments like `database_cleaner`'s
truncation strategy.

*Note: Given how many moving parts there are here (the majority from 3rd
parties), for future projects I will be experimenting with sticking to the stock
Rails test framework to measure approximately the difference in productivity.
The tests may (or may not!) take longer to write, but I would expect some gains
from the reduction in lost time due to needing to fix the test harness.*

An additional complication that I have is that I am testing a subdomain-based
multi-tenant system--this is something that is not well documented when it comes
to RSpec and particularly Capybara so if you are in a similar situation then the
following should hopefully be of use to you.

The roadblocks I have hit are:

 * Database inconsistencies when running a mix of plain and Javascript/Selenium
   tests.
 * Assets (CSS/JS) not being served in Selenium tests.
 * The requesting of assets causing RSpec/Capybara to hang, with no hint as to
   what is wrong.

The setup:

 * rspec-rails 2.14.2
 * capybara 2.4.1
 * selenium-webdriver 2.42.0
 * factory_girl_rails 4.4.1
 * database_cleaner 1.3.0

The characteristics that we want the setup to have:

 * Running a test in a real browser should be as simple as specifying `:js` for
   a feature or scenario.
 * Tests should pass/fail reliably, not sporadically, i.e. there should be no
   race conditions.
 * All tests will be carried out using a single tenant--we are testing the
   application functionality; the multi-tenant aspect of the system will be
handled by a completely separate set of tests as mixing the two in one suite is
more trouble than it's worth.

## The Solution

### Gemfile

```ruby
group :development, :test do
  gem 'rspec-rails', '2.14.2'
  gem 'capybara', '2.4.1'
  gem 'selenium-webdriver'
  gem 'factory_girl_rails', '~> 4.0'
  gem 'database_cleaner', '1.3.0'
end
```

### Spec Helper

Here is a full spec helper file (which should live in `spec/spec_helper.rb`); I
would recommend reading the comments and incorporating the settings as
appropriate (i.e. don't copy and paste this whole file into your project):

```ruby
# This file is copied to spec/ when you run 'rails generate rspec:install'
ENV["RAILS_ENV"] ||= 'test'
require File.expand_path("../../config/environment", __FILE__)
require 'rspec/rails'
require 'rspec/autorun'
require 'capybara'

# Requires supporting ruby files with custom matchers and macros, etc,
# in spec/support/ and its subdirectories.
Dir[Rails.root.join("spec/support/**/*.rb")].each { |f| require f }

# Checks for pending migrations before tests are run.
# If you are not using ActiveRecord, you can remove this line.
ActiveRecord::Migration.check_pending! if defined?(ActiveRecord::Migration)

RSpec.configure do |config|
  # Remove this when upgrading to RSpec 3
  # Allows us to write just `:js` instead of `js: true` in tests
  config.treat_symbols_as_metadata_keys_with_true_values = true

  # Remove this line if you're not using ActiveRecord or ActiveRecord
fixtures
  config.fixture_path = "#{::Rails.root}/spec/fixtures"

  # If you're not using ActiveRecord, or you'd prefer not to run each of
your
  # database_cleaner below
  config.use_transactional_fixtures = false

  # If true, the base class of anonymous controllers will be inferred
  # automatically. This will be the default behavior in future versions of
  # rspec-rails.
  config.infer_base_class_for_anonymous_controllers = false

  # Run specs in random order to surface order dependencies. If you find an
  # order dependency and want to debug it, you can fix the order by
providing
  # the seed, which is printed after each run.
  #     --seed 1234
  config.order = "random"

  # Insist on 'expect' syntax rather than 'should'
  config.expect_with :rspec do |c|
    c.syntax = :expect
  end

  config.before(:suite) do
    # First, in case the database contains data from a previous run (e.g.
    # from a run that crashed), run a full clean using the truncation
    # strategy.
    DatabaseCleaner.clean_with(:truncation)

    # *** The following is specific to this project
    #     Here we set up the default tenant to be used in each test

    # Drop the default tenant if it exists
    Apartment::Database.drop(PracticeManager::DEFAULT_TENANT_NAME) rescue
nil

    # Create the default tenant
    default_account = Subscriptions::Account.create!(name:"Test Customer
#1", subdomain: PracticeManager::DEFAULT_TENANT_NAME)
    default_account.create_schema

    owner = Subscriptions::User.create!(email: "customer@example.com",
password: "password", password_confirmation: "password")

    default_account.owner = owner
    default_account.save!

    default_account.users << default_account.owner

    # *** End of project-specific portion
  end

  config.before(:each) do
    # If the test is a Javascript test, set the strategy to truncation
    # as transactional cleaning will not work due to the test runner
    # and app not sharing the same process when testing from a browser.
    # For non-Javascript tests use the transaction strategy as it is faster.
    if example.metadata[:js]
      DatabaseCleaner.strategy = :truncation
    else
      DatabaseCleaner.strategy = :transaction
    end

    DatabaseCleaner.start

    # Before each test, switch into the schema of the default tenant
    Apartment::Database.switch(PracticeManager::DEFAULT_TENANT_NAME)
  end

  config.after(:each) do |group|
    DatabaseCleaner.clean
    Apartment::Database.reset
  end

  # Load FactoryGirl helpers
  config.include FactoryGirl::Syntax::Methods
end

# Explicitly set the test server process to a particular port
# so that we can access it directly at will.
Capybara.server_port = 10000

# To ensure that browser tests can find the test server process,
# always include the port number in URLs.
Capybara.always_include_port = true

# For all tests except Javascript tests we will use :rack_test
# (the default) as it is the fastest. For Javascript tests we will
# use Selenium as it is the most robust/mature browser driver
# available.
Capybara.javascript_driver = :selenium
```

#### Database Cleaner

The `database_cleaner` gem feels intuitively like an unpleasant hack, but with
the right configuration it can work in such a way that we only change the test
runner behaviour slightly for JS tests.

It is well known that once you are using Factory Girl and/or browser testing
then you will want to stop using transactional fixtures as they won't work as
expected:

```ruby
config.use_transactional_fixtures = false
```

Then, when cleaning up after tests it is common to use the cleaner's
`truncation` strategy after browsers tests and the `transaction` strategy
(basically the same as that enabled by `config.use_transactional_fixtures`) for
all other tests.

Many tutorials and StackOverflow answers out there have JS and non-JS tests
mixed together, with the strategy being switched between `transaction` and
`truncation` on each test.

In my experience this has resulted in database access race conditions that have
caused otherwise well-written, independent tests to fail intermittently.

The solution I have settled on is to run all non-JS tests first (shuffled), and
then all JS tests (also shuffled). This allows for discovery of tests that
incorrectly expect certain state, or fail to clean up after themselves (by
virtue of the random execution order), while not attempting to freely mix JS and
non-JS tests. As these different classes of test have different purposes, I see
no disadvantage in this approach, with the exception of it being somewhat
non-standard.

In RSpec 3 we would be able to provide our own ordering scheme to achieve this,
but as we are still on RSpec 2 this needs to be set up outside of the spec
helper.

First, in `.rspec` we set the default runner to exclude all Javascript tests:

    --color --tag ~js

*(We also colour all output)*

Now when we run `rspec` in the shell it will run all tests not marked with
`:js`.

To run our browser tests, we now need to run `rspec --tag js`. While this could
perhaps be seen as inconvenient, I find this preferable as generally I am not
looking to run the (very slow) browser tests unless I am working on a
Javascript-based feature, or I am committing code.

I have a git pre-commit hook which handles the running of both suites of tests,
and only allows the commit to go ahead if both suites pass.

In a file symlinked to `.git/hooks/pre-commit`:

```bash
#!/bin/sh

# Based on
http://codeinthehole.com/writing/tips-for-using-a-git-pre-commit-hook/

clean_up_and_exit () {
  git stash pop -q
  exit $1
}

# Run all tests and ensure that they pass before continuing
git stash -q --keep-index

rspec --tag ~js
RESULT=$?

[ $RESULT -ne 0 ] && clean_up_and_exit 1

rspec --tag js
JS_RESULT=$?

[ $JS_RESULT -ne 0 ] && clean_up_and_exit 1

# All tests passed
clean_up_and_exit 0
```

So in general I'll be running individual test files as I work, and then before
committing I will be sure to exercise the whole test suite.

#### Why Selenium?

Initially I was using the
[capybara-webkit](https://github.com/thoughtbot/capybara-webkit) gem to allow
for headless testing but unfortunately ran into some sporadic errors/hangs which
were hard to nail down.

Running the tests without a live browsers instance is both faster, easier to
integrate with a CI server and has the advantage of not popping up windows on
your machine when testing. That said, as my primary goal here is to set up the
most reliable test harness I have gone for Selenium as the driver as it is a
more mature offering by virtue of it having been in production use for longer.

#### FactoryGirl

FactoryGirl makes generating test data trivially easy, but it also makes tests
unpredictable if the cleanup of this data is not handled appropriately.

By completely separating our JS and non-JS test suites, and using the
`truncation` strategy for DatabaseCleaner on JS tests we get the best of both
worlds: JS tests are run cleanly with no database-related race conditions, and
all other tests run as quickly as possible using the `transaction` strategy.

#### Multi-tenant / Subdomain Testing

The multi-tenant aspect of the system is provided by a Rails engine,
custom-built for the purpose of managing accounts, users, authentication and
scoping of data based on the current subdomain. The engine has its own test
suite which exercises this logic and tests the data scoping in particular. The
test suite runs on `rack_test` only.

When testing, FactoryGirl is used to create multiple accounts, each with their
own subdomain and database (or schema, in the case of Postgres).

Initially when testing the behaviour of the host application I continued with
this approach. Now however, to greatly simplify the testing process, all tests
**for the host app** (and not the engine) run against a single default tenant.

One of the key reasons for doing this is to avoid needing to resolve potentially
hundreds of subdomains to `localhost` when running browser tests. In `rack_test`
it doesn't matter that `customer20.example.com` doesn't resolve as it doesn't
resolve names by DNS in the first place.

When using Selenium however the address must resolve. Setting up a local DNS
server like `dnsmasq` is one option, but now your development and test
environments need to have this external software installed and running. An
alternative could be to use [Pow](http://pow.cx), which I am already using for
development, to automatically resolve all subdomains at an appropriate `.dev`
domain.

While both approaches work, both would behave erratically on occasion. The
solution in the case of Pow was to upgrade to the latest version, but this
highlighted the fragility of the approach. With `dnsmasq` again you have a
fragile solution that could break the next time you update your OS, making it
undesirable too.

By sticking to the one tenant in all tests, all that needs to be done to have
browser tests work is to map the one tenant domain to `localhost` in
`/etc/hosts`, e.g. `customer1.yourappname-test.com`.

To make this work in your Capybara tests, you can define a helper function that
will temporarily override `Capybara.app_host` to point all requests at this
domain.

For example in `lib/testing_support/subdomain_helpers.rb` I have:

```ruby
module TestingSupport
  module SubdomainHelpers
    def within_account_subdomain
      before {
        if Capybara.current_driver != :rack_test
          Capybara.app_host =
"http://#{account.subdomain}.yourappname-test.com"
        else
          Capybara.app_host = "http://#{account.subdomain}.example.com"
        end
      }

      after { Capybara.app_host = "http://www.example.com" }

      yield
    end

      RSpec.configure do |config|
      config.extend SubdomainHelpers, :type => :feature
      end
  end
end
```

So now on each test invocation, `Capybara.app_host` will be set appropriately
and the correct tenant (which in our case is always the default tenant) will be
targeted.

**Note:** While this approach works and has been reliable, the author of
Capybara does not recommend or explicitly support the setting of
`Capybara.app_host` more than once. We can alternatively set this value just
once, in our spec helper, and it will work fine with the caveat that we will
truly be locked to using just the one tenant in our test suite.

The relevant RSpec settings from the spec helper are:

```ruby
# Explicitly set the test server process to a particular port
# so that we can access it directly at will.
Capybara.server_port = 10000

# To ensure that browser tests can find the test server process,
# always include the port number in URLs.
Capybara.always_include_port = true
```

With the port set and Capybara ensuring that every request includes the port, we
can be sure that all browser tests will hit
`http://customer1.yourappname-test.com:10000`.

#### Asset Generation and Serving in Tests

Chances are, with your stock Rails/RSpec/Capybara setup, when you call
`save_and_open_page` in a feature spec you will be presented with a completely
unstyled version of your app, as both the CSS and JS asset creation and serving
will be broken.

This isn't particularly important when you're testing non-Javascript interaction
with your app but as soon as you get into browser testing then this will be an
issue for two key reasons:

 * If Javascript from the asset pipeline is not served then it will of course
   not be available to test.
 * The stock asset settings for the test environment can cause RSpec/Capybara to
   hang (requiring `kill -9` to halt the process).

Searching online you will find various solutions, such as precompiling all
assets on each test run and serving them using `file://` URLs pointing to the
`public` directory of your Rails app.

The simplest approach that I have found, and which works well, is simply to set
`config.assets.debug = true` in `config/environments/test.rb`.

Doing so causes assets to be generated and served just as if you were using the
development server; all assets (JS and CSS) should now be generated and served
correctly.

## Wrapping Up

This is more than likely a work in progress. While the combination of RSpec,
Capybara, FactoryGirl and Selenium have provided a welcome productivity boost on
the development side, they have also caused a great deal of time to be spent on
getting them all to work together reliably, particularly in a multi-tenant
setting.

Your mileage may vary but I hope these notes will help you to avoid some of the
less-well-documented pitfalls, and so spend more time on actually shipping
features.

As previously mentioned, I will be taking an in-depth look at the stock Rails
testing framework and methodologies again in the near future. While I appreciate
the neatness of these various 3rd party libraries that have generally made me
more productive, Rails is always on the move and the more moving parts we have
that must keep up, the more points of failure we have, and so the more time we
spend debugging issues that have nothing to do with shipping features and
providing value to users and customers.

I think it's valuable to keep going back to the *point* of testing, which is (in
my mind) to ship reliable software and to make refactoring safer, so that we can
always be working to reduce the technical debt that we normally incur when
evolving software over time. While tools like RSpec and FactoryGirl seem on the
surface to be neat solutions to real problems, it is perhaps the case that
sticking with the standard Rails testing methods will better serve that goal.
