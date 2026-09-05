### [⤺ Back to Table of Contents](README.md#divergence-framework-documentation)

# Media

The media system pairs a database record with files on disk. `Divergence\Models\Media\Media` inspects the MIME type and selects the registered image, document, video, or audio model. `MediaRequestHandler` provides the HTTP endpoints.

## Server Setup

The features you use determine what needs to be installed:

| Feature | Dependencies used by the implementation |
| --- | --- |
| MIME detection | PHP Fileinfo |
| Image loading and thumbnails | PHP GD; EXIF for JPEG orientation |
| PDF, SVG, and some image conversions | ImageMagick's `convert` command with the required format support |
| PostScript conversion | Ghostscript's `gs` command |
| Video analysis, frames, encoding, and audio previews | `ffprobe` and `ffmpeg` |

Media is stored under `ApplicationPath/media/`, outside `public/`. The original is normally `media/original/<ID>.<extension>`. Thumbnails and encoded variants have their own subdirectories. The application needs permission to create and write those files, plus temporary-file space.

Database backups alone are not enough. Back up the media directory too.

## Creating Media

After bootstrapping the App and database connection:

```php
use Divergence\Models\Media\Media;

$media = Media::createFromFile('/path/to/photo.jpg', [
    'Caption' => 'A very good photo',
]);

if ($media) {
    echo $media->ID;
    echo $media->getFilesystemPath();
    echo $media->getMIMEType();
}
```

This analyzes the file, creates and saves the model, and writes the file. It is not just an unsaved model factory. Failures can throw exceptions. Database writes and filesystem writes are separate operations, not one atomic transaction.

`Media::getSupportedTypes()` returns the registered MIME types. Don't infer support from a filename extension alone.

`createFromFile()` also accepts URLs and fetches them from the server. Do not pass an arbitrary user-supplied URL to it; restrict remote imports at the application boundary.

For an actual PHP upload, validate the upload error first and pass the temporary filename to `createFromUpload()`. The HTTP upload endpoint already follows that path. Prefer the filename form: the array overload currently rejects an array whenever its `error` key is set, even if the value is `UPLOAD_ERR_OK`.

## HTTP Routes

Route your `/media` branch to `MediaRequestHandler`, or an application subclass with your access policy. The action comes before the identifier:

| Route | Purpose |
| --- | --- |
| `/media/json/browse` | Browse media as JSON |
| `/media/json/info/1` | Metadata for media ID 1 |
| `/media/open/1` | Open the media stream |
| `/media/download/1/example.jpg` | Download with an optional filename |
| `/media/json/upload` | Upload media and return JSON |
| `/media/json/caption/1` | Read or update the caption |
| `/media/json/delete/1` | Delete endpoint |
| `/media/thumbnail/1/100x100` | Thumbnail through the media handler |

POST uploads use multipart field `mediaFile` by default:

```bash
curl -X POST http://localhost:8080/media/json/upload \
    -F 'mediaFile=@photo.jpg' \
    -F 'Caption=A very good photo'
```

PUT uploads read the request body as the file:

```bash
curl -X PUT http://localhost:8080/media/json/upload \
    --data-binary @photo.jpg
```

Those examples assume your application's authentication and permissions allow the request. See [Security](security.md#binding-permissions), especially the media access-hook caveat.

## Thumbnails and Variants

```php
$thumbnailPath = $media->getThumbnail(200, 200);
$available = $media->isVariantAvailable('original');
```

`getThumbnail()` returns a filesystem path and can create the thumbnail when it is missing. A thumbnail request is therefore potentially processing work, not just a file read.

The default `Media::$thumbnailRequestFormat` generates `/thumbnail/...` URLs. If you keep that format, route a top-level `/thumbnail` branch to the media handler as shown in the [controller examples](controllers.md#intro-to-tree-routing). If you only mount the media handler under `/media`, change the format to match that route.

Video variants depend on the configured encoding profiles and may not be ready immediately after upload. Check availability before assuming an encoded file exists.

## Streaming

Media responses use a stream and the emitter sends its contents. The handler supports a single byte range, such as `bytes=0-1023`, and rejects multiple ranges with 416. Do not assume full HTTP range semantics from the presence of an `Accept-Ranges` header; verify the request forms your client uses against the handler.

Keep files outside the public web root when access must go through the application. A direct static-file URL bypasses controller permissions. Upload limits, permitted media types, conversion resource limits, authorization, and cleanup policy belong in the application and server configuration.
