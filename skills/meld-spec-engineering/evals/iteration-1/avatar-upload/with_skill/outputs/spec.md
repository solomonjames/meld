# User Avatar Upload — Implementation Spec

## Overview

Add profile picture upload to the user settings page. Users can upload a JPEG or PNG image (max 5MB) that is stored via Active Storage on S3. The avatar is displayed as a 100x100 thumbnail in the navigation bar and a 300x300 image on the profile page.

---

## Implementation Tasks

- [ ] Task 1: Add avatar attachment to User model
  - File: `app/models/user.rb`
  - Action: Add `has_one_attached :avatar` association. Add validation method for content type (JPEG/PNG) and file size (max 5MB). Add helper methods `avatar_thumbnail` and `avatar_profile` that return resized variants (100x100 and 300x300 respectively).
  - Notes: Use `avatar.variant(resize_to_fill: [100, 100])` for thumbnail and `resize_to_fill: [300, 300]` for profile. Variants require the `image_processing` gem with `vips` or `mini_magick` processor. Validations should use `avatar.content_type` and `avatar.byte_size`.

- [ ] Task 2: Verify image processing dependency is configured
  - File: `Gemfile`
  - Action: Confirm `image_processing` gem is present. If not, add `gem "image_processing", "~> 1.2"` and run `bundle install`.
  - Notes: Rails 7 Active Storage variants require this gem. The app likely already has it if Active Storage is configured, but verify.

- [ ] Task 3: Add avatar field to settings controller
  - File: `app/controllers/settings_controller.rb` (or `app/controllers/users/registrations_controller.rb` if using Devise)
  - Action: Permit `:avatar` in strong parameters. Add `purge_avatar` handling if the user wants to remove their avatar. In the update action, ensure avatar is attached via standard Active Storage parameter assignment.
  - Notes: If the controller uses `user_params`, add `:avatar` to the permitted list. Also permit `:remove_avatar` (a virtual attribute) to support avatar removal.

- [ ] Task 4: Add remove_avatar virtual attribute to User model
  - File: `app/models/user.rb`
  - Action: Add `attribute :remove_avatar, :boolean, default: false`. Add a `before_save` callback that calls `avatar.purge` if `remove_avatar` is truthy and avatar is attached.
  - Notes: This supports the "Remove avatar" checkbox on the settings form.

- [ ] Task 5: Add avatar upload form to settings page
  - File: `app/views/settings/edit.html.erb` (or equivalent settings view)
  - Action: Add a file input field for avatar upload inside the existing settings form. Include a preview of the current avatar (if attached) with a "Remove avatar" checkbox. Set `form_with` to use `multipart: true` if not already set.
  - Notes: Use `<%= form.file_field :avatar, accept: "image/jpeg,image/png" %>` for client-side filtering. Display current avatar with `<%= image_tag user.avatar_thumbnail if user.avatar.attached? %>`. Fall back to a default placeholder image when no avatar is attached.

- [ ] Task 6: Display avatar thumbnail in navigation bar
  - File: `app/views/layouts/_navbar.html.erb` (or equivalent navigation partial)
  - Action: Replace or augment the current user identifier (name/icon) with the avatar thumbnail. Use `current_user.avatar_thumbnail` if avatar is attached, otherwise render a default placeholder.
  - Notes: Render as `<%= image_tag current_user.avatar_thumbnail, class: "nav-avatar", size: "100x100" %>`. Add CSS class `nav-avatar` for styling (border-radius for circular display, etc.). Only render for authenticated users.

- [ ] Task 7: Display avatar on profile page
  - File: `app/views/profiles/show.html.erb` (or equivalent profile view)
  - Action: Display the 300x300 avatar variant. Fall back to a default placeholder if no avatar is attached.
  - Notes: Use `<%= image_tag @user.avatar_profile, class: "profile-avatar", size: "300x300" %>`. This view may be visible to other users, so use `@user` (the profile being viewed), not `current_user`.

- [ ] Task 8: Add avatar CSS styles
  - File: `app/assets/stylesheets/avatars.css` (new file, or append to existing component stylesheet)
  - Action: Add styles for `.nav-avatar` (100x100, border-radius: 50%, object-fit: cover) and `.profile-avatar` (300x300, border-radius: 50%, object-fit: cover). Add `.avatar-placeholder` style for the default state.
  - Notes: Use `object-fit: cover` to prevent distortion of non-square source images.

- [ ] Task 9: Add default avatar placeholder
  - File: `app/assets/images/default_avatar.png`
  - Action: Add a default placeholder image (simple silhouette or initials-based icon) used when no avatar is uploaded.
  - Notes: Should work at both 100x100 and 300x300 display sizes. Can use an SVG alternatively at `app/assets/images/default_avatar.svg`.

---

## Acceptance Criteria

**AC-1: Successful avatar upload**
- Given a signed-in user on the settings page
- When they select a valid JPEG file under 5MB and submit the form
- Then the avatar is stored in S3 via Active Storage, the page reloads with a success flash, and the uploaded image is visible as a preview on the settings page

**AC-2: PNG upload accepted**
- Given a signed-in user on the settings page
- When they select a valid PNG file under 5MB and submit the form
- Then the avatar is stored successfully and displayed correctly

**AC-3: Navigation bar thumbnail display**
- Given a signed-in user with an uploaded avatar
- When they view any page in the application
- Then their avatar is displayed as a 100x100 thumbnail in the navigation bar

**AC-4: Profile page display**
- Given a user with an uploaded avatar
- When any user views that user's profile page
- Then the avatar is displayed as a 300x300 image

**AC-5: Avatar replacement**
- Given a signed-in user who already has an avatar
- When they upload a new image on the settings page
- Then the old avatar is replaced by the new one, and the new image is displayed in all locations

**AC-6: Avatar removal**
- Given a signed-in user who has an avatar
- When they check "Remove avatar" and submit the settings form
- Then the avatar is purged from storage and the default placeholder is shown in all locations

**AC-7: File type validation — invalid type rejected**
- Given a signed-in user on the settings page
- When they attempt to upload a GIF, BMP, WebP, or other non-JPEG/PNG file
- Then the upload is rejected, a validation error is displayed ("Avatar must be a JPEG or PNG image"), and the existing avatar (if any) is unchanged

**AC-8: File size validation — oversized file rejected**
- Given a signed-in user on the settings page
- When they attempt to upload a JPEG or PNG file larger than 5MB
- Then the upload is rejected, a validation error is displayed ("Avatar must be less than 5MB"), and the existing avatar (if any) is unchanged

**AC-9: Default placeholder when no avatar**
- Given a user who has never uploaded an avatar
- When their profile page or the navigation bar is rendered
- Then a default placeholder image is displayed instead of a broken image

**AC-10: Unauthenticated user cannot upload**
- Given a user who is not signed in
- When they attempt to access the settings page or submit an avatar upload
- Then they are redirected to the sign-in page

**AC-11: Variant generation**
- Given a user uploads a large (e.g., 4000x3000) JPEG image
- When the thumbnail or profile variant is requested
- Then Active Storage generates the correctly sized variant (100x100 or 300x300) using `resize_to_fill`, and the variant is cached for subsequent requests

**AC-12: Non-square image handling**
- Given a user uploads a non-square image (e.g., 1200x800)
- When the avatar is displayed
- Then the image is cropped to center and fills the display area without distortion (via `resize_to_fill` and `object-fit: cover`)

---

## Testing Strategy

### Unit tests

- **User model validations** (`test/models/user_test.rb` or `spec/models/user_spec.rb`):
  - Validates that JPEG and PNG content types are accepted
  - Validates that GIF, BMP, WebP, and other types are rejected
  - Validates that files under 5MB are accepted
  - Validates that files over 5MB are rejected
  - Validates that `avatar_thumbnail` returns a variant with 100x100 dimensions
  - Validates that `avatar_profile` returns a variant with 300x300 dimensions
  - Validates that `remove_avatar = true` triggers purge on save

### Feature / integration tests

- **Avatar upload flow** (`test/integration/avatar_upload_test.rb` or `spec/features/avatar_upload_spec.rb`):
  - Upload a valid JPEG, verify it persists and renders on settings page
  - Upload a valid PNG, verify it persists
  - Upload an invalid file type, verify rejection with error message
  - Upload an oversized file, verify rejection with error message
  - Replace an existing avatar, verify the new image is shown
  - Remove avatar via checkbox, verify placeholder is shown
  - Verify avatar thumbnail appears in navigation bar after upload
  - Verify avatar appears on profile page after upload

- **Authorization** (`test/integration/avatar_authorization_test.rb`):
  - Unauthenticated user is redirected when attempting to access settings
  - User A cannot modify User B's avatar

### Manual testing

- Upload various image dimensions (portrait, landscape, square) and verify cropping looks correct
- Verify S3 storage by checking the bucket after upload
- Test on mobile viewport to confirm avatar rendering in nav bar
- Verify variant caching by loading the thumbnail multiple times and checking S3/CDN behavior

---

## Dependencies

- **Gems:** `image_processing` (~> 1.2) — required for Active Storage variant generation
- **System:** `libvips` (preferred) or `imagemagick` must be installed on the server for image processing
- **Infrastructure:** Active Storage already configured with S3 (per requirements) — no additional S3 setup needed
- **Config:** Verify `config/storage.yml` has the S3 service configured and `config.active_storage.service` is set to the S3 service in production

## Notes

- **Pre-mortem risk:** Large image uploads may time out on slow connections. Consider adding a client-side file size check (JavaScript) to provide immediate feedback before the form is submitted.
- **Pre-mortem risk:** Variant generation on first request adds latency. For high-traffic apps, consider using `avatar.variant(...).processed` in a background job after upload to pre-generate variants.
- **Trade-off:** `resize_to_fill` crops the image to fill the exact dimensions, which may cut off parts of the image. This is standard for avatar use cases but could be surprising if users upload group photos. The settings page preview mitigates this by showing the result immediately.
- **Future consideration:** Adding client-side image cropping (e.g., via Croppr.js) before upload would give users control over the crop area.
- **Future consideration:** Support for WebP could reduce storage and bandwidth costs but is not in scope for this iteration.

---

## Ready-for-Dev Checklist

| Criterion | Status |
|-----------|--------|
| **Actionable** | All 9 tasks have specific file paths and actions |
| **Logical** | Tasks ordered by dependency: model -> controller -> views -> styles |
| **Testable** | 12 ACs in Given/When/Then covering happy path, errors, edge cases, authorization |
| **Complete** | No placeholders or TBDs — all implementation details inlined |
| **Self-Contained** | A fresh agent can implement from this spec without additional context |
