# Coding conventions

## Linters

Consul Democracy includes linters that define code conventions for Ruby, JavaScript, SCSS and Markdown. Following these conventions makes the code consistent, easier to read and easier to maintain.

When you open a pull request, our continuous integration (CI) system automatically checks that the code in the pull request follows these conventions (you can have a look at the `.github/workflows/linters.yml` file in order to check the exact commands the CI executes). Please check the pull request report generated by these linters.

Since you probably want to check these conventions at all times during development instead of waiting until you've already opened a pull request, we recommend you check whether your text editor has support for these linters. When an editor supports these linters, every time you save a file you'll get immediate feedback about whether you're following these conventions.

A current limitation is that there are two Rubocop (the linter we use for Ruby) rules that aren't followed by old code and one rule that contains false positives. These rules are marked with `Severity: refactor` in the `.rubocop.yml` file; you might want to configure your editor so it ignores them.

If your editor doesn't support these linters, you can also run the `bundle exec pronto run` command, which will only analyze the changes in the current development branch, meaning that you'll get feedback much faster than you'd do if you ran the commands to analyze every file in the project.

## Beyond linters

There are code conventions we follow that can't be analyzed by linters. Moreover, old code doesn't always follow these conventions, so sometimes it'll be hard to figure out which way you should proceed.

Here are a few hints that will hopefully help. Don't try to memorize them all at once; instead, check whether these conventions are related to the code you are currently writing.

### Only add English and Spanish texts

Most Consul Democracy developers are fluent in English and Spanish but don't know much about other languages. On the other hand, there are many people who are fluent in other languages but don't have the technical knowledge to work with Consul Democracy's source code. That's why we use [Consul Democracy's Crowdin](https://crwd.in/consul) to manage translations for languages other than English and Spanish.

Currently, the downside of this approach is that if you include translations for other languages in your pull request, they will be overwritten by the content in Crowdin. So don't bother including texts in other languages in your pull request; once your pull request is merged, use Crowdin to add translations to other languages instead.

### Use components instead of views and helpers

You might notice that HTML/ERB code appears in both `app/views/` and `app/components/`. For new code, use `app/components/`. We're gradually moving existing code from `app/views/` to `app/components/`.

There's one scenario when you still need to add code to `app/views/`. Suppose you're creating a new controller named `AwesomeThingsController` that contains the action `index`. In this case, you need to create the `app/views/awesome_things/views/index.html.erb` file, with the following content (note that, in the code below, you might need to pass one or more parameters to the `new` method if the component needs them):

```erb
<%= render AwesomeThings::IndexComponent.new %>
```

Then you would create the `app/components/awesome_things/index_component.rb` and `app/components/awesome_things/index_component.html.erb` files.

Similarly, whenever possible, avoid adding helper methods and add methods to a component instead. Since helper methods are globally available, writing a helper method with the same name as an existing one (even if the existing one is present in a different module or even in one of our gem dependencies) will silently overwrite the existing helper. Component methods, on the other hand, are only available inside one component class, which makes them more reliable and easier to refactor.

### Structure (S)CSS and JavaScript like components

If you're adding some CSS or JavaScript code that only affects one component, create a file following the same structure as the component. For example, the CSS related to the code in `app/components/admin/budgets/form_component.html.erb` is located at `app/assets/stylesheets/admin/budgets/form.scss`.

Also note that the HTML classes follow a similar structure; in this case, the `app/components/admin/budgets/form_component.html.erb` file contains an HTML element with the `budgets-form` class (note that we aren't always consistent regarding when to add an `admin-` prefix, though).

### Use the Header component concern in new admin pages

When writing a new page in the admin area , using the `Header` concern in components makes it easier to add a proper title and heading to the page. Define a `title` method like:

```ruby
class Admin::AwesomeThings::IndexComponent < ApplicationComponent
  include Header

  private

    def title
      t("admin.awesome_things.index.title")
    end
end
```

Then you can add `<%= header %>` to the `.html.erb` file, which will set the correct `<title>` tag (which is very important for accessibility) as well as generating the page heading.

### Use buttons for non-GET HTTP requests

By default, clicking an HTML link generates a GET request. However, the `link_to` helper provided by Rails accepts a `method:` option; using it will allow you to render links that generate non-GET requests (POST, PUT/PATCH, DELETE, ...). **Do not do this**.

Instead, use the `button_to` helper, which generates a form with a button with native support for non-GET requests. Buttons offer many advantages over links in this scenario:

* People using screen readers will know what the button is supposed to do, but might be confused about what the link is supposed to do.
* Buttons work when JavaScript hasn't loaded or is disabled.
* Most browsers allow opening links in a new tab; for non-GET requests, that will usually generate an error.

Similarly, use buttons instead of links for controls that don't generate any HTTP requests but modify the content of the page using JavaScript (for example, to hide or show certain content).

### Use Tenant secrets instead of Rails secrets

Rails manages confidential information by storing it in the `config/secrets.yml` file. Usually, Rails applications access the contents of this file using the `Rails.application.secrets` method.

Consul Democracy, however, is a [multitenant application](../features/multitenancy.md). With a few exceptions, most secrets allow different values for different tenants. In this case, using `Tenant.current_secrets` instead of `Rails.application.secrets` will correctly find different values depending on the current tenant.

### Browser support

We try to support 100% of the browsers whenever possible. That means that we don't use ECMAScript 2015 (or later) but use the ECMAScript 5 syntax instead.

However, sometimes, particularly when styling with CSS, supporting 100% of the browsers isn't feasible. In this case, we aim to make the page look as expected on browser versions that are 7 years old or newer (about 98% of the browsers out there), while making sure things don't look too broken everywhere else.

### Right-To-Left support

In order to support Right-To-Left (RTL) languages, like Arabic or Hebrew, when writing CSS, use properties like `margin-#{$global-left}` or `margin-#{$global-right}` instead of `margin-left` or `margin-right`. Same with padding and other positional properties. The `$global-left` variable is set to `left` on languages written from left to right and to `right` on languages written from right to left; the `$global-right` variable takes the opposite value.

Until there's universal browser support for them, don't use logical properties like `margin-inline`.

### Use rem units instead of rem-calc

In our (S)CSS code, you'll probably find many places using Foundation's `rem-calc` function to convert pixels to rems. Other places directly use `rem` to define rems. At some point, we'll remove every usage of `rem-calc`, so use `rem` instead.

### Write component tests for scenarios with no user interaction

For years, we almost exclusively wrote system tests to check the content displayed on a page. However, system tests are very slow and, when Consul Democracy started to grow, our test suite reached a point where it takes too long to run it on a development machine and we need to rely on a continuous integration system. To prevent the test suite from becoming even slower, only write system tests when testing some kind of user interaction like clicking links or filling in forms. To test that, for example, certain content is rendered for verified users but not rendered for unverified ones, write a component test instead.

### Don't check the database after a system test

In general, system tests have four parts:

1. Setting up the database
2. Starting the browser using the `visit` method
3. Doing some kind of interaction
4. Checking the results of the interaction

When checking the results of the interaction, don't check the content of the database (for example, by checking `Proposal.count` after creating a proposal) because it might result in a flaky database exception when the process running the test and the process that started the browser both access the database. Instead, check the content of the page as seen from the user's point of view (for example, check the number of proposals appearing on the proposals index page instead of checking `Proposal.count`).

Checking the database instead of checking the user's point of view can also lead to usability and accessibility issues. For example, a button which modifies the database without giving users any feedback is a usability issue that will be detected when checking the user's point of view. For the same reason, you should try to use reference texts instead of HTML attributes; for example, instead of writing `fill_in "residence_document_number", with: "12345678Z"`, you should write `fill_in "Document number", with: "12345678Z"`. The latter checks that a label is correctly associated with the input, while the former does not.

### Avoid simultaneous requests in system tests

When using methods like `click_link`, make sure the request has finished before generating a new one. Getting this point right is really hard (sometimes we don't do it right either), so we really appreciate your efforts regarding this point.

Here's an example:

```ruby
scenario "maintains the navigation link" do
  visit admin_root_path
  click_link "Proposals"

  within("#side_menu") { expect(page).to have_link "Proposals" }
end
```

The problem of this test is that the expectation is also true without clicking the "Proposals" link. When writing this code, we probably want the browser to click on the "Proposals" link, wait for the request to finish, and then check the content of the page. That is not what's happening here, though.

Instead, what's happening is: the browser clicks on the link and the test immediately checks the content of the page. If the expectation is met, the test doesn't wait for the request to finish. That means that the request might finish when a different test has started and, after that, anything can happen. For example, the data from the two different tests might get mixed up.

This sometimes results in a so-called "flaky spec", which is a test that fails sometimes but not 100% of the time. Flaky specs are really harmful because, when facing a test suite with a failure during a pull request, you no longer know the cause of the failure, which might not even be related to the pull request.

The example above would be solved with:

```ruby
scenario "maintains the navigation link" do
  visit admin_root_path
  click_link "Proposals"

  within("#side_menu") { expect(page).to have_css "[aria-current]", exact_text: "Proposals" }
end
```

This time, the expectation isn't true before clicking the "Proposals" link, so the test waits until the request has finished.

By the way, you might notice that here we're checking an HTML attribute, which seems to be the opposite of what we recommended in the [don't check the database after a system test](#dont-check-the-database-after-a-system-test) section. However, people using screen readers will be notified about the `aria-current` attribute, so we're actually testing the page from the user's point of view.
