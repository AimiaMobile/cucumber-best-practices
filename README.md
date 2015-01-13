Cucumber Best Practices
=======================

A compilation of best practices and tips for using Cucumber.

## Best Practices

A series of best practices collected from the following sites:

> https://github.com/cucumber/cucumber/wiki/Cucumber-Backgrounder

> http://coryschires.com/ten-tips-for-writing-better-cucumber-steps/

> http://andrewvos.com/2011/06/15/writing-better-cucumber-features/

> https://blog.engineyard.com/2009/15-expert-tips-for-using-cucumber/

#### Use flexible pluralization

```ruby
Then /^the users? should receive an email$/ do
# ...
end
```

The ? specifies that your looking for zero or more of the proceeding character. So the above example will capture both user and users.

#### Use non-capturing groups to help steps read naturally

You can create non-capturing groups by adding a ?: to the beginning of a otherwise normal group (e.g. (?:some text) rather than (some text)). This is treated exactly like a normal group except that the result will is not captured and thus not passed as an argument to your step definition. This often useful in conjunction with alternation

```ruby
And /^once the files? (?:have|has) finished processing$/ do
# ...
end
```

#### Consolidate step definitions by capturing optional groups

Often I find myself writing essentially the same step with both positive and negative assertions. You can remove this duplication by capturing an optional group:

```ruby
Then /^I should( not)? see the following columns: "([^"]*)"$/ do |negate, columns|
  within('table thead tr') do
    columns.split(', ').each do |column|
      negate ? page.should_not(have_content(column)) : page.should(have_content(column))
    end
  end
end
```

Here we’re capturing an optional group (note ( not)? using the ? mentioned above). We then pass that into our step as the negate variable which we can use to write conditional assertions.

#### Not all matched phrases need to be surrounded by quotes

Don’t assume that matched phrases must (or should) be enclosed in double quotes. Often quotes are a good idea. They make it visually clear which phrases will be passed to the step definition. For example:

```ruby
Given I have "potatoes" in my cart
```

That’s reasonable. The quotes highlight the parts that change without hurting readability. But sometimes, quotes are just poor style:

```ruby
Given I have "7" items in my cart
```

It should be pretty obvious that the number is variable. The quotes add nothing except noise. A better step would read:

```ruby
Then /^I have (\d+) items? in my cart$/ do |item_count|
  item_count.to_i.times { ... }
end
```

#### Use transforms to make smarter, DRYer regular expressions

There’s an opportunity to refactor the previous example. Do you see it? Cucumber passes everything as a string, so you must remember to convert types within your step definition (e.g. above we use `to_i` to transform `item_count` into a proper integer). That’s annoying and easy to forget.

Fortunately, Cucumber gives us a way to avoid this peskiness by using `Transform`:

```ruby
CAPTURE_A_NUMBER = Transform /^\d+$/ do |number|
  number.to_i
end
```

And we can use this in our steps:

```ruby
Then /^I have (#{CAPTURE_A_NUMBER}) items? in my cart$/ do |item_count|
  item_count.times { ... }
end
```

Not only have we removed the need to call `to_i` but we’ve also moved our regex into a reusable constant.

#### Define methods to DRY up your step definitions

Sometimes it’s a good idea to remove duplication by defining methods. For example, it’s common to define a `current_user` method:

```ruby
def current_user
  User.find_by_email('current_user@example.com')
end
```

Similarly, if you find yourself wrapping several steps with the same logic, you might consider making a helper method:

```ruby
def within_lightbox(opts = {sleep: 0} )
sleep(opts[:sleep])
within_frame("prettyPhotoIframe") { yield }
end
```

And our step definitions stay nice and clean:

```ruby
Then /^some stuff should be visible in the lightbox$/ do
  within_lightbox { page.should have_content('Some Stuff') }
end
```

By convention, these methods should live in features/support/world_extensions.rb and be included in the Cucumber World module. But keep in mind this is a tradeoff: you’re removing duplication but adding indirection. You should be reluctant to define methods until the code makes it very obvious that it’s a good idea.

#### Improve readability with unanchored regular expressions

Most step definitions look something like:

```ruby
Given /^I am an admin user$/ do |item_count|
# ...
end
```

Note we’re using `^ `and `$` to anchor our regex to the start and end of the captured string. This ensures the regular expression exactly matches “I am an admin user” (i.e. allows no additional words at the beginning or end of the step). Most of the time, this is exactly what want.

Occasionally, however, it makes sense to omit the final `$`. Take this step for example:

```ruby
Then /^wait (\d+) seconds/ do |seconds|
  sleep(seconds.to_i)
end
```

Now you can use this definition to write flexible, expressive steps:

```ruby
Then wait 2 seconds for the revenue statistics to finish loading
Then wait 5 seconds while the document is converted
```

#### When your steps must include data, use tables

Generally, steps should be human readable and that means they shouldn’t include loads of cryptic data. But sometimes, you have no other choice. In those cases, use tables to clearly represent the data:

```ruby
Given I sent "$25" to "test@example.com" from my "Bank account"
Then I should see the following transaction history:
  | create     | complete    |
  | deposit    | in_progress |
  | transfer   | pending     |
  | withdrawal | pending     |
```

The step definition looks like the following:

```ruby
Then "I should see the following transaction history:" do |table|
  table.raw.each do |event, state|
    page.should have_css("tr.#{event}.#{state}")
  end
end
```

#### Step definitions that call other steps

A rrule of thumb is that if a step definition is called from another step definition then its contents probably should be extracted out into a custom method. For example a logon step definition is likely to be used repeatedly throughout many features. Turning it into a method is probably called for.

You can either keep these custom methods in the same file as the regular steps or stick them in any convenient file ending in .rb that is located in the support directory.

#### Scenarios should not be dependant on other scenarios

There should never be a situation where one scenario needs to be run before another scenario.

I've personally seen feature files that had scenarios with names that had number prefixes in order to make sure that they ran in a certain order. This is evil and should never happen.

Never ever ever let this happen:

```ruby
Scenario: 0TestSetup
  Given some data is in the database

Scenario: 1UserNavigatesToProfilePage
  Given the user has logged in
  When I click Profile
  Then I should see the Profile page

Scenario: 2UserChangesTheirPassword
  When I enter a new password
  And I click Save
  Then I should see "Password changed"
```

#### Use Background for setup when there's more than one scenario

Remember that whatever is more readable is always better. In the first code sample we only have one scenario so we don't need to DRY up the feature yet, but in the second sample it makes sense to add a Background section.

```ruby
Scenario: Logged in user visits the user registration page
  Given I am logged in
  And I visit the user registration page
  Then I should be redirected to the home page
```

```ruby
Background:
  Given I am logged in

Scenario: Logged in user visits the user registration page
  And I visit the user registration page
  Then I should be redirected to the home page

Scenario: Logged in user visits the logout page
  And I visit the logout page
  Then I should be redirected to the home page
```

#### Feature names are features not processes

Feature names should describe actual features. An example of a bad feature name is "A user registers an account". This feature should probably be called "Registration" or "User Registration".

#### Don't use multiple 'thens'

Each step should do one thing. You should not generally have step patterns containing "and." For example:

```ruby
Given A and B
```

should be split into two steps:

```ruby
Given A
And B
```

#### What is a good Step Definition?

Opinions vary of course, but for me a good step definition has the following attributes:

- The matcher is short.
- The matcher handles both positive and negative (true and false) conditions.
- The matcher has at most two value parameters
- The parameter variables are clearly named
- The body is less than ten lines of code
- The body does not call other steps

## Tags

In the simplest case, Cucumber runs all the scenarios in all the features that you point it at. By using tags you can be more specific about what is run. 

Some useful ways you can use Cucumber tags to classify features and scenarios:
- Tagging the size of the feature/scenario, e.g. fast, slow, glacial
- Tagging how often a feature or scenario should be run (every build, hourly, daily etc)
- Tagging whether the scenario is destructive or invasive (or indeed non-destructive or non-invasive: good for running in production!)
- Tagging the feature or scenario with some form of meta-data for requirements traceability, or symbolic link to some other document or system
- What external dependencies they have: @local, @database, @network
- Level: @functional, @system, @smoke
- etc

**Reference:**

> http://watirmelon.com/2011/07/04/use-cucumber-feature-folders-for-functional-organization-tags-for-non-functional-classification/

> https://blog.engineyard.com/2009/cucumber-more-advanced

> https://blog.engineyard.com/2009/15-expert-tips-for-using-cucumber/

## Hooks

Cucumber provides the ability to supply hooks to modify the behavior of executing features. Hooks can be used at the various levels of granularity, generally before and after, and are often defined in your env.rb file. Hooks are executed whenever the event they are defined for occurs.

#### Global Hooks
Global hooks run when Cucumber begins and exits. Begin hooks are informal: simply put the desired code in a file named 'hooks.rb' in the features/support directory
An exit hook is more formal, using at_exit. Here's an example of a pair of global hooks:

#### Scenario Hooks
You can add blocks that will run before and after each scenario:

The After block will be passed the scenario that just ran. You can use this to inspect its result status by using the failed?, passed? and exception methods. For example:

#### Step Hooks
Cucumber gives you the ability to define a block that will be executed after each (and every) step:

#### Tagged Hooks
If you've tagged some of your scenarios, you can also tag scenario and step hooks. You simply pass the tags as arguments to the hook methods). These tagged hooks will only be executed before/after scenarios that are tagged the same and after steps in scenarios tagged the same. Here's an example where the hooks will only be run before scenarios tagged with @cucumis or @sativus.

**A note on Background steps**

A feature Background is very much like a scenario in that it consists of a series of steps. The difference is that its steps are executed before the steps of each scenario in the feature. It's basically a factoring out of a set of common lead-in steps for the features scenarios. One thing to remember is that a Background is run after any Before hooks

**Reference:**

> https://blog.engineyard.com/2009/cucumber-more-advanced

## Folder and File Organisation

#### Organizing the Code

We've found that our favorite way to organize step definition files is to organize them with one file per domain entity. So, in our example, we’d have three files:

```
features/step_definitions/account_steps.rb
features/step_definitions/teller_steps.rb
features/step_definitions/cash_slot_steps.rb
```

#### Group features by feature

```
$ ls features/scheduling_appointment/
canceling_appointment.feature
client_requesting_appointments_with_insurance.feature
client_requesting_chat_appointment.feature
client_requesting_phone_appointment.feature
client_requesting_video_appointment.feature
client_responding_to_rescheduled_appointment.feature
expiring_pending_appointments.feature
```

#### Code loading order

1. The file `features/support/env.rb` is always the very first file to be loaded when Cucumber starts a test run. You use it to prepare the environment for the rest of your support and step definition code to operate.
2. When Cucumber first starts a test run, before it loads the step definitions, it loads the files in a directory called `features/support`.
3. Cucumber will then load all the Ruby files it finds in `features/step_definitions`.

#### features/support

Cucumber will load all the Ruby files it finds in `features/support`. This is convenient, because it means you don’t need to pepper require statements everywhere, but it does mean that the precise order in which files are loaded is hard to control. 

In practice, this means you can’t have dependencies between files in the support directory, because you can’t expect that another file will have already been loaded. There is one exception to that, a special file called env.rb

#### Adding custom helper methods to the world

Just before it executes each scenario, Cucumber creates a new object. We call this object the World. The step definitions for the scenario execute in the context of the World, effectively as though they were methods of that object. Just like methods on a regular Ruby class, we can use instance variables to pass state between step definitions

To add custom methods to the World, you define them in a module and then tell Cucumber you want them to be mixed into your World

**Reference:**

> [The cucumber book](https://pragprog.com/book/hwcuc/the-cucumber-book)

> http://collectiveidea.com/blog/archives/2010/09/13/practical-cucumber-organization/

#### Page Object

Use the Page Object pattern to abstract platform UI differences. Define a base Page Object class and derive child classes to implement platform specific UI logic, e.g. iOS uses accessibility labels to locate UI elements, whereas Android uses resource ids.

- The basic rule of thumb for a page object is that it should allow a software client to do anything and see anything that a human can
- It should provide an interface that's easy to program to and hides the underlying widgetry in the window. So to access a text field you should have accessor methods that take and return a string, check boxes should use booleans, and buttons should be represented by action oriented method names
-  A good rule of thumb is to imagine changing the concrete control - in which case the page object interface shouldn't change.
- Despite the term "page" object, these objects shouldn't usually be built for each page, but rather for the significant elements on a page. So a page showing multiple albums would have an album list page object containing several album page objects.
- Page objects should not make assertions themselves. Their responsibility is to provide access to the state of the underlying page. It's up to test clients to carry out the assertion logic.

The below code illustrates what a simple Page Object interaction would look like.

Base class `OfferPage` would have the following interface:
```ruby
selectOfferWithId()
activateOfferWithId()
```

Derive base classes for each relevent platform and override the appropriate methods.

Call the generic interface methods in your step definition. Consider creating a factory of some sort to retrieve the Offer Page for the particular platform under test, e.g. iOS or Android. In the sample below, this is implemented would be obtained from the `newForCurrentPlatformContext` method call.

```ruby
When I activate offer with id: (\d+) do |offerId|
  offerPage = OfferPage.newForCurrentPlatformContext
  offerPage.selectOfferWithId(offerId)
  offerPage.activateOfferWithId(offerId)
end
```
