cucumber-best-practices
=======================

Best practices and tips for using Cucumber

## Best Practices

A collection of best practices collecting from the following sites:

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

