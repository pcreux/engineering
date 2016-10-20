# Brewhouse Engineering Principles and Practices


# Table of Contents

1. [Brewhouse Engineering Principles](#brewhouse-engineering-principles)
   * [We write code for our peers](#we-write-code-for-our-peers)
   * [We trust our peers](#we-trust-our-peers)
   * [We experiment (bleeding edge)](#we-experiment-bleeding-edge)
   * [We ship good, old, stable](#we-ship-good-old-stable)
   * [We collaborate](#we-collaborate)
   * [We reflect](#we-reflect)
   * [We contribute](#we-contribute)
2. [Brewhouse Engineering Practices](#brewhouse-engineering-practices)
   * [Git](#git)
   * [Code](#code)
   * [Pull Requests](#pull-requests)
   * [CI](#ci)
   * [Deployment](#deployment)
   * [Ruby on Rails](#ruby-on-rails)
     * [Services](#services)
     * [Controllers](#controllers)
     * [Models](#models)
     * [Views](#views)
     * [Confident Ruby - Robust code](#confident-ruby-robust-code)
     * [Data integrity](#data-integrity)
     * [RSpec](#rspec)
     * [Cucumber](#cucumber)
     * [Rake tasks](#rake-tasks)
     * [APIs](#apis)
     * [Charts](#charts)
     * [Rails and JS](#rails-and-js)

# Brewhouse Engineering Principles

## We write code for our peers

We care about the person who will have to read, debug or extend our code. We want them to have a good time. We take extra time to turn "code that works" into code that’s clean, explicit, simple, robust, confident and tested… obviously!

## We trust our peers

We trust our peers for making the best decisions and the best work they can.

## We experiment (bleeding edge)

We experiment with new shiny technologies, tools and languages. We challenge ourselves, get out of our zone of comfort and expand our horizon. That’s what Friday times are for.

## We ship good, old, stable

We ship software built on top of mainstream languages and tools that are stable and should be supported for years to come.

## We collaborate

We know where engineering fits in the big picture and do our best to collaborate with product owners, designers and customers. We explain our constraints in words anyone can understand.

## We make pragmatic decisions

When we evaluate decisions, we take into account the requirements, unknowns, experience, team, and deadlines. We know the tradeoffs implied by the decisions we make.

## We reflect

We take time to reflect on our practice and improve.

## We contribute

We acknowledge that we sit on shoulders of giants. We give back by contributing to open source. We share knowledge on our blog and on the Brewhouse Software Engineering Guidelines document.

# Brewhouse Engineering Practices

## Git

### Commit messages

The goal of a commit message is to help future developers to find out why a particular change was made to the code.
We write commit messages that explain **Why** we made a change. We can look at the diff to know **What** the change is about.

* Not good: "Made the project container an inline-block"
* Good: "Fix project cards layout"

A commit is structured like this:

```
Short (50 chars or less) summary of changes

More detailed explanatory text, if necessary.
* Bullet points are okay, too.
```

You can read more on ["How to Write a Git Commit Message"](http://chris.beams.io/posts/git-commit/)

### Branches

We prefix our branches with one of the four following keywords:

* `feature/` for new features
* `fix/` for bugs
* `design/` for UI changes or design spikes (a.k.a prototypes)
* `tweak/` for other changes such as refactorings, improvements, ...

We turn a `design` branch into a `feature` one when we turn a prototype into a actual feature.

We rebase remote changes to prevent local merges from polluting the history (such as `Merge branch 'master' of github.com:BrewhouseTeam/goodbits`)

Use `git pull --rebase` and add the following to your `~/.gitconfig` to do it automagically on most branches

```
[branch]
  autosetuprebase = always
```

## Code

We setup our Text Editors to remove trailing whitespaces. This helps making commit diffs clean.

We prefer Clear Code over Comments:
  * we use Good names that reflect the intent of a class or a method
  * we extract complex code into methods or local variable with meaningful names

We order public methods with Requests first (`available?`, `full_name`, ...) then Commands (`available!`, `publish!`, ...).

We order private methods so that they always call a method that's
written below them. It makes it easier to find the method and it prevents
infinite recursions.

## Pull Requests

A couple of rules:

* Any code change should go through a Pull Request.
* A pull request gives a high level overview of what has been accomplished.
* A picture is worth a thousand words, so we add a screenshot if we've added a new feature or changed the UI.
  An animated GIF is worth a million words, so we showcase new UI interactions with one! ([LiceCap](http://licecap.en.softonic.com/) does the job.)
* We review our own changes before creating a pull request: Is there
  code that's commented out? Are there tests? ...
* When we are not done but we want some feedback, we prefix the PR title
  with "[WIP]"
* Most pull requests should be reviewed and merged by another team member.
* When we make a change that's risky (new database migrations, new
  integration with third party service, ...) we (the author) should monitor the
  deploy to production.

Things to look at when reviewing a Pull Request:

* Is the code clear and robust?
* Is it well tested?
* Are the commits well written? Should they be squashed?

We want honest and good enough code to be shipped. We don't want to
spend too many cycles making the code Perfect™.

## CI

We use Circle-CI for continuous integration. The master branch is setup to deploy to a staging server on Heroku.

## Deployment

We use Heroku to serve client apps.

The [brewhouse-rails-template](https://github.com/BrewhouseTeam/brewhouse-rails-template) has a good basic setup.

We use the following addons:

* rollbar for error reporting
* sendgrid to send emails
* newrelic for performance monitoring
* papertrail to get access to the logs
* heroku-redis for sidekiq

We use Cloudflare to get DNS + CDN + free SSL cert.

We setup pg:backups `heroku pg:backups schedule --at '04:00 UTC'`

## Ruby on Rails

We follow the [Airbnb Ruby Styleguide](https://github.com/airbnb/ruby).

We bootstrap apps using our rails template: [brewhouse-rails-template](https://github.com/BrewhouseTeam/brewhouse-rails-template).

### Services

See [Gourmet Service Objects](http://brewhouse.io/blog/2014/04/30/gourmet-service-objects.html) for an introduction.

Whenever an action becomes complex, we extract it from our controller or model and wrap it into a Service.

We namespace services to keep them well organized:

```ruby
# Bad
CreateUser

# Good
User::Create
```

Most services use the [base service mixin](https://github.com/BrewhouseTeam/brewhouse-rails-template/blob/master/app/services/service.rb).
They favor raising exceptions on failure over returning `false`.

Whenever we can we decouple services from ActiveRecord to make them more "functional". They take primitives in and return primitives back. That follows the "functional core & imperative shell" approach.
Your service won't deal with persistence and will be super easy to test!

Example:

```ruby
# Not so good
class Post < ActiveRecord::Base
  def refresh_view_stats!
    Post::RefreshViewStats.call(post: self)
  end
end

# Great!
class Post < ActiveRecord::Base
  def refresh_view_stats!
    update_attribute(:view, Post::GetViewStats.call(post_public_id: public_id))
  end
end
```

### Controllers

Controller actions are responsible for:

* massaging params to pass a clean set of attributes to a service
* calling a service
* redirecting / rendering

Controllers also take care of authentication and authorization.

Authorization via ActiveRecord associations is good enough:
```ruby
current_user.accounts.find(params[:account_id]).campaigns
```

We use [Pundit](https://github.com/elabs/pundit) when fine-grained authorization is required.

We create a `LoggedInController` that ensures that a user is authenticated. Most controllers inherit from `LoggedInController`.

### Models

Models are responsible for associations, scopes, validations and data consistency.

A given model is organized in the following order:

* Mixins
* Associations
* Validations
* Scopes
* Enums
* Queries (ex: `full_name`)
* Commands (ex: `publish!`)

Services and Controllers should not call "where" and "order" methods. Models should provide them with an API that's abstracted from the data-layer:

```ruby
# bad
User.where("created_at > ?", 2.days.ago).order("created_at DESC")

# good
User.created_after(2.days.ago).order_desc
```

Use [enums](http://api.rubyonrails.org/classes/ActiveRecord/Enum.html) to define states. When you get a fair amount of states, using a state machine such as [aasm](https://github.com/aasm/aasm) can help a lot.

Don’t use `has_and_belongs_to_many`. Use `has_many through` instead and define a model for the join table. It gives you access to that join table and to add attributes if needed.

Callbacks can only be used to ensure data integrity (i.e. keeping a `comments_count` cache value up to date, touch records, …). They should not be used for business logic.

Encrypt api and oauth tokens using [attr_encrypted](https://github.com/attr-encrypted/attr_encrypted).

### Views

Use erb because our designers are not big fan of slim / haml. :)

Use Page Objects when views become a bit complex. Page Objects methods tell the view what to hide / display.

```ruby
# Bad
@page.profile_completed?

# Good
@page.display_progress_bar?
```

### Confident Ruby - Robust code

In order to be confident that the app works, makes things easy to debug, and ensures data integrity:

Don't fail silently. Use `save!` , `find_by_...!`, `update!`, `Hash.fetch`, `case ... when ... else raise "...."` whenever the call should not fail.

### Data integrity

Inconsistent data (missing fields, duplicate records) leads to bugs that are hard to track down. The following practices don’t require much extra time and can save you from painful debugging sessions:

* wrap multiple create / updates in a SQL transaction
* use foreign keys to prevent orphan records (add `schema_auto_foreign_keys` to your Gemfile!)
* use `null: false` on fields that should not be empty. This can save you from writing  `validates_presence_of`  constraints for models which are not edited directly by users. It is also pretty good as documentation right in `schema.rb`
* use index with `unique: true` on unique fields or combination of fields.
* use index with  `unique: true` on `*_id` attributes of `has_one`, `has_many through: associations`. Rails does not double check that an association does not exist when creating it, so you can easily get multiple associations between the same objects in a join table.

Also see this article: [Five practices for Robust Ruby on Rails applications](http://brewhouse.io/2016/02/26/five-practices-for-robust-ruby-on-rails-applications.html)

### RSpec

With RSpec, we mostly test models, services and api interaction. Cucumber scenarios are likely to cover the thin controller layer. Once in a while, you might need to test a complex controller, go for it!

We use FactoryGirl to create models without trying to go too far with it. Each factory should be valid on its own. If you need some crazy custom setup, call Services from your specs to create stuff.

Use `describe` blocks to describe the class, method or behaviour. Ex:

```ruby
describe Subscriber::Add
describe "#disable!"
describe "sending via MailChimp"
describe ".active"
```

Instance methods are prefixed with a `#`, class methods are prefixed with a `.` (`#disable!` but `.active1`)

`context` should start with the keyword "when". Ex:

```ruby
context "when email is invalid"
context "when CSV has no content"
```

Use `rspec-set` to speed up the RSpec test suite! Be well aware that `rspec-set` leaves orphan records in the database, so most of your tests should not assume that the database is empty. It’s sometimes tricky to test scopes with an non-empty database. Feel free to truncate the database in a before(:all) block if needed.

Write accurate tests to prevent false positives:

```ruby
# Bad
expect(campaign.active.count).to eq 2

# Good
expect(campaign.active).to match_array(active_campaign_1, active_campaign_2)
```

### Cucumber

Cucumber scenarios should be high level, and contain a few (5-8) steps. They describe a workflow, not UI interactions. Doing so, we spend more time in step definitions (read capybara & ruby) than step definition and cucumber pattern matching hell.

Define steps with strings rather than regex when possible:

```ruby
Given "I am logged in" do
  # ...
end

Then %|I should be logged in as "$name"| do |name|
  # ...
end
```

Given steps should rely on factories and services to set up a context. Going through the UI or reusing other steps are likely to slow things down.

By order of preference (because of speed), use `capybara-rack`, or `capybara-webkit`... `capybara-selenium` is slow!

### Rake tasks

We use rake tasks to trigger recurring jobs.

`rake -T` counts about 40 tasks by default. We namespace all the application tasks under the `app` namespace so that they are easy to find.

We use Heroku Scheduler to run tasks every 10 minutes, hourly and daily. In order to make it easy to update tasks, we create three generic tasks (`app:cron:daily`, `app:cron:hourly`, `app:cron:every_10_minutes`) and call other tasks within those.
We setup Heroku Scheduler to call the `app:cron:...` tasks.

In order to prevent a failing task to run subsequent ones, each rake task should queue up a job that Sidekiq will happily take care of.

### APIs

We have a good experience with [jsonapi-resources](https://github.com/cerebris/jsonapi-resources). It provides a good framework to build a consistent api. Pair it with rspec-api-documentation and apitome and you’ve got a pretty good setup that makes your mobile friends happy. :)

Authentication has been done so far using Basic Auth ("Authentication" header with login:password base64 encoded).

### Charts

[Chartkick](http://chartkick.com/) + [Chart.js](http://www.chartjs.org/) is a pretty good combo to keep things simple and tidy.

### Rails and JS

JQuery sprinkles and Rails Remote Javascript, when done well, are pretty good.

A couple of good practices here to keep things simple and robust:

* Decouple the javascript function from the view by injecting css selectors when calling a function from the view. This will prevent you from breaking js.
* If you don't inject selectors for some reason, prefix classes with `js-` or use data attributes.
* With Rails Remote Javascript: pass the selector that you went to render in (or append to…) as an http param.
* Make Rails Remote files just set or append html content that you render from another partial.

**Thanks for reading!**

Feel free to create issues and pull-requests to share your feedback or improve this document.


---

The MIT License (MIT)
Copyright (c) 2016 Brewhouse Software

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
