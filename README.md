[![Build Status](https://travis-ci.org/toptal/chewy.png)](https://travis-ci.org/toptal/chewy)
[![Code Climate](https://codeclimate.com/github/toptal/chewy.png)](https://codeclimate.com/github/toptal/chewy)

# Chewy

Chewy is ODM and wrapper for official elasticsearch client (https://github.com/elasticsearch/elasticsearch-ruby)

## Installation

Add this line to your application's Gemfile:

    gem 'chewy'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install chewy

## Usage

### Index file

1. Create `/app/chewy/users_index.rb`

  ```ruby
  class UsersIndex < Chewy::Index

  end
  ```

2. Add one or more types mapping

  ```ruby
  class UsersIndex < Chewy::Index
    define_type User.active # or just model instead_of scope: define_type User
  end
  ```

  Newly-defined index type class is accessible via `UsersIndex.user` or `UsersIndex::User`

3. Add some type mappings

  ```ruby
  class UsersIndex < Chewy::Index
    define_type User.active.includes(:country, :badges, :projects) do
      field :first_name, :last_name # multiple fields without additional options
      field :email, analyzer: 'email' # elasticsearch-related options
      field :country, value: ->(user) { user.country.name } # custom value proc
      field :badges, value: ->(user) { user.badges.map(&:name) } # passing array values to index
      field :projects, type: 'object' do # the same syntax for `multi_field`
        field :title
        field :description # default data type is `string`
      end
      field :rating, type: 'integer' # custom data type
      field :created, type: 'date', include_in_all: false,
        value: ->{ created_at } # value proc for source object context
    end
  end
  ```

  Mapping definitions - http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/mapping.html

4. Add some index- and type-related settings

  ```ruby
  class UsersIndex < Chewy::Index
    settings analysis: {
      analyzer: {
        email: {
          tokenizer: 'keyword',
          filter: ['lowercase']
        }
      }
    }

    define_type User.active.includes(:country, :badges, :projects) do
      root _boost: { name: :_boost, null_value: 1.0 } do
        field :first_name, :last_name
        field :email, analyzer: 'email'
        field :country, value: ->(user) { user.country.name }
        field :badges, value: ->(user) { user.badges.map(&:name) }
        field :projects, type: 'object' do
          field :title
          field :description
        end
        field :about_translations, type: 'object'
        field :rating, type: 'integer'
        field :created, type: 'date', include_in_all: false,
          value: ->{ created_at }
      end
    end
  end
  ```

  Index settings - http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/indices-update-settings.html
  Root object settings - http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/mapping-root-object-type.html

5. Add model observing code

  ```ruby
  class User < ActiveRecord::Base
    update_index('users#user') { self } # specifying index, type and backreference
                                        # for updating after user save or destroy
  end

  class Country < ActiveRecord::Base
    has_many :users

    update_index('users#user') { users } # return single object or collection
  end

  class Project < ActiveRecord::Base
    update_index('users#user') { user if user.active? } # you can return even `nil` from the backreference
  end

  class Bage < ActiveRecord::Base
    has_and_belongs_to_many :users

    update_index('users') { users } # if index has only one type
                                    # there is no need to specify updated type
  end
  ```

  Also, you can use second argument for method name passing:

  ```ruby
  update_index('users#user', :self)
  update_index('users#user', :users)
  ```

### Types access

You are able to access index-defined types with the following API:

```ruby
UsersIndex::User # => UsersIndex::User
UsersIndex.types_hash['user'] # => UsersIndex::User
UsersIndex.user # => UsersIndex::User
UsersIndex.types # => [UsersIndex::User]
UsersIndex.type_names # => ['user']
```

### Index manipulation

```ruby
UsersIndex.delete # destroy index if exists
UsersIndex.delete!

UsersIndex.create
UsersIndex.create! # use bang or non-bang methods

UsersIndex.purge
UsersIndex.purge! # deletes then creates index

UsersIndex::User.import # import with 0 arguments process all the data specified in type definition
                        # literally, User.active.includes(:country, :badges, :projects).find_in_batches
UsersIndex::User.import User.where('rating > 100') # or import specified users scope
UsersIndex::User.import User.where('rating > 100').to_a # or import specified users array
UsersIndex::User.import [1, 2, 42] # pass even ids for import, it will be handled in the most effective way

UsersIndex.import # import every defined type
UsersIndex.reset # purges index and imports default data for all types
```

Also if passed user is #destroyed? or specified id is not existing in the database, import will perform `delete` index for this it

### Observing strategies

There are 3 strategies for index updating: do not update index at all, update right after save and cummulative update. The first is by default.

#### Updating index on-demand

By default Chewy indexes are not updated when the observed model is saved or destroyed.
This depends on the `Chewy.urgent_update` (false by default) or on the per-model update config.
If you will perform `Chewy.urgent_update = true`, all the models will start to update elasticsearch
index right after save. Also

```ruby
class User < ActiveRecord::Base
  update_index 'users#user', 'self', urgent: true
end
```

will make the same effect for User model only.
Note than urgent update options affects only outside-atomic-block behavour. Inside
the `Chewy.atomic { }` block indexes updates as described below.

#### Using atomic updates

To perform atomic cummulative updates, use `Chewy.atomic`:

```ruby
Chewy.atomic do
  user.each { |user| user.update_attributes(name: user.name.strip) }
end
```

Index update will be performed once per Chewy.atomic block for every affected type.
This strategy is highly usable for rails actions:

```ruby
class ApplicationController < ActionController::Base
  around_action { |&block| Chewy.atomic(&block) }
end
```

Also atomic blocks might be nested and don't affect each other.

### Index querying

```ruby
scope = UsersIndex.query(term: {name: 'foo'})
  .filter(numeric_range: {rating: {gte: 100}})
  .order(created: :desc)
  .limit(20).offset(100)

scope.to_a # => will produce array of UserIndex::User or other types instances
scope.map { |user| user.email }
scope.total_count # => will return total objects count

scope.per(10).page(3) # supports kaminari pagination
scope.explain.map { |user| user._explanation }
scope.only(:id, :email) # returns ids and emails only

scope.merge(other_scope) # queries could be merged
```

Also, queries can be performed on a type individually

```ruby
UsersIndex::User.filter(term: {name: 'foo'}) # will return UserIndex::User collection only
```

See [query.rb](lib/chewy/query.rb) for more info.

### Filters query DSL.

There is a test version of filters creating DSL:

```ruby
UsersIndex.filter{ name == 'Fred' } # will produce `term` filter.
UsersIndex.filter{ age <= 42 } # will produce `range` filter.
```

See [context.rb](lib/chewy/query/context.rb) for more info.

### Objects loading

It is possible to load source objects from database for every search result:

```ruby
scope = UsersIndex.filter(numeric_range: {rating: {gte: 100}})

scope.load # => will return User instances array (not a scope because )
scope.load(user: { scope: ->(_) { includes(:country) }}) # you can also pass loading scopes for each
                                                         # possibly returned type
scope.load(user: { scope: User.includes(:country) }) # the second scope passing way
scope.only(:id).load # it is optimal to request ids only if you are not planning to use type objects
```

### Rake tasks

Inside Rails application some index mantaining rake tasks are defined.

```bash
rake chewy:reset:all # resets all the existing indexes, declared in app/chewy
rake chewy:reset[users] # resets UsersIndex
rake chewy:update[users] # updates UsersIndex
```

### Rspec integration

Just add `require 'chewy/rspec'` to your spec_helper.rb and you will get additional features:

#### `update_index` matcher

```ruby
# just update index expectation. Used type class as argument.
specify { expect { user.save! }.to update_index(UsersIndex.user) }
# expect do not update target index. Type for `update_index` might be specified via string also
specify { expect { user.name = 'Duke' }.not_to update_index('users#user') }
# expecting update specified objects
specify { expect { user.save! }.to update_index(UsersIndex.user).and_reindex(user) }
# you can specify even id
specify { expect { user.save! }.to update_index(UsersIndex.user).and_reindex(42) }
# expected multiple objects to be reindexed
specify { expect { [user1, user2].map(&:save!) }
  .to update_index(UsersIndex.user).and_reindex(user1, user2) }
specify { expect { [user1, user2].map(&:save!) }
  .to update_index(UsersIndex.user).and_reindex(user1).and_reindex(user2) }
# expect object to be reindexed exact twice
specify { expect { 2.times { user.save! } }
  .to update_index(UsersIndex.user).and_reindex(user, times: 2) }
# expect object in index to be updated with specified fields
specify { expect { user.update_attributes!(name: 'Duke') }
  .to update_index(UsersIndex.user).and_reindex(user, with: {name: 'Duke'}) }
# combination of previous two
specify { expect { 2.times { user.update_attributes!(name: 'Duke') } }
  .to update_index(UsersIndex.user).and_reindex(user, times: 2, with: {name: 'Duke'}) }
# for every object
specify { expect { 2.times { [user1, user2].map { |u| u.update_attributes!(name: 'Duke') } } }
  .to update_index(UsersIndex.user).and_reindex(user1, user2, times: 2, with: {name: 'Duke'}) }
# for every object splitted
specify { expect { 2.times { [user1, user2].map { |u| u.update_attributes!(name: "Duke#{u.id}") } } }
  .to update_index(UsersIndex.user)
    .and_reindex(user1, with: {name: 'Duke42'}) }
    .and_reindex(user2, times: 1, with: {name: 'Duke43'}) }
# object deletion same abilities as `and_reindex`, except `:with` option
specify { expect { user.destroy! }.to update_index(UsersIndex.user).and_delete(user) }
# double deletion, whatever it means
specify { expect { 2.times { user.destroy! } }.to update_index(UsersIndex.user).and_delete(user, times: 2) }
# alltogether
specify { expect { user1.destroy!; user2.save! } }
  .to update_index(UsersIndex.user).and_reindex(user2).and_delete(user1)
```

```ruby
# strictly specifing updated and deleted records
specify { expect { [user1, user2].map(&:save!) }
  .to update_index(UsersIndex.user).and_reindex(user1, user2).only }
specify { expect { [user1, user2].map(&:destroy!) }
  .to update_index(UsersIndex.user).and_delete(user1, user2).only }
# this will fail
specify { expect { [user1, user2].map(&:save!) }
  .to update_index(UsersIndex.user).and_reindex(user1).only }
```

## TODO a.k.a coming soon:

* Dynamic templates additional DSL
* Typecasting support
* Advanced (simplyfied) query DSL: `UsersIndex.query { email == 'my@gmail.com' }` will produce term query
* update_all support
* Other than ActiveRecord ORMs support (Mongoid)
* Maybe, closer ORM/ODM integration, creating index classes implicitly
* Better facets support

## Contributing

1. Fork it ( http://github.com/toptal/chewy/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Implement your changes, cover it with specs and make sure old specs are passing
4. Commit your changes (`git commit -am 'Add some feature'`)
5. Push to the branch (`git push origin my-new-feature`)
6. Create new Pull Request
