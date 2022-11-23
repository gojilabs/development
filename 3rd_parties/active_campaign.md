# ActiveCampaign

ActiveCampaign is a subscription service. It helps customers manage subscriber lists, track subscribers, monitor the effectiveness of email campaigns, etc.

For application developers, ActiveCampaign offers various [integration guides](https://developers.activecampaign.com/docs), [documented API](https://developers.activecampaign.com/reference) and application management dashboards. In addition to the usual production environment, they offer a [sandbox environment](https://developers.activecampaign.com/page/developer-sandbox-accounts) to facilitate development. As of now, the service does not offer any Ruby gem that makes it easy to integrate the service into a Rails application.

The provided guide is based on the ActiveCampaign API version 3 documentation, November 2022.

## Prerequisites

In order to work with the ActiveCampaign API, the developer needs to use a set of environment variables. Typically they are placed in the `.env` file for development.

- `BASE_URL` - the URL to access the API. There are different URLs for staging and production environments.
- `TOKEN` - generic developer token for using ActiveCampaign endpoints.

Usually in a Rails application we store these tokens with the prefix `ACTIVE_CAMPAIGN_`: `ACTIVE_CAMPAIGN_BASE_URL`, `ACTIVE_CAMPAIGN_TOKEN`.

## Usage

### The wrapper class

Since there is no gem that provides a wrapper around ActiveCampaign API calls and handles errors, the wrapper must be written yourself. The wrapper can be a class in the `lib` folder that contains public methods for accessing service endpoints and service methods for low-level access to the API. Here's an excerpt:

```ruby
# lib/integrations/active_campaign.rb

module Integrations
  class ActiveCampaign
    class << self
      ...

      private

      def headers(token = nil)
        {
          Accept:      'application/json',
          'Api-Token': token || ENV.fetch('ACTIVE_CAMPAIGN_TOKEN', nil)
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
                     ::HTTParty.get(url, headers: headers)
                   when :post
                     ::HTTParty.post(url, headers: headers, body: body.to_json)
                   when :patch
                     ::HTTParty.patch(url, headers: headers, body: body.to_json)
                   when :delete
                     ::HTTParty.delete(url, headers: headers)
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
        if response_body[:errors]
          response_body[:errors].pluck(:title).join(' ')
        elsif response_body[:message]
          response_body[:message]
        else
          'Generic error'
        end
      rescue StandardError
        'Failed to capture error'
      end
    end
  end
end
```

### API usage

To access ActiveCampaign API endpoints, public methods are added to the `Integrations::ActiveCampaign` wrapper class. Because the number of endpoints does not allow us to list them all here, we will show a few uses.

The API follows the RestAPI specifications in naming endpoints and using HTTP request types, so metaprogramming is used to simplify writing wrappers for each endpoint:

```ruby
# lib/integrations/unit.rb

module Integrations
  class ActiveCampaign
    class << self
      %i[
        user
        list
        contact
        message
      ].each do |resource|
        handle = resource.to_s.camelize(:lower)
        endpoint_url = "#{ENV.fetch('ACTIVE_CAMPAIGN_BASE_URL')}/#{handle.pluralize}"

        # https://developers.activecampaign.com/reference/list-all-users
        # https://developers.activecampaign.com/reference/retrieve-all-lists
        # https://developers.activecampaign.com/reference/list-all-contacts
        # https://developers.activecampaign.com/reference/list-all-messages
        define_method(resource.to_s.pluralize.to_sym) do |params = {}|
          uri = URI(endpoint_url)
          uri.query = params.to_query

          get_request(uri.to_s)
        end

        # https://developers.activecampaign.com/reference/get-user
        # https://developers.activecampaign.com/reference/retrieve-a-list
        # https://developers.activecampaign.com/reference/get-contact
        # https://developers.activecampaign.com/reference/retrieve-a-message
        define_method(resource) do |id, scope = nil|
          uri = URI("#{endpoint_url}/#{id}#{scope && "/#{scope.to_s.camelize(:lower)}"}")

          get_request(uri.to_s)
        end

        # https://developers.activecampaign.com/reference/create-user
        # https://developers.activecampaign.com/reference/create-new-list
        # https://developers.activecampaign.com/reference/create-a-new-contact
        # https://developers.activecampaign.com/reference/create-a-new-message
        define_method("create_#{resource}".to_sym) do |attributes|
          uri = URI(endpoint_url)

          body = {
            handle => attributes
          }

          post_request(uri.to_s, body)
        end

        # https://developers.activecampaign.com/reference/delete-user
        # https://developers.activecampaign.com/reference/delete-a-list
        # https://developers.activecampaign.com/reference/delete-contact
        # https://developers.activecampaign.com/reference/delete-a-message
        define_method("delete_#{resource}".to_sym) do |id|
          uri = URI("#{endpoint_url}/#{id}")

          delete_request(uri.to_s)
        end
      end

      ...
    end
  end
end
```

In an application, these methods can be used in several scenarios.

To get an active user ID that can manage subscriptions:

```ruby
users_response = Integrations::ActiveCampaign.users(limit: 1)
@user_id = users_response&.dig(:users)&.first&.dig(:id)
```

To get user group ID:

```ruby
group_response = Integrations::ActiveCampaign.user(user_id, :user_group)
@group_id = group_response&.dig(:userGroup, :groupid)
```

To create a list (subscription):

```ruby
list_response = Integrations::ActiveCampaign.create_list(attributes)
@list_id = list_response.dig(:list, :id)
```

To create a contact (subscriber):

```ruby
attributes = {
  email: email
}
contact_response = Integrations::ActiveCampaign.create_contact(attributes)
@contact_id = contact_response.dig(:contact, :id)
```

To subscribe a contact to some list:

```ruby
Integrations::ActiveCampaign.subscribe_contact(ac_contact_id, ac_list_id)
```

To remove a subscription:

```ruby
Integrations::ActiveCampaign.delete_list(ac_list_id)
```

For more examples see [the wrapper class from IowaLeague project](https://github.com/gojilabs/iowa-league/blob/main/api/lib/integrations/active_campaign.rb).

### The webhooks

The ActiveCampaign service allows you to track changes to lists / contacts / campaigns asynchronously using webhooks. For example, when you create a new list of subscribers in the ActiveCampaign dashboard, the service can notify the application so that the application can react to it and create an internal representation of the list. This can be done with [webhooks](https://developers.activecampaign.com/page/webhooks).

The idea is to allow developers to register some callback URLs that will be requested when some event occurs on the ActiveCampaign side (in the dashboard, such as creating a new subscriber list, or otherwise, such as unsubscribing from the list). In these calls, the service will pass information about the event-the data of the list just created, or contact information.

Unfortunately, the service does not add any level of security to these calls, such as HTTP headers with secret tokens that need to be validated. Any user knowing this URL may misuse it. As a result, using this option is a violation of application security and is not allowed.
