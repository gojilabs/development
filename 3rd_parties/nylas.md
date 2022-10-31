# Nylas

Nylas is a SAAS that helps applications to connect their users with their mailboxes, contact lists and calendars in other services. For example, we can use Nylas API to let our users to connect their calendars in Google Calendar, and use them in our app.

For application developers Nylas offers:
- a [documented API](https://developer.nylas.com/)
- dashboards to manage applications and their connections to other services
- a [gem](https://github.com/nylas/nylas-ruby) for the seamless integrations with the app

The provided guideline is based on Nylas API documentation from September 2022 and on using 
**nylas** gem version _5.12.1_.

## Usage

### API keys

Depending on the application, the set of the keys for using Nylas API (and their Ruby gem) includes:

- `CLIENT_ID` - The Client ID found on the dashboard page for the Nylas App
- `CLIENT_SECRET` - The Client Secret found on the dashboard page for the Nylas App
- `ACCESS_TOKEN` - The Access Token that is provided to authenticate an account to the Nylas App

When we let users connect their own calendars to their user profiles in the application, every connection has its own access token. In this case global `ACCESS_TOKEN` key is not necessary.

Usually in Rails application we keep these keys with the prefix `NYLAS_`: `NYLAS_CLIENT_ID`, `NYLAS_CLIENT_SECRET`, `NYLAS_ACCESS_TOKEN`

### API wrapper

The Nylas API gem is easy to use and usually does not require a special wrapper (a class in `lib` folder that helps to integrate their API into the app). For example this is how the API client is initiated in the controller:
```ruby
@nylas_client = Nylas::API.new(
  app_id:     ENV.fetch('NYLAS_CLIENT_ID'),
  app_secret: ENV.fetch('NYLAS_CLIENT_SECRET')
)
```

### Connecting user calendar

To connect user calendar the app uses OAuth authentication and passes control to Nylas service (so called hosted authentication). The first step to connect is to redirect the user to the special URL:

```ruby
redirect_uri = url_for(
  host:       ENV.fetch('APP_HOST', 'localhost'),
  protocol:   Rails.env.staging? || Rails.env.production? ? 'https' : 'http',
  port:       Rails.env.staging? || Rails.env.production? ? 443 : ENV.fetch('PORT', 3000),
  controller: 'calendars',
  action:     'callback'
)

url = @nylas_client.authentication_url(
  redirect_uri:  redirect_uri,
  scopes:        ['calendar'],
  response_type: 'code',
  login_hint:    current_user.email,
  state:         @pod.id,
  provider:      'outlook'
)
```

When redirected, the user will see a special page from Nylas domain, that asks them to authenticate in 3rd party calendar service. In provided example this is MS Outlook.

After successful authentication the user will be redirected back to the application - to the `redirect_uri` which was provided in the original URL. The code of this handler may look like:

```ruby
raise UnprocessableEntityError, :code_missing if params[:code].blank?
raise UnprocessableEntityError, :not_authenticated if params[:state].blank?

pod = current_user.admin_pods.find(params[:state])

raise UnprocessableEntityError, :calendar_connected unless pod.calendar_data.empty?

pod.calendar_data = {
  access_token: @nylas_client.exchange_code_for_token(params[:code].to_s),
  user_id:      current_user.id
}

calendar_nylas_client = Nylas::API.new(access_token: pod.calendar_data[:access_token])
calendars = calendar_nylas_client.calendars

```

For more examples see [the controller from Healthpod project](https://github.com/gojilabs/healthpod-api/blob/main/app/controllers/calendars_controller.rb).
