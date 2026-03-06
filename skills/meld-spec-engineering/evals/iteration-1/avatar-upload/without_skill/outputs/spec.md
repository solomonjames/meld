# Implementation Spec: User Avatar Upload

## Overview

Add profile picture upload functionality to the user settings page. Users can upload JPEG or PNG images (max 5MB) that are displayed as thumbnails in the nav bar and larger images on the profile page. Uses Active Storage with the existing S3 backend.

## Requirements

### Functional Requirements

- Users can upload a profile picture from their settings page
- Accepted formats: JPEG, PNG
- Maximum file size: 5MB
- Avatar displayed as 100x100 thumbnail in the navigation bar
- Avatar displayed as 300x300 image on the profile page
- Users can replace an existing avatar with a new upload
- Users can remove their avatar (reverts to a default/placeholder)
- A default placeholder avatar is shown when no avatar has been uploaded

### Non-Functional Requirements

- Image variants are generated lazily on first request and cached
- Uploads go directly to S3 via Active Storage
- Avatar display does not degrade page load performance (variants served via CDN/S3)

## Technical Design

### 1. Model Layer

**File:** `app/models/user.rb`

Add the Active Storage attachment and validation:

```ruby
class User < ApplicationRecord
  has_one_attached :avatar

  validates :avatar,
    content_type: ['image/jpeg', 'image/png'],
    size: { less_than: 5.megabytes }
end
```

**Dependency:** Add the `active_storage_validations` gem for declarative content type and size validations.

**Gemfile addition:**

```ruby
gem 'active_storage_validations', '~> 1.1'
```

Run `bundle install` after adding.

### 2. Image Variants

Define named variants for consistent usage across the app.

**File:** `app/models/user.rb`

```ruby
class User < ApplicationRecord
  has_one_attached :avatar do |attachable|
    attachable.variant :thumbnail, resize_to_fill: [100, 100], format: :webp, saver: { quality: 80 }
    attachable.variant :profile, resize_to_fill: [300, 300], format: :webp, saver: { quality: 85 }
  end
end
```

**Dependency:** Ensure `image_processing` gem (with `vips` or `mini_magick`) is available. Rails 7 includes `image_processing` by default but confirm it is uncommented in the Gemfile.

### 3. Controller

**File:** `app/controllers/users/avatars_controller.rb`

```ruby
module Users
  class AvatarsController < ApplicationController
    before_action :authenticate_user!

    def update
      if current_user.update(avatar_params)
        redirect_to edit_user_settings_path, notice: "Avatar updated successfully."
      else
        redirect_to edit_user_settings_path, alert: current_user.errors.full_messages.to_sentence
      end
    end

    def destroy
      current_user.avatar.purge_later
      redirect_to edit_user_settings_path, notice: "Avatar removed."
    end

    private

    def avatar_params
      params.require(:user).permit(:avatar)
    end
  end
end
```

### 4. Routes

**File:** `config/routes.rb`

Add within the appropriate scope:

```ruby
namespace :users do
  resource :avatar, only: [:update, :destroy]
end
```

This produces:
- `PATCH /users/avatar` -- update (upload/replace)
- `DELETE /users/avatar` -- destroy (remove)

### 5. View Helper

**File:** `app/helpers/avatar_helper.rb`

```ruby
module AvatarHelper
  DEFAULT_AVATAR_PATH = "default_avatar.png"

  def user_avatar(user, variant:, **options)
    if user.avatar.attached?
      image_tag user.avatar.variant(variant), **options
    else
      image_tag DEFAULT_AVATAR_PATH, **options
    end
  end
end
```

### 6. Views

#### Settings Page (Upload Form)

**File:** `app/views/users/settings/_avatar_form.html.erb`

```erb
<div class="avatar-upload">
  <h3>Profile Picture</h3>

  <div class="avatar-preview">
    <%= user_avatar(current_user, variant: :profile, class: "avatar avatar--profile") %>
  </div>

  <%= form_with model: current_user, url: users_avatar_path, method: :patch, class: "avatar-form" do |f| %>
    <div class="avatar-form__input">
      <%= f.file_field :avatar, accept: "image/jpeg,image/png", direct_upload: true %>
      <p class="avatar-form__hint">JPEG or PNG, max 5MB</p>
    </div>

    <div class="avatar-form__actions">
      <%= f.submit "Upload Avatar", class: "btn btn--primary" %>
    </div>
  <% end %>

  <% if current_user.avatar.attached? %>
    <%= button_to "Remove Avatar", users_avatar_path, method: :delete, class: "btn btn--danger",
        data: { turbo_confirm: "Are you sure you want to remove your avatar?" } %>
  <% end %>
</div>
```

#### Nav Bar Thumbnail

**File:** Update the existing nav bar partial (e.g., `app/views/shared/_navbar.html.erb`)

Add where the user menu/icon is rendered:

```erb
<%= user_avatar(current_user, variant: :thumbnail, class: "avatar avatar--nav") %>
```

#### Profile Page

**File:** Update the profile show view (e.g., `app/views/users/show.html.erb`)

```erb
<%= user_avatar(@user, variant: :profile, class: "avatar avatar--profile") %>
```

### 7. CSS

**File:** Add to the appropriate stylesheet (e.g., `app/assets/stylesheets/components/_avatar.css` or equivalent)

```css
.avatar {
  border-radius: 50%;
  object-fit: cover;
}

.avatar--nav {
  width: 32px;
  height: 32px;
}

.avatar--profile {
  width: 300px;
  height: 300px;
}

.avatar-upload {
  max-width: 400px;
}

.avatar-form__hint {
  font-size: 0.875rem;
  color: #6b7280;
  margin-top: 0.25rem;
}
```

### 8. Default Avatar Asset

**File:** `app/assets/images/default_avatar.png`

Provide a neutral placeholder image (e.g., silhouette icon) at 300x300 resolution. This will be used whenever a user has no uploaded avatar.

### 9. Direct Upload Configuration

Active Storage direct uploads require the JavaScript package. Confirm the following are in place:

**File:** `app/javascript/application.js` (or equivalent entry point)

```javascript
import * as ActiveStorage from "@rails/activestorage"
ActiveStorage.start()
```

This enables client-side direct upload to S3, avoiding the app server as an intermediary for large files.

### 10. Active Storage S3 Configuration

Confirm the existing S3 service is configured in:

**File:** `config/storage.yml`

```yaml
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: us-east-1
  bucket: your-bucket-name
```

**File:** `config/environments/production.rb`

```ruby
config.active_storage.service = :amazon
```

Ensure the S3 bucket CORS configuration allows direct uploads from the app domain.

## Migration Checklist

No database migrations are required. Active Storage uses its own tables (`active_storage_blobs`, `active_storage_attachments`, `active_storage_variant_records`) which should already exist if Active Storage is installed.

If Active Storage tables do not exist, run:

```
bin/rails active_storage:install
bin/rails db:migrate
```

## Implementation Steps

1. **Add gem dependency** -- Add `active_storage_validations` to Gemfile and run `bundle install`.
2. **Update User model** -- Add `has_one_attached :avatar` with variants and validations.
3. **Create avatar helper** -- Add `AvatarHelper` with the `user_avatar` method.
4. **Create avatars controller** -- Add `Users::AvatarsController` with `update` and `destroy` actions.
5. **Add routes** -- Add the `resource :avatar` route under the `users` namespace.
6. **Add default avatar asset** -- Place a placeholder image in the assets directory.
7. **Create settings form partial** -- Build the avatar upload form for the settings page.
8. **Update nav bar** -- Add the thumbnail avatar display to the navigation partial.
9. **Update profile page** -- Add the profile-size avatar display to the user profile view.
10. **Add CSS** -- Style avatar components.
11. **Verify direct upload** -- Confirm Active Storage JS is loaded and S3 CORS is configured.

## Acceptance Criteria

Given a logged-in user on the settings page,
When they select a valid JPEG or PNG file under 5MB and click "Upload Avatar",
Then the avatar is uploaded to S3 and displayed on the settings page as a 300x300 image.

Given a logged-in user with an uploaded avatar,
When they view any page with the navigation bar,
Then their avatar is displayed as a 100x100 thumbnail.

Given a logged-in user on the settings page,
When they click "Remove Avatar" and confirm,
Then the avatar is removed and the default placeholder is shown.

Given a logged-in user on the settings page,
When they attempt to upload a file that is not JPEG or PNG,
Then the upload is rejected with a validation error message.

Given a logged-in user on the settings page,
When they attempt to upload a file larger than 5MB,
Then the upload is rejected with a validation error message.

Given a visitor viewing another user's profile page,
When the viewed user has an uploaded avatar,
Then the avatar is displayed at 300x300.

Given a visitor viewing another user's profile page,
When the viewed user has no avatar,
Then the default placeholder image is displayed.

## Edge Cases and Considerations

- **Animated GIFs:** Rejected by content type validation (only JPEG/PNG allowed).
- **Very small images:** `resize_to_fill` will upscale. Acceptable for MVP; consider minimum dimension validation later.
- **Concurrent uploads:** Active Storage handles replacement atomically; the old blob is purged after the new one attaches.
- **S3 bucket cleanup:** `purge_later` enqueues an Active Job to delete the blob from S3. Ensure a job backend (Sidekiq, etc.) is running.
- **CDN caching:** If using CloudFront or similar, variant URLs include content hashes and are cache-safe by default.
- **HEIC/HEIF format:** Common on iOS but not accepted per requirements. Users will need to convert before uploading. Could be added in a future iteration.
