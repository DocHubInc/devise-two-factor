# Devise-Two-Factor Authentication
By [Tinfoil Security](https://www.tinfoilsecurity.com/) (acq. [Synopsys](https://www.synopsys.com/) 2020). Interested in [working with us](https://www.synopsys.com/careers.html)? We're hiring!

![Build Status](https://github.com/tinfoil/devise-two-factor/actions/workflows/ci.yml/badge.svg)

Devise-Two-Factor is a minimalist extension to Devise which offers support for two-factor authentication, through the [TOTP](https://en.wikipedia.org/wiki/Time-based_One-Time_Password) scheme. It:

* Allows you to incorporate two-factor authentication into your existing models
* Is opinionated about security, so you don't have to be
* Integrates easily with two-factor applications like [Google Authenticator](https://support.google.com/accounts/answer/1066447?hl=en) and [Authy](https://authy.com/)
* Is extensible, and includes two-factor backup codes as an example of how plugins can be structured

## Contributing
We welcome pull requests, bug reports, and other contributions. We're especially looking for help getting this gem fully compatible with Rails 5+ and squashing any deprecation messages.

## Example App
An example Rails 4 application is provided in the `demo` directory. It showcases a minimal example of Devise-Two-Factor in action, and can act as a reference for integrating the gem into your own application.

For the demo app to work, create an encryption key and store it as an environment variable. One way to do this is to create a file named `local_env.yml` in the application root. Set the value of `ENCRYPTION_KEY` in the YML file. That value will be loaded into the application environment by `application.rb`.

## Getting Started

Devise-Two-Factor doesn't require much to get started, but there are two prerequisites before you can start using it in your application:

1. A Rails application with [devise](https://github.com/heartcombo/devise) installed
1. Secrets configured for ActiveRecord encrypted attributes

First, you'll need a Rails application setup with Devise. Visit the Devise [homepage](https://github.com/plataformatec/devise) for instructions.

Devise-Two-Factor uses [ActiveRecord encrypted attributes](https://edgeguides.rubyonrails.org/active_record_encryption.html) which in turn uses Rails' encrypted credentials. [The Rails encrypted attributes guide](https://edgeguides.rubyonrails.org/active_record_encryption.html) has full details of how to set these up but briefly:

```bash
# generate suitable encryption secrets to stdout
$ ./bin/rails db:encryption:init

# Add the output from the command above to your encrypted credentials file via
# Setting the EDITOR environment variable is optional, without it, your default editor will open
$ EDITOR=code ./bin/rails credentials:edit
```

Add Devise-Two-Factor to your Gemfile with:

```ruby
# Gemfile

gem 'devise-two-factor'
```

There is a generator which automates most of the setup:

```bash
# MODEL is the name of the model you wish to configure devise_two_factor e.g. User or Admin
./bin/rails generate devise_two_factor MODEL
```

Where `MODEL` is the name of the model you wish to add two-factor functionality to (for example `user`)

This generator will:

1. Create a new migration which adds a few columns to the specified model:
    ```ruby
    add_column :users, :otp_secret, :string
    add_column :users, :consumed_timestep, :integer
    add_column :users, :otp_required_for_login, :boolean
    ```
1. Edit `app/models/MODEL.rb` (where MODEL is your model name):
    * add the `:two_factor_authenticatable` devise module
    * remove the `:database_authenticatable` if present because it is incompatible with `:two_factor_authenticatable`
1. Add a Warden config block to your Devise initializer, which enables the strategies required for two-factor authentication.

Remember to apply the new migration after you run the generator:

```bash
./bin/rails db:migrate
```

Next you need to whitelist `:otp_attempt` as a permitted parameter in Devise `:sign_in` controller. You can do this by adding the following to your `application_controller.rb`:

```ruby
# app/controllers/application_controller.rb

  before_action :configure_permitted_parameters, if: :devise_controller?

  # ...

  protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_in, keys: [:otp_attempt])
  end
```

Finally you should verify that `:database_authenticatable` is **not** being loaded by your model. The generator will try to remove it, but if you have a non-standard Devise setup, this step may fail.

**Loading both `:database_authenticatable` and `:two_factor_authenticatable` in a model is a security issue** It will allow users to bypass two-factor authenticatable due to the way Warden handles cascading strategies!

## Designing Your Workflow
Devise-Two-Factor only worries about the backend, leaving the details of the integration up to you. This means that you're responsible for building the UI that drives the gem. While there is an example Rails application included in the gem, it is important to remember that this gem is intentionally very open-ended, and you should build a user experience which fits your individual application.

There are two key workflows you'll have to think about:

1. Logging in with two-factor authentication
2. Enabling two-factor authentication for a given user

We chose to keep things as simple as possible, and our implementation can be found by registering at [Tinfoil Security](https://www.tinfoilsecurity.com/), and enabling two-factor authentication from the [security settings page](https://www.tinfoilsecurity.com/account/security).


### Logging In
Logging in with two-factor authentication works extremely similarly to regular database authentication in Devise. The `TwoFactorAuthenticatable` strategy accepts three parameters:

1. email
2. password
3. otp_attempt (Their one-time password for this session)

These parameters can be submitted to the standard Devise login route, and the strategy will handle the authentication of the user for you.

### Disabling Automatic Login After Password Resets
If you use the Devise `recoverable` strategy, the default behavior after a password reset is to automatically authenticate the user and log them in. This is obviously a problem if a user has two-factor authentication enabled, as resetting the password would get around the two-factor requirement.

Because of this, you need to set `sign_in_after_reset_password` to `false` (either globally in your Devise initializer or via `devise_for`).

### Enabling Two-Factor Authentication
Enabling two-factor authentication for a user is easy. For example, if my user model were named User, I could do the following:

```ruby
current_user.otp_required_for_login = true
current_user.otp_secret = User.generate_otp_secret
current_user.save!
```

Before you can do this however, you need to decide how you're going to transmit two-factor tokens to a user. Common strategies include sending an SMS, or using a mobile application such as Google Authenticator.

At Tinfoil Security, we opted to use the excellent [rqrcode-rails3](https://github.com/samvincent/rqrcode-rails3) gem to generate a QR-code representing the user's secret key, which can then be scanned by any mobile two-factor authentication client.

If you decide to do this you'll need to generate a URI to act as the source for the QR code. This can be done using the `User#otp_provisioning_uri` method.

```ruby
issuer = 'Your App'
label = "#{issuer}:#{current_user.email}"

current_user.otp_provisioning_uri(label, issuer: issuer)

# > "otpauth://totp/Your%20App:user@example.com?secret=[otp_secret]&issuer=Your+App"
```

If you instead to decide to send the one-time password to the user directly, such as via SMS, you'll need a mechanism for generating the one-time password on the server:

```ruby
current_user.current_otp
```

The generated code will be valid for the duration specified by `otp_allowed_drift`. This value can be modified by adding a config in `config/initializers/devise.rb`.
```ruby
Devise.otp_allowed_drift = 240 # value in seconds
Devise.setup do |config|
...
end
```

However you decide to handle enrollment, there are a few important considerations to be made:

* Whether you'll force the use of two-factor authentication, and if so, how you'll migrate existing users to system, and what your on-boarding experience will look like
* If you authenticate using SMS, you'll want to verify the user's ownership of the phone, in much the same way you're probably verifying their email address
* How you'll handle device revocation in the event that a user loses access to their device, or that device is rendered temporarily unavailable (This gem includes `TwoFactorBackupable` as an example extension meant to solve this problem)

It sounds like a lot of work, but most of these problems have been very elegantly solved by other people. We recommend taking a look at the excellent workflows used by Heroku and Google for inspiration.

### Filtering sensitive parameters from the logs
To prevent two-factor authentication codes from leaking if your application logs get breached, you'll want to filter sensitive parameters from the Rails logs. Add the following to `config/initializers/filter_parameter_logging.rb`:

```ruby
Rails.application.config.filter_parameters += [:otp_attempt]
```

## Backup Codes
Devise-Two-Factor is designed with extensibility in mind. One such extension, `TwoFactorBackupable`, is included and serves as a good example of how to extend this gem. This plugin allows you to add the ability to generate single-use backup codes for a user, which they may use to bypass two-factor authentication, in the event that they lose access to their device.

To install it, you need to add the `:two_factor_backupable` directive to your model.

```ruby
devise :two_factor_backupable
```

You'll also be required to enable the `:two_factor_backupable` strategy, by adding the following line to your Warden config in your Devise initializer, substituting :user for the name of your Devise scope.

```ruby
manager.default_strategies(:scope => :user).unshift :two_factor_backupable
```

The final installation step is dependent on your version of Rails. If you're not running Rails 4, skip to the next section. Otherwise, create the following migration:

```ruby
class AddDeviseTwoFactorBackupableToUsers < ActiveRecord::Migration
  def change
    # Change type from :string to :text if using MySQL database
    add_column :users, :otp_backup_codes, :string, array: true
  end
end
```

You can then generate backup codes for a user:

```ruby
codes = current_user.generate_otp_backup_codes!
current_user.save!
# Display codes to the user somehow!
```

The backup codes are stored in the database as bcrypt hashes, so be sure to display them to the user at this point. If all went well, the user should be able to login using each of the generated codes in place of their two-factor token. Each code is single-use, and generating a new set of backup codes for that user will invalidate all of the old ones.

You can customize the length of each code, and the number of codes generated by passing the options into `:two_factor_backupable` in the Devise directive:

```ruby
devise :two_factor_backupable, otp_backup_code_length:     32,
                               otp_number_of_backup_codes: 10
```

### Help! I'm not using Rails 4.0!
Don't worry! `TwoFactorBackupable` stores the backup codes as an array of strings in the database. In Rails 4.0 this is supported natively, but in earlier versions you can use a gem to emulate this behavior: we recommend [activerecord-postgres-array](https://github.com/tlconnor/activerecord-postgres-array).

You'll then simply have to create a migration to add an array named `otp_backup_codes` to your model. If you use the above gem, this migration might look like:

```ruby
class AddTwoFactorBackupCodesToUsers < ActiveRecord::Migration
  def change
    # Change type from :string_array to :text_array if using MySQL database
    add_column :users, :otp_backup_codes, :string_array
  end
end
```

Now just continue with the setup in the previous section, skipping the generator step.

## Testing
Devise-Two-Factor includes shared-examples for both `TwoFactorAuthenticatable` and `TwoFactorBackupable`. Adding the following two lines to the specs for your two-factor enabled models will allow you to test your models for two-factor functionality:

```ruby
require 'devise_two_factor/spec_helpers'

it_behaves_like "two_factor_authenticatable"
it_behaves_like "two_factor_backupable"
```

## Troubleshooting
If you are using Rails 4.x and Ruby >= 2.7, you may get an error like

```
An error occurred while loading ./spec/devise/models/two_factor_authenticatable_spec.rb.
Failure/Error: require 'devise'

NoMethodError:
  undefined method `new' for BigDecimal:Class
```
see https://github.com/ruby/bigdecimal#which-version-should-you-select and https://github.com/ruby/bigdecimal/issues/127
for more details, but you should be able to solve this
by explicitly requiring an older version of bigdecimal in your gemfile like
```
gem "bigdecimal", "~> 1.4"
```
