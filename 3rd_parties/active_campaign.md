# Active Campaign

Active Campaign is the subscription service. It helps customers to manage their subscribe list, trace subscribers, monitor effectiveness of mailing campaigns, etc.

For application developers Active Campaign offers various [guides on integration](https://developers.activecampaign.com/docs), [documented API](https://developers.activecampaign.com/reference) and dashboards to manage applications. In addition to a regular production environment they offer [sandbox environment](https://developers.activecampaign.com/page/developer-sandbox-accounts) to facilitate the development. As it is known, there is no Ruby gem that helps integrate the service into the Rails application.

The provided guideline is based on Active Campaign API documentation version 3, November 2022.

## Prerequisites

To work with Active Campaign API the developer needs to use the set of environment variables. Usually for development they are placed to `.env` file.

- `BASE_URL` - The url to access the API. There are different URLs for staging and production environments
- `TOKEN` - The common developer's token for using Active Campaign endpoints

Usually in Rails application we keep these keys with the prefix `ACTIVE_CAMPAIGN_`: `ACTIVE_CAMPAIGN_BASE_URL`, `ACTIVE_CAMPAIGN_TOKEN`

## Usage

### The wrapper class

Since there is no a gem that provides a wrapper around Active Campaign API calls and handle its errors, the wrapper needs to be written by ourselves. The wrapper can be a class in `lib` folder that contains public methods for accessing Unit endpoints and service methods to low-level access to Unit API. Here is the extract:

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

To access Active Campaign API endpoints public methods are added to `Integrations::ActiveCampaign` wrapper class. Since the number of endpoints does not allow us to list them all here, let's show a couple of uses.

The API follows RestAPI specifications in naming endpoints and using HTTP request types, so to simplify the writing of wrappers for each endpoint the metaprogramming is used:

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

In the application these methods can be used in multiple scenarios.

To get active user id who can manage subscriptions:

```ruby
users_response = Integrations::ActiveCampaign.users(limit: 1)
@user_id = users_response&.dig(:users)&.first&.dig(:id)
```

To get user group id:

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

To subscribe the contact to some list:

```ruby
Integrations::ActiveCampaign.subscribe_contact(ac_contact_id, ac_list_id)
```

To delete subscription:

```ruby
Integrations::ActiveCampaign.delete_list(ac_list_id)
```

For more examples see [the wrapper class from IowaLeague project](https://github.com/gojilabs/iowa-league/blob/main/api/lib/integrations/active_campaign.rb).
