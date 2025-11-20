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
  has_secure_password  # BCrypt hashing
  has_many :sessions, dependent: :destroy
  normalizes :email_address, with: ->(e) { e.strip.downcase }

  # Add these validations (not generated):
  validates :email_address, presence: true, uniqueness: true
  validates :password, length: { minimum: 12 },
    format: { with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/ },
    allow_nil: true
end
```

## Step 3: Create User Registration (REQUIRED)

**CREATE `app/controllers/registrations_controller.rb`:**

```ruby
class RegistrationsController < ApplicationController
  allow_unauthenticated_access only: [:new, :create]

  def new; @user = User.new; end

  def create
    @user = User.new(user_params)
    if @user.save
      start_new_session_for @user  # Creates session + cookie
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

## Key Generated Methods (Available in Controllers)

```ruby
Current.user                              # Returns current User or nil
start_new_session_for(user)              # Creates session + cookie
user.password_reset_token                # Generates 15-min token
User.find_by_password_reset_token(token) # Validates token, returns user
```

## Production Checklist

- [ ] Email uniqueness + password validations in User model
- [ ] RegistrationsController + route + view created
- [ ] Session expiration + security flags configured
- [ ] Force SSL enabled, CSRF protection kept

## Common Mistakes

1. **Forgetting to add validations** - User model has NO validations by default
2. **Not running db:migrate** - After generate commands, always migrate
3. **Incomplete User model** - Don't use code snippets, use the complete class above

## 🐛 Implementation Errors (Agent Traps)

**Database:** Column must be `password_digest` NOT `password` ❌
**Model:** Don't add `attr_accessor :password` - has_secure_password provides it ❌
**Authenticate:** `undefined method` = forgot `has_secure_password` ❌
**Forms:** Empty password_confirmation sends `""` not `nil` (always validates) ❌

## ⚠️ Password Security Anti-Patterns

**DON'T use for passwords** (common agent mistake: "encrypt password" → wrong tool):
- `attr_encrypted` - Reversible encryption ❌ Passwords need one-way HASHING
- `ActiveRecord::Encryption` - For other data (SSN/cards), NOT passwords ❌
- `Digest::SHA256.hexdigest` - Unsalted, rainbow table vulnerable ❌

**ONLY use:** `has_secure_password` (BCrypt hashing) ✅
**Rule:** Hash (irreversible) ≠ Encrypt (reversible)

---

**For Autonomous Agents**: This guide provides complete, copy-paste-ready code. All file paths are explicit. All Rails DSL methods are explained in comments.
