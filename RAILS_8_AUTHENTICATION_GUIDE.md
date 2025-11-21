# Rails 8 Authentication Guide - For Autonomous Agents

## Overview

Rails 8 includes authentication generator. This guide shows complete, working code for agents unfamiliar with Rails conventions.

## Prerequisites

```bash
rails --version  # Requires Rails 8.0.0+
# Ensure gem 'bcrypt', '~> 3.1.7' is in Gemfile
```

## Generated vs Manual

| Auto-Generated | Must Create Manually |
|-------------------|-------------------------|
| User/Session models, Login/Logout | User registration (controller + view + route) |
| Password reset (15min tokens) | Email/password validations in User model |
| Rate limiting (10/3min) | Session expiration config |

## Step 1: Generate Authentication System

```bash
bin/rails generate authentication
bin/rails db:migrate
```

**Creates:** User/Session models (with `password_digest`), SessionsController, PasswordsController, Authentication concern (`start_new_session_for`, `Current.user`), views.

**WARNING: Working code but NO validations!**

## Step 2: Add Validations to User Model

**OPEN `app/models/user.rb` and ADD these lines after `normalizes`:**

```ruby
# Generator doesn't create validations - ADD these:
validates :email_address, presence: true, uniqueness: true
validates :password, length: { minimum: 12 },
  format: { with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d).+\z/ },
  allow_nil: true
```

**Result:** User model now has `has_secure_password` (from generator) + validations (you added).

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
<% if @user.errors.any? %>
  <ul><% @user.errors.full_messages.each do |msg| %><li><%= msg %></li><% end %></ul>
<% end %>
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
  expire_after: 12.hours, secure: Rails.env.production?,
  httponly: true, same_site: :lax
```

**EDIT `config/environments/production.rb` - add this line:**

```ruby
config.force_ssl = true  # Force HTTPS in production
```

## Key Generated Methods (Available in Controllers)

```ruby
Current.user                              # Returns current User or nil
start_new_session_for(user)              # Creates session + cookie
user.password_reset_token                # Generates 15-min token
User.find_by_password_reset_token(token) # Validates token, returns user
```

## Production Checklist

- [ ] User validations (email uniqueness + password complexity)
- [ ] Registration (controller + route + view)
- [ ] Session security (expiration + SSL + CSRF)

## Common Mistakes

1. Forgetting validations (User has NONE by default)
2. Not running db:migrate after generator
3. Using code snippets instead of complete class (Step 2)

## Errors (Only if Deviating from Guide)

**Steps 1-4 prevent these errors. They happen when improvising:**
- Creating `password` column manually (generator uses `password_digest`)
- Adding `attr_accessor :password` (has_secure_password provides it)
- Skipping `has_secure_password` (no authenticate method)

**Follow exact steps = zero errors. ONE path.**

## Password Security Anti-Patterns

**DON'T use for passwords** (common agent mistake: "encrypt password" → wrong tool):
- `attr_encrypted` - Reversible encryption. Passwords need one-way HASHING
- `ActiveRecord::Encryption` - For other data (SSN/cards), NOT passwords
- `Digest::SHA256.hexdigest` - Unsalted, rainbow table vulnerable

**ONLY use:** `has_secure_password` (BCrypt hashing)
**Rule:** Hash (irreversible) ≠ Encrypt (reversible)

---

## ONE Path Strategy

**Follow Steps 1-4 sequentially. NO alternatives. Don't improvise.**

Generator (Step 1) → Validations (Step 2) → Registration (Step 3) → Security (Step 4)

**For Agents**: Copy-paste ready. Explicit paths. Explained DSL.
