# WordPress Media Processing Diagnostics

A practical troubleshooting guide for diagnosing and resolving WordPress media upload and image processing problems.

This repository covers media upload failures, image editor compatibility, thumbnail generation, GD, Imagick, MIME types, attachment metadata, WebP processing, and server resource limitations.

## Table of Contents

- [Overview](#overview)
- [Common Media Processing Issues](#common-media-processing-issues)
- [Requirements](#requirements)
- [1. Check PHP Image Extensions](#1-check-php-image-extensions)
- [2. Check PHP Upload Configuration](#2-check-php-upload-configuration)
- [3. Diagnose Media Upload Failures](#3-diagnose-media-upload-failures)
- [4. Check WordPress Image Editor](#4-check-wordpress-image-editor)
- [5. Diagnose Thumbnail Generation](#5-diagnose-thumbnail-generation)
- [6. Inspect Registered Image Sizes](#6-inspect-registered-image-sizes)
- [7. Inspect Attachment Metadata](#7-inspect-attachment-metadata)
- [8. Diagnose MIME Type Issues](#8-diagnose-mime-type-issues)
- [9. Troubleshoot WebP Processing](#9-troubleshoot-webp-processing)
- [10. Fix Image Orientation](#10-fix-image-orientation)
- [11. Diagnose Large Image Problems](#11-diagnose-large-image-problems)
- [12. Regenerate Missing Thumbnails](#12-regenerate-missing-thumbnails)
- [13. Check Upload Directory Permissions](#13-check-upload-directory-permissions)
- [Useful WP-CLI Commands](#useful-wp-cli-commands)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [Best Practices](#best-practices)
- [Contributing](#contributing)
- [License](#license)

## Overview

WordPress includes a media handling system that manages file uploads, attachment records, image editing, metadata generation, and image resizing.

Media processing problems can affect both administrators and website visitors.

Common causes include:

- Missing PHP image processing extensions
- Incorrect upload configuration
- Unsupported image formats
- Insufficient PHP memory
- Image editor compatibility problems
- Incorrect file permissions
- Failed thumbnail generation
- Missing attachment metadata
- Server execution timeouts
- Conflicts with image optimization plugins

This repository provides practical diagnostic commands and troubleshooting examples for identifying these problems.

## Common Media Processing Issues

| Issue | Possible Cause |
|---|---|
| Image upload fails | PHP upload limits or permissions |
| HTTP error during upload | Server or image processing failure |
| Thumbnails are missing | Image editor or memory issue |
| Images appear rotated | EXIF orientation handling |
| WebP upload is rejected | MIME type or format support |
| Image editing fails | GD or Imagick configuration |
| Media library image is broken | Missing file or attachment metadata |
| Large images fail to process | Memory or execution limits |

## Requirements

Before troubleshooting, check the environment:

- WordPress installation
- Supported PHP version
- WP-CLI (recommended)
- PHP GD or Imagick extension
- Writable WordPress uploads directory
- Sufficient disk space
- Access to PHP and web server error logs

## 1. Check PHP Image Extensions

WordPress uses PHP image processing libraries to handle image uploads and resizing.

Check installed PHP extensions:

```bash
php -m
```

Look for:

```text
gd
imagick
```

Check extension details:

```bash
php -i | grep -i -E "gd|imagick"
```

On Windows:

```powershell
php -m | findstr /i "gd imagick"
```

**Important:** The PHP version used by the command line may differ from the version used by Apache or PHP-FPM. Always verify the web server environment as well.

## 2. Check PHP Upload Configuration

The following PHP settings can affect media uploads:

```ini
file_uploads = On
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 120
max_input_time = 120
```

These are example values. Actual requirements depend on the server and website.

Check the active PHP configuration:

```bash
php -i | grep -E "upload_max_filesize|post_max_size|memory_limit|max_execution_time"
```

On Windows:

```powershell
php -i | findstr /i "upload_max_filesize post_max_size memory_limit max_execution_time"
```

Things to verify:

- `file_uploads` must be enabled.
- `post_max_size` should be greater than or equal to `upload_max_filesize`.
- PHP must have enough memory to process large images.
- Web server and hosting-level upload limits must also be checked.

## 3. Diagnose Media Upload Failures

When WordPress cannot upload an image, investigate the following:

1. Confirm that the file format is supported.
2. Check the original file size.
3. Verify PHP upload limits.
4. Check the uploads directory permissions.
5. Confirm sufficient disk space.
6. Review PHP and web server error logs.
7. Temporarily test without image optimization plugins.

Find the WordPress uploads directory:

```bash
wp eval 'echo wp_upload_dir()["basedir"] . PHP_EOL;'
```

Inspect its permissions on Linux:

```bash
ls -ld wp-content/uploads
```

Avoid using unrestricted `777` permissions. Use the ownership and permissions recommended by your hosting provider.

## 4. Check WordPress Image Editor

WordPress supports image processing through available image editor implementations, including GD and Imagick.

Use WP-CLI to check whether WordPress can initialize an image editor:

```bash
wp eval '
$file = ABSPATH . "wp-includes/images/media/default.png";

$editor = wp_get_image_editor( $file );

if ( is_wp_error( $editor ) ) {
    echo "Image editor error: ";
    echo $editor->get_error_message() . PHP_EOL;
} else {
    echo "Active editor: ";
    echo get_class( $editor ) . PHP_EOL;
}
'
```

Possible results:

```text
WP_Image_Editor_Imagick
```

or:

```text
WP_Image_Editor_GD
```

If the editor cannot initialize, investigate:

- Missing PHP extensions
- Unsupported image formats
- Invalid image files
- Image library configuration
- Insufficient server memory

## 5. Diagnose Thumbnail Generation

WordPress generates additional image sizes according to the registered image size settings.

When thumbnails are missing or incomplete:

1. Verify that the original image exists.
2. Confirm that WordPress can read the original image.
3. Check the active image editor.
4. Inspect registered image sizes.
5. Review PHP memory and execution limits.
6. Check server error logs.
7. Test thumbnail generation with one attachment.

Do not repeatedly regenerate the entire media library before identifying the underlying problem.

## 6. Inspect Registered Image Sizes

WordPress supports built-in and custom image sizes.

Check registered image sizes:

```bash
wp media image-size
```

Example:

| Image Size | Width | Height |
|---|---:|---:|
| thumbnail | 150 | 150 |
| medium | 300 | 300 |
| medium_large | 768 | 0 |
| large | 1024 | 1024 |
| full | Original | Original |

Register a custom image size:

```php
add_action( 'after_setup_theme', function () {
    add_image_size( 'custom-card', 600, 400, true );
} );
```

The custom size will be available for newly processed images. Existing attachments may require thumbnail regeneration.

## 7. Inspect Attachment Metadata

WordPress stores attachment information, including generated image sizes and dimensions.

Inspect an attachment using its ID:

```bash
wp eval '
$id = 123;

$file = get_attached_file( $id );
$metadata = wp_get_attachment_metadata( $id );

echo "Attachment ID: " . $id . PHP_EOL;
echo "Original File: " . $file . PHP_EOL;
echo "File Exists: " . ( file_exists( $file ) ? "Yes" : "No" ) . PHP_EOL;

echo "Metadata:" . PHP_EOL;
print_r( $metadata );
'
```

Replace `123` with a valid attachment ID.

Check for:

- Missing original files
- Empty or incomplete metadata
- Missing generated image sizes
- Incorrect image dimensions
- Invalid attachment file paths

If the original image is missing, thumbnail regeneration cannot restore it.

## 8. Diagnose MIME Type Issues

WordPress checks file types before accepting uploads.

Inspect a file's detected MIME type:

```bash
wp eval '
$file = wp_check_filetype( "sample.webp" );

print_r( $file );
'
```

Check the allowed upload MIME types:

```bash
wp eval 'print_r( get_allowed_mime_types() );'
```

A custom MIME type can be registered with the `upload_mimes` filter:

```php
add_filter( 'upload_mimes', function ( $mimes ) {
    $mimes["webp"] = "image/webp";

    return $mimes;
} );
```

Only allow trusted formats that are appropriate for the website.

Allowing a file extension does not guarantee that the server can process the file.

## 9. Troubleshoot WebP Processing

WebP support depends on WordPress, PHP, and the available image editor.

Check whether WordPress recognizes WebP:

```bash
wp eval '
$result = wp_check_filetype( "sample.webp" );

print_r( $result );
'
```

Check installed image processing extensions:

```bash
php -m
```

If WebP processing fails, investigate:

- GD or Imagick WebP support
- PHP image library configuration
- WordPress version compatibility
- File integrity
- Memory limitations
- Image optimization plugin conflicts

Remember that accepting a WebP upload and generating WebP images are separate capabilities.

## 10. Fix Image Orientation

Images captured by mobile devices and cameras may contain EXIF orientation metadata.

Incorrect orientation handling can cause images to appear rotated.

WP-CLI provides an orientation correction command:

```bash
wp media fix-orientation 123
```

Replace `123` with the attachment ID.

Check command options:

```bash
wp help media fix-orientation
```

Test the command on a small number of images before running it across a large media library.

## 11. Diagnose Large Image Problems

Large images can consume significant memory during decoding and resizing.

The memory required to process an image can be much higher than its original file size.

Check PHP resource limits:

```bash
php -i | grep -E "memory_limit|max_execution_time"
```

Investigate:

- PHP memory limits
- Maximum execution time
- Image dimensions
- Available server memory
- Image editor compatibility
- Hosting-level resource restrictions

Do not increase resource limits blindly. Identify which operation is failing before changing server configuration.

## 12. Regenerate Missing Thumbnails

WP-CLI can regenerate image sizes for existing attachments.

Regenerate thumbnails for one attachment:

```bash
wp media regenerate 123
```

Generate only missing image sizes:

```bash
wp media regenerate --only-missing
```

Regenerate a specific image size:

```bash
wp media regenerate --image_size=large
```

Regenerate thumbnails for all attachments:

```bash
wp media regenerate --yes
```

Before bulk regeneration:

- Back up the uploads directory.
- Confirm that original images are available.
- Check available disk space.
- Test with a few attachment IDs.
- Run the operation during a low-traffic period.

## 13. Check Upload Directory Permissions

WordPress needs permission to write uploaded files and generated image sizes.

Find the configured uploads directory:

```bash
wp eval 'print_r( wp_upload_dir() );'
```

Inspect permissions:

```bash
ls -ld wp-content/uploads
```

Check disk space:

```bash
df -h
```

Verify that the web server user has the required access.

Do not change ownership or permissions without understanding the hosting environment.

## Useful WP-CLI Commands

| Command | Purpose |
|---|---|
| `wp media image-size` | List registered image sizes |
| `wp media regenerate 123` | Regenerate sizes for one attachment |
| `wp media regenerate --only-missing` | Generate missing image sizes |
| `wp media regenerate --image_size=large` | Regenerate a specific size |
| `wp media fix-orientation 123` | Correct image orientation |
| `wp media import image.jpg` | Import an image as an attachment |
| `wp help media` | Display media command help |

Replace example attachment IDs and file paths with actual values.

## Troubleshooting Checklist

Use this checklist before changing server or WordPress configuration.

- [ ] Confirm that the original image exists.
- [ ] Verify the image format.
- [ ] Check PHP upload limits.
- [ ] Check PHP memory limits.
- [ ] Verify GD or Imagick availability.
- [ ] Confirm the uploads directory is writable.
- [ ] Check available disk space.
- [ ] Review PHP and web server error logs.
- [ ] Verify registered image sizes.
- [ ] Inspect attachment metadata.
- [ ] Test image editor initialization.
- [ ] Test thumbnail generation.
- [ ] Check WebP support when required.
- [ ] Test without image optimization plugins.
- [ ] Back up files before bulk regeneration.

## Best Practices

- Keep PHP and image processing extensions updated.
- Use supported image formats.
- Avoid unnecessary custom image sizes.
- Keep original uploaded images backed up.
- Test bulk operations before running them on large media libraries.
- Avoid unrestricted MIME type changes.
- Monitor server memory and disk usage.
- Use compatible image optimization tools.
- Review server logs when image processing fails.
- Use WP-CLI for controlled media maintenance.

## Contributing

Contributions are welcome.

You can help improve this repository by:

- Reporting media processing issues.
- Adding troubleshooting examples.
- Improving diagnostic commands.
- Documenting hosting-specific problems.
- Correcting outdated instructions.

When reporting an issue, include:

- WordPress version
- PHP version
- GD or Imagick version
- Hosting environment
- Error message
- Steps to reproduce the problem

Do not include passwords, API keys, or other sensitive server information.

## License

This project is licensed under the MIT License.

You are free to use, modify, and distribute this material with appropriate attribution.

---

**WordPress Media Processing Diagnostics**

A practical reference for diagnosing and troubleshooting WordPress media upload and image processing problems.
