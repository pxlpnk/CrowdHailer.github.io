---
layout: post
title: A persistence problem
description: Solutions with entities, Repositories and ruby
date: 2015-07-23 17:20:06
tags: ruby design
author: Peter Saxton
---

### The worst advice I have followed

> Fat Model, Skinny controller

This advice is only good advice in that a fat model is slightly preferable to a fat controller. When I started creating projects with Rails I found that I would have a single model for a single database table. As OOP is about packaging behaviour with data as a database table grows then a data in the model grows and the bahviour on the model grows. Quite soon you end up with a fat model and shorty afterwards a God Object. God objects are objects that seam to know and act upon all parts of the system. They are an antipattern that are in violation of the Single responsibility principle.

Recently I have seen people blaiming Rails and particularly ActiveRecord(the library) for the encouraging god objects. ActiveRecord is a beast but also a fantastic piece of software. Most importantly it is a tool and doesn't encourage god object however it certainly does faciliate making that mistake if you are unaware. What does encourage god objects is simplistic advice like the fat model skiney controller.

So what is the solution? well I think that it is small, well defined, single responsibility objects everywhere.

### The persistence problem

Handling the data in a single database table, sounds like a single responsibility. That is a valid starting point but with anything beyond a trivial application it is too simplistic. To break it down further lets look at some of the key functionality we need when handling with a database table

- Data serialization. The database will not understand our domain objects, so we need to transform them too and from a format that the database does understand. We should do this as near the database as possible so that the database has the smallest possible influence on the rest of our code.

- Querying. We will want to be able to fetch the correct items from out database. This may be simple or it may involve filtering on many fields, such as find all the declined orders for this month for customers who signed up more than a year ago.

- Buisness Logic. What you want to acomplish with your data, this is specific to your problem an as such there is no limit to how complex this might be.

### Building a solution
First we want to separate querying, this functionality acts on the whole database table while the other two parts need to act of individual entries. To handle querying we will introduce repositories in the new post.

To split the last two I employ the decorator pattern. I have a core Record object that formats the data saved to it. Record objects are just dumb data buckets. Business logic is build in to entities each entity wraps a single record and has no state other that the record object it is decorating.

#### Records
These are the only objects anywhere in my system that know how data is stored in the database. We often don't think about databases as out of our control but we did not build them an so the api is not ours and may not be the best representation of data in our domain. For this reason record objects act as gatekeepers in a way analagous to form objects.

The only logic a record object should have should be packing domain objects into the database and initializing domain objects from entries in the database.

Talking to a database is a solved problem and so I use ActiveRecord or more recently Sequel to talk to the database. Both have many features that allow them to act as far more than a data bucket. It is only by being disciplined that the functionality of a record remains serialization only.

The benefits or using an ORM library is that I can write records that are very simple often looking like

```rb
# {% highlight ruby %}
class UserRecord < Sequel::Model(:users)
end
# {% endhighlight %}
```

#### Entities

These objects represent business logic and so have very little in common with each other. The only consistent feature is that the are initialized with a record and that they cannot have any state on them selves they must always update the record if something changes.

Multiple entity class can be created for a single record class. These entities are separated by business concerns and because there is not limit to how many can be made a system can be built up of small cohesive parts regardless of total scope.

Code should never need to interact with a record directly but only with an entity that is decorating it. Good DDD code should communicate in a language understood by the client. They will understand user but not 'user entity' or 'user record' for this reason the user is represented by the user entity and the user record is just an implementation detail

### Examples
A very typical example would be similar to the following. A user represents the information kept in the users database table. There is a good amount of meta information associated with the account such as authentication credentials.

Following good design principles we have dedicated value objects for email, password and name. This means we have moved validation away from our entities. Our record layer will happily dump and load these value objects to the database. However features continue to be added our user needs a combined full name and also the ability to authenticate. These methods are not part of the same set of responsibility and so we look for a way to separate them. Authentication is often a good candidate to be separated to a single purpose entity. In this example we have done just that and named the extracted class 'Credentials'.  

```rb
# {% highlight ruby %}
class UserRecord < Sequel::Model(:users)
  # Turns database friendly representations of values in to rich objects specific to our domain
  plugin :serialization

  serialize_attributes [Email.method(:dump), Email.method(:load)], :email
  serialize_attributes [Password.method(:dump), Password.method(:load)], :password
  serialize_attributes [Name.method(:dump), Name.method(:load)], :first_name, :last_name
end

class User
  # Enities make no sense if they don't also have a record to preserve state
  def initialize(record)
    @record = record
  end

  attr_reader :record
  private :record

  # Behaviour that the user can handle itself.
  def full_name
    "#{record.first_name} #{record.last_name}"
  end

  def full_name=(full_name)
    first_name, last_name = full_name.split ' '
    record.first_name = first_name
    record.last_name = last_name
  end

  # Behaviour that the user delegates to a dedicated credentials object
  def authenticate(submitted_password)
    credentials.authenticate submitted_password
  end

  # The credentials object also uses the same record
  def credentials
    @credentials ||= Credentials.new record
  end
end

class Credentials
  # Logic for authentication can grow and is cleanly separated from the main user object
  def authenticate(submitted_password)
    return nil unless correct_password? submitted_password
    record_login
    self
  end

  private

  def correct_password?(submitted_password)
    record.password == submitted_password
  end

  def record_login_at(date_time = DateTime.now)
    self.last_login_at = date_time
  end
end

# {% endhighlight %}
```

### Testing
Testing records is done in much the same way as you would have tested methods on any model. The main difference is that there should be much fewer tests and they should be simply setting a value and testing the same value is retrieved. Tests that hit the database are slower because of there interaction with the database. I don't stub the database out at this level as a recod is only about talking to the database. The tests are simple and not too numerous and so these slower tests are not a problem.

Testing entities however can easily be separated from the database. In my test I always initialize the enitity with an open struct to stand in for the record. There are then two kinds of tests, setting values on the mock record and testing the entities behaviour, or setting values on the entity and checking the correct values are represented on the mock record

```rb
# {% highlight ruby %}
class UserTest < Minitest::Test
  def record
    @record ||= OpenStruct.new
  end

  def user
    @user = User.new record
  end

  def teardown
    @user = nil
    @record = nil
  end

  def test_user_has_correct_full_name
    record.first_name = 'Peter'
    record.last_name = 'Saxton'
    assert_equal 'Peter Saxton', user.full_name
  end

  def test_user_sets_first_name_on_record
    user.full_name = 'Peter Saxton'
    assert_equal 'Peter', record.first_name
  end

  def test_user_sets_last_name_on_record
    user.full_name = 'Peter Saxton'
    assert_equal 'Saxton', record.last_name
  end
end
# {% endhighlight %}
```

### Conclusion

This is my take on entities and records. I encourage you to give them a try as multiple entities can be an effective cure for god objects. This is not quite the whole persistence problem solved as we still need to pull the correct records out of our database. That is my next post and it's on repositories.

### Resources

- [How I use ActiveRecord serialize with custom data types]( http://viget.com/extend/how-i-used-activerecord-serialize-with-a-custom-data-type)  
  Explaining the serialize method in active record by [Zachary Porter](https://twitter.com/zporter9)


http://smashingboxes.com/ideas/domain-logic-in-rails
https://medium.com/@KamilLelonek/why-is-your-rails-application-still-coupled-to-activerecord-efe34d657c91
http://victorsavkin.com/post/41016739721/building-rich-domain-models-in-rails-separating