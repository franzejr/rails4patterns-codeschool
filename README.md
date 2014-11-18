rails4patterns-codeschool
=========================


## Level 1 : Models

##### Fat Controllers are bad

- Hard to understand
- Business logic not encapsuled
- More conflicts in code
- New features hard to implement


##### Avoid calling other domain objects from callbacks

Callbacks are methods that get called at certain moments of an object's life cycle.

Referencing other models in a callback:
- Introduces tight coupling
- Affects the object's database lifecycle


##### Callbacks should only be for modifying internal state

Remembering that it's a convention to set callback methods as protected

```ruby
class Item < ActiveRecord::Base
  before_save :set_pretty_url
  
  protected
  def set_pretty_url
    self.pretty_url =self.name.parameterize
  end
end
```


Refactoring this controller:
```ruby
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)

    if valid_background_check?
      @user.is_approved = true
    end

    if @user.save
      redirect_to @user
    else
      render :new
    end
  end

  private

  def valid_background_check?
    !!(@user.valid_ssn? || @user.valid_address?)
  end

  def user_params
    params.require(:user).permit(:name, :email, :ssn, :address)
  end
end
```

Creating the following class(non-AR class):

```ruby
class UserRegistration
  attr_reader :user
  
  def initialize user
    @user = User.new user
  end

  private
  def valid_background_check?
		!!(@user.valid_ssn? || @user.valid_address?)
  end
end
```


##### Refactoring

```ruby
class UsersController < ApplicationController
  def create
    registration = UserRegistration.new(user_params)
    @user = registration.user
    if registration.create
      redirect_to @user
    else
      render :new
    end
  end

  private

  def user_params
    params.require(:user).permit(:name, :email, :ssn, :address)
  end
end
```

```ruby
class UserRegistration
  attr_reader :user

  def initialize(params)
    @user = User.new(params)
  end

  def create
    if valid_background_check?
      user.is_approved = true
    end

    user.save
  end

  private

  def valid_background_check?
    !!(@user.valid_ssn? || @user.valid_address?)
  end
end
```


## Level 2 : Scopes and methods

##### Extract Query


```ruby
class ItemsController < ApplicationController
  def index
    @items = Item.where('rating > ? AND published_on > ?', 5, 2.days.ago)
  end
end
```

We can extract the query into a model
```ruby
class Item < ActiveRecord::Base
  
  def self.featured
    where('rating > ? AND published_on > ?', 5, 2.days.ago)
  end
end
```

##### Class methods vS Scopes

Two class methods:
```ruby
class Item < ActiveRecord::Base
  def self.by_name(name)
    where(name: name) if name.present?
  end

  def self.recent
    where('created_on > ?', 2.days.ago)
  end
end
```

```ruby
class Item < ActiveRecord::Base
  scope :by_name,->(name){where(name: name) if name.present?}
  scope :recent, -> { where('created_on > ?', 2.days.ago)}
end
```


##### Use Merge to combine conditions from different Models

```ruby
class Item < ActiveRecord::Base
  has_many :reviews
  
  scope :recent, ->{
    where('published_on > ?', 2.days.ago)
    .joins(:reviews).merge(Review.approved)
  }
end
```


```ruby
class Review < ActiveRecord::Base
  belongs_to :item
  scope :approved, -> { where(approved: true) }
end
```

##### Merging scopes

When upgradign to Rails 4, you run into this bug because Rails no longe automatically removes duplication from SQL conditions.

Example:

```ruby
class User < ActiveRecord::Base
	scope :active, -> {where(state:'active')}
	scope :inactive, -> {where(state:'inactive')}
end
```

RAILS 3:
(query in the last scope overrides the first one)

User.active.inactive

RAILS 4:

(appends conditions)

User.active.inactive
SELECT * from users where state = 'active' AND state='inactive'


User.active.merge(User.inactive)
SELECT * from users where state = 'inactive'


## Level 3 : Concerns

https://signalvnoise.com/posts/3372-put-chubby-models-on-a-diet-with-concerns

##### Move shared model code into model concerns


app/models/concerns/commentable.rb
```ruby
module Commentable 
	extend ActiveSupport::Concern
	
	module ClassMethods
		def upvote(comment)
			...
		end
	end

end
```

Methods declared inside of this inner module become class methods on target class

Post.upvote(@comment)
Image.upvote(@comment)



ActiveSupport::Concern automatically includes methods from the ClassMethods module as class methods on the target class


##### Controllers concerns

```ruby
class UsersController < ApplicationController
  include ImageExportable
  def file
    user = User.find(params[:id])
    send_image(user.image)
  end
end
```


```ruby
module ImageExportable
  def send_image(image_path)
    send_file(image_path, :type => 'image/jpeg',  :disposition => 'inline')
  end
end
```


## Level 4 : Decorators

http://robots.thoughtbot.com/tidy-views-and-beyond-with-decorators


Decorators attach presentation logic to an object dynamically
- Transparent to clients
- "Wrap" another object
- Delegate most methods to the wrapped object
- Provide one or two methods of their own

##### Logic view doest not belong in model

```ruby
class Item < ActiveRecord::Base
  def is_featured?
    self.ratings > 5
  end
end
```


##### Refactoring to a decorator
```ruby
class ItemDecorator
  def initialize(item)
    @item = item
  end
  
  def is_featured?
    @item.ratings > 5
  end
  
  def respond_to_missing?(method_name, include_private=false)
    @item.respond_to?(method_name, include_private) || super
  end
  
  def method_missing(method_name, *args, &block)
    @item.send(method_name,*args,&block)
  end
end
```


Examples

##### First Example

```html
<h2><%= @item_decorator.title %></h2>

<% if @item_decorator.is_featured? %>
  <h3><%= featured_image %></h3>
<% end %>

<p><%= @item_decorator.description %></p>
```


```ruby
class ItemController < ApplicationController
  def show
    item = Item.find(params[:id])
    @item_decorator = ItemDecorator.new(item)
  end
end
```

```ruby
class ItemDecorator
  def initialize(item)
    @item = item
  end

  def is_featured?
    @item.ratings > 5
  end

  def method_missing(method_name, *args, &block)
    @item.send(method_name, *args, &block)
  end

  def respond_to_missing?(method_name, include_private = false)
    @item.respond_to?(method_name, include_private) || super
  end
end
```

##### Second Example

```ruby
class ItemDecorator

  def initialize(item)
    @item = item
  end

  def status
    if @item.sold?
      "Sold on #{@item.sold_on.strftime('%A, %B %e')}"
    else
      "Available"
    end
  end

  def method_missing(method_name, *args, &block)
    @item.send(method_name, *args, &block)
  end

  def respond_to_missing?(method_name, include_private = false)
    @item.respond_to?(method_name, include_private) || super
  end
end
```

```ruby
class Item < ActiveRecord::Base
  def sold?
    sold_on.present?
  end
end
```

```ruby
class ItemsController < ApplicationController
	def show
  		@item_decorator = ItemDecorator.new(Item.find(params[:id]))
	end
end
```

And we can use the decorator in our view

```ruby
<li>
  <%= @item_decorator.name %> <i><%= @item_decorator.status %></i>
</li>
```


## Level 5: Activemodel: Serializers part one

Serialization code should not be in controller.


ActiveModelSerializers replace "hash-driven development" with object-oriented development.
- Decouples serialization code from the model
- Convention Over Configuration
- Access to url helpers methods
- Support for associations


##### Active Model Serializers

Gemfile

```ruby
gem 'active_model_serializers', github: 'rails-api/active_model_serializers'
# gem 'jbuilder'
```


## Level 6: ActiveModel Serializers part two

##### Associations

```ruby
class ItemSerializer < ActiveModel::Serializer
  attributes :id, :name, :url
￼  has_many :comments
  def url
    item_url(object)
end end
```

For associations like these, we can use embed in each association. Using that, we're gonna have "comments_ids" rather than the entire object for each comment. By using the embed class method, all existing associations will embed ids instead objects.


```ruby
class ReviewSerializer < ActiveModel::Serializer
  attributes :id, :description, :approve_url

  has_many :comments, embed: :id

  def approve_url
    approve_review_url(object)
  end
end
```


The client-side JavaScript library we are using automatically connects side-loaded objects with their respective ids embeded in parent associations. On our ItemSerializer, let's add an option to the embed method to make sure our reviews are side-loaded.


```ruby
class ItemSerializer < ActiveModel::Serializer
  attributes :id, :name
  
  has_many :reviews
  embed :ids, include: true
end
```


##### Override association methods to filter records

```ruby
class ItemSerializer < ActiveModel::Serializer
  attributes :id, :name

  has_many :reviews
  embed :ids, include: true
  
  def reviews
    object.reviews.approved
  end
end
```

## Level 7: Fine tuning

##### Pluck

Rails 4 with multiple arguments

```ruby
class Item < ActiveRecord::Base

  def self.recent
    where('published_on > ?', 2.days.ago).pluck(:id,:name)
  end
end
```

##### Filter Sensitive Parameters


For security purposes, our application asks a security question during sign up. Just like passwords, we can't let those answers be written to the log. Make sure we are filtering the value for security_answer.

```ruby
class Rails4Patterns::Application
  # hiding other config for brevity...
  config.filter_parameters += [:password,:security_answer]
end
```


##### Replace webrick on production


```ruby
gem 'puma'
```

##### Specify ruby version

In our GemFile it's a good practice to specify our ruby version

##### Procfile


```ruby
gem 'foreman'
```

Example of a procfile:

```shell
web: bundle exec rails s -p $PORT
```

```ruby
web: bundle exec rails s -p $PORT
worker: bundle exec rake worker
urgent_worker: bundle exec rake urgent_worker
scheduler: bundle exec rake scheduler
```

lib/tasks/run_app
```ruby
desc 'Start worker'
task worker: :environment do
  # starts worker...
end

desc 'Start urgent worker'
task urgent_worker: :environment do
  # starts urgent worker...
end

desc 'Start scheduler'
task scheduler: :environment do
  # starts scheduler..
end
```

Foreman is a command line tool for running Procfile-backend apps.
De



