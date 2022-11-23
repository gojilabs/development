# Plaid

Plaid is a financial service that connects users to their banking accounts and let them perform financial operations, such as bill payments and money transfers. Plaid is an interface between a Rails application and a "real banking".

For application developers, Plaid offers:
- [documented API](https://plaid.com/docs/api/)
- dashboards to manage applications. There are 3 different environments available.
- [gem](https://github.com/plaid/plaid-ruby) for seamless integration with the application.

This guide is based on the September 2022 Plaid API documentation and the use of **plaid** gem version _16.0.0_.

## Prerequisites

1. Add gem to the `Gemfile`:
   ```ruby
   gem 'plaid'
   ```
   and run `bundle install`

2. Add the necessary keys to the environment variables. Usually for development they are placed in the `.env` file. A set of keys to use the Plaid API (and their Ruby gem) include:

   - `ENVIRONMENT` - the Plaid environment. Can be __production__, __development__ or __sandbox__
   - `CLIENT_ID` - the developer ID issued by Plaid.
   - `SECRET_KEY` - the secret key for accessing Plaid API.

   Usually in Rails application we store these keys with prefix `PLAID_`: `PLAID_ENVIRONMENT`, `PLAID_CLIENT_ID`, `PLAID_SECRET_KEY`.

## Usage

### The wrapper class

Because Plaid deals with users' real bank accounts, calls to the service can often fail. A transaction is rejected as suspicious, the user's access key is expired, etc. Such exceptional situations need to be handled, so all Plaid calls are enclosed in a wrapper class that initializes the Plaid client operation and handles errors when working with it. The initialization of the client looks like this:

```ruby
# lib/integrations/plaid.rb

require 'plaid'

module Integrations
  class Plaid
    class << self
      ...

      private

      def api_client
        return @api_client if @api_client

        configuration = ::Plaid::Configuration.new
        configuration.server_index = ::Plaid::Configuration::Environment[ENV.fetch('PLAID_ENVIRONMENT', nil)]
        configuration.api_key['PLAID-CLIENT-ID'] = ENV.fetch('PLAID_CLIENT_ID', nil)
        configuration.api_key['PLAID-SECRET'] = ENV.fetch('PLAID_SECRET_KEY', nil)
        @api_client = ::Plaid::ApiClient.new(configuration)
      end

      def plaid_api
        @plaid_api ||= ::Plaid::PlaidApi.new(api_client)
      end
    end
  end
end
```

### API usage

To access Plaid API endpoints, public methods are added to the `Integrations::Plaid` wrapper class. Since the number of endpoints does not allow us to list them all here, we will show a few use cases.

The following methods allow you to create a link token to retrieve access token for having access to user's bank account, get the info about user bank accounts (including balances) and list of transactions:

```ruby
# lib/integrations/plaid.rb

require 'plaid'

module Integrations
  class Plaid
    class << self
      def link_token(user_id)
        request = ::Plaid::LinkTokenCreateRequest.new(
          user:          { client_user_id: user_id },
          client_name:   'Grind Finance',
          products:      ['auth'],
          country_codes: ['US'],
          language:      'en',
          webhook:       "#{ENV.fetch('API_BASE_URL', 'localhost')}/webhooks/plaid"
        )
        plaid_api.link_token_create(request)
      end

      def access_token(public_token)
        request = ::Plaid::ItemPublicTokenExchangeRequest.new(
          public_token: public_token
        )
        access_response = plaid_api.item_public_token_exchange(request)

        access_response.access_token
      end

      def access_token_info(access_token, institution_id)
        request = ::Plaid::InstitutionsGetByIdRequest.new(
          institution_id: institution_id,
          country_codes:  ['US'],
          options:        { include_optional_metadata: true }
        )
        institution = plaid_api.institutions_get_by_id(request)

        request = ::Plaid::AuthGetRequest.new(
          access_token: access_token
        )
        accounts = plaid_api.auth_get(request)

        {
          institution: institution.institution,
          accounts:    accounts.accounts
        }
      end

      def transactions(access_token, from, to)
        request = ::Plaid::TransactionsGetRequest.new(
          access_token: access_token,
          start_date:   from,
          end_date:     to
        )
        transaction_response = plaid_api.transactions_get(request)

        transaction_response.transactions
      end

      ...
    end
  end
end
```

They can be used in the application as follows.

To get the `link_token` that can be passed to the frontend for retrieving the public token:

```ruby
link_token = Integrations::Plaid.link_token(current_user.external_id)
```

To generate the access token of the associated bank selected by the user in Plaid:

```ruby
access_token = Integrations::Plaid.access_token(params[:public_token])
```

To get the info about user's bank accounts:

```ruby
res = {}

begin
  plaid_info = Integrations::Plaid.access_token_info(connection.access_token, connection.institution_id)

  res[:institution] = plaid_info[:institution].to_hash.slice(*%i[institution_id name country_codes url primary_color logo])
  res[:accounts]    = plaid_info[:accounts]

rescue ::Plaid::ApiError => e
  Rollbar.error(e)
end

res
```

For more examples, see [the wrapper class from the Grind24 project](https://github.com/BoB-Company/grind-banking-api/blob/main/lib/integrations/plaid.rb) and [its Plaid controller](https://github.com/BoB-Company/grind-banking-api/blob/main/app/controllers/plaid_controller.rb).

### The webhooks

Plaid creates a special `access_token` to establish a connection to the bank. This token can expire for various reasons: for example, the bank specifically limits the lifetime of such a token, or the user has changed the password to the banking application. In any case it is desirable that Plaid informs our application about expiration of the token. To do that Plaid API has webhooks.

We register webhooks when connecting to the account (see `Integrations::Plaid.link_token` method). If at some point the specified situation happens, Plaid requests the registered webhook (callback URL) and informs the application about it.

The important part of this flow is request verification - we need to make sure that the request is initiated by Plaid service. This is how it is performed in the controller:

```ruby
# app/controllers/plaid_webhooks_controller.rb

class PlaidWebhooksController < ApplicationController
  skip_before_action :authenticate!
  before_action :verify_webhook

  ...

  private

  def verify_webhook
    body = request.body.read
    signed_jwt = request.headers['Plaid-Verification'] || ''

    verified = Integrations::Plaid.verify_webhook(body, signed_jwt)

    raise UnauthorizedError unless verified
  end
end
```

The corresponding method in the wrapper class is:

```ruby
# lib/integrations/plaid.rb

require 'digest'
require 'jwt'
require 'time'
require 'base64'
require 'openssl'
require 'json'
require 'jose'
require 'active_support/security_utils'

module Integrations
  class Plaid
    class << self
      ...

      # https://plaid.com/docs/api/webhooks/webhook-verification/
      def verify_webhook(body, signed_jwt)
        @key_cache ||= {}

        decoded_token = ::JWT.decode(signed_jwt, nil, false)
        current_key_id = decoded_token[1]['kid']

        unless @key_cache.key?(current_key_id)
          key_ids_to_update = []

          @key_cache.each do |key_id, key|
            next unless key.expired_at.nil?

            key_ids_to_update.append(key_id)
          end

          key_ids_to_update.append(current_key_id)

          key_ids_to_update.each do |key_id|
            response = plaid_api.webhook_verification_key_get(key_id)
            @key_cache[key_id] = response.key
          end
        end

        # If the key ID is not in the cache, the key ID may be invalid
        return false unless @key_cache.key?(current_key_id)

        key = @key_cache[current_key_id]

        # Reject expired keys
        return false unless key.expired_at.nil?

        # Validate the signature
        begin
          return false unless ::JOSE::JWT.verify(key, signed_jwt)[0]
        rescue StandardError
          return false
        end

        # Compare hashes
        body_hash = Digest::SHA256.hexdigest(body)
        claimed_body_hash = decoded_token[0]['request_body_sha256']
        return false unless ActiveSupport::SecurityUtils.secure_compare(body_hash, claimed_body_hash)

        # Validate that token is not expired
        iat = decoded_token[0]['iat']
        return false if Time.now.to_i - iat > 60 * 5

        true
      end

      ...
    end
  end
end
```

For more examples, see [the controller itself from Grind24 project](https://github.com/BoB-Company/grind-banking-api/blob/main/app/controllers/plaid_webhooks_controller.rb).
