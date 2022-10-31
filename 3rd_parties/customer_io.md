# Customer.IO

Customer.io is an automated messaging platform for collecting and processing info about application customers. We used Customer.io as an outer database for tracking users of the banking application.

For application developers Customer.io offers:
- a [documented API](https://customer.io/docs/api/)
- dashboards to manage applications
- a [gem](https://github.com/customerio/customerio-ruby) for the seamless integrations with the app

The provided guideline is based on Customer.io API documentation from September 2022 and on using
**customerio** gem version _4.3.0_.

## Usage

### API keys

The set of the keys for using Customer.io API (and their Ruby gem) includes:

- `SITE_ID` - The internal Id of the application in Customer.io
- `API_KEY` - The API key (for using from backend)
- `APP_KEY` - The APP key (for using from frontend)

The `APP_KEY` is not required when the Customer.io is used for tracking user actions from backend application only.

Usually in Rails application we keep these keys with the prefix `CUSTOMERIO_`: `CUSTOMERIO_SITE_ID`, `CUSTOMERIO_API_KEY`, `CUSTOMERIO_APP_KEY`

### API wrapper

The Customer.io developers recommend to initialize connection to the service (their client) in some Rails initializer. This method suggests using global variables, which is considered as a bad practice. The other way to use Customer.io API through their gem is with some wrapper class. The wrapper can be a class in `lib` folder that contains public methods for accessing Customer.io endpoints. Here is how it may look like:

```ruby
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

There are some examples of usage of Customer.io service using the wrapper class provided above.

To initialize the application user the record in Customer.io needs to be created. Usually it happens right after creating user records in the app itself. The code could look like:

```ruby
attributes = {
  email:      customer[:email],
  first_name: customer[:fullName][:first],
  last_name:  customer[:fullName][:last],
  phone:      phone
}
Integrations::Customerio.identify(params[:external_id], attributes)
```

To track user activity and add a record about it to user profile in Customer.io the `track` method can be invoked:

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
