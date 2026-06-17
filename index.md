# Even Better Specs

> Guidelines for Maintainable Tests — <https://evenbetterspecs.github.io>

This is a plain-text/markdown version of the home page, intended for AI assistants and other tools.

## What is it?

Even Better Specs is an *opinionated* set of best practices to support the creation of tests that are easy to read and maintain.

It focuses on the [Ruby](https://www.ruby-lang.org/en/) testing framework [RSpec](https://rspec.info/), but some of it could potentially be applied to other frameworks and languages (like [sus](https://github.com/sus-rb/sus) and [Crystal](https://crystal-lang.org/reference/1.13/guides/testing.html)).

### Guiding principles

- tests must be [self-contained](https://thoughtbot.com/blog/the-self-contained-test), not DRY
- tests should follow the [Arrange-Act-Assert](https://automationpanda.com/2020/07/07/arrange-act-assert-a-pattern-for-writing-good-tests/) pattern

---

## Describe what you are testing

Be clear about what you are testing.

Bad:

```ruby
describe 'User' do
  describe 'the authenticate method for User' do
  end

  describe 'if the user is an admin' do
  end
end
```

Use the Ruby documentation convention of `.` when referring to a class method's name and `#` when referring to an instance method's name.

Good:

```ruby
describe User do
  describe '.authenticate' do
  end

  describe '#admin?' do
  end
end
```

### Describing request tests

In request tests, describe the controller constant and its actions so one can easily spot what is being tested.

Bad:

```ruby
describe '/users', type: :request do
  describe 'GET #index' do
    it 'returns a successful response' do
      get users_url

      expect(response.code).to eq('200')
    end
  end

  describe 'POST /users' do
    it 'creates a user' do
      post users_url, params: { user: { name: 'Tom Jobim' } }

      expect(User.last.name).to eq('Tom Jobim')
    end
  end
end
```

Good:

```ruby
describe UsersController, type: :request do
  describe '#index' do
    it 'returns a successful response' do
      get users_url

      expect(response.code).to eq('200')
    end
  end

  describe '#create' do
    it 'creates a user' do
      post users_url, params: { user: { name: 'Tom Jobim' } }

      expect(User.last.name).to eq('Tom Jobim')
    end
  end
end
```

RuboCop: You can enforce this guideline using the cop [RSpec/DescribeClass](https://docs.rubocop.org/rubocop-rspec/cops_rspec.html#rspecdescribeclass) from the [rubocop-rspec](https://docs.rubocop.org/rubocop-rspec/installation.html) gem.

---

## Use contexts

Contexts are a powerful way to make your tests clear and well organized. They should start with `when`.

Bad:

```ruby
it 'has 200 status code if logged in' do
  expect(response.code).to eq('200')
end

describe 'it returns 401 status code if not logged in' do
  it { expect(response.code).to eq('401') }
end
```

Good:

```ruby
context 'when logged in' do
  it 'returns 200 status code' do
    expect(response.code).to eq('200')
  end
end

context 'when logged out' do
  it 'returns 401 status code' do
    expect(response.code).to eq('401')
  end
end
```

RuboCop: You can enforce this guideline using the cops [RSpec/ContextMethod](https://docs.rubocop.org/rubocop-rspec/cops_rspec.html#rspeccontextmethod) and [RSpec/ContextWording](https://docs.rubocop.org/rubocop-rspec/cops_rspec.html#rspeccontextwording) from the [rubocop-rspec](https://docs.rubocop.org/rubocop-rspec/installation.html) gem.

---

## Factories, not fixtures

[Factories](https://github.com/thoughtbot/factory_bot) are more flexible and easier to work with. Understand more [here](https://thoughtbot.com/blog/we-need-to-talk-about-fixtures).

Example:

```ruby
def full_name
  "#{first_name} #{last_name}"
end
```

Good:

```ruby
it 'returns the full name' do
  user = create(:user, first_name: 'Santos', last_name: 'Dumont')

  expect(user.full_name).to eq('Santos Dumont')
end
```

If your code doesn't hit the database, prefer `build_stubbed` over `create` since the former will make your [tests faster](https://thoughtbot.com/blog/use-factory-bots-build-stubbed-for-a-faster-test) 🔥

Good:

```ruby
it 'returns the full name' do
  user = build_stubbed(:user, first_name: 'Santos', last_name: 'Dumont')

  expect(user.full_name).to eq('Santos Dumont')
end
```

---

## Leverage described class

Tests are not supposed to be DRY, but that doesn't mean we need to repeat ourselves in vain. Leverage `described_class` to make your tests maintainable over the years.

Bad:

```ruby
describe Pilot do
  describe '.most_successful' do
    it 'returns the most successful pilot' do
      senna = create(:pilot, name: 'Ayrton Senna')
      create(:pilot, name: 'Alain Prost')
      create(:race, winner: senna)

      most_successful_pilot = Pilot.most_successful

      expect(most_successful_pilot.name).to eq('Ayrton Senna')
    end
  end
end
```

Good:

```ruby
describe Pilot do
  describe '.most_successful' do
    it 'returns the most successful pilot' do
      senna = create(:pilot, name: 'Ayrton Senna')
      create(:pilot, name: 'Alain Prost')
      create(:race, winner: senna)

      most_successful_pilot = described_class.most_successful

      expect(most_successful_pilot.name).to eq('Ayrton Senna')
    end
  end
end
```

If the class `Pilot` ever gets renamed, one just need to change it at the top level `describe`.

RuboCop: You can enforce this guideline using the cop [RSpec/DescribedClass](https://docs.rubocop.org/rubocop-rspec/cops_rspec.html#rspecdescribedclass) from the [rubocop-rspec](https://docs.rubocop.org/rubocop-rspec/installation.html) gem.

---

## Use subject

Use `subject` instead of `described_class.new` when no argument is given to the initializer.

Example:

```ruby
class Calculator
  attr_reader :base

  def initialize(base = 0)
    @base = base
  end

  def add(number1, number2)
    base + number1 + number2
  end
end
```

Bad:

```ruby
describe '#add' do
 it 'sums two numbers' do
   calculator = described_class.new

   expect(calculator.add(1, 2)).to eq(3)
 end
end
```

Good:

```ruby
describe '#add' do
  it 'sums two numbers' do
    expect(subject.add(1, 2)).to eq(3)
  end
end
```

It can be overridden when argument is given to the initializer.

Good:

```ruby
describe '#add' do
  it 'sums two numbers' do
    subject = described_class.new(5)

    expect(subject.add(1, 2)).to eq(8)
  end
end
```

---

## Short description

A spec description must not be longer than 100 characters neither span many lines. If this happens you should split it using `context`.

Bad:

```ruby
it 'returns 422 when user first_name is missing from params and when user last_name is missing from params' do
end

it 'returns 422 when user first_name is missing from params \
    and when user last_name is missing from params' do
end
```

Good:

```ruby
context 'when user first_name is missing from params' do
  it 'returns 422' do
  end
end

context 'when user last_name is missing from params' do
  it 'returns 422' do
  end
end
```

---

## Group expectations

Having an `it` for each expectation can lead to a terrible test performance. Group expectations that use a similar data setup to improve performance and make the tests more readable.

Bad:

```ruby
it 'returns 200' do
  user = create(:user)

  get user_path(user)

  expect(response.code).to eq('200')
end

it 'returns JSON content type' do
  user = create(:user)

  get user_path(user)

  expect(response.content_type).to eq('application/json')
end
```

Good:

```ruby
it 'responds with 200 http status and a JSON content type' do
  user = create(:user)

  get user_path(user)

  expect(response.code).to eq('200')
  expect(response.content_type).to eq('application/json')
end
```

We recommend enabling [failure aggregation globally](https://rspec.toolboxforweb.xyz/docs/rspec-core/expectation_framework_integration/aggregating_failures#enable-failure-aggregation-globally-using-define_derived_metadata) to make it list all failures at once.

---

## All possible cases

Testing is a good practice, but if you do not test the edge cases, it will not be useful. Test valid, edge and invalid cases.

Note: If you have way too many cases to test, it might be an indication your subject class is doing too much and must be break down into other classes.

Example:

```ruby
before_action :authenticate_user!
before_action :find_product

def destroy
  @product.destroy
  redirect_to products_path
end
```

Bad:

```ruby
describe '#destroy' do
  context 'when the product exists' do
    it 'deletes the product' do
    end
  end
end
```

Good:

```ruby
describe '#destroy' do
  context 'when the product exists' do
    it 'deletes the product' do
    end
  end

  context 'when the product does not exist' do
    it 'raises 404' do
    end
  end

  context 'when user is not authenticated' do
    it 'raises 404' do
    end
  end
end
```

---

## Request vs controller specs

Do request tests instead of controller tests.

> The official recommendation of the Rails team and the RSpec core team is to write request specs instead. Request specs allow you to focus on a single controller action, but unlike controller tests involve the router, the middleware stack, and both rack requests and responses. This adds realism to the test that you are writing, and helps avoid many of the issues that are common in controller specs. In Rails 5, request specs are significantly faster than either request or controller specs were in Rails 4.

See [source](https://rspec.info/blog/2016/07/rspec-3-5-has-been-released/).

Bad:

```ruby
describe UsersController, type: :controller do
  describe "#index" do
    it "returns a successful response" do
      get :index

      expect(response.code).to eq('200')
    end
  end
end
```

Good:

```ruby
describe UsersController, type: :request do
  describe "#index" do
    it "returns a successful response" do
      user = create(:user, name: "Carlos Chagas")

      get users_path

      expect(response.code).to eq('200')
      expect(response.body).to include("Carlos Chagas")
    end
  end
end
```

---

## Expect vs should syntax

Always use `expect` instead of `should`.

Bad:

```ruby
it 'creates a resource' do
  response.code.should eq('200')
end
```

Good:

```ruby
it 'creates a resource' do
  expect(response.code).to eq('200')
end
```

---

## Redundant require

Remove any redundant `require` in your spec files. Use `.rspec` file instead.

Bad:

```ruby
# spec/models/user_spec.rb
require 'rails_helper'

describe User do
end
```

Good:

```ruby
# spec/models/user_spec.rb
describe User do
end
```

```
# .rspec
--require rails_helper
```

RuboCop: You can enforce this guideline with the custom cop [RSpec/RedundantRequireRailsHelper](https://gist.github.com/glaucocustodio/36e0fafb9137a482ec8cfdc794ca36a7).

---

## Instance double over double

Instance double ensures the object responds to methods being called, which is not true for double.

Example:

```ruby
class User
  def full_name
    "#{first_name} #{last_name}"
  end
end
```

Bad:

```ruby
it "passes" do
  user = double(:user, name: "Gustavo Kuerten")
  puts user.name
end
```

Good:

```ruby
it "fails" do
  user = instance_double(User, name: "Gustavo Kuerten")
  puts user.name
end
```

The above test will fail with the following message since the `name` method really doesn't exist.

```
the User class does not implement the instance method: name. Perhaps you meant to use `class_double` instead?
```

RuboCop: You can enforce this guideline using the cop [RSpec/VerifiedDoubles](https://docs.rubocop.org/rubocop-rspec/cops_rspec.html#rspecverifieddoubles) from the [rubocop-rspec](https://docs.rubocop.org/rubocop-rspec/installation.html) gem.

---

## Let's not

Do not use `let` / `let!`. These tend to turn your tests very complicated over time as one needs to look up variables defined then apply deltas to figure their current state. Understand more [here](https://thoughtbot.com/blog/lets-not).

Note: Tests are not supposed to be DRY, but easy to read and maintain.

Bad:

```ruby
describe '#full_name' do
  let(:user) { build(:user, first_name: 'Edson', last_name: 'Pelé') }

  context 'when first name and last name are present' do
    it 'returns the full name' do
      expect(user.full_name).to eq('Edson Pelé')
    end
  end

  context 'when last name is not present' do
    it 'returns the first name' do
      user.last_name = nil
      expect(user.full_name).to eq('Edson')
    end
  end
end
```

Good:

```ruby
describe '#full_name' do
  context 'when first name and last name are present' do
    it 'returns the full name' do
      user = build(:user, first_name: 'Edson', last_name: 'Pelé')

      expect(user.full_name).to eq('Edson Pelé')
    end
  end

  context 'when last name is not present' do
    it 'returns the first name' do
      user = build(:user, first_name: 'Edson', last_name: nil)

      expect(user.full_name).to eq('Edson')
    end
  end
end
```

RuboCop: You can enforce this guideline using the cop [RSpec/MultipleMemoizedHelpers](https://docs.rubocop.org/rubocop-rspec/cops_rspec.html#rspecmultiplememoizedhelpers) from the [rubocop-rspec](https://docs.rubocop.org/rubocop-rspec/installation.html) gem:

```ruby
# .rubocop.yml
RSpec/MultipleMemoizedHelpers:
  Max: 0
  AllowSubject: false
```

---

## Avoid hooks

Avoid hooks since they usually cause your tests to become more complex in the long run.

Bad:

```ruby
describe '#index' do
  context 'when user is authenticated' do
    before do
      @user = create(:user)
      sign_in @user
      get profile_path
    end

    context 'when user has a profile' do
      it 'returns 200' do
        create(:profile, user: @user)
        expect(response.code).to eq('200')
      end
    end

    context 'when user does not have a profile' do
      it 'returns 404' do
        expect(response.code).to eq('404')
      end
    end
  end
end
```

Good:

```ruby
describe '#index' do
  context 'when user is authenticated' do
    context 'when user has a profile' do
      it 'returns 200' do
        user = create(:user)
        create(:profile, user: user)
        sign_in user

        get profile_path

        expect(response.code).to eq('200')
      end
    end

    context 'when user does not have a profile' do
      it 'returns 404' do
        user = create(:user)
        sign_in user

        get profile_path

        expect(response.code).to eq('404')
      end
    end
  end
end
```

---

## Don't use shared examples

[Shared examples](https://rspec.info/features/3-12/rspec-core/example-groups/shared-examples/) is a feature that allow us to not repeat code, but tests are not real code, so introducing it increases the complexity of your suite.

Bad:

```ruby
shared_examples 'a barking animal' do
  it 'barks' do
    expect(animal.able_to_bark?).to eq(true)
  end
end

describe Dog do
  let(:animal) { described_class.new(able_to_bark: true) }
  it_behaves_like 'a barking animal'
end

describe Fox do
  let(:animal) { described_class.new(able_to_bark: true) }
  it_behaves_like 'a barking animal'
end
```

Good:

```ruby
describe Dog do
  describe '#able_to_bark?' do
    it 'barks' do
      subject = described_class.new(able_to_bark: true)

      expect(subject.able_to_bark?).to eq(true)
    end
  end
end

describe Fox do
  describe '#able_to_bark?' do
    it 'barks' do
      subject = described_class.new(able_to_bark: true)

      expect(subject.able_to_bark?).to eq(true)
    end
  end
end
```

---

## Mock external dependencies

Mock external dependencies to your test subject. By external dependencies I mean code that's not the main responsibility of the subject under test.

Example:

```ruby
def github_stars(repository_id)
  stars = Github.fetch_repository_stars(repository_id)
  "Stars: #{stars}"
end
```

Bad:

```ruby
describe '#github_stars' do
  it 'displays the number of stars' do
    expect(subject.github_stars(1)).to eq('Stars: 10')
  end
end
```

Good:

```ruby
describe '#github_stars' do
  it 'displays the number of stars' do
    expect(Github).to receive(:fetch_repository_stars).with(1).and_return(10)

    expect(subject.github_stars(1)).to eq('Stars: 10')
  end
end
```

Unless you intentionally want to test the underlying dependencies. This type of test must be limited to very few cases though otherwise it will make your test suite slow and hard to maintain.

Example:

```ruby
def process(payload)
  UserCreator.new(payload).create
  ProductCreator.new(payload).create
end
```

Good:

```ruby
describe '#process' do
  it 'creates a user and a product' do
    payload = { user: { name: 'Isabel' }, product: { name: 'Book' } }

    subject.process(payload)

    expect(User.count).to eq(1)
    expect(Product.count).to eq(1)
  end
end
```

---

## Stub HTTP requests

Sometimes you need to access external services. In these cases you can't rely on the real service but you should stub it with solutions like [http_decoy](https://github.com/jibranusman95/http_decoy), [webmock](https://github.com/bblimke/webmock) or [VCR](https://github.com/vcr/vcr).

Example:

```ruby
def request
  uri = URI.parse("http://www.example.com/")
  request = Net::HTTP::Post.new(uri.path)
  request['Content-Length'] = 3

  Net::HTTP.start(uri.host, uri.port) do |http|
    http.request(request, "abc")
  end
end
```

Bad:

```ruby
describe '#request' do
  response = subject.request

  expect(response.code).to eq('200')
end
```

Good:

```ruby
describe '#request' do
  stub_request(:post, "www.example.com").with(body: "abc", headers: { 'Content-Length' => 3 })

  response = subject.request

  expect(response.code).to eq('200')
end
```

---

## Stub environment variables

Prefer to stub environment variables whenever possible so your tests become more self-contained, the [stub_env](https://rubygems.org/gems/stub_env) gem might help you with that.

Example:

```ruby
def notify_sales_team
  mail(
    to: ENV['SALES_TEAM_EMAIL'],
    subject: 'New sale'
  )
end
```

Bad:

```ruby
describe '#notify_sales_team' do
  it 'prepares the email' do
    subject = described_class.notify_sales_team

    expect(subject.to).to eq(['sales-group@company.com'])
    expect(subject.subject).to eq('New sale')
  end
end
```

Bad:

```ruby
# .env.test
SALES_TEAM_EMAIL='sales-group@company.com'
```

Good:

```ruby
describe '#notify_sales_team' do
  it 'prepares the email' do
    stub_env('SALES_TEAM_EMAIL', 'sales-group@company.com')

    subject = described_class.notify_sales_team

    expect(subject.to).to eq(['sales-group@company.com'])
    expect(subject.subject).to eq('New sale')
  end
end
```

---

## Create only the data you need

Create only the data you need for each test. Having more data than necessary might make your suite slow.

Bad:

```ruby
describe '.featured_product' do
  it 'returns the featured product' do
    create_list(:product, 5)
    product_featured = create(:product, featured: true)

    expect(described_class.featured_product).to eq(product_featured)
  end
end
```

Good:

```ruby
describe '.featured_product' do
  it 'returns the featured product' do
    create(:product, featured: false)
    product_featured = create(:product, featured: true)

    expect(described_class.featured_product).to eq(product_featured)
  end
end
```

---

## Who is using it

Here are a few examples of organizations using Even Better Specs:

- [Streetbees](https://www.streetbees.com/)
- [Flowis](https://www.flowis.com/)

Does your team also use it? [Let us know!](https://github.com/evenbetterspecs/evenbetterspecs.github.io/edit/main/_includes/who_is_using_it.html)
