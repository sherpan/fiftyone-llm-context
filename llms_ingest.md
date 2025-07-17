<!-- TOC depthFrom:2 depthTo:3 -->
- [SDK Connections and Configurations](#sdk-connections-and-configurations)
  - [API Connection](#api-connection)
  - [Direct MongoDB Connection](#direct-mongodb-connection)
  - [Cloud Media Credentials (AWS)](#cloud-media-credentials-aws)
- [Loading Datasets](#loading-datasets)
  - [Image Folders](#image-folders)
  - [Custom Formats](#custom-formats)
  - [Setting Values](#setting-values)
  - [Datasets with cloud-backed media](#datasets-with-cloud-backed-media)
  - [Uploading Data to the Cloud](#uploading-data-to-the-cloud)
- [Dataset Setup](#dataset-setup)
  - [Computing Metadata](#computing-metadata)
  - [Multiple Media Fields](#multiple-media-fields)
- [Grouped Datasets](#grouped-datasets)
  - [Grouped Datasets](#grouped-datasets-1)
  - [Creating Grouped Datasets](#creating-grouped-datasets)
  - [Active Slice and Changing Slices](#active-slice-and-changing-slices)
  - [Flattened Views](#flattened-views)
- [3D Datasets](#3d-datasets)
  - [3D samples](#3d-samples)
  - [Example: Grouped Dataset with 3D](#example-grouped-dataset-with-3d)
- [Miscellaneous](#miscellaneous)
  - [Saved Views](#saved-views)
  - [Updating Samples](#updating-samples)
- [What’s Next?](#whats-next)

<!-- /TOC -->

---

# SDK Connections and Configurations

## [API Connection](https://docs.voxel51.com/enterprise/api_connection.html#api-connection) 


To connect your SDK to your FiftyOne Enterprise deployment, click the link above to generate an API Key and configure this key locally.


## [Direct MongoDB Connection](https://docs.voxel51.com/user_guide/config.html#configuring-a-mongodb-connection)

For high-volume or production workloads, you may configure a direct MongoDB connection instead of using an API key. This can be useful for larger workloads and production pipelines.

## [Cloud Media Credentials (AWS)](https://docs.voxel51.com/enterprise/installation.html#amazon-s3) 

To work with cloud media locally with your SDK, see the above link to configure your cloud credentials.

# Loading Datasets 
## [Image Folders](https://docs.voxel51.com/user_guide/dataset_creation/index.html#loading-images)

If you’re just getting started with a project and all you have is a bunch of image files, you can easily load them into a FiftyOne dataset and start visualizing them in the App:


```
import fiftyone as fo

# Create a dataset from a list of images
dataset = fo.Dataset.from_images(
    ["/path/to/image1.jpg", "/path/to/image2.jpg", ...]
)

# Create a dataset from a directory of images
dataset = fo.Dataset.from_images_dir("/path/to/images")

# Create a dataset from a glob pattern of images
dataset = fo.Dataset.from_images_patt("/path/to/images/*.jpg")

```

You can name your Dataset and mark it as persistent
```
dataset.name = 'your-dataset-name'
dataset.persistent = True
``` 

You can also use the [IO Plugin](https://github.com/voxel51/fiftyone-plugins/tree/main/plugins/io) to easily import datasets directly in the App.

## [Custom Formats](https://docs.voxel51.com/user_guide/dataset_creation/index.html#custom-formats)

To ingest datasets with labels or metadata, the simplest approach is to iterate over your data in a simple Python loop, create a Sample for each data + label(s) pair, and then add those samples to a Dataset.

FiftyOne provides label types for common tasks such as classification, detection, segmentation, and many more. The example below illustrates creating detection objects.

```
import glob
import fiftyone as fo

images_patt = "/path/to/images/*"

# Ex: your custom label format
annotations = {
    "/path/to/images/000001.jpg": [
        {"bbox": ..., "label": ...},
        ...
    ],
    ...
}

# Create samples for your data
samples = []
for filepath in glob.glob(images_patt):
    sample = fo.Sample(filepath=filepath)

    # Convert detections to FiftyOne format
    detections = []
    for obj in annotations[filepath]:
        label = obj["label"]

        # Bounding box coordinates should be relative values
        # in [0, 1] in the following format:
        # [top-left-x, top-left-y, width, height]
        bounding_box = obj["bbox"]

        detections.append(
            fo.Detection(label=label, bounding_box=bounding_box)
        )

    # Store detections in a field name of your choice
    sample["ground_truth"] = fo.Detections(detections=detections)

    samples.append(sample)

# Create dataset
dataset = fo.Dataset("my-detection-dataset")
dataset.add_samples(samples)
```

## [Setting Values](https://docs.voxel51.com/user_guide/using_datasets.html#setting-values)

Another strategy is to load Samples with no metadata, and then add metadata afterwords. This can be done efficiently in bulk 
using set_values() to set a field (or embedded field) on each sample in the dataset in a single batch operation.


```
# Populate the field on each sample in the dataset
values = [random.random() for _ in range(len(dataset))]
dataset.set_values("random", values)

print(dataset.count("random"))  # 50
print(dataset.bounds("random")) # (0.0041, 0.9973)
```

When possible, using set_values() is often more efficient than performing the equivalent operation via an explicit iteration over the Dataset because it avoids the need to read Sample instances into memory and sequentially save them.

Note, along with setting values via set_values(), you can also get values in bulk using values().


## [Datasets with cloud-backed media](https://docs.voxel51.com/user_guide/dataset_creation/index.html#custom-formats)

Note, almost all filepath-related methods in the FIftyOne Enterprise SDK are cloud aware. Here is an example of 
uploading all files of a certain type within a cloud bucket to your Fiftyone Dataset.

```
import fiftyone as fo
import fiftyone.core.storage as fos
s3_files = fos.list_files(dirpath="s3://YOUR_BUCKET/YOUR_PREFIX", abs_path=True)
dataset = fo.Dataset('YOUR_VIDEO_DATASET')
for s3_uri in s3_files:
    if s3_uri.lower().endswith('.mp4'):
        sample = fo.Sample(filepath=s3_uri)
        samples.append(sample)

dataset.add_samples(samples)
dataset.persistent = True # will render the dataset in the UI
```

## [Uploading Data to the Cloud](https://docs.voxel51.com/enterprise/cloud_media.html?highlight=upload_media#:~:text=or%20local%20media.-,The%20upload_media(),-method%20provides%20a)


If you have your data stored locally, you can use FiftyOne to create a dataset and then upload the local data to a specified bucket. 

```
import fiftyone.core.storage as fos

# Create a dataset from media stored locally
dataset = fo.Dataset.from_dir("/tmp/local", ...)

# Upload the dataset's media to the cloud
fos.upload_media(
    dataset,
    "s3://voxel51-test/your-media",
    update_filepaths=True,
    progress=True,
)
```

# Dataset Setup

## [Computing Metadata](https://docs.voxel51.com/api/fiftyone.core.collections.html#fiftyone.core.collections.SampleCollection.compute_metadata)

A best practice when creating dataset is to compute metadata so that image heights/widths are available. This can be useful for end-users as well as for optimizing App grid layout performance.


## [Multiple Media Fields](https://docs.voxel51.com/user_guide/app.html#multiple-media-fields)

There are use cases where you may want to associate multiple media versions with each sample in your dataset, such as:

Thumbnail images

Anonymized (e.g., blurred) versions of the images

You can work with multiple media sources in FiftyOne by simply adding extra field(s) to your dataset containing the paths to each media source and then configuring your dataset to expose these multiple media fields in the App.

For example, let’s create thumbnail images for use in the App’s grid view and store their paths in a thumbnail_path field:

```
import fiftyone as fo
import fiftyone.utils.image as foui
import fiftyone.zoo as foz

dataset = foz.load_zoo_dataset("quickstart")

# Generate some thumbnail images
foui.transform_images(
    dataset,
    size=(-1, 32),
    output_field="thumbnail_path",
    output_dir="/tmp/thumbnails",
)

print(dataset)
```

We can expose the thumbnail images to the App by modifying the dataset’s App config:


```
# Modify the dataset's App config
dataset.app_config.media_fields = ["filepath", "thumbnail_path"]
dataset.app_config.grid_media_field = "thumbnail_path"
dataset.save()  # must save after edits

session = fo.launch_app(dataset)
```

Adding thumbnail_path to the media_fields property adds it to the Media Field selector under the App’s settings menu, and setting the grid_media_field property to thumbnail_path instructs the App to use the thumbnail images by default in the grid view:

# Grouped Datasets

## [Grouped Datasets](https://docs.voxel51.com/user_guide/groups.html#grouped-datasets)

FiftyOne supports the creation of grouped datasets, which contain multiple slices of samples of possibly different modalities (e.g., image, video, or 3D scenes) that are organized into groups.

Grouped datasets can be used to represent multiview scenes, where data for multiple perspectives of the same scene can be stored, visualized, and queried together. In sequential acquisitions, each group commonly represents a single moment in time.

Here is a basic example of creating and working with grouped datasets via Python.Let’s start by creating some test data. We’ll use the quickstart dataset to construct some mocked triples of left/center/right images:

```
import fiftyone as fo
import fiftyone.utils.random as four
import fiftyone.zoo as foz

groups = ["left", "center", "right"]

d = foz.load_zoo_dataset("quickstart")
four.random_split(d, {g: 1 / len(groups) for g in groups})
filepaths = [d.match_tags(g).values("filepath") for g in groups]
filepaths = [dict(zip(groups, fps)) for fps in zip(*filepaths)]

print(filepaths[:2])
```
```
[
    {
        'left': '~/fiftyone/quickstart/data/000880.jpg',
        'center': '~/fiftyone/quickstart/data/002799.jpg',
        'right': '~/fiftyone/quickstart/data/001599.jpg',
    },
    {
        'left': '~/fiftyone/quickstart/data/003344.jpg',
        'center': '~/fiftyone/quickstart/data/001057.jpg',
        'right': '~/fiftyone/quickstart/data/001430.jpg',
    },
]
```

## [Creating Grouped Datasets](https://docs.voxel51.com/user_guide/groups.html#creating-grouped-datasets)

To create a grouped dataset, simply use add_group_field() to declare a Group field on your dataset before you add samples to it:


```
dataset = fo.Dataset("groups-overview")
dataset.add_group_field("group", default="center")
```

The optional default parameter specifies the slice of samples that will be returned via the API or visualized in the App’s grid view by default. If you don’t specify a default, one will be inferred from the first sample you add to the dataset.


To populate a grouped dataset with samples, create a single Group instance for each group of samples and use Group.element() to generate values for the group field of each Sample object in the group based on their slice’s name. The Sample objects can then simply be added to the dataset as usual:


```
samples = []
for fps in filepaths:
    group = fo.Group()
    for name, filepath in fps.items():
        sample = fo.Sample(filepath=filepath, group=group.element(name))
        samples.append(sample)

dataset.add_samples(samples)

```

## [Active Slice and Changing Slices](https://docs.voxel51.com/user_guide/groups.html#:~:text=You%20can%20change%20the%20active%20group%20slice%20in%20your%20current%20session%20by%20setting%20the%20group_slice%20property%3A)

SDK operations on a grouped dataset often operate on the current slice. Here is how you can retrieve the current slice 
```
print(dataset.group_slices)
```
To change the current slice, simply set the property to your desired new slice
```
dataset.group_slice = your_new_slice
```

## [Flattened Views](https://docs.voxel51.com/api/fiftyone.core.dataset.html?highlight=select_group_slices#fiftyone.core.dataset.Dataset.select_group_slices)
Sometimes you might want to operate on all group slices. For this you can form a flattened view:
```
view = dataset.select_group_slices(media_type='image')
view.set_values('new_field',range(len(view)))
```

# 3D Datasets
## [3D samples](https://docs.voxel51.com/user_guide/using_datasets.html#d-datasets)

Any Sample whose filepath is a file with extension .fo3d is recognized as a 3D sample, and datasets composed of 3D samples have media type 3d.

An FO3D file encapsulates a 3D scene constructed using the Scene class, which provides methods to add, remove, and manipulate 3D objects in the scene. A scene is internally represented as a n-ary tree of 3D objects, where each object is a node in the tree. A 3D object is either a 3D mesh, point cloud, or a 3D shape geometry.

A scene may be explicitly initialized with additional attributes, such as camera, lights, and background. By default, a scene is created with neutral lighting, and a perspective camera whose up is set to Y axis in a right-handed coordinate system.

After a scene is constructed, it should be written to the disk using the scene.write() method, which serializes the scene into an FO3D file.

FiftyOne supports the PCD point cloud format. A code snippet to create a PCD object that can be added to a FiftyOne 3D scene is shown below:

```
import fiftyone as fo

pcd = fo.PointCloud("my-pcd", "point-cloud.pcd")
pcd.default_material.shading_mode = "custom"
pcd.default_material.custom_color = "red"
pcd.default_material.point_size = 2

scene = fo.Scene()
scene.add(pcd)

scene.write("/path/to/scene.fo3d")
```

Here’s how a typical PCD file is structured:


```
import numpy as np
import open3d as o3d

points = np.array([(x1, y1, z1), (x2, y2, z2), ...])
colors = np.array([(r1, g1, b1), (r2, g2, b2), ...])

pcd = o3d.geometry.PointCloud()
pcd.points = o3d.utility.Vector3dVector(points)
pcd.colors = o3d.utility.Vector3dVector(colors)

o3d.io.write_point_cloud("/path/to/point-cloud.pcd", pcd)
```
## [Example: Grouped Dataset with 3D](https://docs.voxel51.com/user_guide/groups.html#toy-dataset)

The snippet below generates a toy dataset containing 3D cuboids filled with points that demonstrates how 3D detections are represented:

```
import fiftyone as fo
import numpy as np
import open3d as o3d

detections = []
point_cloud = []

for _ in range(10):
    dimensions = np.random.uniform([1, 1, 1], [3, 3, 3])
    location = np.random.uniform([-10, -10, 0], [10, 10, 10])
    rotation = np.random.uniform(-np.pi, np.pi, size=3)

    detection = fo.Detection(
        dimensions=list(dimensions),
        location=list(location),
        rotation=list(rotation),
    )
    detections.append(detection)

    R = o3d.geometry.get_rotation_matrix_from_xyz(rotation)
    points = np.random.uniform(-dimensions / 2, dimensions / 2, size=(1000, 3))
    points = points @ R.T + location[np.newaxis, :]
    point_cloud.extend(points)

pc = o3d.geometry.PointCloud()
pc.points = o3d.utility.Vector3dVector(np.array(point_cloud))
o3d.io.write_point_cloud("/tmp/toy.pcd", pc)

scene = fo.Scene()
scene.add(fo.PointCloud("point cloud", "/tmp/toy.pcd"))
scene.write("/tmp/toy.fo3d")

# Please note: a new fo.Group() object must be created for every separate group of samples!
group = fo.Group()
samples = [
    fo.Sample(
        filepath="/tmp/toy.png",  # non-existent
        group=group.element("image"),
    ),
    fo.Sample(
        filepath="/tmp/toy.fo3d",
        group=group.element("pcd"),
        detections=fo.Detections(detections=detections),
    )
]

dataset = fo.Dataset()
dataset.add_samples(samples)

dataset.app_config.plugins["3d"] = {
    "defaultCameraPosition": {"x": 0, "y": 0, "z": 20}
}
dataset.save()

session = fo.launch_app(dataset)
```


# Miscellaneous

## [Saved Views](https://docs.voxel51.com/user_guide/using_views.html#saving-views)

```
dataset = foz.load_zoo_dataset("quickstart")
view = dataset.filter_labels("ground_truth", F("label") == "cat")

dataset.save_view("cats", view)
```
To see the list of saved views on your dataset 
```
dataset.list_saved_views()
```
Load a specific saved view
```
also_view = dataset.load_saved_view("cats")
```


## [Updating Samples](https://docs.voxel51.com/user_guide/using_datasets.html#updating-samples)

```
import fiftyone as fo
import fiftyone.zoo as foz

dataset = foz.load_zoo_dataset("cifar10", split="train")
view = dataset.select_fields("ground_truth")

def update_fcn(sample):
    sample.ground_truth.label = sample.ground_truth.label.upper()

view.update_samples(update_fcn)
print(dataset.count_values("ground_truth.label"))
# {'DEER': 5000, 'HORSE': 5000, 'AIRPLANE': 5000, ..., 'DOG': 5000}
```


# What’s Next?

Where should you go from here? You could…

- Try one of the [tutorials](https://docs.voxel51.com/tutorials/index.html) that demonstrate the unique
capabilities of FiftyOne

- Check out the [cheat sheets](https://docs.voxel51.com/cheat_sheets/index.html) for topics you may
want to master quickly

- Consult the [user guide](https://docs.voxel51.com/user_guide/index.html) for detailed instructions on
how to accomplish various tasks with FiftyOne