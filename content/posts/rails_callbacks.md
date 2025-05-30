+++
date = '2025-05-30T16:33:52-06:00'
draft = true
title = 'Rails Callbacks'
+++

## What is a callback?
In Ruby on Rails, a callback is a method or unit of code you can connect to the lifecycle events of Active Record models. For example, if in your application you want to send a welcome email after a user is first created, you could use a callback for that. It would look something like this:

### Example 1
```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  def create
    user = User.create(user_params)
    if user.save
      render user
    else
      render user.errors
    end
  end
end

# app/models/user.rb
class User < ApplicationRecord
  after_create :send_welcome_email

  def send_welcome_email
    UserMailer.with(user: self)
      .welcome
      .deliver_later
  end
end
```

## Why are callbacks used?
If you didn't use a callback in the example above, you would need to find every place in your codebase a user was created and add a call to the UserMailer after. In a simple application, this might be 1 place in the code, the UsersController, but it could be more in practice. In addition, many would consider calling a method to send an email a side effect of creating a user that does not belong in a controller. It makes sense to use a callback so that logic belongs to the User model, rather than the controller.

### Example 2
```ruby
class UsersController < ApplicationController
  def create
    user = User.create(user_params)
    if user.save
      UserMailer.with(user: user)
          .welcome
          .deliver_later
      render user
    else
      render user.errors
    end
  end
end
```

## What problems do callbacks cause? 
If you have every read opinions online on the use of callbacks in Ruby on Rails applications, you know they are divisive. The simple example presented above may not cause problems, but let's examine some problems callbacks can cause as your application scales

### Indirection 
The code in example 2 reads top to bottom without needing to switch files. A User is created, and if it is created successfully, the welcome mailer is sent. In example 1, if you didn't know about callbacks you might now know when or where in the code the users mailer is being sent. Even though this approach follows the best practice of reducing side effects in controllers, the reader must 'know' that side effects of creating a User may be in the User model.
### Complexity 
Example 1 contains a bug. When the job to send the user mailer runs, it might be before the transaction to commit the user record to the database has finished. You would see an ActiveRecord::RecordNotFound error then, and the job would retry. The correct solution to this would be to use the after_commit callback, since it runs after the transaction has committed. This is an example of one of the many gotchas around callbacks. Here are some more complications around callbacks that have caused production errors in applications I maintained:

#### Execution Order 
In the following script, what order will the callbacks run in?

```ruby
class User < ActiveRecord::Base
  after_create :this_happens_first
  after_create :then_this

  def this_happens_first
    puts 'first'
  end

  def then_this
    puts 'second'
  end
end

User.create(name: 'John Lennon', age: 90)
# => 'first'
# => 'second'
```
If the example used after_commit callbacks, they would still run in the same order, right? Wrong! In the example below, this didn't change the outcome, but in real production applications it can. It's a good illustration of the kinds of problems associated with having multiple callbacks

```ruby
class User < ActiveRecord::Base
  after_commit :this_happens_first
  after_commit :then_this

  def this_happens_first
    puts 'first'
  end

  def then_this
    puts 'second'
  end
end

User.create(name: 'John Lennon', age: 90)
# => 'second'
# => 'first'
```

#### Error Behavior 
Errors in 1 callback will affect others in the callback chain. An unhandled exception, as are so common in legacy applications, could result in your critical business processes never running at all.

```ruby
class User < ActiveRecord::Base
  after_commit :this_never_happens
  after_commit :because_this_raises_an_error

  def because_this_raises_an_error 
    raise 'an error occurred'
  end

  def this_never_happens 
    puts 'super important business process'
  end
end

begin
  User.create(name: 'John Lennon', age: 90)
rescue RuntimeError => e
  puts e  
end
# => 'an error occurred'
```

### Too sharp a knife
It's easy to think that you want the welcome email to send every time a user is created, but exceptions will pop up over time. Long after you have implemented the callback to send the email after creating a user, your business or operations team will need to bulk onboard some users. No problem, you say, as you write a quick script to iterate over their spreadsheet and create some users. However, when you run it in production you end up overwhelming your mail provider, erroring out, and sending some unexpected emails. Sure, adding logic to skip the callback in certain cases is easy, but you are adding complexity to what seemed so simple.

## Alternatives
To me, callbacks are Rails' answer to the question of where do you put stuff that isn't a CRUD action, AKA your business logic. There are a lot of opinions on alternatives to using callbacks for business logic, which I will explore below.

### Service Objects
Service objects can open up a can of worms as to what they even are, but in this case I am referring to encapsulating the business process of creating a user, for example, into a piece of procedural code, like so:

```ruby
class CreateUser
  def initialize(user_params)
    @user_params = user_params
  end
  def call
    user = User.create!(user_params)
    UserMailer.with(user: user)
      .welcome
      .deliver_later unless user.import? 
    # ...
  end
end
```

This has the advantage of being able to read top to bottom without changing files, avoids any complication due to database transactions, and will be easier to test than each other example presented so far.

Refactoring to this approach in an existing application would involve finding where you are creating records and calling this class instead, and maintaining that strategy moving forward.

### Event Based Architecture

Some would disagree with the Rails convention of coupling business logic and CRUD altogether and propose an event-based architecture. According to the Single Responsibility Principle, none of the examples above *really* have one responsibility since they trigger the welcome email in one way or another. An event-based solution might look like the User model publishing a 'User #1 Created Event' to a stream and a process separate from Rails consuming that event stream and running code based on the events. I'm not going to include an example here as I haven't too many well-supported libraries in the Ruby ecosystem supporting event-based logic that would also be succinct enough to make a good example. It seems like the most likely choice moving forward will be something like the [Socketry Async framework](https://github.com/socketry/async) as the concept of Fibers becomes more fleshed out. 

To wrap it up, I would make the argument that callbacks are fine when limited to 1 or 2 callbacks per model lifecycle but the increase in complexity as applications grow requires care to avoid the problems explained here. 

