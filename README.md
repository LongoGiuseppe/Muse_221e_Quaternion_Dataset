# Muse 221e aligned quaternion dataset

This repository contains aligned quaternion data from the Muse sensor and the robot reference system. The data are intended for AI-based compensation of quaternion errors.

The exported data are already processed through the same pipeline used for the Muse 221e analysis:

```text
raw Muse + raw robot
→ 221e compensation
→ quaternion SLERP resampling
→ onset-based alignment
→ robot quaternion transformed into the Muse frame
→ dataset export
```

## Dataset status

Current export summary:

```text
successful trajectory-axis exports: 83
failed trajectory-axis exports: 1
total manifest entries: 84
axes: x, y, z
nominal sample rate: 125 Hz
summary/all_samples.csv rows: 235,204
NaN/Inf values in exported quaternion fields: removed before export
```

Some trajectory-axis entries are marked as invalid/questionable in `dataset_manifest.csv`. For standard AI training and evaluation, use only rows with:

```text
valid = 1
```

The invalid rows are kept in the manifest for traceability, but they should normally be excluded from model training.

## Folder structure

```text
DATASET/
├── README.md
├── dataset_manifest.csv
├── dataset_metadata.json
├── export_summary.json
├── export_log.txt
├── summary/
│   ├── all_samples.csv
│   └── all_trials_metrics.csv
├── trajectories/
│   └── <trajectory_name>/
│       └── <axis>/
│           ├── aligned_quaternions.csv
│           ├── aligned_quaternions.mat
│           └── metadata.json
└── Inspection/
    ├── README_INSPECTION.md
    ├── summary/
    │   └── global inspection figures
    └── trajectories/
        └── per-trial inspection figures
```

## Main files

### `dataset_manifest.csv`

This is the main index of the dataset. It contains one row for each trajectory-axis pair.

Important columns:

```text
trial_id              unique trial-axis identifier
trajectory_name       original trajectory name
axis                  x, y, or z
k_index               trajectory index used in the original MATLAB pipeline
relative_csv_path     path to the per-trial CSV file
relative_mat_path     path to the per-trial MAT file
n_samples             number of exported valid samples
n_raw_samples         number of samples before validity filtering
n_removed_samples     number of removed samples
valid_fraction        n_samples / n_raw_samples
sample_rate_hz        nominal sampling rate after alignment/resampling
source_Muse_file      original Muse file name
source_robot_file     original robot file name
valid                 1 = usable by default, 0 = invalid/questionable
notes                 status note
```

Recommended use:

```text
Use only rows where valid = 1 for default model training and testing.
```

### `summary/all_samples.csv`

This is a single flattened table containing all exported samples from all trajectory-axis pairs. It is useful for quick Python/Pandas analysis.

Each row is one time sample.

Important columns:

```text
trial_id
trajectory_name
axis
k_index
sample_idx
time_s

q_Muse_w
q_Muse_x
q_Muse_y
q_Muse_z

q_robot_w
q_robot_x
q_robot_y
q_robot_z

q_err_w
q_err_x
q_err_y
q_err_z

eul_err_roll_deg
eul_err_pitch_deg
eul_err_yaw_deg
eul_err_norm_deg
eul_err_azimuth_deg
eul_err_elevation_deg
error_angle_so3_deg
```

Use this file for sample-level analysis. For sequence models, prefer the per-trial files under `trajectories/`.

### `summary/all_trials_metrics.csv`

This file contains one row per exported trajectory-axis pair, with summary error metrics.

Useful columns:

```text
mean_eul_err_norm_deg
rms_eul_err_norm_deg
max_eul_err_norm_deg
p95_eul_err_norm_deg
mean_error_angle_so3_deg
rms_error_angle_so3_deg
max_error_angle_so3_deg
rms_roll_err_deg
rms_pitch_err_deg
rms_yaw_err_deg
dominant_error_component
```

Use this file to inspect which axes or trajectories are more difficult before training an AI model.

### `trajectories/<trajectory>/<axis>/aligned_quaternions.csv`

This is the recommended file format for sequence-based AI work. Each file contains one aligned trajectory for one axis.

Example path:

```text
trajectories/t1/x/aligned_quaternions.csv
```

The columns are the same as in `summary/all_samples.csv`, but restricted to one trajectory-axis pair.

### `trajectories/<trajectory>/<axis>/aligned_quaternions.mat`

MATLAB version of the same per-trial data. Use this if working directly in MATLAB.

### `trajectories/<trajectory>/<axis>/metadata.json`

Per-trial metadata. It documents the processing applied to that specific trajectory-axis pair.

Important fields:

```text
trial_id
trajectory_name
axis
k_index
sample_rate_hz
n_samples
n_raw_samples
n_removed_samples
valid_fraction
is_questionable
quaternion_order
q_err_definition
euler_convention
robot_frame_after_export
alignment_method
resampling_method
q_F2M_wxyz
q_W2B_wxyz
idx_start_robot
idx_start_Muse
i0_robot
i0_Muse
source_Muse_file
source_robot_file
```

### `dataset_metadata.json`

Global dataset metadata. It defines dataset version, sensor type, quaternion convention, Euler convention, validity policy, and output-file list.

### `Inspection/`

This folder contains quality-control figures. These figures are not needed for AI training, but they are useful to verify that compensation and alignment worked correctly.

Important inspection files:

```text
Inspection/summary/01_global_error_by_axis.png
Inspection/summary/02_trial_rms_distribution.png
Inspection/summary/03_dominant_component_counts.png
Inspection/summary/04_export_status_by_axis.png

Inspection/trajectories/<trajectory>/<axis>/00_alignment_onset_detection.png
Inspection/trajectories/<trajectory>/<axis>/01_quaternion_overlay.png
Inspection/trajectories/<trajectory>/<axis>/02_error_norm_angle.png
Inspection/trajectories/<trajectory>/<axis>/03_component_rms_contribution.png
Inspection/trajectories/<trajectory>/<axis>/04_alignment_activity_overlay.png
```

Use `Inspection/README_INSPECTION.md` for a description of each plot.

## Quaternion convention

All quaternions are stored in this order:

```text
[w x y z]
```

The Muse quaternion after compensation/alignment is:

```text
q_Muse
```

The robot quaternion transformed into the Muse frame is:

```text
q_robot
```

The correction quaternion is:

```text
q_err = conjugate(q_Muse) * q_robot
```

Therefore:

```text
q_robot ≈ q_Muse * q_err
```

For AI compensation, the recommended supervised-learning formulation is:

```text
input  = q_Muse
output = q_err
```

An alternative is:

```text
input  = q_Muse
output = q_robot
```

but predicting the residual correction `q_err` is usually cleaner than predicting the full robot orientation.

## Euler-error convention

Euler angles use the MATLAB ZYX yaw-pitch-roll convention, in degrees.

The exported Euler-error vector is stored as:

```text
[roll_x, pitch_y, yaw_z]
```

The Euler-error norm is:

```text
eul_err_norm_deg = sqrt(roll_error^2 + pitch_error^2 + yaw_error^2)
```

The Euler-error direction angles are:

```text
eul_err_azimuth_deg   = atan2(pitch_error, roll_error)
eul_err_elevation_deg = atan2(yaw_error, sqrt(roll_error^2 + pitch_error^2))
```

The SO(3) angular error is computed from the relative quaternion error and is stored as:

```text
error_angle_so3_deg
```

## Recommended Python loading

Load the manifest and keep only valid trajectory-axis pairs:

```python
import pandas as pd
from pathlib import Path

root = Path("DATASET")

manifest = pd.read_csv(root / "dataset_manifest.csv")
valid_manifest = manifest[manifest["valid"] == 1].copy()

print(valid_manifest[["trial_id", "trajectory_name", "axis", "n_samples", "valid_fraction"]])
```

Load all valid samples from the flattened file:

```python
samples = pd.read_csv(root / "summary" / "all_samples.csv")

valid_trial_ids = set(valid_manifest["trial_id"])
samples_valid = samples[samples["trial_id"].isin(valid_trial_ids)].copy()

X = samples_valid[["q_Muse_w", "q_Muse_x", "q_Muse_y", "q_Muse_z"]].to_numpy()
y = samples_valid[["q_err_w", "q_err_x", "q_err_y", "q_err_z"]].to_numpy()
```

Load per-trial sequences:

```python
sequences = []

for _, row in valid_manifest.iterrows():
    csv_path = root / row["relative_csv_path"]
    trial = pd.read_csv(csv_path)

    q_Muse = trial[["q_Muse_w", "q_Muse_x", "q_Muse_y", "q_Muse_z"]].to_numpy()
    q_err = trial[["q_err_w", "q_err_x", "q_err_y", "q_err_z"]].to_numpy()

    sequences.append({
        "trial_id": row["trial_id"],
        "trajectory_name": row["trajectory_name"],
        "axis": row["axis"],
        "q_Muse": q_Muse,
        "q_err": q_err,
    })
```

## Recommended MATLAB loading

Load the manifest:

```matlab
root = "DATASET";
manifest = readtable(fullfile(root, "dataset_manifest.csv"));
valid_manifest = manifest(manifest.valid == 1, :);
```

Load the flattened sample table:

```matlab
samples = readtable(fullfile(root, "summary", "all_samples.csv"));
```

Load one per-trial CSV:

```matlab
csv_path = fullfile(root, valid_manifest.relative_csv_path{1});
trial = readtable(csv_path);

q_Muse = trial{:, {"q_Muse_w", "q_Muse_x", "q_Muse_y", "q_Muse_z"}};
q_err  = trial{:, {"q_err_w",  "q_err_x",  "q_err_y",  "q_err_z"}};
```

Load one per-trial MAT file:

```matlab
mat_path = fullfile(root, valid_manifest.relative_mat_path{1});
S = load(mat_path);
```

## Suggested train/test split policy

Do not randomly split individual samples from the same trajectory between train and test. Adjacent samples are strongly correlated.

Prefer splitting by trajectory or by trajectory-axis pair:

```text
recommended: split by trial_id or trajectory_name
avoid: random row-wise split of all_samples.csv
```

For example, use complete trajectory-axis sequences for training and reserve different complete trajectories for validation/testing.

## Practical recommendation for AI compensation

Default input/output definition:

```text
input features:
    q_Muse_w, q_Muse_x, q_Muse_y, q_Muse_z

target:
    q_err_w, q_err_x, q_err_y, q_err_z
```

Optional additional input features:

```text
axis
trajectory type / k_index
time_s or normalized time within trajectory
```

Default filtering:

```text
1. keep only manifest rows with valid = 1
2. use per-trial files for sequence models
3. use summary/all_samples.csv only for quick sample-level baselines
4. keep quaternion order [w x y z]
5. normalize predicted quaternions before applying them
```

## Notes for users of this repository

This repository contains the generated dataset, not the original raw acquisition folders.

The `Inspection/` figures are included to allow visual sanity checks before training models. They are not used directly by the AI pipeline.

The manifest is the authoritative file for deciding which trajectory-axis pairs should be used by default.
