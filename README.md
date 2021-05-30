# Active Record Validations

## Learning Goals

After this lesson, you should be able to:

- Explain the purpose of validation
- Identify when validation occurs in the lifespan of an object
- Use built-in validation methods
- Add custom validation errors

## Context: Databases and Data Validity

What is a "validation"?

In the context of Rails, **validations** are special method calls that go at the top
of model class definitions and prevent them from being saved to the database if
their data doesn't look right.

In general, **validations** consist of code that performs the job of protecting
the database from invalid data.

Active Record can validate our models for us before they even touch the database.
This means it's harder to end up with bad data, which can cause problems later
even if our code is technically bug-free.

We can use `ActiveRecord::Base` helper methods like `#validates` to set things
up.

### Active Record Validations vs Database Constraints

Many relational databases, such as SQLite and PostgreSQL, have data validation
features that check things like length and data type. While Active Record
validations are added in the model files, these validations are typically added
via migrations; depending on the specific validation, they may or may not be
reflected in the `schema.rb` file.

Database constraints and model validations are also functionally different.
Database constraints will ALWAYS be checked when adding or updating data in the
database, while Active Record validations will only be checked when adding or
updating data through Ruby/Rails (e.g. if we use SQL code in the command line to
modify the database, Active Record validations are not run).

Some developers use both database constraints and Active Record validations,
while others rely on Active Record validations alone. Ultimately, it depends on
how the developer plans to add and update data in the database. In this lesson,
we'll be focusing on Active Record validations.

### What is "invalid data"?

Suppose you get a new phone and you ask all of your friends for their phone
number again. One of them tells you, "555-868-902". If you're paying attention,
you'll probably wrinkle your nose and think, "Wait a minute. That doesn't sound
like a real phone number."

"555-868-902" is an example of **invalid data**... for a phone number. It's
probably a valid account number for some internet service provider in Alaska,
but there's no way to figure out what your friend's phone number is from those
nine numbers. It's a showstopper, and even worse, it kind of looks like valid
data if you're not looking closely.

### Validations Protect the Database

Invalid data is the bogeyman of web applications: it hides in your database
until the worst possible moment, then jumps out and ruins everything by causing
confusing errors.

Imagine the phone number above being saved to the database in an application
that makes automatic calls using the Twilio API. When your system tries to call
this number, there will be an error because no such phone number exists, which
means you need to have an entire branch of code dedicated to handling _just_
that edge case.

It would be much easier if you never have bad data in the first place, so you
can focus on handling edge cases that are truly unpredictable.

That's where validations come in.

## Basic Usage

For more examples of basic validation usage, see the Rails Guide for
[Active Record Validations][active record validations]. Take a few minutes to
browse the helpers listed in Section 2.

```ruby
class Person < Active Record::Base
  validates :name, presence: true
end

Person.create(name: "John Doe").valid?
# => true
Person.create(name: nil).valid?
# => false
```

`#validates` is our Swiss Army knife for validations. It takes two arguments:
the first is the **name of the attribute** we want to validate, and the second
is a **hash of options** that will include the details of how to validate it.

In this example, the options hash is `{ presence: true }`, which implements the
most basic form of validation, preventing the object from being saved if its
`name` attribute is empty.

## Lifecycle Timing

Before proceeding, keep the answer to this question in mind:

**What is the difference between `#new` and `#create`?**

If you've forgotten, `#new` instantiates a new Active Record model _without_
saving it to the database, whereas `#create` immediately attempts to save it, as
if you had called `#new` and then `#save`.

**Database activity triggers validation**. An Active Record model instantiated
with `#new` will not be validated, because no attempt to write to the database
was made. Validations won't run unless you call a method that actually hits the
DB, like `#save`.

The only way to trigger validation without touching the database is to call the
`#valid?` method.

For a full list of methods that trigger validation, see
[Section 4][active record callbacks] of the Rails Guide for Active Record
Callbacks. Don't worry about the rest of the information in that guide just yet;
we'll go into callbacks later!

[active record callbacks]: http://guides.rubyonrails.org/active_record_callbacks.html#running-callbacks

## Validation Failure

Here it is, the moment of truth. What can we do when a record fails validation?

### How can you tell when a record fails validation?

**Pay attention to return values!**

By default, Active Record does not raise an exception when validation fails. DB
operation methods (such as `#save`) will simply return `false` and decline to
update the database.

Every database method has a sister method with a `!` at the end which will raise
an exception (`#create!` instead of `#create` and so on).

And of course, you can always check manually with `#valid?`.

```ruby
class Person < Active Record::Base
  validates :name, presence: true
end

person = Person.new
person.valid? #=> false
person.save #=> false
person.save! #=> EXCEPTION
```

### Finding out why validations failed

To find out what went wrong, you can look at the model's `#errors` object.

You can check all errors at once by examining `errors.messages`.

```ruby
person = Person.new
person.errors.messages #=> empty
person.save #=> false
person.valid? #=> false
person.errors.messages #=> name: can't be blank
```

You can check one attribute at a time by passing the name to `errors` as a key,
like so:

```ruby
person.errors[:name]
```

## Rendering Validation Errors As JSON

When a validation error occurs on the server, we can give our users more details
about what went wrong by returning the validation errors from our controllers:

```rb
def create
  person = Person.create(person_params)
  if person.valid?
    render json: person, status: :created
  else
    render json: { error: person.errors }, status: :unprocessable_entity
  end
end
```

Since we're sending these errors in the response, we can then use the errors
object on the frontend to display these errors to our users.

We can also get an array of nicely-formatted errors with the `.full_messages`
method:

```rb
def create
  person = Person.create(person_params)
  if person.valid?
    render json: person, status: :created
  else
    render json: { errors: person.errors.full_messages }, status: :unprocessable_entity
  end
end
```

Also, if you're a fan of using `rescue` for control flow instead of `if/else`,
you can use the `ActiveRecord::RecordInvalid` exception class along with
`create!` or `update!`:

```rb
def create
  person = Person.create!(person_params)
  render json: person, status: :created
rescue ActiveRecord::RecordInvalid => invalid
  render json: { errors: invalid.record.errors.full_messages }, status: :unprocessable_entity
end
```

We'll cover error handling in controllers in more detail the next lesson.

## Other Built-in Validators

Rails has a host of built-in helpers.

### Length

`length` is one of the most versatile:

```ruby
class Person < Active Record::Base
  validates :name, length: { minimum: 2 }
  validates :bio, length: { maximum: 500 }
  validates :password, length: { in: 6..20 }
  validates :registration_number, length: { is: 6 }
end
```

The `in` argument makes use of a [Range][ruby-core-range].

[ruby-core-range]: http://ruby-doc.org/core/Range.html

Remember that there's no syntactical magic happening with any of these method
calls. If we weren't using Ruby's "poetry mode" (which is considered standard
for Rails), the above code would look like this:

```ruby
class Person < Active Record::Base
  validates(:name, { :length => { :minimum => 2 } })
  validates(:bio, { :length => { :maximum => 500 } })
  validates(:password, { :length => { :in => 6..20 } })
  validates(:registration_number, { :length => { :is => 6 } })
end
```

Phew!

### Uniqueness

Another common built-in validator is `uniqueness`:

```ruby
class Account < Active Record::Base
  validates :email, uniqueness: true
end
```

This will prevent any account from being created with the same email as another already-existing account.

### Custom Messages

This isn't a validator in its own right, but a handy convenience option for specifying your own error messages:

```ruby
class Person < Active Record::Base
  validates :not_a_robot, acceptance: true, message: "Humans only!"
end
```

### Custom Validations

There are two ways to implement custom validations, with examples in [Section
6][active record custom validations] of the Rails Guide.

Of the two, adding custom methods is the simplest. Here's an example of creating
a custom validation method to check the validity of an email address:

```ruby
class Person
  validate :must_have_flatiron_email

  def must_have_flatiron_email
    unless email.match?(/flatironschool.com/)
      errors.add(:email, "We're only allowed to have people who work for the company in the database!")
    end
  end
end
```

Note that here, we are calling the `#validate` method, rather than `#validates`,
and passing it a method we write ourselves to perform our custom validation.

[active record custom validations]: http://guides.rubyonrails.org/active_record_validations.html#performing-custom-validations

## Conclusion

In this lesson, we learned the importance of validating data to ensure that no
bad data ends up in our database. We also discussed the difference between model
validations and database constraints. Finally, we saw some common methods for
implementing validations on our models using Active Record.

## Resources

- [Active Record Validations][active record validations]

[active record validations]: http://guides.rubyonrails.org/active_record_validations.html
