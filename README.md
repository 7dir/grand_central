# GrandCentral

GrandCentral is a state-management and action-dispatching library for Ruby apps. It was created with [Clearwater](https://github.com/clearwater-rb/clearwater) apps in mind, but there's no reason you couldn't use it with other types of Ruby apps.

GrandCentral is based on ideas similar to [Redux](http://rackt.github.io/redux/). You have a central store that holds all your state. This state is updated via a handler block when you dispatch actions to the store.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'grand_central'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install grand_central

## Usage

First, you'll need a store. You'll need to seed it with initial state and give it a handler block:

```ruby
require 'grand_central'

store = GrandCentral::Store.new(a: 1, b: 2) do |state, action|
  case action
  when :a
    # Notice we aren't updating the state in-place. We are returning a new
    # value for it by passing a new value for the :a key
    state.merge a: state[:a] + 1
  when :b
    state.merge b: state[:b] + 1
  else # Always return the given state if you aren't updating it.
    state
  end
end

store.dispatch :a
store.dispatch :b
store.dispatch "You can dispatch anything you want, really"
```

### Actions

The actions you dispatch to the store can be anything. We used symbols in the above example, but GrandCentral also provides a class called `GrandCentral::Action` to help you set up your actions:

```ruby
module Actions
  include GrandCentral

  # The :todo becomes a method on the action, similar to a Struct
  AddTodo    = Action.with_attributes(:todo)
  DeleteTodo = Action.with_attributes(:todo)
  ToggleTodo = Action.with_attributes(:todo) do
    # We don't want to toggle the Todo in place. We want a new instance of it.
    def toggled_todo
      Todo.new(
        id: todo.id,
        name: todo.name,
        complete: !todo.complete?
      )
    end
  end
end
```

Then your handler can use these actions to update the state more easily:

```ruby
store = GrandCentral::Store.new(todos: []) do |state, action|
  case action
  when Actions::AddTodo
    state.merge todos: state[:todos] + [action.todo]
  when Actions::DeleteTodo
    state.merge todos: state[:todos] - [action.todo]
  when Actions::ToggleTodo
    state.merge todos: state[:todos].map { |todo|
      # We want to replace the todo in the array
      if todo.id == action.todo.id
        action.todo
      else
        todo
      end
    }
  else
    state
  end
end
```

### Performing actions on dispatch

You may want your application to do something in response to a dispatch. For example, in a Clearwater app, you might want to re-render the application when the store's state has changed:

```ruby
store = GrandCentral::Store.new(todos: []) do |state, action|
  # ...
end

app = Clearwater::Application.new(component: Layout.new)

store.on_dispatch do |old_state, new_state|
  app.render unless old_state.equal?(new_state)
end
```

Notice the `unless old_state.equal?(new_state)` clause. This is one of the reasons we recommend you update state by returning a new value instead of mutating it in-place. It allows you to do cache invalidation in O(1) time.

## Models

We can use the `GrandCentral::Model` base class to store our objects:

```ruby
class Person < GrandCentral::Model
  attributes(
    :id,
    :name,
    :location,
  )
end
```

This will set up a `Person` class we can instantiate with a hash of attributes:

```ruby
jamie = Person.new(name: 'Jamie')
```

### Immutable Models

The attributes of a model cannot be modified once set. That is, there's no way to say `person.name = 'Foo'`. If you need to change the attributes of a model, there's a method called `update` that returns a new instance of the model with the specified attributes:

```ruby
jamie = Person.new(name: 'Jamie')
updated_jamie = jamie.update(location: 'Baltimore')

jamie.location         # => nil
updated_jamie.location # => "Baltimore"
```

This allows you to use the `update` method in your store's handler without mutating the original reference:

```ruby
store = GrandCentral::Store.new(person) do |person, action|
  case action
  when ChangeLocation
    person.update(location: action.location)
  else person
  end
end
```

This keeps each version of your app state intact if you need to roll back to a previous version. In fact, the app state itself can be a `GrandCentral::Model`:

```ruby
class AppState < GrandCentral::Model
  attributes(
    :todos,
    :people,
  )
end

initial_state = AppState.new(
  todos: [],
  people: [],
)

store = GrandCentral::Store.new(initial_state) do |state, action|
  case action
  when AddPerson
    state.update(people: state.people + [action.person])
  when DeleteTodo
    state.update(todos: state.todos - [action.todo])

  else
    state
  end
end
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/clearwater-rb/grand_central. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](contributor-covenant.org) code of conduct.

