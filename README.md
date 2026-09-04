# Lidar_Camera_Calibration

Velodyne VLP-16 LiDAR 데이터 기록 및 카메라-라이다 외부 파라미터 캘리브레이션 작업 저장소입니다.

이 저장소에는 VLP-16에서 수집한 **원본 UDP 패킷 단위 rosbag 기록**이 포함되어 있습니다.

---

## 이게 뭔가요?

Velodyne VLP-16 3D LiDAR가 회전하며 초당 약 10회 주변을 스캔합니다. 이 저장소의 bag 파일은 그 스캔 데이터를 **센서가 내보낸 원본 그대로** 15초간 담아둔 것입니다.

재생하면 녹화 당시의 3D 공간이 시간 순서대로 그대로 복원됩니다.

---

## 왜 포인트클라우드가 아니라 패킷을 저장했나

VLP-16 데이터는 두 가지 형태로 저장할 수 있습니다.

| 토픽 | 내용 | 크기 | 시간당 |
|---|---|---|---|
| `/velodyne_packets` | 센서 원본 UDP 패킷 | 92.45 KB/msg | **3.3 GB** |
| `/velodyne_points` | 변환된 3D 포인트클라우드 | 0.64 MB/msg | 22.9 GB |

`/velodyne_packets`를 선택한 이유는 두 가지입니다.

**1. 용량이 7배 작으면서 무손실입니다.**
패킷을 재생해서 `velodyne_transform_node`에 통과시키면 포인트클라우드가 100% 동일하게 복원됩니다. 정보 손실이 전혀 없습니다.

**2. 캘리브레이션을 나중에 바꿀 수 있습니다.**
포인트클라우드로 저장하면 거리·각도 보정값이 이미 좌표에 반영되어 굳어버립니다. 패킷으로 저장하면 보정 파라미터(`VLP16db.yaml`)를 수정한 뒤 다시 변환할 수 있습니다. 캘리브레이션 작업에서는 이 점이 결정적입니다.

---

## 기록된 데이터

```
velodyne_packets_15s/
├── metadata.yaml
└── velodyne_packets_15s_0.db3
```

| 항목 | 값 |
|---|---|
| 기록 시각 | 2026-09-04 18:50:13 ~ 18:50:28 (KST) |
| 지속 시간 | 14.93 초 |
| 메시지 수 | 149 |
| 실측 주기 | 9.98 Hz |
| 파일 크기 | 13.4 MiB |
| 토픽 | `/velodyne_packets` |
| 메시지 타입 | `velodyne_msgs/msg/VelodyneScan` |
| 저장 포맷 | sqlite3 (rosbag2, CDR 직렬화) |

### 센서 설정

| 항목 | 값 |
|---|---|
| 모델 | Velodyne VLP-16 (Puck) |
| 회전 속도 | 600 RPM (10 Hz) |
| 채널 | 16 |
| IP / 포트 | 192.168.1.201 / UDP 2368 |
| `frame_id` | `velodyne` |
| 스캔당 패킷 | 76 |

---

## 재생 방법

### 1. 준비

ROS 2 Humble과 [`ros-drivers/velodyne`](https://github.com/ros-drivers/velodyne) 패키지가 필요합니다.

```bash
mkdir -p ~/velodyne_ws && cd ~/velodyne_ws
git clone https://github.com/ros-drivers/velodyne.git
colcon build --symlink-install
source install/setup.bash
```

### 2. 변환 노드 실행

패킷을 포인트클라우드로 바꿔주는 노드를 먼저 띄웁니다. **실제 라이다 하드웨어는 필요 없습니다.**

```bash
ros2 launch velodyne_pointcloud velodyne_transform_node-VLP16-launch.py
```

### 3. bag 재생

새 터미널에서:

```bash
ros2 bag play velodyne_packets_15s
```

반복 재생하려면 `--loop`, 느리게 보려면 `--rate 0.5`를 붙입니다.

### 4. 시각화

또 다른 터미널에서:

```bash
rviz2
```

RViz에서 아래 두 가지만 설정하면 포인트클라우드가 보입니다.

- **Fixed Frame** → `velodyne`
- **Add** → **PointCloud2** → Topic을 `/velodyne_points`로 지정

> `/velodyne_points`는 Best Effort QoS로 발행됩니다. RViz의 Reliability Policy도 **Best Effort**로 맞춰야 데이터가 표시됩니다.

---

## 데이터 구조

재생 후 나오는 `/velodyne_points`의 포인트 하나는 22바이트이며, 스캔당 29,184개(1824 × 16채널)입니다.

| 필드 | 타입 | 크기 | 설명 |
|---|---|---|---|
| `x`, `y`, `z` | float32 | 12 B | 센서 기준 3D 좌표 (미터) |
| `intensity` | float32 | 4 B | 반사 강도 |
| `ring` | uint16 | 2 B | 레이저 채널 번호 (0~15) |
| `time` | float32 | 4 B | 스캔 시작 시점 대비 상대 시각 (초) |

`ring`과 `time` 필드 덕분에 포인트 단위 시계열 복원이 가능합니다. 메시지 헤더의 절대 타임스탬프에 포인트별 `time` 오프셋을 더하면 각 점이 정확히 언제 측정되었는지 알 수 있습니다. 회전형 LiDAR는 한 스캔 안에서도 점마다 측정 시각이 다르기 때문에, 움직이는 플랫폼에서는 이 값이 모션 보정(deskewing)에 필수입니다.

---

## 다른 형식으로 내보내기

포인트클라우드를 CSV나 PCD로 뽑고 싶다면 bag을 재생하면서 저장하면 됩니다.

```bash
# PCD 파일로 (스캔당 1개 파일)
ros2 run pcl_ros pointcloud_to_pcd --ros-args -r input:=/velodyne_points
```

pandas 분석용 테이블이 필요하면 `sensor_msgs_py.point_cloud2.read_points()`로 numpy 배열을 얻어 Parquet으로 저장하는 방식을 권장합니다. CSV는 같은 데이터가 5배 이상 커집니다.

---

## 수집 환경

| 항목 | 내용 |
|---|---|
| OS | Ubuntu 22.04 (Linux 6.8.0-138-generic) |
| ROS | ROS 2 Humble |
| 드라이버 | [ros-drivers/velodyne](https://github.com/ros-drivers/velodyne) |
| 호스트 | ASUS TUF Gaming A14 (FA401UM) |
| 네트워크 | Belkin Thunderbolt 5 Dock 내장 2.5GbE (RTL8156b), 40 Gb/s USB4 링크 |
| 호스트 IP | 192.168.1.100/24 |

수집 중 패킷 유실은 없었습니다 (NIC RX errors 0 / dropped 0 / missed 0).
