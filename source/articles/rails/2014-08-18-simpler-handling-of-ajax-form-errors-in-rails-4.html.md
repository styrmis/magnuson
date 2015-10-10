---
title: Simpler Handling of AJAX Form Errors in Rails 4
category: Rails
---

## The Problem

With unobtrusive Javascript, converting a form in Rails to use AJAX is as easy as setting `remote: true`:

```haml
= form_for Client.new, remote: true do |form|
  = form.text_field :name
```

This will cause our form to submit asynchronously, and expect a Javascript response, which will be executed directly to e.g. update the page appropriately.

[DHH recommends](https://signalvnoise.com/posts/3697-server-generated-javascript-responses) that we design our apps in this manner, where in the Javascript response we re-render the model using its template and use jQuery to update the HTML on the page.

This method has the advantage of using the same template for both initial page generation and updates, which if used carefully can result in an easy and unobtrusive way to make your app allow for asynchronous interaction.

One downside however is that every page containing a form for this action which uses `remote: true` must be happy to accept the same Javascript response, i.e. that Javascript needs to make sense in all contexts. We could start to put conditional logic in the Javascript response to handle other cases but this would get unwieldy quickly--overall it would seem best to only use this approach when the model in question is to always be represented in the same manner, with the same kind of interaction, wherever it appears.

Rails was initially extracted from Basecamp and [DHH generally evaluates new Rails development from the perspective of what it will mean for Basecamp](http://david.heinemeierhansson.com/2012/the-parley-letter.html). This pattern of Server-generated Javascript Responses being used to enable remote forms is a pattern that fits Basecamp well. If your app is very similar to Basecamp then not only will this pattern likely serve you well, the rest of Rails will too.

In my case, I am currently working on a system that doesn't fit so neatly into the Basecamp mould. In today's example I have a case where on one page I want to show the full form for the creation of a Client object (with around 10 fields), and on another page a minimal form with only 4 fields. Both forms should point to the same `create` action, which is to be the sole point of Client creation whether we are POSTing data synchronously, using AJAX or communicating via the API.

As such, rather than return Javascript (which is the default), I want the `create` action to return JSON.

As [DHH notes](https://signalvnoise.com/posts/3697-server-generated-javascript-responses), there is a duplication of effort when working with JSON in that you write your template once on the server side and then again on the client side. In this case, this decoupling is desirable to allow for greater flexibility of presentation on the client side.

This is straightforward to achieve, **with the exception of handling form errors**, which is the topic of this post.

By default, when a JSON `create` request fails to create the entity in question, errors are returned in the form of field/message pairs, e.g.:

    {"name":["can't be blank"],"year_end":["can't be blank"],"date_of_incorporation":["can't be blank"]}

What I want is for those error messages to be applied to the appropriate form fields automatically, highlighting them in red and with the error message in the right place.

I would be happy to discover that this is already catered for in Rails but it appears to be something that developers are generally rolling on their own.

In this post I'll present a relatively general way to handle the mapping of JSON errors to input fields for any form in your Rails application that is tied to a model.

### A Little Background

Regarding the adding of `remote: true` to a form, all that this does is to add `data-remote="true"` to the form HTML:

```html
<form accept-charset="UTF-8" action="/clients" class="new_client" data-remote="true" id="new_client" method="post">
```

This in itself doesn't cause a form to submit asynchronously, but this data attribute will cause Rails' unobtrusive Javascript to bind to the submit function on page load, and use jQuery to submit the form data using an AJAX request.

The unobtrusive part comes naturally from the fact that if Javascript is not available on the client for whatever reason, the form will submit synchronously and your app will return an HTML response instead of Javascript or JSON.

## Implementation

*Note: In this example I am using Bootstrap but the technique will work no matter what framework you use (if any), you will just need to change the CSS classes in the code samples.*

In this example we are creating a Client and the form has four fields. I'm using an alternative form helper [rails-bootstrap-forms](https://github.com/bootstrap-ruby/rails-bootstrap-forms). All this does is apply bootstrap CSS classes so everything in this post will work fine with the regular form helper (or others) if you tweak the CSS classes accordingly.

### The Form

```haml
= bootstrap_form_for Client.new, remote: true do |form|
  = form.text_field :name
  = form.text_field :company_registration_no, label: "Company Registration No."
  = form.date_select :date_of_incorporation, start_year: 1980, end_year: Date.today.year, include_blank: true, default: nil, label: "Date of Incorporation"
  = form.date_select :year_end, order: [ :day, :month ], include_blank: true, default: nil

  = form.submit
```

### The Controller

The form is set to POST (asynchronously or otherwise) to the ClientsController which is a standard scaffold/REST controller. The `create` action is largely unchanged from when it was generated:

```ruby
# POST /clients
# POST /clients.json
def create
  @client = Client.new(client_params)

  respond_to do |format|
    if @client.save
      format.html { redirect_to clients_path, notice: 'Client was successfully created.' }
      format.json
    else
      format.html { render action: 'new' }
      format.json { render json: @client.errors, status: :unprocessable_entity }
    end
  end
end
```

When the client can be saved successfully it returns the new client's details as JSON, rendered by `app/views/clients/create.json.jbuilder`:

```ruby
json.id @client.id
json.name @client.name
```

When the client fails validation and cannot be saved, the errors are returned as JSON, e.g.

    {"name":["can't be blank"],"year_end":["can't be blank"],"date_of_incorporation":["can't be blank"]}

### Validation

We have the form which is ready to be submitted asynchronously, and the controller is ready to return JSON, but the default behaviour in Rails AJAX requests is to ask for a JS (Javascript) response. So, we reconfigure all AJAX requests to request JSON instead, as in this application this is what we always want by default:

```coffeescript
# Default to JSON responses for remote calls
$.ajaxSetup({
  dataType: 'json'
})
```

Finally, we hook into the AJAX request cycle and specify behaviour for when the Client is created successfully (append it to a list on the page) and for when validation fails (render the errors in the form):

```coffeescript
# New Client
$("#new_client").on("ajax:success", (e, data, status, xhr) ->
  $("#clients").append("<li>" + data['name'] + "</li>")
  $("#clients").effect("highlight")
).on("ajax:error", (e, data, status, xhr) ->
  $("#step-clients form").render_form_errors('client', data.responseJSON)
)
```

The crucial part here, which appears to not be provided by Rails, is this `render_form_errors` function. Here is my naïve implementation which looks for form elements with a `name` that *starts with* the field name provided in the errors hash:

```coffeescript
$.fn.render_form_errors = (model_name, errors) ->
  form = this
  this.clear_form_errors()

  $.each(errors, (field, messages) ->
    input = form.find('input, select, textarea').filter(->
      name = $(this).attr('name')
      if name
        name.match(new RegExp(model_name + '\\[' + field + '\\(?'))
    )
    input.closest('.form-group').addClass('has-error')
    input.parent().append('<span class="help-block">' + $.map(messages, (m) -> m.charAt(0).toUpperCase() + m.slice(1)).join('<br />') + '</span>')
  )
```

Scoping the search to the form in question, we look for **any element** with a `name` that starts with the given field name. For example for the "year end" field, its HTML representation is:

```html
<input id="client_year_end_1i" name="client[year_end(1i)]" type="hidden" value="1">
```

We can't match it exactly as it is suffixed with `(1i)` due to it being composed of multiple select drop downs. Instead, this function will look for any form elements with a name that matches the regex `/client\[year_end\(?/` which matches strings like `"client[year_end]"` or in this case `"client[year_end(1i)]"`.

Importantly, it **does not** match strings like `"client[year_end_another_field_name]"` as this would cause errors for certain fields to find their way on to the elements of other fields where the names share a common beginning.

### Resetting Forms

We define two more helper functions that will be used when working with remote forms, one to clear all errors from the form and the other to clear all form data (which we might do when we successfully save a Client):

```coffeescript
$.fn.clear_form_errors = () ->
  this.find('.form-group').removeClass('has-error')
  this.find('span.help-block').remove()

$.fn.clear_form_fields = () ->
  this.find(':input','#myform')
      .not(':button, :submit, :reset, :hidden')
      .val('')
      .removeAttr('checked')
      .removeAttr('selected')
```

## Wrapping Up

With the above we now have a fairly general implementation of remote forms for our app where to fully handle validation we only need to call one helper method.

In doing so we have left the controller largely untouched, and thanks to a custom form helper and Bootstrap we need need do very little to have all controls appear as they should when we need to show an error.

Here's what the result looks like when we click submit on an empty form, thus triggering validation errors:

<img src="http://stefan.haflidason.com/assets/2014/08/Screen-Shot-2014-08-18-at-16.42.21.png" alt="Bootrap Rails Form Validation" width="491" height="444" class="aligncenter size-full wp-image-784" />

If you have any questions or suggestions for improvement you can [drop me a line on Twitter](https://twitter.com/styrmis).
