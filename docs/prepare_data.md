# Preparing NeRF On-The-Road (NOTR) Dataset

This doc is heavily borrowed from [S3Gaussian](https://github.com/nnanhuang/S3Gaussian) and [emernerf](https://github.com/NVlabs/EmerNeRF/blob/main/docs/NOTR.md?plain=1), thanks for the author!

The key difference is the way we use SAM to separate the dynamic object from the static object.
DeepLab was used to segment Sky in our paper, but subsequent findings showed that it could be learned well without special processing of Sky, so it could not be used for convenience.
Of course, you can also use other ways to separate dynamic objects from static objects.

## 1. Register on Waymo Open Dataset

### Sign Up for a Waymo Open Dataset Account and Install gcloud SDK

To download the Waymo dataset, you need to register an account at [Waymo Open Dataset](https://waymo.com/open/). You also need to install gcloud SDK and authenticate your account. Please refer to [this page](https://cloud.google.com/sdk/docs/install) for more details.

### Set Up the Data Directory

Once you've registered and installed the gcloud SDK, create a directory to house the raw data:

```shell
# Create the data directory or create a symbolic link to the data directory
mkdir -p ./data/waymo/raw   
mkdir -p ./data/waymo/processed 
```

## 2. Download the raw data

Start by downloading the necessary data samples as follows:

### Downloading Specific Scenes from Waymo Open Dataset

For example, to obtain the 114th, 700th, and 754th scenes from the Waymo Open Dataset, execute:

```shell
python data/download_waymo.py \
    --target_dir ./data/waymo/raw \
    --scene_ids 114 700 754
```

### Downloading Different Splits of the NOTR Dataset

Our NOTR dataset comes in multiple splits. Specify the `split_file` argument to download your desired split:

- **Static32 Split:**

```shell
python datasets/download_waymo.py --split_file data/waymo_splits/static32.txt
```

- **Dynamic32 Split:**

```shell
python datasets/download_waymo.py --split_file data/waymo_splits/dynamic32.txt
```

Ensure you modify the paths and filenames to align with your project directory structure and needs.

### Manual download
If you cannot download by this way, you can download raw data from [here](https://console.cloud.google.com/storage/browser/waymo_open_dataset_scene_flow/train?pageState=(%22StorageObjectListTable%22:(%22f%22:%22%255B%255D%22))&prefix=&forceOnObjectsSortingFiltering=true) manually.

### Dataset split

For the Waymo Open Dataset, we first organize the scene names alphabetically and store them in `data/waymo_train_list.txt`. The scene index is then determined by the line number minus one. The splits for the NOTR dataset are as follows:

**Static-32**: 3, 19, 36, 69, 81, 126, 139, 140, 146, 148, 157, 181, 200, 204, 226, 232, 237, 241, 245, 246, 271, 297, 302, 312, 314, 362, 482, 495, 524, 527, 753, 780

**Dynamic-32**: 16, 21, 22, 25, 31, 34, 35, 49, 53, 80, 84, 86, 89, 94, 96, 102, 111, 222, 323, 323, 382, 382, 402, 402, 427, 427, 438, 438, 546, 581, 592, 620, 640, 700, 754, 795, 796

**NOTA-64**: both 

- Ego-static: 1, 23, 24, 37, 66, 108, 114, 115
- Dusk/Dawn: 124, 147, 206, 213, 574, 680, 696, 737
- Gloomy: 47, 205, 220, 284, 333, 537, 699, 749
- Exposure mismatch: 58, 93, 143, 505, 545, 585, 765, 766
- Nighttime: 7, 15, 30, 51, 130, 133, 159, 770
- Rainy: 44, 56, 244, 449, 688, 690, 736, 738
- High-speed: 2, 41, 46, 62, 71, 73, 82, 83

For further information, refer to the `data/waymo_splits` directory.

## 3. Preprocess the data

After downloading the raw dataset, you'll need to preprocess this compressed data to extract and organize various components.

### Prepare SAM Segmentation
We use the official [SAM](https://github.com/facebookresearch/segment-anything) code and model.
You need to download the model code and model parameters to this folder.
```
├── DriveRecon
│   ├── scene
│   ├── data
│   ├── segment_anything (copy from Official code)
│   ├── checkpoints (Download from Official code)
│   │   ├── sam_vit_h_4b8939.pth
```


### Running the Preprocessing Script

To preprocess specific scenes of the dataset, use the following command:

```shell
python preprocess.py \
    --data_root data/waymo/raw/ \
    --target_dir data/waymo/processed \
    --split training \
    --process_keys images lidar calib pose dynamic_masks segmentation \
    --workers 2 \
    --scene_ids 114 700
```

Alternatively, preprocess different splits of the NOTR dataset by providing the split file:

```shell
# preprocess the static split
python preprocess.py \
    --data_root data/waymo/raw/ \
    --target_dir data/waymo/processed \
    --split training \
    --process_keys images lidar calib pose dynamic_masks segmentation \
    --workers 16 \
    --split_file data/waymo_splits/static32.txt # change to dynamic32.txt or diverse56.txt to preprocess different splits
```

**Troubleshooting**: if you encounter `TypeError: 'numpy._DTypeMeta' object is not subscriptable`, use `pip install numpy==1.26.1` and ignore the warnings.

This command performs the following tasks:

- Extract camera poses, images, LiDAR data, calibration matrices, dynamic masks and point cloud flows from the raw dataset.
- Stores the extracted data in the `data/waymo/processed` directory.

### Data Components

After preprocessing, the dataset will be organized into the following components:

- **Images**: All frame images named as  `{timestep:03d}_{cam_id}.jpg`, where cam_id is 0, 1, 2, 3, 4 for FRONT, FRONT_LEFT, FRONT_RIGHT, SIDE_LEFT, SIDE_RIGHT cameras respectively.
- **Ego Poses**: - **Ego Poses**: Each file is named `{timestep:03d}.txt` and contains a 4x4 ego to world transformation matrix.
- **Camera Intrinsics**: Each file is named `{cam_id}.txt` and contains a 1d array of [f_u, f_v, c_u, c_v, k{1, 2}, p{1, 2}, k{3}].
- **Camera Extrinsics**: Each file is named `{cam_id}.txt` and contains a 4x4 camera to ego transformation matrix, i.e., `frame.context.camera_calibrations.extrinsic.transform` from the Waymo Open Dataset.
- **Lidar Data**: Each file is named {timestep:03d}.bin and contains an Nx14 array with:
  - **Origins** (3 dims): Origins of LiDAR rays in the ego-vehicle coordinate system.
  - **Points** (3 dims): (x, y, z) coordinates of LiDAR points in the ego-vehicle coordinate system.
  - **Flows** (4 dims): Flow vectors (dx, dy, dz, flow_class). Refer to lines 676-682 of `datasets/waymo_preprocess.py` for flow_class definition. Used for evaluating the flow prediction performance.
  - **Ground Labels** (1 dim): the ground labels of all LiDAR points. 1 means ground and 0 means non-ground. This is used for training neural scene flow priors, which is not used in EmerNeRF.
  - **Intensities** (1 dim): Intensity values of LiDAR points.
  - **Elongations** (1 dim): Elongations of LiDAR points.
  - **Laser_ids** (1 dim): Laser IDs of LiDAR points with 0: TOP, 1: FRONT, 2: SIDE_LEFT, 3: SIDE_RIGHT, 4: REAR.
- **Dynamic Mask**: Binary mask images named `{timestep:03d}_{cam_id}.png` to indicate the dynamic regions in the scene. 1 means dynamic and 0 means static. These are obtained by filtering ground truth 2D object bounding boxes by excluding the bounding boxes with velocity less than 1m/s, so as
to include meaningful moving objects without introducing too much background noise. This is used for evaluation, which will not be used during training.
- **Sky Mask**: Binary mask images named `{timestep:03d}_{cam_id}.png` to indicate the sky regions in the scene. 1 means sky and 0 means non-sky.
- **sam_masks**: Binary mask images named `{timestep:03d}_{cam_id}.png` to indicate the dynamic regions in the scene. 1 means dynamic and 0 means static.


### Directory Structure

The organized dataset will follow this directory structure:

```
data/waymo/processed
├── training
│   ├── SCENE_ID
│   │   ├── dynamic_masks      # Dynamic masks: `{timestep:03d}_{cam_id}.png`
│   │   ├── sam_masks          # Sam masks: `{timestep:03d}_{cam_id}.png`
│   │   ├── ego_pose           # Ego vehicle poses: `{timestep:03d}.txt`
│   │   ├── extrinsics         # Camera extrinsics: `{cam_id}.txt`
│   │   ├── images             # Images: `{timestep:03d}_{cam_id}.jpg`
│   │   ├── intrinsics         # Camera intrinsics: `{cam_id}.txt`
│   │   ├── lidar              # LiDAR data: `{timestep:03d}.bin`
│   │   ├── sky_masks          # Sky masks: we don't use here
│   │   ├── FEATURE_NAME       # Features: `{timestep:03d}_{cam_id}.npy` 
│   │   └── occ3d              # 3D semantic occupancy grids: `{timestep:03d}.npz` or `{timestep:03d}_04.npz`
```

Note that the `FEATURE_NAME` folder will be generated when call the training script.