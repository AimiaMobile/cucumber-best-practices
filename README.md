cucumber-best-practices
=======================

Best practices and tips for using Cucumber

** Best Practices
http://coryschires.com/ten-tips-for-writing-better-cucumber-steps/
http://andrewvos.com/2011/06/15/writing-better-cucumber-features/

- Scenarios should not be dependant on other scenarios
- Don't use multiple 'thens'

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


- Improve readability with unanchored regular expressions
- When your steps must include data, use tables e.g.

Given I sent "$25" to "mukmuk@example.com" from my "Bank account"
Then I should see the following transaction history:
  | create     | complete    |
  | deposit    | in_progress |
  | transfer   | pending     |
  | withdrawal | pending     |

The step definition looks like the following:

Then "I should see the following transaction history:" do |table|
  table.raw.each do |event, state|
    page.should have_css("tr.#{event}.#{state}")
  end
end

- Scenario outlines
- Background
