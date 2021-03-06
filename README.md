# chip-from-vrt

A GBDX task for generating image chips from a mosaic or an arbitrary number of image strips on S3 using [the GDAL virtual format](http://www.gdal.org/gdal_vrttut.html) (vrt). By creating a vrt that points to the imagery location on S3, the task can extract pixels from an unlimited number of images without having to mount these onto the worker node, thus reducing overhead and bypassing disc space limitations.

The chip geometries are provided in a geojson file. Chips are saved to a user-defined S3 location along with a reference geojson (ref.geojson) which contains the geometry of each chip, along with the feature id and class name (if provided in the input geojson); chips that can not be extracted are not included in the chips or ref.geojson. If there is spatial overlap between images, the chip is extracted from the last image to be listed as input.


## Run

There are two ways to run chip-from-vrt; chip from a group of tiles that comprise a mosaic, or from a group of individual image strips.

### Chip from FLAME mosaic

<img src='images/chip-from-mosaic.png' width=750>

1. In a Python terminal create a GBDX interface:

    ```python
    from gbdxtools import Interface
    from os.path import join
    import uuid

    gbdx = Interface()

    # Specify location of input files and aws creds
    input_location = 's3://gbd-customer-data/32cbab7a-4307-40c8-bb31-e2de32f940c2/platform-stories/chip-from-vrt/'
    info = gbdx.s3.info
    access_key = info['S3_access_key']
    secret_key = info['S3_secret_key']
    session_token = info['S3_session_token']
    ```

2. Create a task instance and set the required [inputs](#inputs):

    ```python
    chip_mosaic = gbdx.Task('chip-from-vrt')
    chip_mosaic.inputs.geojson = join(input_location, 'mosaic-geojson/')
    chip_mosaic.inputs.images = join(input_location, 'mosaic/')
    chip_mosaic.inputs.mosaic = 'True'
    chip_mosaic.inputs.tar = 'True'     # Output chip dir as a tar file
    chip_mosaic.inputs.aws_access_key = access_key
    chip_mosaic.inputs.aws_secret_key = secret_key
    chip_mosaic.inputs.aws_session_token = session_token
    ```

3. Set the domain to raid if generating more than 10000 chips to speed up execution:

    ```python
    chip_mosaic.domain = 'raid'
    ```

4. Create a workflow from the task and specify where to save the output chips:

    ```python
    # Specify output location with random string
    random_str = str(uuid.uuid4())
    output_location = join('platform-stories/trial-runs', random_str)

    # Create the workflow and save the output to output_location
    chip_mosaic_wf = gbdx.Workflow([chip_mosaic])
    chip_mosaic_wf.savedata(chip_mosaic.outputs.chips, join(output_location, 'mosaic-chips'))
    ```

5. Execute the workflow:

    ```python
    chip_mosaic_wf.execute()
    ```

### Chip from strips

<img src='images/chip-from-strips.png' width=750>

1. In a Python terminal create a GBDX interface:

    ```python
    from gbdxtools import Interface
    from os.path import join
    import uuid

    gbdx = Interface()
    input_location = 's3://gbd-customer-data/32cbab7a-4307-40c8-bb31-e2de32f940c2/platform-stories/chip-from-vrt/'

    ```

2. For images on the gbd-customer-data bucket you will need to set S3 credentials (including a session token). It is recommended to request credentials that last for 36 hours to ensure they last for the duration of the task. Do so by sending a post to ```https://geobigdata.io/s3creds/v1/prefix?duration=129600``` and using these values as the aws credentials below.

    ```python
    info = gbdx.s3.info          # Skip this if using 36 hr credentials
    access_key = info['S3_access_key']
    secret_key = info['S3_secret_key']
    session_token = info['S3_session_token']
    ```

3. Create a task instance and set the required [inputs](#inputs):

    ```python
    # Define images input location
    image_input = join(input_location, 'strip-imagery')

    # Create task and set inputs
    chip_strips = gbdx.Task('chip-from-vrt')
    chip_strips.inputs.geojson = join(input_location, 'strip-geojson')
    chip_strips.inputs.images = ', '.join([join(image_input, '1040010014BCA700.tif'), join(image_input, '1040010014800C00.tif')])
    chip_strips.inputs.tar = 'True'     # Output chip dir as a tar file
    chip_strips.inputs.aws_access_key = access_key
    chip_strips.inputs.aws_secret_key = secret_key
    chip_strips.inputs.aws_session_token = session_token
    ```

4. Set the domain to raid if generating more than 10000 chips to speed up execution:

    ```python
    chip_strips.domain = 'raid'
    ```

5. Create a workflow from the task and specify where to save the output chips:

    ```python
    # Specify output location with random string
    random_str = str(uuid.uuid4())
    output_location = join('platform-stories/trial-runs', random_str)

    chip_strips_wf = gbdx.Workflow([chip_strips])
    chip_strips_wf.savedata(chip_strips.outputs.chips, join(output_location, 'chips'))
    ```

6. Execute the workflow

    ```python
    chip_strips_wf.execute()
    ```


## Input ports

GBDX input ports can only be of 'Directory' or 'String' type. Booleans, integers and floats are passed to the task as strings, e.g., 'True', '10', '0.001'.

| Name  | Type | Description | Required |
|-----------------|-----------------|-----------------|-----------------|
| geojson | Directory | Contains one geojson file containing AOIs to extract from the mosaic. If chips are to be used for training each feature must have a class_name property. Features will be saved as feature_id.tif in the output directory. If no feature_id property is present, ids will be generated and saved to the reference gejoson in the output directory. | True |
|  images | String | S3 image locations. Note that if the bucket is private you must enter valid AWS keys and a token to access the images. If using a FLAME mosaic this should be the location of the project directory as follows: bucket_name/path/to/project_name/. This directory must contain the a subdirectory with the mosaic tiles and a wms/ subdirectory with a shapefile 'vsitindex_z12.shp'. Otherwise, this should be the exact S3 location of any image strips being used. Different strips should be separated by a comma as follows: 'bucket_name/path/to/image1.tif, bucket_name/path/to.image2.tif', ...| True |
|  tar | String | If True, outputs chips directory as a tar file. Defaults to False. | False |
|  mosaic | String | True if the images comprise a FLAME mosaic; else False. Defaults to False.| False |
|  shapefile_location | String | Location of the shapefile pointing to image raster location within the mosaic directory. Only relevant if using a flame mosaic. This file must be in the project directory. Defaults to wms/vsitindex_z12.shp. | False |  
|  aws_access_key | String | AWS access key. The account associated with this key should have read access to the bucket containing the mosaic. | False |
|  aws_secret_key | String | AWS secret access key. The account associated with this key should have read access to the bucket containing the mosaic. | False |
|  aws_session_token | String | AWS session token. This is necessary if the images input use IAM credentials such as is in the gbd_customer_data bucket. | False |  
|  mask | String | If True, blackfill pixels outside the polygon. Otherwise entire bounding box will be included in the output chip. Defaults to False.| False |  
|  bit_depth | String | Bit depth to set the output chips to if larger than 8. Defaults to 8.| False |
|  bands | String | List of bands to include in the chips, each separated by a comma. Note that indexing starts at 1. For example, to use the first three bands use input "1,2,3". Defaults to all bands. | False |  

## Output ports

| Name  | Type | Description |
|-------|---------|---------------------------------------------------|
| chips | Directory | Contains chipped AOIs from input geojson in tif format. Each chip is named after its feature_id value. A reference geojson file with feature ids for each geometry is also saved in this directory. |

## Advanced

### Internal Tiling for Faster Chipping

When generating a large number of chips(>10000) from image strips, it is recommended to use internal tiling to speed up the task. To accomplish this you may use the [tile-strips](https://github.com/PlatformStories/tile-strips) gbdx task on each image as follows:

```python
from gbdxtools import Interface()
gbdx = Interface()

tiler = gbdx.Task('tile-strips', images = 's3://bucket/prefix/path/to/images/')
tiler_wf = gbdx.Task([tiler])

# This will overwrite original images with tiled versions
tiler_wf.savedata(tiler.outputs.tiled_images, 'path/to/images/')

tiler_wf.execute()
```

### Virtual Datasets and Image Overlap

The task uses [gdalbuildvrt](http://www.gdal.org/gdalbuildvrt.html) to create a virtual dataset that combines all input images and points to the various locations of each strip on S3. The task uses this vrt as a reference for where to find the pixel data of each AOI. If there is spatial overlap between images, data is extracted from the latest image listed. The order that the images are input to the task is maintained when calling gdalbuildvrt.


## Development

### Build the Docker image

You need to install [Docker](https://docs.docker.com/engine/installation/).

Clone the repository:

```bash
git clone https://github.com/platformstories/chip-from-vrt
```

Then:

```bash
cd chip-from-vrt
docker build -t chip-from-vrt .
```

### Try out locally

Create a container in interactive mode and mount the sample input under `/mnt/work/input/`:

```bash
docker run -v full/path/to/sample-input:/mnt/work/input -it chip-from-vrt
```

Then, within the container:

```bash
python /chip-from-vrt.py
```

Confirm that the chips are in the `/mnt/work/output/chips/` directory.

### Docker Hub

Login to Docker Hub:

```bash
docker login
```

Tag your image using your username and push it to DockerHub:

```bash
docker tag chip-from-vrt yourusername/chip-from-vrt
docker push yourusername/chip-from-vrt
```

The image name should be the same as the image name under containerDescriptors in chip-from-vrt.json.

Alternatively, you can link this repository to a [Docker automated build](https://docs.docker.com/docker-hub/builds/). Every time you push a change to the repository, the Docker image gets automatically updated.

### Register on GBDX

In a Python terminal:

```python
import gbdxtools
gbdx = gbdxtools.Interface()
gbdx.task_registry.register(json_filename='chip-from-vrt.json')
```

Note: If you change the task image, you need to reregister the task with a higher version number in order for the new image to take effect. Keep this in mind especially if you use Docker automated build.
