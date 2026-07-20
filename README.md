# duatic_scan_2d_merger

[![Jazzy](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/mbloechli/d2906ca9c367e68f47e636da2059be24/raw/duatic_scan_2d_merger-jazzy.json)](https://github.com/Duatic/duatic_scan_2d_merger/actions/workflows/ci.yml)
[![Kilted](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/mbloechli/d2906ca9c367e68f47e636da2059be24/raw/duatic_scan_2d_merger-kilted.json)](https://github.com/Duatic/duatic_scan_2d_merger/actions/workflows/ci.yml)
[![Lyrical](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/mbloechli/d2906ca9c367e68f47e636da2059be24/raw/duatic_scan_2d_merger-lyrical.json)](https://github.com/Duatic/duatic_scan_2d_merger/actions/workflows/ci.yml)
[![Rolling](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/mbloechli/d2906ca9c367e68f47e636da2059be24/raw/duatic_scan_2d_merger-rolling.json)](https://github.com/Duatic/duatic_scan_2d_merger/actions/workflows/ci.yml)

A Duatic-maintained fork of [2D_Scan_Merger_ROS2](https://github.com/ali-pahlevani/2D_Scan_Merger_ROS2)
by Ali Pahlevani. This fork tracks the ROS 2 distros used at [Duatic](https://duatic.com) and
adds our CI/packaging. For the full design documentation and to contribute to the core
project, please refer to the upstream repository.


## Overview


A ROS 2 composable node that merges any number of 2D LiDAR scans into a single unified
`sensor_msgs/LaserScan`, with approximate-time synchronization and parallel ray projection.

The node is implemented as a **composable node** (`rclcpp_components`), so it can run inside
a shared component container with zero-copy intra-process communication, or as a standalone
process depending on your deployment needs.

![2D-Scan Merger Preview](https://github.com/user-attachments/assets/63ef8f72-5476-4905-8f3e-35a4d9702792)


## Features

- **Unlimited input LiDARs**: the custom approximate-time synchronizer imposes no cap on the
  number of input topics (unlike `message_filters::ApproximateTime`, which is bounded by
  compile-time template constraints).
- **Parallel ray projection**: for N > 1 LiDARs, a persistent thread pool (one thread per
  LiDAR) projects all scans simultaneously using `std::barrier` (C++20). Threads are created
  once at startup, eliminating per-callback overhead.
- **No intermediate point cloud**: rays are projected directly from polar coordinates to
  angular bins in the merged frame. No `PointCloud2` conversion and no PCL dependency.
- **Per-topic QoS**: each input topic can independently use `reliable` or `best_effort`
  delivery, making it straightforward to mix LiDARs with different publishers.
- **Static and dynamic scanner mounts**: with `moving_frames: false` (default) TF transforms
  are looked up once and cached; set `true` for manipulator-mounted or otherwise moving scanners.
- **Height filtering**: points outside a configurable `[min_height, max_height]` band in the
  merged frame are discarded, useful for filtering ground returns or ceiling reflections.
- **Composable node**: runs inside `component_container` for efficient multi-node deployments.
  A `MutuallyExclusiveCallbackGroup` ensures serial callback execution on any executor.


## Configuration

The example config file (`config/example/param.yaml`) contains values for a specific robot.
For your own robot, adapt the values to your use case (defaults are listed below as a starting
point).

### Parameter Reference

| Parameter | Type | Default | Description |
|---|---|---|---|
| `scan_topics` | `string[]` | `[]` | **Required.** List of input `LaserScan` topic names. Supports 1 to N topics. |
| `scan_policies` | `int64[]` | `[]` | QoS reliability per topic: `0` = reliable, `1` = best effort. Shorter than `scan_topics` defaults missing entries to reliable. |
| `merged_frame_id` | `string` | `"base_link"` | TF frame of the merged output scan. |
| `output_topic` | `string` | `"merged_scan"` | Topic name for the merged `LaserScan`. |
| `sync_slop` | `double` | `0.1` | Maximum timestamp difference [s] for two scans to be considered synchronised. Ignored when only one topic is configured. |
| `queue_size` | `int` | `20` | Per-topic scan buffer depth. Older scans are dropped when the buffer is full. |
| `moving_frames` | `bool` | `false` | Set to `true` if scanner frames move relative to `merged_frame_id` at runtime. When `false`, transforms are looked up once and cached. |
| `tolerance` | `double` | `0.01` | TF lookup timeout [s]. |
| `angle_min` | `double` | `-π` | Start angle of the output scan [rad]. |
| `angle_max` | `double` | `π` | End angle of the output scan [rad]. |
| `angle_increment` | `double` | `π/180` | Angular resolution of the output scan [rad]. Smaller values increase resolution but also message size. |
| `range_min` | `double` | `0.1` | Minimum valid range [m]. Closer readings are discarded. |
| `range_max` | `double` | `float max` | Maximum valid range [m]. Farther readings are discarded. |
| `min_height` | `double` | `-∞` | Minimum point height [m] in `merged_frame_id`. Points below this are discarded. Useful for filtering ground returns. |
| `max_height` | `double` | `+∞` | Maximum point height [m] in `merged_frame_id`. Points above this are discarded. Useful for filtering ceiling reflections. |
| `use_inf` | `bool` | `true` | When `true`, bins with no reading are set to `+inf`. When `false`, they are set to `range_max + inf_epsilon`. |
| `inf_epsilon` | `double` | `1.0` | Offset added to `range_max` for the no-reading fill value when `use_inf` is `false` [m]. |
| `scan_time` | `double` | `1/30` | Nominal scan period [s], written into the output message header. |
| `debug` | `bool` | `false` | Logs each subscribed topic and its QoS policy at startup. |


## Running

### Demo (with included rosbag)

The demo launch file plays the bundled rosbag (3 LiDAR scans), starts the merger node, and
opens RViz2 with a preconfigured display:

```bash
ros2 launch scan_2d_merger demo_launcher.launch.py
```

### Custom robot configuration

```bash
ros2 launch scan_2d_merger merger_launcher.launch.py robotname:=my_robot
```

This loads `config/my_robot/param.yaml` and starts the merger inside a composable node
container. Replace `my_robot` with the name of the directory you created under `config/`.

### Directly as a composable node

If you already have a component container running, load the node into it:

```bash
ros2 component load /component_manager_node scan_2d_merger util::LaserScanMerger \
  --param scan_topics:='["/scan0", "/scan1"]' \
  --param merged_frame_id:=base_link
```

## Topics

### Subscribed

| Topic | Type | Description |
|---|---|---|
| As configured in `scan_topics` | `sensor_msgs/LaserScan` | One subscription per entry in `scan_topics`. QoS set per `scan_policies`. |

### Published

| Topic | Type | QoS | Description |
|---|---|---|---|
| As configured in `output_topic` (default: `merged_scan`) | `sensor_msgs/LaserScan` | Reliable, depth 1 | The merged scan in the `merged_frame_id` frame. |

The node looks up the transform from each scanner's `header.frame_id` to `merged_frame_id`
using TF2 (listening on `/tf` and `/tf_static`). Make sure your robot's TF tree is
broadcasting the required transforms before starting the node.

## License

The contents are licensed under the MIT [license](LICENSE).\
This package is a fork of [2D_Scan_Merger_ROS2](https://github.com/ali-pahlevani/2D_Scan_Merger_ROS2),
© Ali Pahlevani.

## Contributing

This repository is a downstream fork maintained by Duatic. For contributions to the core
algorithm, please open a pull request against the
[upstream repository](https://github.com/ali-pahlevani/2D_Scan_Merger_ROS2). For issues
specific to Duatic packaging, CI, or distro support, please open an issue here.
