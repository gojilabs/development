# Unit

Unit is a banking-as-a-service platform that allows developers to embed financial functions into applications. Unit is used as a backbone service to provide customers with banking features, payments, money transfers.

For application developers, Unit offers a [documented API](https://docs.unit.co/) and dashboards to manage applications. There are [2 environments](https://docs.unit.co/#environments), _staging_ and _production_. As of now, there is no Ruby gem to help integrate the Unit service into a Rails application.

This guide is based on the August 2022 Unit API documentation.

## Prerequisites

In order to work with the Unit API, the developer needs to use a set of environment variables. Typically they are placed in the `.env` file for development.

- `BASE_URL` - the URL to access the Unit API. There are different URLs for staging and production environments.
- `ORG_TOKEN` - generic developer token for using Unit endpoints.
- `WEBHOOK_TOKEN` - a token for checking Unit webhook callbacks.

Normally in a Rails application we store these tokens with the prefix `UNIT_`: `UNIT_BASE_URL`, `UNIT_ORG_TOKEN`, `UNIT_WEBHOOK_TOKEN`.

## Usage

### The wrapper class

Since there is no gem that provides a wrapper around the Unit's API calls and handles its errors, the wrapper must be written yourself. The wrapper can be a class in the `lib` folder that contains public methods for accessing Unit endpoints and service methods for low-level access to the Unit API. Here is an excerpt:

```ruby
# lib/integrations/unit.rb

module Integrations
  class Unit
    class << self
      ...

      private

      def headers(token = nil)
        {
          'Content-Type': 'application/vnd.api+json',
          Authorization:  "Bearer #{token || ENV.fetch('UNIT_ORG_TOKEN', nil)}"
        }
      end

      def get_request(url)
        request(:get, url)
      end

      def post_request(url, body)
        request(:post, url, body)
      end

      def patch_request(url, body)
        request(:patch, url, body)
      end

      def delete_request(url)
        request(:delete, url)
      end

      def request(method, url, body = {})
        response = case method
                   when :get
                     ::HTTParty.get(url, body: body.to_json, headers: headers)
                   when :post
                     ::HTTParty.post(url, body: body.to_json, headers: headers)
                   when :patch
                     ::HTTParty.patch(url, body: body.to_json, headers: headers)
                   when :delete
                     ::HTTParty.delete(url, body: body.to_json, headers: headers)
                   end

        response_body = response.body.present? ? JSON.parse(response.body, symbolize_names: true) : {}

        raise ApiError.new(response.code, error_message(response_body)) unless response.success?

        response_body

      rescue ::JSON::JSONError
        raise UnprocessableEntityError, 'Malformed JSON response'

      rescue ::HTTParty::Error
        raise UnprocessableEntityError, 'Request could not be processed'
      end

      def error_message(response_body)
        response_body[:errors].map { |e| [e[:title], e[:detail]].compact.join(', ') }.join(' ')
      rescue StandardError
        'Failed to capture error'
      end
    end
  end
end
```

### API usage

To access Unit API endpoints, public methods are added to the `Integrations::Unit` wrapper class. Since the number of endpoints does not allow us to list them all here, we will show a few use cases.

The following methods allow you to access information about the client application, namely: get the application by its ID, create the application and update it:

```ruby
# lib/integrations/unit.rb

module Integrations
  class Unit
    class << self
      # https://docs.unit.co/applications#get-application-by-id
      def application(application_id)
        uri = URI("#{ENV.fetch('UNIT_BASE_URL')}/applications/#{application_id}")

        get_request(uri.to_s)
      end

      # https://docs.unit.co/applications#create-individual-application
      def create_application(attributes)
        uri = URI("#{ENV.fetch('UNIT_BASE_URL')}/applications")

        body = {
          data: {
            type:       'individualApplication',
            attributes: attributes
          }
        }

        post_request(uri.to_s, body)
      end

      # https://docs.unit.co/applications/#update-individual-application
      def update_application(application_id, attributes)
        uri = URI("#{ENV.fetch('UNIT_BASE_URL')}/applications/#{application_id}")

        body = {
          data: {
            type:       'individualApplication',
            attributes: attributes
          }
        }

        patch_request(uri.to_s, body)
      end

      ...
    end
  end
end
```

They can be used in the application as follows.

To get the status of an application:

```ruby
@application ||= Integrations::Unit.application(application_id)
application_status = @application.dig(:data, :attributes, :status)
```

To create application:

```ruby
customer = params[:customer].permit(
  :ssn, :email, :dateOfBirth, :idempotencyKey,
  { fullName: %i[first last] },
  { phone: %i[countryCode number] },
  { address: %i[street street2 city state postalCode country] }
).to_h
customer[:tags] = { externalId: params[:external_id] }

application_data = Integrations::Unit.create_application(customer)[:data]
```

To update the application record in the Unit API:

```ruby
attributes = {
  tags: { externalId: user.external_id }
}
Integrations::Unit.update_application(user.unit_application_id, attributes)
```

For more examples, see [the wrapper class from the Grind24 project](https://github.com/BoB-Company/grind-banking-api/blob/main/lib/integrations/unit.rb).

### The webhooks

The Unit service can perform some operations asynchronously. For example, checking KYC for new customers or activating customers' debit cards. To inform the application of these changes, the [webhook](https://docs.unit.co/webhooks) mechanism is designed.

Developers register a webhook callback URL in the Unit dashboard. The types of events in the Unit that initiate webhook requests can be configured in the dashboard as well. In response, they receive a special token to acknowledge incoming webhook requests that come from the Unit.

 This is what a webhook controller might look like:

```ruby
# app/controllers/unit_webhooks_controller.rb

class UnitWebhooksController < ApplicationController
  skip_before_action :authenticate!
  before_action :verify_webhook

  def callback
    data = params[:data][0]

    case data[:type]
    # https://docs.unit.co/events/#cardactivated
    # https://docs.unit.co/events/#cardstatuschanged
    when 'card.activated', 'card.statusChanged'
      # Cache flag update for cards
      current_user.update(cards_updated_at: DateTime.now)

    # https://docs.unit.co/events/#applicationpendingreview
    when 'application.pendingReview'
      application_id = data.dig(:relationships, :application, :data, :id)
      @application ||= Integrations::Unit.application(application_id)

    # https://docs.unit.co/events/#accountfrozen
    when 'account.frozen'
      customer_id   = data.dig(:relationships, :customer, :data, :id)
      account_id    = data.dig(:relationships, :account, :data, :id)
      freeze_reason = data.dig(:attributes, :freezeReason)
      
    ...
    end

    head :ok
  end

  private

  def current_user
    ...
  end

  def verify_webhook
    body = request.body.read
    signature = request.headers['x-unit-signature'] || ''

    calculated_hmac = Base64.encode64(OpenSSL::HMAC.digest(OpenSSL::Digest.new('sha1'), ENV.fetch('UNIT_WEBHOOK_TOKEN', nil), body)).strip

    raise UnauthorizedError unless Rack::Utils.secure_compare(calculated_hmac, signature)
  end
end
```

For more examples, see [the controller itself from Grind24 project](https://github.com/BoB-Company/grind-banking-api/blob/main/app/controllers/unit_webhooks_controller.rb).
