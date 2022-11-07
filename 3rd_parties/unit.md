# Unit

Unit is the banking-as-a-service platform that lets developers embed financial features into the apps. Unit is used as a backbone service for providing customers banking features, payments, money transfers.

For application developers Unit offers a [documented API](https://docs.unit.co/) and dashboards to manage applications. There are [2 environments](https://docs.unit.co/#environments) available, staging and production. As it is known, there is no Ruby gem that helps integrate Unit service into the Rails application.

The provided guideline is based on Unit API documentation from August 2022.

## Prerequisites

To work with Unit API the developer needs to use the set of environment variables. Usually for development they are placed to `.env` file.

- `BASE_URL` - The url to access the Unit API. There are different URLs for staging and production environmants
- `ORG_TOKEN` - The common developer's token for using Unit endpoints
- `WEBHOOK_TOKEN` - The token for verifying Unit webhook callbacks

Usually in Rails application we keep these keys with the prefix `UNIT_`: `UNIT_BASE_URL`, `UNIT_ORG_TOKEN`, `UNIT_WEBHOOK_TOKEN`

## Usage

### The wrapper class

Since there is no a gem that provides a wrapper around Unit API calls and handle its errors, the wrapper needs to be written by ourselves. The wrapper can be a class in `lib` folder that contains public methods for accessing Unit endpoints and service methods to low-level access to Unit API. Here is the extract:

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

To access Unit API endpoints public methods are added to `Integrations::Unit` wrapper class. Since the number of endpoints does not allow us to list them all here, let's show a couple of uses.

The following methods let access customer application information, namely: get application by its Id, create application and update it:

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

In the application they can be used in the following way.

To get the status of the application:

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

To update application record in Unit API:

```ruby
attributes = {
  tags: { externalId: user.external_id }
}
Integrations::Unit.update_application(user.unit_application_id, attributes)
```

For more examples see [the wrapper class from Grind24 project](https://github.com/BoB-Company/grind-banking-api/blob/main/lib/integrations/unit.rb).

### The webhooks

The Unit service may perform some operations asynchronously. For example, KYC check for new customer applications, or activating customer debit cards. To let the app know about these changes, the [webhook](https://docs.unit.co/webhooks) mechanism is developed.

The developers register webhook callback URL in Unit dashboard. In response they receive a special token to validate incoming webhook requests that are coming from Unit.

The types of events in Unit that initiate webhook request can be set up in the dashboard. This is how webhook controller may look like:

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

For more examples see [the controller itself from Grind24 project](https://github.com/BoB-Company/grind-banking-api/blob/main/app/controllers/unit_webhooks_controller.rb).
