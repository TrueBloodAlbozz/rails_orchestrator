# Rails 8 Authentication Guide - For Autonomous Agents

## Overview

Rails 8 includes authentication generator. This guide shows complete, working code for agents unfamiliar with Rails conventions.

## Prerequisites

```bash
rails --version  # Requires Rails 8.0.0+
# Ensure gem 'bcrypt', '~> 3.1.7' is in Gemfile
```

## Generated vs Manual

| Auto-Generated ✅ | Must Create Manually ❌ |
|-------------------|-------------------------|
| User/Session models, Login/Logout | User registration (controller + view + route) |
| Password reset (15min tokens) | Email/password validations in User model |
| Rate limiting (10/3min) | Session expiration config |

## Step 1: Generate Authentication System

```bash
bin/rails generate authentication
bin/rails db:migrate
```

Creates User/Session models, SessionsController, PasswordsController, Authentication concern.

## Step 2: Complete User Model (Add Validations)

**REPLACE `app/models/user.rb` with this complete version:**

```ruby
class User < ApplicationRecord
  has_secure_password  # BCrypt hashing, creates password= and password_confirmation=
  has_many :sessions, dependent: :destroy
  normalizes :email_address, with: ->(e) { e.strip.downcase }

  # NOT generated - you must add these:
  validates :email_address, presence: true, uniqueness: true
  validates :password,
    length: { minimum: 12 },
    format: { with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
              message: "needs upper/lower/number" },
    allow_nil: true  # Only validates on password change
end
```

## Step 3: Create User Registration (REQUIRED)

**CREATE `app/controllers/registrations_controller.rb`:**

```ruby
class RegistrationsController < ApplicationController
  allow_unauthenticated_access only: [:new, :create]  # From Authentication concern

  def new; @user = User.new; end

  def create
    @user = User.new(user_params)
    if @user.save
      start_new_session_for @user  # From Authentication concern - creates session + cookie
      redirect_to root_path, notice: "Welcome!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private
  def user_params
    params.require(:user).permit(:email_address, :password, :password_confirmation)
  end
end
```

**ADD to `config/routes.rb` inside `Rails.application.routes.draw do` block:**

```ruby
Rails.application.routes.draw do
  resource :registration, only: [:new, :create]  # ADD THIS LINE
  # ... other routes ...
end
```

**CREATE `app/views/registrations/new.html.erb`:**

```erb
<h1>Sign Up</h1>
<%= form_with model: @user, url: registration_path do |f| %>
  <%= f.email_field :email_address, required: true, placeholder: "Email" %>
  <%= f.password_field :password, required: true, placeholder: "Password (12+ chars)" %>
  <%= f.password_field :password_confirmation, required: true, placeholder: "Confirm" %>
  <%= f.submit "Sign Up" %>
<% end %>
```

## Step 4: Configure Session Security (Production)

**CREATE `config/initializers/session_store.rb`:**

```ruby
Rails.application.config.session_store :cookie_store,
  expire_after: 12.hours,        # Sessions expire after 12 hours
  secure: Rails.env.production?, # HTTPS-only in production
  httponly: true,                # JavaScript cannot access (XSS protection)
  same_site: :lax                # CSRF protection
```

**EDIT `config/environments/production.rb` - add this line:**

```ruby
config.force_ssl = true  # Force HTTPS in production
```

## Step 5: Optional - Account Lockout

```bash
rails generate migration AddLockableToUsers failed_attempts:integer locked_at:datetime
rails db:migrate  # Don't forget to run migration!
```

**ADD to User model:**

```ruby
def locked?
  locked_at.present? && locked_at > 1.hour.ago
end
```

## What's NOT Generated (Must Add Manually)

1. Registration controller + view + route
2. Email uniqueness + password complexity validations
3. Session expiration configuration

## Key Generated Methods (Available in Controllers)

```ruby
Current.user                              # Returns current User or nil
start_new_session_for(user)              # Creates session + cookie
user.password_reset_token                # Generates 15-min token
User.find_by_password_reset_token(token) # Validates token, returns user
```

## Production Checklist

- [ ] User model has email uniqueness + password complexity validations
- [ ] RegistrationsController created with route
- [ ] Session store configured with expiration + security flags
- [ ] Force SSL enabled in production.rb
- [ ] CSRF protection NOT disabled (enabled by default)

## Common Mistakes

1. **Forgetting to add validations** - User model has NO validations by default
2. **Not running db:migrate** - After generate commands, always migrate
3. **Incomplete User model** - Don't use code snippets, use the complete class above

---

**For Autonomous Agents**: This guide provides complete, copy-paste-ready code. All file paths are explicit. All Rails DSL methods are explained in comments.
