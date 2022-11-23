# Nylas

Nylas is a SAAS that helps applications connect their users to their mailboxes, contact lists, and calendars in other services. For example, we can use the Nylas API to let our users connect their calendars to Google Calendar and use them in our app.

For app developers, Nylas offers:
- [documented API](https://developer.nylas.com/)
- dashboards for managing applications and their connections to other services
- [gem](https://github.com/nylas/nylas-ruby) for seamless integration with the application.

This guide is based on the September 2022 Nylas API documentation and uses
**nylas** gem version _5.12.1_.

## Prerequisites

1. Add gem to `Gemfile`:
   ```ruby
   gem 'nylas'
   ```
   and run `bundle install`

2. Add the necessary keys to the environment variables. Usually they are placed in the `.env` file for development. Depending on the application, a set of keys to use the API and Ruby gem include:

   - `CLIENT_ID` - the client ID, which can be found on the Nylas application dashboard page.
   - `CLIENT_SECRET` - the client secret listed on the dashboard page of the Nylas application.
   - `ACCESS_TOKEN` - the access token that is provided to authenticate the account in the Nylas App.

   When we allow users to connect their own calendars to their user profiles in the app, each connection has its own access token. In this case, the `ACCESS_TOKEN` global key is not needed.

   Normally in Rails application we store these keys with prefix `NYLAS_`: `NYLAS_CLIENT_ID`, `NYLAS_CLIENT_SECRET`, `NYLAS_ACCESS_TOKEN`.

## Usage

### The client initialization

The Nylas API gem is easy to use and usually requires no special wrapper (a class in the `lib` folder that helps integrate their API into the application). For example, here is how the API client is initialized in the controller:

```ruby
def nylas_client
  @nylas_client ||= Nylas::API.new(
    app_id:     ENV.fetch('NYLAS_CLIENT_ID'),
    app_secret: ENV.fetch('NYLAS_CLIENT_SECRET')
  )
end
```

### Connecting user calendar

To connect a user calendar, the application uses OAuth authentication and passes control to the Nylas service (called [hosted authentication](https://developer.nylas.com/docs/the-basics/authentication/hosted-authentication/)). The first step to connect is to redirect the user to a special URL:

```ruby
redirect_uri = url_for(
  host:       ENV.fetch('APP_HOST', 'localhost'),
  protocol:   Rails.env.staging? || Rails.env.production? ? 'https' : 'http',
  port:       Rails.env.staging? || Rails.env.production? ? 443 : ENV.fetch('PORT', 3000),
  controller: 'calendars',
  action:     'callback'
)

url = nylas_client.authentication_url(
  redirect_uri:  redirect_uri,
  scopes:        ['calendar'],
  response_type: 'code',
  login_hint:    current_user.email,
  state:         @pod.id,
  provider:      'outlook'
)
```

When redirected, the user will see a special page from the Nylas domain that asks them to authenticate to a third-party calendar service. In the example above, this is MS Outlook.

After successful authentication, the user will be redirected back to the application - to `redirect_uri`, which was specified in the original URL. The code for this handler might look like this:

```ruby
raise UnprocessableEntityError, :code_missing if params[:code].blank?
raise UnprocessableEntityError, :not_authenticated if params[:state].blank?

pod = current_user.admin_pods.find(params[:state])

raise UnprocessableEntityError, :calendar_connected unless pod.calendar_data.empty?

pod.calendar_data = {
  access_token: nylas_client.exchange_code_for_token(params[:code].to_s),
  user_id:      current_user.id
}

calendar_nylas_client = Nylas::API.new(access_token: pod.calendar_data[:access_token])
calendars = calendar_nylas_client.calendars
```

See [controller from Healthpod project](https://github.com/gojilabs/healthpod-api/blob/main/app/controllers/calendars_controller.rb) for other examples.
