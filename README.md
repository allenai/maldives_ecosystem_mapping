# Maldives Ecosystem Mapping

As part of the Global Ecosystem Atlas Maldives Accelerator project (https://earthobservations.org/solutions/incubators/global-ecosystems-atlas), Ai2
has work with partners to create the first version of an AI segmentation model (building off the foundation of Satlas) that can be use to classify teresterial and near land  ecosystem 
in the Maldives 

This repository contains the workflow and configuration files needed to reproduce the training of the ecosystem category segmentation model, and supports applying the model on new images.

You will also find in the Data Access Information section below how to access the classification created for various sensors, checkpoint file and the Annotations created for this project in geojson format.

# Training
## Requirements

* Python (3.12 is recommended)
* GPU 

Notes:

Testing was done on a Google Cloud VM with following specification 
* n1-standard-8  ( 8 vCPUs and 30 GB RAM)
* GPU: 1 x NVIDIA T4
* Image: mage Deep Learning VM with CUDA 11.8, M125 (Debian 11, Python 3.10. With CUDA 11.8 preinstalled.)

## Download Dataset

To train and apply the model, first obtain the rslearn dataset from the release bucket on Google Cloud Storage:

    export DATASET_PATH=/local/dataset/path/
    mkdir $DATASET_PATH
    cd $DATASET_PATH
    wget https://storage.googleapis.com/ai2-earthsystem-release/Release/GlobalEcosystemSegmenter/202410/rslearn_dataset.zip
    unzip rslearn_dataset.zip

Above, replace `/local/dataset/path/` with the local path of your choice.

The dataset contains:
- Sentinel-2 images spanning each island in the Maldives. There are six images per island, with all being used by the model, to provide robustness to cloud cover and other artifacts.
- Crops of the images that were annotated with segmentation labels.
- The corresponding labels.

The image crops and labels are used to train and validate the model, while the island-spanning images are used afterward to obtain predictions over the entire Maldives.

The dataset is intended for use with [rslearn](https://github.com/allenai/rslearn), AI2's remote sensing model development tool. Training and prediction examples are stored in the `windows/` directory; each example is a "spatiotemporal window" with a defined projection, spatial bounds, and time range, and includes "layers" containing images and labels. They are divided into four groups of windows:

- `images_sentinel2`: Sentinel-2 images of entire islands.
- `crops_sentinel2`: the portions of islands that have segmentation labels.
- `images_planetscope`: windows that reference commercial PlanetScope images of entire islands. The windows are included so that you can easily download the commercial images using rslearn in case you have a Planet Labs account (requires purchase).
- `crops_planetscope`: portions of islands with segmentation labels, coupled with PlanetScope image crops.

The labels (`windows/crops_sentinel2/*/layers/label/label/geotiff.tif`) are GeoTIFFs with integer values specifying an [ecosystem class](https://global-ecosystems.org/):

0. Unknown
1. FM_1_3_INTERMITTENTLY_CLOSED_AND_OPEN_LAKES_AND_LAGOONS
2. F_2_2_SMALL_PERMANENT_FRESHWATER_LAKES
3. MFT_1_2_INTERTIDAL_FORESTS_AND_SHRUBLANDS
4. MFT_1_3_COASTAL_SALTMARSHES_AND_REEDBEDS
5. MT_1_1_ROCKY_SHORELINES
6. MT_1_3_SANDY_SHORELINES
7. MT_2_1_COASTAL_SHRUBLANDS_AND_GRASSLANDS
8. MT_3_1_ARTIFICIAL_SHORELINES
9. M_1_1_SEAGRASS_MEADOWS
10. M_1_3_PHOTIC_CORAL_REEFS
11. M_1_6_SUBTIDAL_ROCKY_REEFS
12. M_1_7_SUBTIDAL_SAND_BEDS
13. TF_1_3_PERMANENT_MARSHES
14. T_7_1_ANNUAL_CROPLANDS
15. T_7_3_PLANTATIONS
16. T_7_4_URBAN_AND_INDUSTRIAL_ECOSYSTEMS

You can open Sentinel-2 images and corresponding labels in GIS software like qgis, e.g.:

    qgis $DATASET_PATH/windows/crops_sentinel2/HaaAlifu_Baarah_LD0987_30081_-75425_sentinel2/layers/sentinel2/B02_B03_B04_B08/geotiff.tif $DATASET_PATH/windows/crops_sentinel2/HaaAlifu_Baarah_LD0987_30081_-75425_sentinel2/layers/label/label/geotiff.tif

## Install RSLearn

    git clone https://github.com/allenai/rslearn.git
    python3 -m venv your_env_name
    source your_env_name/bin/activate
    cd rslearn
    pip install .[extra]

## Train the model:

    cd /path/to/maldives_ecosystem_mapping/
    rslearn model fit --config model_configs/config_sentinel2.yaml --data.init_args.path $DATASET_PATH

After training, find the best checkpoint and copy it for convenience:

    cp lightning_logs/version_0/checkpoints/epoch*ckpt best.ckpt

You may need to change the `version_0` to the correct subfolder in `lightning_logs` if you ran `model fit` multiple times.


You can visualize the model's outputs over the validation set:

    rslearn model test --config model_configs/config_sentinel2.yaml --data.init_args.path $DATASET_PATH --model.init_args.visualize_dir vis/ --ckpt_path best.ckpt

Note:  You will need to create the vis directory under maldives_ecosystem_mapping directory if it does not exist
(example /path/to/maldives_ecosystem_mapping/vis/) 

## Prediction

Now we can generate GeoTIFF outputs across all the islands:

    rslearn model predict --config model_configs/config_sentinel2.yaml --data.init_args.path $DATASET_PATH --ckpt_path best.ckpt --data.init_args.num_worker 8

Note:  You can set --data.init_args.num_workers to value that is approriate to your environment, for example for 8 cpu machine
you might want to set this to 8 as in the example above.

The outputs will appear in paths like `windows/images_sentinel2/*/layers/output/output/geotiff.tif` (overwriting the outputs included in the rslearn dataset download).

    qgis $DATASET_PATH/windows/images_sentinel2/HaaAlifu_Baarah_LD0987_sentinel2/layers/sentinel2/B02_B03_B04_B08/geotiff.tif $DATASET_PATH/windows/images_sentinel2/HaaAlifu_Baarah_LD0987_sentinel2/layers/output/output/geotiff.tif


Note: 

As an alternative to training the model, you can download the released model we have made available, see the Check Point 
File section in the Data Access Information section below for more information these models.  Example: 

    wget https://storage.googleapis.com/ai2-earthsystem-release/Release/GlobalEcosystemSegmenter/202410/weights/sentinel2.ckpt -O best.ckpt

## Obtaining Images

The dataset includes Sentinel-2 images, but if you want to obtain PlanetScope images, or get new Sentinel-2 images, first swap out the dataset config with one that specifies a data source (which is used by rslearn to automatically retrieve images).

    cp dataset_configs/config_planetscope.json $DATASET_PATH/config.json
    cp dataset_configs/config_sentinel2.json $DATASET_PATH/config.json

Then use rslearn to populate the dataset:

    rslearn dataset prepare --root $DATASET_PATH --workers 8 --group crops_planetscope
    rslearn dataset ingest --root $DATASET_PATH --workers 8 --group crops_planetscope
    rslearn dataset materialize --root $DATASET_PATH --workers 8 --group crops_planetscope

For PlanetScope, the PL_API_KEY environment variable must be set.

Before training the model, swap the configuration back:

    cp dataset_configs/config_train.json $DATASET_PATH/config.json

## Apply the Model on a New Location

The model is only trained on ecosystem labels in the Maldives, so it likely will not give accurate results in other regions, but it is nevertheless possible to apply the model elsewhere.

Start by adding a new window to rslearn. Here we put the window in a different group called "new_locations".

    rslearn dataset add_windows --root $DATASET_PATH --group new_locations --utm --resolution 10 --src_crs EPSG:4326 --box=-122.414,47.587,-122.286,47.704 --start 2024-05-01T00:00:00+00:00 --end 2024-10-01T00:00:00+00:00 --name name

Replace the box with coordinates corresponding to the desired location. Then obtain images here:

    cp dataset_configs/config_sentinel2.json $DATASET_PATH/config.json
    rslearn dataset prepare --root $DATASET_PATH --workers 8 --group new_locations
    rslearn dataset ingest --root $DATASET_PATH --workers 8 --group new_locations --no-use-initial-job --jobs-per-process 1
    rslearn dataset materialize --root $DATASET_PATH --workers 8 --group new_locations --no-use-initial-job

Then apply the model:

    cp dataset_configs/config_train.json $DATASET_PATH/config.json
    rslearn model predict --config model_configs/config_sentinel2.yaml --data.init_args.path $DATASET_PATH --ckpt_path best.ckpt --data.init_args.predict_config.groups '["new_locations"]'

Visualize in qgis:

    qgis $DATASET_PATH/windows/new_locations/name/layers/sentinel2/B02_B03_B04_B08/geotiff.tif $DATASET_PATH/windows/new_locations/name/layers/output/output/geotiff.tif

# Data Access Information
All data listed below is located in a Google Cloud Service Bucket named ai2-earthsystem-release, this bucket is open
to the public.  See https://cloud.google.com/storage/docs/downloading-objects#cli-download-object to learn more about how to use Google Cloud tools for the downloading of data

## Annotation Data:
Annotation data is stored in geojson format, it is located in  gs://ai2-earthsystem-release/Release/GlobalEcosystemSegmenter/202410/  folder in geojson format. 

Name of the file is [IslandName]_[DateOfCollect]_labels.geojson 

Example for the Island named baarh we annotated image that was collected on 2024-02-24-05-48 gs://ai2-earthsystem-release/Release/GlobalEcosystemSegmenter/202410/TrainingData/baarah_2024-02-24-05-48_labels.geojson

## Classification Prediction Data
Data is available at gs://ai2-earthsystem-release/Release/GlobalEcosytemAtlas/[version].  We have named version (example 202410) that include date of generation, and we also have a folder called latest which will always have the most recent generated files.

Our models produce a single-band geotiff, with that band's value corresponding to an ecosystem type (see below section for mapping) for each island we have data available for each of the following sensors as well as having a geotiff of the annotations.  Details below

* Sentinel-2: We have processed 1247 islands and placed data in the folder named sentinel2  (example: gs://ai2-earthsystem-release/Release/GlobalEcosytemAtlas/202410/Maldives/sentinel2/Baarah.tif).  Note, the number of islands processed is less then PlanetScope because very small islands in Sentinel-2 (less then 40 m x 40 m) would not produce any pixels of classifications. 
* PlanetScope: We have processed 1419 islands and placed data in the folder named planetscope (example: gs://ai2-earthsystem-release/Release/GlobalEcosytemAtlas/202410/Maldives/planetscope/Baarah.tif)
* Labels:  These are annotation in  geotiff format, this can be helpful for analyze predictions against annotations. 

The following are some additional notes on the data.
* The source of the Island Name was from data provided by Maldives, we have a copy of this data in geojson format at gs://ai2-earthsystem-release/Release/GlobalEcosytemAtlas/202410/Maldives/island_geo.geojson. 
* Some islands in this file did not have names, for these, we used the FCODE property that was in this file.  Below are sources of data we have output for

# Checkpoint files (Models and Weights)
As mentioned above we have made available checkpoint files that allow you to run predictions using rslearn without the need to train
the models yourself. 

The following Checkpoint files are available at gs://ai2-earthsystem-release/Release/GlobalEcosystemSegmenter/[version]/weights/[satelitte_source].ckpt
* Sentinel-2 (exampl path gs://ai2-earthsystem-release/Release/GlobalEcosystemSegmenter/202410/weights/sentinel2.ckpt
* PlanetScope (example path gs://ai2-earthsystem-release/Release/GlobalEcosystemSegmenter/202410/weights/planetscope.ckpt

# License
The following are the license for artifacts mentioned in this document
* Code: Apache License 2.0.
* Checkpoint files:  Apache License 2.0
* Annotation Data:  Creative Commons (https://creativecommons.org/publicdomain/zero/1.0/)
* Classification Prediction Data: Open Data Commons (https://opendatacommons.org/licenses/by/1-0/)

# Contact
For questions and suggestions, please open an issue on GitHub.
