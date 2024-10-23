# LiipImagineBundle Integration

## Auto Optimize files on Upload Using an Event Listener
This assumes you have a default filter setup already in `liip_imagine.yaml`. You can view an example configuration below.

```php
<?php

namespace App\EventListener;

use Adeliom\EasyMediaBundle\Entity\Media;
use Adeliom\EasyMediaBundle\Event\EasyMediaBeforeSetMetas;
use Adeliom\EasyMediaBundle\Event\EasyMediaFileUploaded;
use Adeliom\EasyMediaBundle\Service\EasyMediaManager;
use Doctrine\ORM\EntityManagerInterface;
use Liip\ImagineBundle\Imagine\Data\DataManager;
use Liip\ImagineBundle\Imagine\Filter\FilterManager;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\HttpFoundation\File\File;


class FileOptimizerListener
{
    protected Media|null $entity;
    protected $isImage = false;

    public function __construct(
        private FilterManager $filterManager,
        private DataManager $dataManager,
        private EasyMediaManager $easyMediaManager,
        private EntityManagerInterface $entityManager,
    ) {}

    #[AsEventListener(event: EasyMediaBeforeSetMetas::NAME)]
    public function mediaBeforeSetMetas(EasyMediaBeforeSetMetas $easyMediaBeforeSetMetas)
    {
        $this->entity = $easyMediaBeforeSetMetas->getEntity();
        $source = $easyMediaBeforeSetMetas->getSource();

        if ($source && $source instanceof File) {
            // Check to see if file is an image
            $this->isImage = @exif_imagetype($source->getPathname());
        }
    }

    #[AsEventListener(event: EasyMediaFileUploaded::NAME)]
    public function mediaFileUploaded(EasyMediaFileUploaded $event)
    {
        $path = $event->getFilePath();
        $filesystem = $this->easyMediaManager->getFilesystem();
        $entity = $this->entity;

        // File must be an image
        if ($this->isImage) {
            // Find the file from the path
            $binaryFile = $this->dataManager->find('default', $path);

            // Apply the default filter
            $media = $this->filterManager->applyFilter($binaryFile, 'default');

            // Write the file back to the filesystem
            $filesystem->write($entity->getPath(), $media->getContent());

            // Update the entities size for the file
            $entity->setSize($filesystem->fileSize($entity->getPath()));
            
            // Flush the database and update the record
            $this->entityManager->flush();
        }

    }
}

```

```yaml
# Documentation on how to configure the bundle can be found at: https://symfony.com/doc/current/bundles/LiipImagineBundle/basic-usage.html
# Example implementation
liip_imagine:
    # valid drivers options include "gd" or "gmagick" or "imagick"
    driver: "gd"
    data_loader: easy_media_data_loader
    # configure webp
    default_filter_set_settings:
        format: webp
    webp:
        generate: true
        quality: 75
    # configure resolvers
    resolvers :

        # setup the default resolver
        default :

            # use the default web path
            web_path :
                # use %kernel.project_dir%/web for Symfony prior to 4.0.0
                web_root: "%kernel.project_dir%/public"
                cache_prefix: "media/cache"

    # your filter sets are defined here
    filter_sets :
        # use the default cache configuration
        cache : ~
        default:
            quality: 75
            filters:
                auto_rotate: ~
        # the name of the "filter set"
        homepage_thumb:
            # adjust the image quality to 75%
            quality: 75

            # list of transformations to apply (the "filters")
            filters:
                auto_rotate: ~
                # create a thumbnail: set size to 120x90 and use the "outbound" mode
                # to crop the image when the size ratio of the input differs
                thumbnail: { size: [610, 400], mode: outbound }
        portfolio_thumb:
            # adjust the image quality to 75%
            quality: 75

            # list of transformations to apply (the "filters")
            filters:
                auto_rotate: ~
                # create a thumbnail: set size to 120x90 and use the "outbound" mode
                # to crop the image when the size ratio of the input differs
                thumbnail: { size: [625, 400], mode: outbound }
```
