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

