---
layout: post
title: "OpenStack Deployment - Iso to Image"
date: 2021-01-24 09:00:00 -0500
categories: [Training-SOC, OpenStack]
tags: [openstack, cloud, deployment, soc, guide, instance, image, volume]
---

## Installing an ISO on a Volume

1. Download an ISO version of an OS e.g Ubuntu_Focal_Desktop.iso
2. Upload this image to glance in openstack using the openstack dashboard, by navigating to the images tab under compute, then selecting Create Image.
3. Create a new volume, ensuring the size is large enough to install the OS. (E.g: 25 GB for ubuntu desktop).
4. Instantiate a new instance with the ISO intended to be installed.
5. Attatch the newly created volume to the instance.
    - This can be done in the instances section, using the right drop down menu under actions and selecting 'attatch volume'.
6. Switch to the console view of the instance and proceed to install the OS with the destination being the newly attatched volume.
7. After installing the OS to the volume, delete the instance.

## Converting the Volume to an Image

Navigate to the Volumes section and use the right drop down menu to select upload as image.

- This newly created image can be used to create new instances with the OS installed.
- Ensure to re-partition the OS to utilize the larger instance size. (For Ubuntu Desktop the utility program works well)
