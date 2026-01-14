# Dynamic Indoor Robot Navigation — Analysis & Baselines

This repository contains exploratory analysis and simple baseline methods for a dataset focused on **real-time autonomous navigation** in **dynamic indoor environments**. The robot navigates in a simulated indoor space with moving obstacles and multiple conflicting objectives (time, energy, safety, smoothness).

## Dataset Overview

**File:** `autonomous_navigation_dataset.csv`

Each row represents a robot observation at a specific **timestamp** within a navigation **run**.

### Columns
- `run_id`: Unique navigation session ID  
- `timestamp`: Time step within a run  
- `lidar_min`, `lidar_max`: Minimum/maximum LIDAR range readings  
- `ultrasonic_left`, `ultrasonic_right`: Proximity sensor readings  
- `x`, `y`: Robot 2D position  
- `orientation`: Heading (degrees)  
- `battery_level`: Remaining battery (%)  
- `collision_flag`: 1 if collision detected at that timestamp  
- `path_smoothness`: Score indicating curvature/smoothness of movement  
- `time_taken`: Final time duration for the run  
- `target`: Final run label (`success`, `partial`, `collision`, `timeout`)

### Labels (`target`)
- **success**: Goal reached with optimal performance  
- **partial**: Goal reached, but higher energy use and/or worse smoothness  
- **collision**: Collision occurred during navigation  
- **timeout**: Goal not reached within time limit  

---

## Objectives of This Project

1. **EDA (Exploratory Data Analysis)**  
   - Basic statistics, distributions (LIDAR/ultrasonic/battery), collision frequency  
   - Visualization of robot trajectories `(x, y)` per `run_id`

2. **Rule-based Navigation Baseline**  
   A simple reactive controller based on ultrasonic proximity:
   - If both sides are closer than `SAFE_DISTANCE` → `stop`
   - If left is too close → `turn_right`
   - If right is too close → `turn_left`
   - Else → `forward`

   We evaluate how different `SAFE_DISTANCE` values change:
   - movement distribution
   - collisions per movement decision

3. **Collision / Near-Miss Analysis**
   - Collision counts and their association with sensor values  
   - Definition of **near-miss events** (close call without collision), e.g.:
     - `lidar_min < SAFE_DISTANCE` AND `collision_flag == 0`

4. **Unsupervised Anomaly Detection (Isolation Forest)**
   - Build **sliding window** features from past sensor readings  
   - Train on “normal” data (non near-miss)  
   - Detect anomalous patterns and compare anomaly predictions vs near-miss

---

## Methods

### 1) Exploratory Data Analysis (EDA)
- Summary stats: `df.info()`, `df.describe()`
- Missing values: `df.isnull().sum()`  
- Distribution plots for:
  - `ultrasonic_left`, `ultrasonic_right`
  - `lidar_min`, `lidar_max`
  - `battery_level`
- Collision imbalance inspection (`collision_flag`)

### 2) Rule-based Controller (Reactive Baseline)

**Decision logic:**
- `stop` if both ultrasonic readings are below threshold
- `turn_right` if left is below threshold
- `turn_left` if right is below threshold
- `forward` otherwise

**Parameter sweep:**  
`SAFE_DISTANCE ∈ {0.3, 0.5, 0.7, 1.0}`

For each threshold we report:
- collisions per predicted movement  
- non-collision movement distribution  

> Note: This baseline is intentionally simple and does not use full state (e.g., velocities, orientation, lidar patterns).

### 3) Near-Miss Definition
We define near-miss states as “dangerously close but no collision”:
- `near_miss = 1` if `lidar_min < SAFE_DISTANCE` AND `collision_flag == 0`
- else `near_miss = 0`

This is useful because collisions are rare, so near-miss events provide more frequent safety-related signals.

### 4) Isolation Forest with Sliding Window Features
We construct a temporal context using a sliding window of size `W=5`.

For each time step we build:
- `lidar_min_t-1 ... lidar_min_t-5`
- `ultrasonic_left_t-1 ... ultrasonic_left_t-5`
- `ultrasonic_right_t-1 ... ultrasonic_right_t-5`

Then:
- Fit Isolation Forest on normal samples (`near_miss == 0`)
- Predict anomalies on all samples
- Compare anomaly output vs near-miss using a confusion-style table

---

## Key Findings (from current analysis)

- `collision_flag` is **highly imbalanced** (collisions are rare compared to non-collisions).
- Increasing `SAFE_DISTANCE` shifts decisions toward more `stop/turn` behaviors (more conservative).
- Simple thresholding alone does **not** strongly separate collision vs non-collision states (sensor distributions overlap).
- Near-miss events are much more frequent than collisions and can be used as a richer safety proxy.
- Isolation Forest flags a subset of near-miss points as anomalous, but also detects some anomalies in normal behavior (expected in unsupervised settings).

---

## How to Run

### Requirements
- Python 3.x  
- `pandas`, `numpy`, `matplotlib`, `seaborn`
- `scikit-learn`
- (optional) `plotly` for 3D visualization

### Example
1. Load the dataset:
```python
import pandas as pd
df = pd.read_csv("autonomous_navigation_dataset.csv")
```

2. Run EDA + baselines in a notebook (recommended).

---

## Project Structure (suggested)

- `notebooks/`
  - `01_eda.ipynb`
  - `02_rule_based_baseline.ipynb`
  - `03_near_miss_anomaly.ipynb`
- `data/`
  - `autonomous_navigation_dataset.csv` *(or Kaggle input path)*
- `README.md`

---

## Notes / Limitations

- The rule-based controller uses only ultrasonic readings; it ignores:
  - LIDAR spatial patterns (beyond min/max)
  - robot kinematics (velocity/acceleration constraints)
  - goal direction / global planning
- `movement_decision` is a **derived label** from a handcrafted policy and will be trivially predictable by ML models trained on the same rules.
- For learning meaningful policies, consider:
  - predicting `target` per run (session-level classification)
  - predicting collision/near-miss risk (time-step classification)
  - reinforcement learning / imitation learning setups

---

## Future Work Ideas

- Session-level feature engineering (aggregate per `run_id`):
  - total distance traveled, energy drop, avg smoothness, collision count
- Time-series models:
  - LSTM/GRU/Temporal CNN on sliding windows
- Safety-aware metrics:
  - near-miss rate, minimum clearance statistics, risk curves
- Multi-objective scoring:
  - combine time, energy, smoothness, and safety into a weighted objective

---

## License / Credits
- Dataset: Dynamic Indoor Robot Navigation Dataset (Kaggle source)  
- This repository: analysis + baseline implementations for research and educational use.
