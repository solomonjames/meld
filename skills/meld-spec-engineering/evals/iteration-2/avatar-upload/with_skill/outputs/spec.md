# User Avatar Upload - Implementation Spec

**Feature:** Users can upload a profile picture (JPEG or PNG, max 5MB) from their settings page. The avatar displays as a 100x100 thumbnail in the nav bar and a 300x300 image on the profile page.

**Tech Stack:** Rails 7, Active Storage with S3, `image_processing` gem (libvips).

**Status:** ready-for-dev

---

## Implementation Tasks

- [ ] Task 1: Add the `image_processing` gem
  - File: `Gemfile`
  - Action: Add `gem "image_processing", "~> 1.2"` if not already present. Run `bundle install`.
  - Notes: This gem is required for Active Storage variants. Rails 7 defaults to libvips as the variant processor; confirm `config.active_storage.variant_processor = :vips` is set in `config/application.rb` (it is the default, but verify).

- [ ] Task 2: Attach avatar to the User model with validations
  - File: `app/models/user.rb`
  - Action: Add `has_one_attached :avatar`. Add custom validations for content type (JPEG/PNG only) and file size (max 5MB). Add helper methods `avatar_thumbnail` and `avatar_profile` that return pre-defined variants.
  - Notes: Use `avatar.content_type.in?(%w[image/jpeg image/png])` for type validation. Use `avatar.byte_size` for size validation. Variant helpers should use `resize_to_fill(100, 100)` for thumbnail and `resize_to_fill(300, 300)` for profile. Using `resize_to_fill` ensures non-square source images are cropped to the target dimensions without distortion.

  ```ruby
  # app/models/user.rb
  has_one_attached :avatar

  validate :acceptable_avatar

  def avatar_thumbnail
    avatar.variant(resize_to_fill: [100, 100])
  end

  def avatar_profile
    avatar.variant(resize_to_fill: [300, 300])
  end

  private

  def acceptable_avatar
    return unless avatar.attached?

    unless avatar.content_type.in?(%w[image/jpeg image/png])
      errors.add(:avatar, "must be a JPEG or PNG image")
    end

    if avatar.byte_size > 5.megabytes
      errors.add(:avatar, "must be less than 5MB")
    end
  end
  ```

- [ ] Task 3: Add avatar field to the settings controller
  - File: `app/controllers/settings_controller.rb` (or `app/controllers/users/settings_controller.rb` -- use whichever file currently handles user settings)
  - Action: Permit `:avatar` in the strong parameters for the update action. Add `:avatar` to the `user_params` permit list. Ensure the `update` action calls `current_user.update(user_params)` and handles validation failures by re-rendering the form with error messages.
  - Notes: If the controller uses a different name (e.g., `profiles_controller.rb`, `accounts_controller.rb`), apply the change there. The key requirement is that `:avatar` is permitted in whichever controller handles the settings form submission.

- [ ] Task 4: Add avatar upload form to the settings page
  - File: `app/views/settings/edit.html.erb` (or the corresponding view for the settings controller identified in Task 3)
  - Action: Add a file input field for avatar inside the existing settings form. Include a preview of the current avatar if one is attached, and a "Remove avatar" option. Display validation errors for the avatar field.
  - Notes: Use `form.file_field :avatar, accept: "image/jpeg,image/png"` for client-side filtering. The `accept` attribute is a hint only; server-side validation (Task 2) is the enforcement mechanism. Wrap in `form_with(model: current_user, url: settings_path, method: :patch)` or the existing form structure.

  ```erb
  <div class="avatar-upload">
    <% if current_user.avatar.attached? %>
      <%= image_tag current_user.avatar_profile, alt: "Current avatar", class: "avatar-preview" %>
    <% end %>

    <%= form.label :avatar, "Profile Picture" %>
    <%= form.file_field :avatar, accept: "image/jpeg,image/png", direct_upload: true %>
    <p class="help-text">JPEG or PNG, max 5MB</p>

    <% if current_user.errors[:avatar].any? %>
      <div class="error-messages">
        <% current_user.errors[:avatar].each do |error| %>
          <p class="error"><%= error %></p>
        <% end %>
      </div>
    <% end %>

    <% if current_user.avatar.attached? %>
      <%= button_to "Remove avatar", remove_avatar_settings_path, method: :delete, class: "btn-link" %>
    <% end %>
  </div>
  ```

- [ ] Task 5: Add remove avatar action
  - File: `app/controllers/settings_controller.rb`
  - Action: Add a `remove_avatar` action that calls `current_user.avatar.purge_later` and redirects back to settings with a flash notice.
  - Notes: `purge_later` enqueues an Active Job to delete the blob from S3 asynchronously, avoiding blocking the request.

- [ ] Task 6: Add route for remove avatar
  - File: `config/routes.rb`
  - Action: Add a DELETE route for the remove avatar action. Example: `resource :settings, only: [:edit, :update] do delete :remove_avatar, on: :member end` -- adapt to the existing route structure.
  - Notes: If settings routes already exist as a singular resource, nest the `remove_avatar` action inside. If using a different route structure, add `delete '/settings/remove_avatar', to: 'settings#remove_avatar', as: :remove_avatar_settings`.

- [ ] Task 7: Display avatar thumbnail in the nav bar
  - File: `app/views/layouts/_navbar.html.erb` (or `app/views/shared/_navbar.html.erb`, or wherever the nav bar partial lives)
  - Action: Replace or augment the existing user identifier in the nav bar with the avatar thumbnail. Fall back to a default placeholder when no avatar is attached.
  - Notes: Use `current_user.avatar.attached?` to conditionally render. Serve the variant, not the original. Add CSS class for 100x100 circular display.

  ```erb
  <% if current_user&.avatar&.attached? %>
    <%= image_tag current_user.avatar_thumbnail, alt: current_user.name, class: "nav-avatar", size: "100x100" %>
  <% else %>
    <%= image_tag "default-avatar.png", alt: "Default avatar", class: "nav-avatar", size: "100x100" %>
  <% end %>
  ```

- [ ] Task 8: Display avatar on the profile page
  - File: `app/views/profiles/show.html.erb` (or wherever the user profile is rendered)
  - Action: Display the 300x300 avatar variant on the profile page. Fall back to a default placeholder.
  - Notes: Same conditional pattern as the nav bar, using `avatar_profile` instead of `avatar_thumbnail`.

- [ ] Task 9: Add default avatar placeholder image
  - File: `app/assets/images/default-avatar.png`
  - Action: Add a neutral default avatar image (e.g., a silhouette or initials icon) at 300x300 resolution. The nav bar will scale it down via CSS; the profile page uses it at native size.
  - Notes: Use a simple, brand-appropriate placeholder. PNG with transparent background is preferred.

- [ ] Task 10: Add CSS for avatar display
  - File: `app/assets/stylesheets/avatars.css` (or equivalent in the existing stylesheet structure)
  - Action: Add styles for `.nav-avatar` (100x100, `border-radius: 50%`, `object-fit: cover`), `.avatar-preview` (300x300, `border-radius: 8px`), and `.avatar-upload` form section styling.
  - Notes: Ensure `object-fit: cover` is set so the browser handles any minor aspect ratio differences gracefully. If using Tailwind or another utility framework, use utility classes in the templates instead and skip this file.

---

## Acceptance Criteria

### Happy Path

**AC-1: Successful avatar upload**
- Given a signed-in user on the settings page with no avatar
- When they select a valid JPEG file (under 5MB) and submit the form
- Then the avatar is saved to S3 via Active Storage
- And the settings page re-renders showing the uploaded image as a 300x300 preview
- And a success flash message "Avatar updated" is displayed

**AC-2: Avatar displays in nav bar**
- Given a signed-in user who has uploaded an avatar
- When they navigate to any page in the application
- Then their avatar is displayed in the nav bar as a 100x100 circular thumbnail

**AC-3: Avatar displays on profile page**
- Given a user who has uploaded an avatar
- When any user visits that user's profile page
- Then the avatar is displayed as a 300x300 image

**AC-4: PNG upload accepted**
- Given a signed-in user on the settings page
- When they select a valid PNG file (under 5MB) and submit the form
- Then the avatar is saved successfully
- And the image renders correctly in both the nav bar and profile page

### Error Handling

**AC-5: Oversized file rejected**
- Given a signed-in user on the settings page
- When they attempt to upload a PNG file larger than 5MB
- Then the upload is rejected with the error "Avatar must be less than 5MB"
- And the existing avatar (if any) remains unchanged

**AC-6: Invalid file type rejected**
- Given a signed-in user on the settings page
- When they attempt to upload a GIF file
- Then the upload is rejected with the error "Avatar must be a JPEG or PNG image"
- And the existing avatar (if any) remains unchanged

**AC-7: Invalid file type rejected - non-image**
- Given a signed-in user on the settings page
- When they attempt to upload a PDF or text file
- Then the upload is rejected with the error "Avatar must be a JPEG or PNG image"
- And the existing avatar (if any) remains unchanged

**AC-8: S3 upload failure handled gracefully**
- Given a signed-in user on the settings page
- When they upload a valid image but the S3 upload fails (network error, permissions)
- Then an error message is displayed ("Upload failed, please try again")
- And the existing avatar (if any) remains unchanged

### Edge Cases

**AC-9: Default avatar for users with no upload**
- Given a signed-in user who has never uploaded an avatar
- When they view any page with avatar display (nav bar, profile)
- Then a default placeholder image is shown instead of a broken image

**AC-10: Replacing an existing avatar**
- Given a signed-in user who already has an avatar uploaded
- When they upload a new valid image
- Then the new image replaces the old one in both the nav bar and profile page
- And the old image blob is eventually purged from S3

**AC-11: Removing an avatar**
- Given a signed-in user who has an avatar uploaded
- When they click "Remove avatar" on the settings page
- Then the avatar is removed
- And the default placeholder is shown in the nav bar and profile page
- And a flash notice "Avatar removed" is displayed

**AC-12: File exactly at 5MB boundary**
- Given a signed-in user on the settings page
- When they upload a JPEG file that is exactly 5MB (5,242,880 bytes)
- Then the upload is accepted successfully

**AC-13: File just over 5MB boundary**
- Given a signed-in user on the settings page
- When they upload a JPEG file that is 5,242,881 bytes
- Then the upload is rejected with the error "Avatar must be less than 5MB"

**AC-14: Re-uploading the same image**
- Given a signed-in user who has an avatar
- When they upload the exact same image file again
- Then the upload succeeds without error
- And the avatar continues to display correctly

### Authorization

**AC-15: Unauthenticated user cannot upload**
- Given a user who is not signed in
- When they attempt to access the settings page or POST an avatar upload
- Then they are redirected to the sign-in page

**AC-16: User cannot modify another user's avatar**
- Given signed-in User A
- When User A attempts to update User B's avatar (e.g., by manipulating the form target)
- Then the request is rejected or ignored
- And User B's avatar remains unchanged

**AC-17: Avatar visibility on public profiles**
- Given User A has uploaded an avatar
- When User B (signed-in or not, depending on app visibility settings) visits User A's profile
- Then User A's avatar is displayed (not User B's, not a broken image)

---

## Testing Strategy

### Unit Tests

**File: `test/models/user_test.rb` (or `spec/models/user_spec.rb`)**

- User model accepts JPEG attachment (maps to AC-1)
- User model accepts PNG attachment (maps to AC-4)
- User model rejects GIF attachment with correct error message (maps to AC-6)
- User model rejects PDF attachment with correct error message (maps to AC-7)
- User model rejects file over 5MB with correct error message (maps to AC-5)
- User model accepts file exactly at 5MB (maps to AC-12)
- User model rejects file at 5MB + 1 byte (maps to AC-13)
- `avatar_thumbnail` returns a variant with 100x100 dimensions (maps to AC-2)
- `avatar_profile` returns a variant with 300x300 dimensions (maps to AC-3)
- User without avatar does not raise errors when calling `avatar.attached?` (maps to AC-9)

### Integration / Feature Tests

**File: `test/integration/avatar_upload_test.rb` (or `spec/features/avatar_upload_spec.rb`)**

- Signed-in user uploads a valid JPEG via settings form and it renders in the nav bar as a 100x100 thumbnail (maps to AC-1, AC-2)
- Signed-in user uploads a valid PNG and it displays on the profile page at 300x300 (maps to AC-4, AC-3)
- Uploading a GIF shows validation error and does not change avatar (maps to AC-6)
- Uploading a file over 5MB shows size validation error (maps to AC-5)
- Replacing an existing avatar with a new image updates both nav and profile display (maps to AC-10)
- Removing an avatar shows default placeholder in nav bar (maps to AC-11, AC-9)
- Unauthenticated request to settings redirects to sign-in (maps to AC-15)

### Controller Tests

**File: `test/controllers/settings_controller_test.rb` (or `spec/controllers/settings_controller_spec.rb`)**

- `PATCH /settings` with valid avatar params updates the user's avatar (maps to AC-1)
- `PATCH /settings` with invalid file type returns 422 / re-renders form (maps to AC-6)
- `DELETE /settings/remove_avatar` purges the avatar and redirects (maps to AC-11)
- `PATCH /settings` by unauthenticated user redirects to sign-in (maps to AC-15)
- `PATCH /settings` cannot modify another user's record (maps to AC-16)

### Manual Testing

- Verify the uploaded avatar renders correctly at 100x100 in the nav bar across different browsers (Chrome, Firefox, Safari). Check that circular cropping via `border-radius: 50%` looks correct with various aspect ratios of source images. (maps to AC-2)
- Verify the 300x300 profile image looks sharp and is not pixelated from upscaling a small source image. (maps to AC-3)
- Test with a very large image (e.g., 4000x3000 JPEG at 4.9MB) to confirm variant generation completes in a reasonable time and does not time out. (maps to AC-1)
- Confirm the S3 bucket is receiving blobs by checking the AWS console after upload. (maps to AC-1)

---

## Dependencies

- **`image_processing` gem (~> 1.2):** Required for Active Storage variant generation. Must be in the Gemfile.
- **libvips:** System dependency for image processing. Must be installed on all environments (development, CI, production). Install via `brew install vips` (macOS) or `apt-get install libvips-dev` (Ubuntu).
- **Active Storage:** Must already be installed and configured with S3. Assumes `config/storage.yml` has an S3 service configured and `config.active_storage.service` is set to the S3 service in production.
- **Active Job backend:** `purge_later` (used in remove avatar) requires an Active Job backend (Sidekiq, Delayed Job, etc.) to be configured. If using `async` adapter in development, purging happens in-process.

## Notes

- **Direct uploads:** The template in Task 4 includes `direct_upload: true` on the file field. This uploads directly from the browser to S3, bypassing the Rails server for large files. This requires the Active Storage JavaScript (`@rails/activestorage`) to be loaded. If direct uploads are not desired, remove the `direct_upload: true` attribute and uploads will go through the Rails server.
- **Variant caching:** Active Storage caches processed variants. The first request for a variant will be slow (processing + S3 upload of variant); subsequent requests serve the cached variant directly from S3. No additional caching configuration is needed.
- **Content type spoofing:** The server-side validation checks `content_type` from the upload metadata. A malicious user could spoof the content type header. For stronger security, consider adding `marcel` gem-based content type detection from file bytes, or use Active Storage's built-in `Marcel` analyzer which Rails uses internally.
- **Cleanup of replaced blobs:** When a user uploads a new avatar, Active Storage automatically purges the old blob. When using `purge_later`, the old S3 object is deleted asynchronously via Active Job.
- **Performance risk:** Variant generation for very large images can be CPU-intensive. If this becomes a problem, consider generating variants asynchronously or setting a pixel dimension limit in addition to the file size limit.

---

## Ready-for-Dev Checklist

| Criterion | Status | Evidence |
|-----------|--------|----------|
| **Actionable** | Pass | Every task specifies a file path and concrete action with code samples where relevant |
| **Logical** | Pass | Tasks ordered: gem (1) -> model (2) -> controller (3) -> view (4) -> remove action (5) -> routes (6) -> nav bar (7) -> profile (8) -> assets (9, 10) |
| **Testable** | Pass | 17 ACs in Given/When/Then covering happy path, errors, edge cases, and authorization |
| **Complete** | Pass | No TBD, TODO, or placeholder values. All validation rules, variant dimensions, and file paths specified |
| **Self-Contained** | Pass | Spec includes tech stack, all code snippets, dependencies, and can be implemented without external context |
