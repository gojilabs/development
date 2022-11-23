# Customer.IO

Customer.io is an automated messaging platform for collecting and processing application customer information. We used Customer.io as an external database for tracking banking application users.

For application developers, Customer.io offers:
- [documented API](https://customer.io/docs/api/)
- Application management dashboards
- [gem](https://github.com/customerio/customerio-ruby) for seamless integration with the application.

This guide is based on the September 2022 Customer.io API documentation and the use of **customerio** gem version _4.3.0_.

## Prerequisites

1. Add gem to the `Gemfile`:
   ```ruby
   gem 'customerio'
   ```
   and run `bundle install`

2. Add the necessary keys to the environment variables. Usually for development they are placed in the `.env` file. A set of keys to use the Customer.io API (and their Ruby gem) include:

   - `SITE_ID` - the internal ID of the application in Customer.io
   - `API_KEY` - the API key (for using from backend)
   - `APP_KEY` - the APP key (for using from frontend)

   The `APP_KEY` key is not required if Customer.io is only used to track user actions from the backend application.

   Usually in Rails application we store these keys with prefix `CUSTOMERIO_`: `CUSTOMERIO_SITE_ID`, `CUSTOMERIO_API_KEY`, `CUSTOMERIO_APP_KEY`.

## Usage

### The wrapper class

The developers of Customer.io recommend initializing the connection to the service (their client) in some kind of Rails initializer. This method involves the use of global variables, which is considered bad practice. Another way to use the Customer.io API through their gem is to use a wrapper class. A wrapper can be a class in the `lib` folder that contains public methods for accessing Customer.io endpoints. Here's what it might look like:

```ruby
# lib/integrations/customerio.rb

require 'customerio'

module Integrations
  class Customerio
    class << self
      def identify(id, attributes = {})
        return if ENV.fetch('CUSTOMERIO_SITE_ID', nil).nil?

        client.identify({ id: id }.merge(attributes))
      end

      def track(id, event_name, attributes = {})
        return if ENV.fetch('CUSTOMERIO_SITE_ID', nil).nil?

        client.track(id, event_name, attributes)
      end

      def merge_customers(primary_id_type, primary_id, secondary_id_type, secondary_id)
        return if ENV.fetch('CUSTOMERIO_SITE_ID', nil).nil?

        client.merge_customers(primary_id_type.to_s, primary_id.to_s, secondary_id_type.to_s, secondary_id.to_s)
      end

      private

      def client
        @client ||= ::Customerio::Client.new(ENV.fetch('CUSTOMERIO_SITE_ID'), ENV.fetch('CUSTOMERIO_API_KEY'))
      end
    end
  end
end
```

### API usage

Here are a few examples of how to use the Customer.io service using the wrapper class presented above.

To initialize an application user, an entry must be created in Customer.io. Usually this happens right after creating the user records in the application itself. The code may look like this:

```ruby
attributes = {
  email:      customer[:email],
  first_name: customer[:fullName][:first],
  last_name:  customer[:fullName][:last],
  phone:      phone
}
Integrations::Customerio.identify(params[:external_id], attributes)
```

To track a user's activity and add a record of it to the user's profile in Customer.io, you can call the `track` method:

```ruby
Integrations::Customerio.track(
  current_user.external_id,
  'Account Frozen',
  customer_id:   customer_id,
  account_id:    account_id,
  freeze_reason: freeze_reason
)
```

For more examples see [the webhook controller from Grind24 project](https://github.com/BoB-Company/grind-banking-api/blob/main/app/controllers/unit_webhooks_controller.rb).
