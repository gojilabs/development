# Digital Ocean Spaces and Shrine

Digital Ocean is a hosting provider that offers a cloud storage service called Spaces. It is a good alternative to AWS S3. We use it in some projects to store files uploaded by users.

For developers, storing files in Digital Ocean Spaces is not much different from using AWS S3. DO provides an identical file handling interface. Therefore, the methods of working with DO are exactly the same as when using AWS S3.

## Prerequisites

1. Add gem to the `Gemfile`:
   ```ruby
   gem 'aws-sdk-s3', require: false
   ```
   and run `bundle install`.

2. Generally we use Spaces for storing images. Usually we process images before uploading, so some gems for image uploading and processing should be added as well:
   ```ruby
   gem 'shrine'
   gem 'marcel'
   gem 'image_processing'
   ```

3. Add the necessary keys to the environment variables. Usually for development they are placed in the `.env` file. A set of keys to use the Spaces API include:

   - `ACCESS_KEY_ID` - the access key for the Spaces API
   - `SECRET_ACCESS_KEY` - the secret key for the Spaces API
   - `ENDPOINT_URL` - the URL we use for accessing files. Generally it is https://sfo3.digitaloceanspaces.com
   - `REGION` - the region where the bucket is located. For example, `sfo3`
   - `BUCKET` - the name of the bucket where we store files

   Usually in Rails application we store these keys with prefix `SPACES_`: `SPACES_ACCESS_KEY_ID`, `SPACES_SECRET_ACCESS_KEY`, `SPACES_ENDPOINT_URL`, etc.

## Usage

We do not work with the file storage directly. Usually we use some gem that wraps all interaction with the file storage and provides a convenient interface. These are the following gems:

- [Paperclip](https://github.com/thoughtbot/paperclip) (now deprecated)
- [CarrierWave](https://github.com/carrierwaveuploader/carrierwave)
- [Shrine](https://shrinerb.com/)
- [ActiveStorage](https://guides.rubyonrails.org/active_storage_overview.html) (built-in Rails solution)

For different reasons (maturity of the gem, options available, and flexibility) we decided to use [Shrine](https://shrinerb.com/docs/getting-started).

### Initializer

First of all, we need to add an initializer so that the application knows how to work with the file storage. We can do this as follows:

<details>
  <summary>config/initializers/shrine.rb</summary>

   ```ruby
   require 'shrine'
   
   case Rails.configuration.upload_server
   when :s3
     require 'shrine/storage/s3'
   
     s3_options = {
       access_key_id:     ENV.fetch('SPACES_ACCESS_KEY_ID', nil),
       secret_access_key: ENV.fetch('SPACES_SECRET_ACCESS_KEY', nil),
       endpoint:          ENV.fetch('SPACES_ENDPOINT_URL', nil),
       region:            ENV.fetch('SPACES_REGION', nil),
       bucket:            ENV.fetch('SPACES_BUCKET', nil)
     }
   
     Shrine.storages = {
       cache: Shrine::Storage::S3.new(prefix: 'cache', upload_options: { acl: 'public-read' }, **s3_options),
       store: Shrine::Storage::S3.new(prefix: 'store', upload_options: { acl: 'public-read' }, **s3_options)
     }
   
     Shrine.plugin :url_options, store: {
       host:   "#{ENV.fetch('SPACES_ENDPOINT_URL', nil)}/#{ENV.fetch('SPACES_BUCKET', nil)}/",
       public: true
     }
   
   when :local
     require 'shrine/storage/file_system'
   
     # both `cache` and `store` storages are needed
     Shrine.storages = {
       cache: Shrine::Storage::FileSystem.new('public', prefix: 'uploads/cache'),
       store: Shrine::Storage::FileSystem.new('public', prefix: 'uploads')
     }
   
     Shrine.plugin :url_options, store: { host: 'http://localhost:3000' }
   end
   
   Shrine.plugin :activerecord
   Shrine.plugin :instrumentation
   Shrine.plugin :determine_mime_type, analyzer: :marcel, log_subscriber: nil
   Shrine.plugin :cached_attachment_data
   Shrine.plugin :restore_cached_data
   Shrine.plugin :derivatives
   
   case Rails.configuration.upload_server
   when :s3
     Shrine.plugin :presign_endpoint, presign_options: lambda { |request|
       filename = request.params['filename']
       type     = request.params['type']
       {
         content_disposition:  ContentDisposition.inline(filename), # set download filename
         content_type:         type,                                # set content type
         content_length_range: 0..(10 * 1024 * 1024)                # limit upload size to 10 MB
       }
     }
   when :local
     Shrine.plugin :upload_endpoint
   end
   
   # delay promoting and deleting files to a background job (`backgrounding` plugin)
   Shrine.plugin :backgrounding
   Shrine::Attacher.promote_block { Attachment::PromoteJob.perform_now(record, name, file_data) }
   Shrine::Attacher.destroy_block { Attachment::DestroyJob.perform_now(data) }
   ```
</details>

In addition we need to inform the application to use s3 for storing files on staging and production. For development we usually use the local file storage:

```ruby
# config/application.rb

class Application < Rails::Application
  ...
  config.upload_server = (Rails.env.production? || Rails.env.staging? ? :s3 : :local)
  ...
end
```

### File uploaders

Uploaders are subclasses of `Shrine`, and they wrap the actual upload to the storage. They perform common tasks around upload that aren't related to a particular storage.

We use a tree of uploaders with `BaseUploader` as a base class. The `BaseUploader` class contains the common logic for all uploaders. For example, we can define the location where the file will be stored. We can also define the logic for generating the unique identifier for the file.

```ruby
# app/uploaders/base_uploader.rb

class BaseUploader < Shrine
  plugin :remove_attachment

  def generate_location(io, record: nil, name: nil, derivative: nil, **)
    if record
      table_name = record.class.table_name
      table = table_name.split('.').last
    else
      table = 'uploads'
    end
    prefix = derivative ? "#{derivative}-" : ''

    "#{table}/#{name}/#{prefix}#{super}"
  end

  private

  def generate_uid(_io)
    SecureRandom.urlsafe_base64(64)
  end
end
```

Depending on the app needs we can define different uploaders for different types of files. For example, we can define an uploader for user avatars:

```ruby 
# app/uploaders/avatar_uploader.rb

require 'image_processing/vips'

class AvatarUploader < BaseUploader
  Attacher.derivatives do |original|
    processor = ImageProcessing::Vips.source(original)
    {
      small:  processor.resize_to_limit!(64, 64),
      medium: processor.resize_to_limit!(200, 200),
      large:  processor.resize_to_limit!(400, 400)
    }
  end
end
```

And another one for documents:

```ruby
# app/uploaders/document_uploader.rb

class DocumentUploader < BaseUploader
  plugin :validation_helpers

  Attacher.validate { validate_max_size 15.megabytes }
end
```

To attach uploaded files to database records, Shrine offers an attachment interface built on top of uploaders and uploaded files. For example this is how we attach avatar to user record:

```ruby
# app/models/user.rb

class User < ApplicationRecord
  ...
  include AvatarUploader::Attachment(:avatar)
  ...
end
```

(Make sure the `users` table in the database has a field **avatar_data** of type `jsonb` or `json`.)

For more information about uploaders and attachments see [Shrine documentation](https://shrinerb.com/docs/getting-started#attaching).

### File uploading

When developing applications, we use direct uploading of files to the storage. This means that when a file is uploaded by the user, the file is uploaded directly to Digital Ocean Spaces, bypassing the application server. The upload happens in the background - as soon as the client application has received the file, it is uploaded to the bucket. This significantly reduces interface latency caused by downloading and processing the file.

To enable direct upload we need to add a route to the application:

```ruby
# config/routes.rb

Rails.application.routes.draw do
  ...
  case Rails.configuration.upload_server
  when :s3
    get 'files/presign'
  when :local
    post 'files/upload'
  end
  ...
end
```

The controller for this route is responsible for handling the upload request. The `presign` endpoint is used for getting credentials for the following file upload, and the `upload` endpoint is used for uploading the file itself.

```ruby
# app/controllers/files_controller.rb

class FilesController < ApplicationController
   skip_before_action :authenticate!

   def upload
      setup_rack_response BaseUploader.upload_response(:cache, request.env)
   end

   def presign
      setup_rack_response BaseUploader.presign_response(:cache, request.env)
   end

   private

   def setup_rack_response((status, hdrs, body))
      self.status = status
      headers.merge!(hdrs)
      self.response_body = body
   end
end
```

Mode details about direct upload can be found in [Shrine documentation](https://shrinerb.com/docs/direct-s3).

### File processing

Shrine allows to process attached files eagerly or on-the-fly. For example, if the app is accepting image uploads, we can generate a predefined set of of thumbnails when the image is attached to a record. When the image is deleted these thumbnails can be deleted as well.

We use the `backgrounding` plugin to process files in the background. This means that when a file is attached to a record, the file is uploaded to the storage, and then the processing is performed in the background. This allows us to avoid blocking the main thread while processing the file.

These operations are performed by `ApplicationJob`s:

```ruby
# app/jobs/attachment/promote_job.rb

module Attachment
  class PromoteJob < ApplicationJob
    queue_as :default
    sidekiq_options retry: 5

    def perform(record, name, file_data)
      attacher = Shrine::Attacher.retrieve(model: record, name:, file: file_data)
      attacher.create_derivatives if name&.to_sym != :document
      attacher.atomic_promote
    rescue Shrine::AttachmentChanged, ActiveRecord::RecordNotFound
      # attachment has changed or the record has been deleted, nothing to do
    end
  end
end
```

```ruby
# app/jobs/attachment/destroy_job.rb

module Attachment
  class DestroyJob < ApplicationJob
    queue_as :default
    sidekiq_options retry: 5

    def perform(data)
      attacher = Shrine::Attacher.from_data(data)
      attacher.destroy
    end
  end
end
```

(Make sure these jobs are declared in `config/initializers/shrine.rb` so Shrine knows about them.)

For more information see [Shrine documentation](https://shrinerb.com/docs/processing).
