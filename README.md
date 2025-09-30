# Robot Navigation & YOLO Integration (Course-Only)

> ⚠️ **IMPORTANT**
> 此專案**只能在課程提供的容器與模擬環境**中執行（含預設 ROS 版本、依賴、驅動、模擬器、橋接設定與模型資源）。
> 在其他機器或自行安裝的 ROS/Ubuntu 上**不保證可用**，也**不提供**通用化的安裝支援。

---

## Overview

本專案整合了：

* **YOLOv8 物件偵測**（/yolo/target_info：`[found_flag, depth_m, offset_px]`）
* **LIDAR 360° 環境感知**（/scan）
* **反應式導航策略**：

  * INITIAL_SCAN：起始 360° 掃描
  * EXPLORING：LIDAR 開口探索（含 top-2 接近時以**通道中心“更近”**者為優先）
  * WALL_FOLLOWING：牆隨行（保持側距 `desired_wall_dist`）
  * APPROACHING_TARGET：YOLO 視覺伺服靠近目標
  * TARGET_REACHED / Stuck Recovery：任務完成與卡住復原

輸出對應車輪控制動作：`FORWARD / FORWARD_SLOW / BACKWARD / CLOCKWISE_ROTATION(_SLOW) / COUNTERCLOCKWISE_ROTATION(_SLOW) / STOP`。

---

## Course Environment (Required)

* 課程提供的 **Docker 容器** 與 **模擬器（PROS Twin）**
* 課程預載的 **ROS 2**（以課程版本為準）
* **yolo_activate.sh** 啟動腳本與預設網路/橋接
* 內建之 **Foxglove/RViz**（若有）與 rosbridge 設定

> 若你未使用課程的啟動方式，以下任何步驟都可能失敗（尤其是影像/深度/橋接話題）。

---

## Repo Structure (簡要)

```
src/
  pros_car_py/                 # 車體控制與導航
    nav2_processing.py         # 狀態機、LIDAR探索、牆隨行、視覺伺服
    ros_communicator.py        # ROS 訂閱/發布、車輪控制、工具方法
    nav2_utils.py              # 幾何工具
  yolo_pkg/                    # YOLO 節點（偵測、深度、可視化）
    yolo_detection_node.py     # 主要 YOLO 節點
    ros_communicator.py        # yolo_pkg 專用通訊封裝
  yolo_example_pkg/
    models/                    # *.pt 模型放這裡（課程環境已配置）
```

---

## Quick Start (課程環境)

1. 啟動容器與環境

```bash
./yolo_activate.sh
```

2. 編譯與載入環境

```bash
colcon build
source install/setup.bash
```

3. 啟動 YOLO 節點（偵測 + 輸出影像與 target_info）

```bash
ros2 run yolo_pkg yolo_detection_node
```

4. 啟動車體控制 / 導航（依課程指示）

```bash
ros2 run pros_car_py robot_control
```

5. 在模擬器（PROS Twin）右側 **RosBridge** 填 `localhost`，點 **Reload Button**
   選擇地圖（例如 Pikachu / Door Room），開啟 **DEPTH**。

---

## Topics & Interfaces

**Input / Subscriptions**

* `/camera/image/compressed` — RGB 壓縮影像
* `/camera/depth/image_raw`、`/camera/depth/compressed` — 深度
* `/scan` — LIDAR（sensor_msgs/LaserScan）
* `/yolo/target_info` — YOLO 融合輸出（std_msgs/Float32MultiArray）

**Output / Publications**

* `/yolo/detection/compressed` — 疊框可視化影像
* 車輪控制：`car_C_rear_wheel`、`car_C_front_wheel`（課程定義）
* 其他：導航用診斷、標記等（見程式）

---

## Navigation Modes (摘要)

* **開口探索（EXPLORING）**
  以滑窗卷積在 360° 找連續最寬方向；若 top2 分數接近（≤10%），選**通道中心點更近**者，保守前進；對準後 `FORWARD`。
* **牆隨行（WALL_FOLLOWING）**
  INITIAL_SCAN 後選左或右牆，持續維持側距 `desired_wall_dist ± wall_tol`；前方小於 `front_stop_dist` 先避障轉向。
* **視覺伺服（APPROACHING_TARGET）**
  以 `offset_px` 校正轉向，`depth_m` 判斷停止；前方 LIDAR 若更近則覆蓋 STOP/轉向，避免碰撞。
* **卡住恢復（Stuck Recovery）**
  連續多次非前進同動作 → `BACKWARD → CLOCKWISE_ROTATION → CLOCKWISE_ROTATION` → 回到 EXPLORING。

---

## Key Parameters (程式內可調)

| 參數                    | 作用           | 建議值         |
| --------------------- | ------------ | ----------- |
| `window_size`         | LIDAR 開口滑窗大小 | 20          |
| `align_threshold_deg` | 對準角度門檻       | 15°         |
| `front_stop_dist`     | 前方停止距離       | 0.20–0.30 m |
| `desired_wall_dist`   | 牆隨行目標距離      | 0.23 m      |
| `wall_tol`            | 牆距容忍         | 0.02 m      |
| `stuck_threshold`     | 卡住復原計數       | 12–15       |
| `yolo_stop_dist`      | 視覺停車距離       | 0.01 m      |
| `offset_px_threshold` | 視覺對準像素閾值     | 100 px      |

---

## Verifications / Debug

查看話題是否存在與有資料：

```bash
ros2 topic list
ros2 topic info /yolo/target_info
ros2 topic echo /yolo/target_info
ros2 topic info /scan
ros2 topic echo /scan  # 請小心輸出量
```

常見診斷訊息（在 `robot_control` 終端可見）：

* `[NavProcessing] ...`：狀態切換、決策輸出
* `[WallNav] ...`：牆隨行側距、分支
* `[LIDAR] ...`：開口分數、角度與距離、top2 tie-breaker 結果

---

## Dataset & Model (for YOLO)

* 模型放在：`yolo_example_pkg/models/`（課程已提供）
* data.yaml 與標註依課堂示例；若需替換模型，請保持相同路徑與命名。

---

## Troubleshooting

* **`/yolo/target_info` 沒資料**
  確認 `yolo_pkg/yolo_detection_node` 已啟動、模型可載入、深度話題可用（raw 或 compressed 至少一個要有）。
* **模擬器沒有動作**
  檢查 PROS Twin 的 RosBridge 是否填 `localhost` 並按 **Reload Button**，以及車速滑桿非零。
* **一直旋轉或一直停**
  多半是 LIDAR/深度/YOLO 任一資料流未就緒；或前方距離門檻太嚴；調整 `front_stop_dist / desired_wall_dist`。
* **進入走廊又回頭**
  已在演算法中用 top2 & 近距優先與遮罩避免；如仍發生，適當放寬 `align_threshold_deg`、降低 `window_size` 或加大 `front_stop_dist`。

---

## License & Limitation

* 此專案僅用於**課程作業與演示**。
* **不保證在課程環境外**（不同 OS/ROS/Docker/硬體/話題命名）可用。
* 模型與素材依課堂規範使用；請勿未經允許散佈。

---

## Acknowledgements

感謝課程提供之容器、模擬器、橋接與基礎程式框架；本專案在其上整合 YOLO 與反應式導航以完成任務。

---
