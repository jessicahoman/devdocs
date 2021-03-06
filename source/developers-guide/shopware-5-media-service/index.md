---
layout: default
title: MediaService
github_link: developers-guide/shopware-5-media-service/index.md
shopware_version: 5.1.0
indexed: true
---

The `Shopware\Bundle\MediaBundle` defines how Shopware manages its media files. Shopware 5.1 includes a new media management layer, which abstracts the location of your media files. This applies to both new and existing installations, and there is no possibility to revert to the old behaviour.

## The problem

Since the beginning of Shopware, the media files were organized in a folder called `media` with sub-folders for each media type, e.g. image, video, and so on. Inside a sub-folder, every file has just been thrown in. The problem is that, if you have a huge amount of media files, file operations get very slow, especially on Windows systems.

For that reason, we decided to apply commonly used techniques to our media management and introduce the new MediaService.

## The solution

The main goal of the MediaService is to abstract and handle all file operations, so that you don't have to worry about solving common problems associated with it. The key idea to keep in mind is that you shouldn't directly perform any file system operation on your files. Should you need to read, move or delete your files, you should always use the MediaService. 

In the past, the most common use-case was to load a path like `media/image/my-fancy-image.png` from the database and, during the template render process, prepend the base url to it. These paths are still used, but now as **virtual paths**. These paths identify your files, and are used by the MediaService to access them. Note that the actual file will **not** be in this exact location in your file system. It's up to the MediaService to retrieve the real path from this virtual path.

## Handling your files using the MediaService

The following code snippets should be self-explanatory. They assume you already retrieved both the virtual `$path` from the Media entity and the `$mediaService` instance from the DI container:

```php
$path = 'media/image/my-fancy-image.png';
$mediaService = $container->get('shopware_media.media_service');
```

#### URL generation

The following example shows how to generate a url based on the virtual path.

```php
$url = $mediaService->getUrl($path);
// result: https://www.myshop.com/media/image/0a/20/03/my-fancy-image.png
```

Simply get the `shopware_media.media_service` from the DI container and call `getUrl()` with your virtual path. As a result you'll get a full qualified URL and you are able to bind it to a view. There is no need to use the `{link ...}` Smarty expression.

In your Smarty templates you may use the `{media path=...}` expression to get the fully qualified URL.

```smarty
<img href="{media path="media/image/my-fancy-image.png"}">
```

`{media}` evaluates the given path at template's compile time, so you cannot use runtime variables for its path argument (generally you will use a constant path as in the example above).


#### Check if a files exists

This should be used as replacement for `file_exists()`

```php
$fileExists = $mediaService->has($path);
```

#### Reading

This should be used as replacement for `file_get_contents()` and `fopen()`/`fread()`

```php
$fileContent = $mediaService->read($path);
$fileStream = $mediaService->readStream($path);
```

#### Writing

This should be used as a replacement for `file_put_contents()` and `fopen()`/`fwrite()`

```php
$mediaService->write($path, $fileContent);
$mediaService->writeStream($path, $fileStream);
```

#### Deleting

This should be used as a replacement for `unlink()`

```php
$mediaService->delete($path);
```

#### Moving / Renaming

This should be used as a replacement for `rename()`

```php
$mediaService->rename($path, $newPath);
```

### Handling virtual paths

**Virtual paths**, like the examples shown above, are used by the MediaService to access your files. To better handle all kinds of paths, the MediaService includes a `normalize($path)` method that you can use determine the correct virtual path from different variations of the file's path. As an example, the following 3 paths will all be converted into the correct format (`media/image/my-fancy-image.png`) by the path normalizer:

* `https://www.myshop.com/shop/media/image/my-fancy-image.png`
* `/var/www/shop1/media/image/my-fancy-image.png`
* `media/image/5c/af/3e/my-fancy-image.png`

You can use this normalizer too by calling `$mediaService->normalize($path)`. Note that, when using the MediaService operations, you don't need to explicitly normalize the paths, as this is done for you automatically. As such, the following lines would all produce the same result:

```php
$url = $mediaService->getUrl('media/image/my-fancy-image.png');
$url = $mediaService->getUrl('/var/www/shop1/media/image/my-fancy-image.png');
$url = $mediaService->getUrl($mediaService->normalize('/var/www/shop1/media/image/my-fancy-image.png'));
```

## File system adapters

Using the MediaService allows Shopware to better cope with problems like the one described in this document's first section. However, abstracting the media file system also provides a way to support distributed hosting of content. This can be done by using **file system adapters**. These adapters allow you to store files in places other than your current server, like a FTP server or a CDN service provider. As these adapters are abstracted by the MediaService, they are transparent for the controller logic, and will work out of the box with any Shopware plugin that uses the MediaService.

By default, adapters for local and FTP based file systems are included in Shopware 5.1. We provide an [Amazon S3 adapter](https://github.com/ShopwareLabs/SwagMediaS3) and a [SFTP adapter](https://github.com/ShopwareLabs/SwagMediaSftp). You can download and install them just like any other Shopware plugin. Keep in mind that no official support will be provided for these plugins.

### Migrating your files

To make migrations easy, we've added a new CLI command to migrate all of your media files at once to the location specified by the currently configured adapter. 

```bash
bin/console sw:media:migrate
```

If you are upgrading to Shopware 5.1 or later from 5.0 or previous version, you might also use this command to migrate your files from the legacy location to their new folder. If you can't or decide not to run this command, your media files will still be migrated. The live migration mechanism moves the files to the right place as they get requested. However, we recommend that you use the CLI command to migrate your files, as the live migration might impact your shop's performance.

<div class="alert alert-info">
<strong>Important information for nginx users</strong><br/>
If you are still facing problems with media files, you should update your nginx configuration to the latest version. A working configuration can be found <a href="https://github.com/bcremer/shopware-with-nginx">at GitHub</a>. It adds a location for the media folder and redirects a failed image lookup to a new frontend controller, which then tries to find the requested image.
</div>

#### Example: Migrating all media files to Amazon S3

Assuming you already installed the [Amazon S3 adapter](https://github.com/ShopwareLabs/SwagMediaS3) plugin, you now need to configure it. To do so, you have to edit your Shopware `config.php` and add a new adapter called `s3` and a configuration to access Amazon S3 account like described in the [plugin description on GitHub](https://github.com/ShopwareLabs/SwagMediaS3).

You can now simply run this command to move all files from your local media file system to Amazon S3.

```bash
bin/console sw:media:migrate --from=local --to=s3
```

<div class="alert alert-warning">
After executing this command, you won't have any media file left on your local file system. The migration itself might take some time, depending on the number of media files you have. If you cancel the migration or your server crashes, you can continue the migration by running the command again.
</div>

Should you want to revert the migration to Amazon S3, you can use the following command:

```bash
bin/console sw:media:migrate --from=s3 --to=local
```

Again, keep in mind that the live migration mechanism will still be in place, meaning that, if your Amazon S3 adapter is still configured, files will be migrated back to Amazon's servers when loaded by any incoming frontend request.

#### Build your own adapter

Since our MediaService is built on top of [Flysystem](http://flysystem.thephpleague.com), feel free to create your own adapter and share it with the community. You can also take the provided [Amazon S3 adapter](https://github.com/ShopwareLabs/SwagMediaS3) plugin as an example.
