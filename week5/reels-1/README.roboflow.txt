
reels - v1 qual_reels_1
==============================

This dataset was exported via roboflow.com on April 28, 2026 at 8:36 AM GMT

Roboflow is an end-to-end computer vision platform that helps you
* collaborate with your team on computer vision projects
* collect & organize images
* understand and search unstructured image data
* annotate, and create datasets
* export, train, and deploy computer vision models
* use active learning to improve your dataset over time

For state of the art Computer Vision training notebooks you can use with this dataset,
visit https://github.com/roboflow/notebooks

To find over 100k other datasets and pre-trained models, visit https://universe.roboflow.com

The dataset includes 1170 images.
Real_qual_final are annotated in YOLOv8 format.

The following pre-processing was applied to each image:
* Auto-orientation of pixel data (with EXIF-orientation stripping)
* Resize to 512x512 (Stretch)

The following augmentation was applied to create 3 versions of each source image:
* Randomly crop between 0 and 13 percent of the image
* Random rotation of between -15 and +15 degrees
* Random exposure adjustment of between -8 and +8 percent
* Random Gaussian blur of between 0 and 2.5 pixels
* Salt and pepper noise was applied to 1.05 percent of pixels


