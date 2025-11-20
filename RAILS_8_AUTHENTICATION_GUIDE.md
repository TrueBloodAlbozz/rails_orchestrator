# Rails 8 Authentication Guide - 2024/2025 Best Practices

## Overview

Rails 8 introduces a built-in authentication generator that provides a secure, lightweight alternative to gems like Devise. This guide covers implementation using modern best practices from 2024-2025.

## Why Rails 8 Built-in Authentication?

- **Transparent**: All code is in your app, easy to customize
- **Lightweight**: Only what you need, no bloat
- **Secure**: BCrypt hashing, signed cookies, rate limiting
- **Maintained**: Part of Rails core, official support

## Prerequisites

```bash
# Ensure you have Rails 8+
rails --version  # Should be 8.0.0 or higher

# Install bcrypt gem (add to Gemfile if not present)
gem 'bcrypt', '~> 3.1.7'
```

## Step 1: Generate Authentication

```bash
bin/rails generate authentication
```

This creates:
- **User model**: `app/models/user.rb` with `has_secure_password`
- **Session model**: `app/models/session.rb` for session tracking
- **SessionsController**: Handles login/logout
- **PasswordsController**: Manages password resets
- **Migrations**: For users and sessions tables
- **Views**: Login and password reset forms

## Step 2: Run Migrations

```bash
bin/rails db:migrate
```

## Step 3: Understanding Generated Code

### User Model (`app/models/user.rb`)

```ruby
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy

  normalizes :email_address, with: ->(e) { e.strip.downcase }

  validates :email_address, presence: true, uniqueness: true
  validates :password, length: { minimum: 12 }, on: :create
end
```

**Key Features**:
- `has_secure_password`: BCrypt password hashing, `password` and `password_confirmation` attributes
- `normalizes`: Email normalization (Rails 7.1+)
- Minimum 12-character password (2024 best practice)

### Session Model (`app/models/session.rb`)

```ruby
class Session < ApplicationRecord
  belongs_to :user

  before_create do
    self.token = SecureRandom.urlsafe_base64
  end
end
```

**Security Features**:
- Tracks IP address and user agent
- Unique token per session
- Database-backed for explicit control

## Step 4: Enhance Security Configuration

### Configure Session Store (`config/initializers/session_store.rb`)

```ruby
Rails.application.config.session_store :cookie_store,
  key: '_your_app_session',
  expire_after: 12.hours,
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax
```

**2024/2025 Best Practices**:
- `secure: true`: HTTPS-only in production
- `httponly: true`: Prevents JavaScript access (XSS protection)
- `same_site: :lax`: CSRF protection
- `expire_after: 12.hours`: Session timeout for security

### Force SSL in Production (`config/environments/production.rb`)

```ruby
config.force_ssl = true
```

## Step 5: Add User Registration (Not Included by Default)

Create `app/controllers/registrations_controller.rb`:

```ruby
class RegistrationsController < ApplicationController
  allow_unauthenticated_access only: [:new, :create]

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)

    if @user.save
      session = @user.sessions.create!(
        user_agent: request.user_agent,
        ip_address: request.remote_ip
      )
      cookies.signed.permanent[:session_token] = { value: session.token, httponly: true }
      redirect_to root_path, notice: "Welcome! You have signed up successfully."
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

Add routes (`config/routes.rb`):

```ruby
resource :registration, only: [:new, :create]
```

## Step 6: Strengthen Password Validation

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  validates :password,
    length: { minimum: 12 },
    format: {
      with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/,
      message: "must include uppercase, lowercase, number, and special character"
    },
    on: :create
end
```

## Step 7: Implement Rate Limiting

The generated `SessionsController` includes basic rate limiting:

```ruby
rate_limit to: 10, within: 3.minutes, only: :create
```

**Enhancement for 2025**: Add account lockout:

```ruby
# Add to User model
def increment_failed_attempts
  self.failed_attempts ||= 0
  self.failed_attempts += 1
  self.locked_at = Time.current if failed_attempts >= 5
  save
end

def reset_failed_attempts
  update(failed_attempts: 0, locked_at: nil)
end

def locked?
  locked_at.present? && locked_at > 1.hour.ago
end
```

## Step 8: Password Reset Security

The generated `PasswordsController` uses Rails 8's new `password_reset_token`:

```ruby
# Generate token (automatically expires in 15 minutes)
token = user.password_reset_token

# Find user by token
user = User.find_by_password_reset_token(params[:token])
```

**Key Security Features**:
- Signed tokens using `ActiveSupport::MessageVerifier`
- 15-minute expiration (configurable)
- No database storage needed
- Purpose-limited (can't be used for other purposes)

## Step 9: Log Security Events

```ruby
# app/models/concerns/auditable.rb
module Auditable
  extend ActiveSupport::Concern

  included do
    after_create :log_creation
  end

  private

  def log_creation
    Rails.logger.info "[SECURITY] #{self.class.name} created: #{self.id}"
  end
end

# Include in Session model
class Session < ApplicationRecord
  include Auditable

  after_destroy do
    Rails.logger.info "[SECURITY] Session destroyed: user_id=#{user_id}"
  end
end
```

## Step 10: Testing Authentication

```ruby
# test/models/user_test.rb
require "test_helper"

class UserTest < ActiveSupport::TestCase
  test "should not save user without email" do
    user = User.new(password: "SecurePassword123!")
    assert_not user.save
  end

  test "should require minimum password length" do
    user = User.new(email_address: "test@example.com", password: "short")
    assert_not user.save
  end

  test "should authenticate with correct password" do
    user = users(:one)
    assert user.authenticate("password")
  end
end
```

## Additional Security Best Practices (2024/2025)

### 1. Email Verification
Add email confirmation before allowing login:

```ruby
# Add to users table
add_column :users, :email_confirmed_at, :datetime
```

### 2. Multi-Factor Authentication (MFA)
Consider adding TOTP-based 2FA using `rotp` gem for sensitive applications.

### 3. Session Management
Provide users ability to view and revoke active sessions:

```ruby
# app/controllers/sessions_controller.rb
def index
  @sessions = Current.user.sessions
end

def destroy_all
  Current.user.sessions.where.not(id: Current.session.id).destroy_all
  redirect_to sessions_path, notice: "All other sessions have been logged out."
end
```

### 4. CSRF Protection
Rails enables this by default. Never disable:

```ruby
# Keep this enabled in ApplicationController
protect_from_forgery with: :exception
```

### 5. Content Security Policy
Add CSP headers (`config/initializers/content_security_policy.rb`):

```ruby
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.script_src  :self, :https
  policy.style_src   :self, :https, :unsafe_inline
end
```

## Common Pitfalls to Avoid

1. **Don't store passwords in plain text**: Always use `has_secure_password`
2. **Don't use short session timeouts in development**: It frustrates testing
3. **Don't skip CSRF protection**: Keep `protect_from_forgery` enabled
4. **Don't ignore rate limiting**: Prevents brute force attacks
5. **Don't use `http` in production**: Always use HTTPS with `force_ssl = true`
6. **Don't store sensitive data in cookies**: Use signed/encrypted cookies only
7. **Don't reuse password reset tokens**: They expire after 15 minutes by default

## Production Checklist

- [ ] `force_ssl = true` enabled
- [ ] Session cookies are `secure`, `httponly`, and `same_site`
- [ ] Rate limiting configured on login
- [ ] Password complexity requirements enforced
- [ ] Email verification implemented
- [ ] Security events logged
- [ ] Session expiration configured
- [ ] CSRF protection enabled
- [ ] Content Security Policy configured
- [ ] Regular security audits scheduled

## Resources

- [Rails Security Guide](https://guides.rubyonrails.org/security.html)
- [Rails 8 Authentication Generator Documentation](https://blog.saeloun.com/2025/05/12/rails-8-adds-built-in-authentication-generator/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

---

**Last Updated**: November 2025 | Based on Rails 8.0+ best practices
