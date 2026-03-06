# Implementation Spec: User Avatar Upload

## Overview

Add profile picture upload capability to the user settings page. Users can upload JPEG or PNG images (max 5MB) that are displayed as thumbnails in the nav bar (100x100) and as larger images on the profile page (300x300). Uses Active Storage with S3.

---

## Requirements

### Functional Requirements

1. Users can upload a profile picture from their settings page.
2. Accepted formats: JPEG and PNG only.
3. Maximum file size: 5MB.
4. The avatar displays as a 100x100 thumbnail in the navigation bar.
5. The avatar displays as a 300x300 image on the profile page.
6. Users can replace an existing avatar by uploading a new one.
7. Users can remove their avatar, reverting to a default placeholder.
8. A default placeholder image is shown when no avatar has been uploaded.

### Non-Functional Requirements

1. Images are stored on S3 via Active Storage.
2. Variants (100x100, 300x300) are processed and cached on first access.
3. Upload validation happens both client-side (for fast feedback) and server-side (for security).
4. Image processing uses `image_processing` gem with `vips` (or `mini_magick` as fallback).

---

## Technical Design

### 1. Dependencies

Ensure the following are present in the `Gemfile`:

```ruby
gem "image_processing", "~> 1.2"
```

Run `bundle install`. Verify `libvips` is available in the production environment (or fall back to ImageMagick/MiniMagick).

### 2. Model Changes

**File:** `app/models/user.rb`

```ruby
class User < ApplicationRecord
  has_one_attached :avatar

  validates :avatar,
    content_type: { in: ["image/jpeg", "image/png"], message: "must be a JPEG or PNG" },
    size: { less_than: 5.megabytes, message: "must be less than 5MB" }

  def avatar_thumbnail
    avatar.variant(resize_to_fill: [100, 100])
  end

  def avatar_profile
    avatar.variant(resize_to_fill: [300, 300])
  end
end
```

**Note:** The `validates` calls above require the `active_storage_validations` gem. Add to Gemfile:

```ruby
gem "active_storage_validations", "~> 1.1"
```

### 3. Controller Changes

**File:** `app/controllers/settings_controller.rb` (or equivalent user settings controller)

Add `avatar` to strong parameters:

```ruby
def user_params
  params.require(:user).permit(:name, :email, :avatar)
end
```

Add a `remove_avatar` action:

```ruby
def remove_avatar
  current_user.avatar.purge_later
  redirect_to settings_path, notice: "Avatar removed."
end
```

**Route:**

```ruby
# config/routes.rb
delete "settings/avatar", to: "settings#remove_avatar", as: :remove_avatar
```

### 4. View Changes

#### 4a. Settings Page (Upload Form)

**File:** `app/views/settings/edit.html.erb` (or equivalent)

Add within the existing settings form:

```erb
<div class="avatar-upload">
  <h3>Profile Picture</h3>

  <% if current_user.avatar.attached? %>
    <%= image_tag current_user.avatar_profile, class: "avatar-preview" %>
    <%= button_to "Remove Avatar", remove_avatar_path, method: :delete,
        data: { turbo_confirm: "Remove your profile picture?" },
        class: "btn btn-danger btn-sm" %>
  <% else %>
    <%= image_tag "default_avatar.png", class: "avatar-preview", size: "300x300" %>
  <% end %>

  <div class="avatar-field">
    <%= form.file_field :avatar,
        accept: "image/jpeg,image/png",
        direct_upload: true,
        data: { max_file_size: 5.megabytes } %>
    <p class="help-text">JPEG or PNG, max 5MB</p>
  </div>
</div>
```

#### 4b. Navigation Bar (Thumbnail)

**File:** `app/views/layouts/_navbar.html.erb` (or equivalent nav partial)

```erb
<% if current_user %>
  <div class="nav-avatar">
    <% if current_user.avatar.attached? %>
      <%= image_tag current_user.avatar_thumbnail, class: "avatar-thumb", alt: current_user.name %>
    <% else %>
      <%= image_tag "default_avatar.png", class: "avatar-thumb", size: "100x100", alt: current_user.name %>
    <% end %>
  </div>
<% end %>
```

#### 4c. Profile Page

**File:** `app/views/profiles/show.html.erb` (or equivalent)

```erb
<div class="profile-avatar">
  <% if @user.avatar.attached? %>
    <%= image_tag @user.avatar_profile, class: "avatar-profile", alt: @user.name %>
  <% else %>
    <%= image_tag "default_avatar.png", class: "avatar-profile", size: "300x300", alt: @user.name %>
  <% end %>
</div>
```

### 5. Default Avatar

Place a default avatar placeholder image at:

```
app/assets/images/default_avatar.png
```

This should be a neutral silhouette or initials-based placeholder, 300x300 pixels.

### 6. Active Storage S3 Configuration

Verify `config/storage.yml` has an S3 service configured:

```yaml
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: us-east-1
  bucket: your-bucket-name
```

Verify `config/environments/production.rb` uses it:

```ruby
config.active_storage.service = :amazon
```

Ensure the S3 bucket CORS policy allows direct uploads if using `direct_upload: true`:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["PUT"],
    "AllowedOrigins": ["https://yourdomain.com"],
    "ExposeHeaders": ["Origin", "Content-Type", "Content-MD5", "Content-Disposition"],
    "MaxAgeSeconds": 3600
  }
]
```

### 7. Stimulus Controller for Client-Side Validation (Optional Enhancement)

**File:** `app/javascript/controllers/avatar_upload_controller.js`

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "preview"]

  validate(event) {
    const file = event.target.files[0]
    if (!file) return

    const validTypes = ["image/jpeg", "image/png"]
    const maxSize = 5 * 1024 * 1024 // 5MB

    if (!validTypes.includes(file.type)) {
      alert("Please select a JPEG or PNG image.")
      event.target.value = ""
      return
    }

    if (file.size > maxSize) {
      alert("File must be less than 5MB.")
      event.target.value = ""
      return
    }

    // Show preview
    if (this.hasPreviewTarget) {
      const reader = new FileReader()
      reader.onload = (e) => {
        this.previewTarget.src = e.target.result
      }
      reader.readAsDataURL(file)
    }
  }
}
```

### 8. CSS

```css
.avatar-preview {
  width: 300px;
  height: 300px;
  object-fit: cover;
  border-radius: 50%;
}

.avatar-thumb {
  width: 100px;
  height: 100px;
  object-fit: cover;
  border-radius: 50%;
}

.nav-avatar .avatar-thumb {
  width: 32px;
  height: 32px;
  border-radius: 50%;
}
```

---

## Database Migration

No migration is needed. Active Storage uses its own tables (`active_storage_blobs`, `active_storage_attachments`, `active_storage_variant_records`), which should already exist. If not:

```bash
rails active_storage:install
rails db:migrate
```

---

## Test Plan

### Unit Tests (Model)

| # | Scenario | Given | When | Then |
|---|----------|-------|------|------|
| 1 | Valid JPEG upload | A user with no avatar | They attach a valid JPEG under 5MB | The avatar is attached successfully |
| 2 | Valid PNG upload | A user with no avatar | They attach a valid PNG under 5MB | The avatar is attached successfully |
| 3 | Reject oversized file | A user with no avatar | They attach a 6MB JPEG | Validation fails with size error |
| 4 | Reject invalid type | A user with no avatar | They attach a GIF file | Validation fails with content type error |
| 5 | Replace existing avatar | A user with an existing avatar | They attach a new valid image | The old avatar is replaced |
| 6 | Thumbnail variant | A user with an attached avatar | `avatar_thumbnail` is called | Returns a 100x100 variant |
| 7 | Profile variant | A user with an attached avatar | `avatar_profile` is called | Returns a 300x300 variant |

### Integration Tests (Controller)

| # | Scenario | Given | When | Then |
|---|----------|-------|------|------|
| 8 | Upload via settings | An authenticated user on settings page | They submit the form with an avatar | Avatar is saved; redirects with success notice |
| 9 | Remove avatar | An authenticated user with an avatar | They click "Remove Avatar" | Avatar is purged; placeholder is shown |
| 10 | Upload with invalid file | An authenticated user on settings page | They submit a 10MB BMP | Form re-renders with validation errors |

### System Tests (Browser)

| # | Scenario | Given | When | Then |
|---|----------|-------|------|------|
| 11 | Avatar in nav bar | A user with an uploaded avatar | They visit any page | A 100x100 thumbnail appears in the nav |
| 12 | Avatar on profile | A user with an uploaded avatar | They visit their profile page | A 300x300 image is displayed |
| 13 | Default avatar fallback | A user with no avatar | They visit any page | The default placeholder image is shown |
| 14 | Direct upload progress | A user on settings page | They select a large file | A progress indicator appears during upload |

---

## Rollout Plan

1. **Development:** Implement model, controller, and view changes. Use local disk storage for dev/test.
2. **Staging:** Deploy with S3 configuration. Verify uploads, variants, and direct upload all work.
3. **Production:** Deploy behind no feature flag (low-risk, additive feature). Monitor S3 storage costs and error rates.

## Edge Cases and Considerations

- **Existing users:** All existing users will see the default placeholder until they upload an avatar. No data migration required.
- **Variant generation:** Variants are generated lazily on first request. The first load of a new avatar may be slightly slower.
- **S3 cleanup:** When an avatar is replaced or removed, `purge_later` enqueues an Active Job to delete the blob from S3. Ensure a background job processor (Sidekiq, etc.) is running.
- **CDN caching:** If a CDN sits in front of S3, variant URLs will contain a unique blob key, so cache invalidation is automatic on replacement.
- **HEIC/HEIF:** iOS devices may send HEIC images. These are explicitly not accepted per requirements. The client-side `accept` attribute and server-side validation will reject them. Consider adding HEIC support in a future iteration if user feedback warrants it.
