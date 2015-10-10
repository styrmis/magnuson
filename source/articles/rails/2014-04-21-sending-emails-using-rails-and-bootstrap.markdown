---
title: Sending Emails Using Rails and Bootstrap
category: Rails
---

If you've built your site's look and feel using Bootstrap and you're now looking to start sending emails with similar styling then as you are no doubt aware, email clients are generally extremely limited in their understanding of CSS in particular and so we're going to have to do a bit of work.

The topic of email HTML and CSS rendering is a complex one with an increasing number of caveats as you try to support more email clients. Some common client sins include:

 * Stripping out the `<head>` tag completely.
 * Not loading any linked stylesheets; all CSS must be inline, e.g. `<div style="color: #444444; text-align: center;">` etc.
 * Not understanding any positional CSS, instead requiring tables within tables for layout.
 * Generally behaving like a browser in the 90's.

We'll use `premailer` to convert our linked CSS declarations into inline CSS and `letter_opener_web` to make previewing changes a little easier, then we'll try integrating some parts of Bootstrap in using `less`.

## Letter Opener Web

It would be good to have a way to send emails locally, not worry about accidentally sending to a real person (e.g. a user) and preview the results easily. The `letter_opener` gem by Ryan Bates works well for this, but it would be nicer to be able to easily browse the emails within the app rather than only rely on them launching in a browser on each send. The `letter_opener_web` gem extends `letter_opener` with such an interface, so let's install it.

Add to `Gemfile`:

```ruby
gem 'letter_opener_web', '~> 1.2.0', :group => :development
```

Configure in `config/environments/development.rb`:

If you want emails to open in your browser on send (using the `launchy` gem) and be browsable via the web interface:

```ruby
config.action_mailer.delivery_method = :letter_opener
```

If you only want the emails to be visible in the web interface (I prefer this):

```ruby
config.action_mailer.delivery_method = :letter_opener_web
```

Add to `config/routes.rb`:

```ruby
if Rails.env.development?
  mount LetterOpenerWeb::Engine, at: "/devel/emails"
end
```

Now if you visit `/devel/emails` you should see the web interface; send some emails and they should appear here:

<img src="http://stefan.haflidason.com/assets/2014/04/Screen-Shot-2014-04-21-at-15.10.50.png" alt="Screen Shot 2014-04-21 at 15.10.50" width="782" height="290" class="aligncenter size-full wp-image-684" style="border: 1px solid #444;" />

## Premailer

### Installation

Add `nokogiri` and `premailer-rails` to your `Gemfile`:

```ruby
gem 'nokogiri'
gem 'premailer-rails'
```

(and run `bundle install`)

Premailer will hook in to the email rendering process of `ActionMailer` automatically.

## Testing it Out

To test that it's working as expected, we'll just define a couple of simple classes in a new CSS file, `app/assets/stylesheets/emails.css.less`:

```css
.red {
  color: red;
}

.bold {
  font-weight: bold;
}
```

And then in our email view template (`app/views/notifier/daily_summary.html.haml`), we'll link in the CSS file and make use of the classes:

```haml
!!!
%html
  %head
    %meta(http-equiv="Content-Type" content="text/html; charset=UTF-8")
    =stylesheet_link_tag 'emails.css'
  %body
    %p.red.bold A test email
```

We send this email via the console:

```ruby
pry(main)> Notifier.daily_summary.deliver
```

And the resulting `<p>` tag has the classes defined as expected, including the relevant declarations pulled in as inline styling:

```html
<p class="red bold" style="color: red; font-weight: bold">A test email</p>
```

Inspecting the email, we see that this is basically working, though it isn't a proper test yet as we are using a modern browser and not an email client.

## Incorporating Bootstrap

I'll assume that you already have Bootstrap incorporated into your asset pipeline using the source less files. Using SCSS would work fine too; what we want to do is to import the parts of Bootstrap that would be useful for styling our emails and this could even be achieved using static CSS, though I wouldn't recommend it.

We start by importing some Bootstrap less files into `emails.css.less`:

```css
// Core variables and mixins
@import "twitter/bootstrap/variables.less";
@import "twitter/bootstrap/mixins.less";

// Theme variables and mixins
@import "theme/mixins.less";
@import "theme/variables.less";

// Reset
@import "twitter/bootstrap/normalize.less";

// Core CSS
@import "twitter/bootstrap/scaffolding.less";
@import "twitter/bootstrap/type.less";
@import "twitter/bootstrap/code.less";
@import "twitter/bootstrap/grid.less";
@import "twitter/bootstrap/tables.less";
@import "twitter/bootstrap/forms.less";
@import "twitter/bootstrap/buttons.less";

// Components
@import "twitter/bootstrap/labels.less";
@import "twitter/bootstrap/badges.less";
@import "twitter/bootstrap/panels.less";
```

We've imported the foundational files and a few components; at this stage it isn't clear what components are going to work with this setup, in email clients or otherwise.

Then we extend our email template to make use of some Bootstrap classes:

```haml
!!!
%html
  %head
    %meta(http-equiv="Content-Type" content="text/html; charset=UTF-8")
    =stylesheet_link_tag 'emails.css'
  %body
    %h2 Tables

    %table.table
      %thead
        %tr
          %th Column 1
          %th Column 2
      %tbody
        %tr
          %td 1 / 1
          %td 1 / 2
        %tr
          %td 2 / 1
          %td 2 / 2

    %h2 Links

    %a.btn.btn-primary(href="#")
      Link Styled as Button

    %br
    %br

    %a
      Link with badge
      %span.badge 42

    %br
    %br

    %h2 Panels

    .panel.panel-primary
      .panel-heading
        Panel Heading
      .panel-body
        Panel Body

        .label.label-primary
          Primary Label

        .label.label-warning
          Warning Label

        .label.label-danger
          Danger Label
```

## Results

<img src="http://stefan.haflidason.com/assets/2014/04/Screen-Shot-2014-04-21-at-16.32.33.png" alt="Screen Shot 2014-04-21 at 16.32.33" width="900" height="640" class="aligncenter size-full wp-image-694" />

Some components simply aren't going to work at all, so I'm expecting that the main benefit will be access to variables and mixins. This will help reduce code duplication and it won't add quite as much weight as including the rest of the framework, so overall I would recommend getting as much mileage from simple HTML/CSS and the variables/mixins before spending too much time trying to get components to render properly.

Trying the resulting HTML in various email clients (via [Litmus](https://litmus.com/pub/d831ecc)) results in reasonable results for a first attempt; some clients are nearly there, though some are quite broken. With tweaking this may be viable, though the recommendation to not spend too much time on the components still stands.

<a href="https://litmus.com/pub/d831ecc" target="_blank"><img src="http://stefan.haflidason.com/assets/2014/04/Screen-Shot-2014-04-21-at-16.49.37.png" alt="Screen Shot 2014-04-21 at 16.49.37" width="939" height="448" class="aligncenter size-full wp-image-695" /></a>

In order to get going quickly I'll be starting with one of the battle-tested templates that [Mailchimp has open sourced](https://github.com/mailchimp/email-blueprints). From there it will simply be a case of carefully linking up the Bootstrap style variables (e.g. colours) with the provided HTML. As long as no drastic changes are introduced, and the automatic inlining done by Premailer doesn't break anything, then we should get widely-compatible email templates for our Rails app with a relatively small time investment.
