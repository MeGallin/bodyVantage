# Media Uploads

Profile images (authenticated):
- `POST /api/profileUpload`
  - Auth required.
  - Expects `multipart/form-data` with file field `userProfileImage`.
  - Stores image in Cloudinary `userProfileImage` folder, updates user `profileImage` + `cloudinaryId`, and creates a `UserProfileImages` record.
  - Response: 201 with `{ message, user, profileImage }`.

- `DELETE /api/profile-image/:id`
  - Auth required.
  - Deletes the specified profile image for the current user (removes from Cloudinary and DB).

Note: Cloudinary credentials (`CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_SECRET`) must be set in `api/.env`. Only jpg/jpeg/png files are accepted by the upload middleware.
