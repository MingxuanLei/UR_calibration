# UR5e 眼在手外标定 GUI（ur_eyetohand.py）

本程序从已经可用的 `ur_eyehand_jog.py` 改写，保留 RealSense D435i RGB 实时预览、11×8 棋盘内角点识别、UR5e 关节角/TCP 位姿读取、J1～J6 ± 增量点动（默认锁定）、手动采集、可选保存对齐深度图、删除/载入采样和四种 OpenCV 手眼求解算法。

## 安装与启动

Windows Python 环境：

```powershell
pip install numpy opencv-python pillow pyrealsense2 ur-rtde
python ur_eyetohand.py
```

不接 UR5e 和相机也可以检查运动参数与标定矩阵求解方向：

```powershell
python ur_eyetohand.py --self-test
python ur_eyetohand.py --jog-self-test
```

## 重要区别：眼在手外的安装方式

- **相机必须在机器人外部固定**，整个采样过程中不可移动。求解以 RealSense **彩色相机光学坐标系**为基准；不是深度光学坐标系。
- **12 × 9 方格棋盘格须刚性固定到机器人末端**（法兰/固定工具支架）；11 × 8 是程序检测的内部角点规格。棋盘格不可滑动、变形，夹爪夹持需要确保足够牢固。
- 棋盘格**单个方格边长**须用卡尺实测后输入 GUI 的“方格边长 mm”，默认 20 mm 仅是占位示例。
- 标定全过程不要更改 UR 活动 TCP 设置；如果使用自定义 TCP，`T_base_tcp` 是当前 TCP，而不是法兰。
- 不可把旧版眼在手上采集的棋盘格固定、相机随末端移动的数据直接导入新版。新版将主动拒绝加载旧版没有 `eye_to_hand` 模式标记的 `session.json`。

## 采样步骤

1. 固定外部相机、在 UR5e 末端安装棋盘格，检查法兰/夹爪/棋盘格、周边设备和工件的碰撞风险。
2. 连接 UR5e 与 RealSense。GUI 默认 UR IP 为 `192.168.3.15`，相机以序列号选择。
3. 使用 UR 示教器或 GUI 六关节独立 ± 点动调整末端姿态（默认点动锁定，必须主动启用）；确保所有棋盘角点都进入 RGB 图像。
4. 机械臂停稳，图像清晰、重投影误差合格，再点击“记录当前姿态 + 棋盘格”。
5. 先采 10 组验证流程，建议 15～25 组并包含**多个明显不同的末端旋转方向**；只有平移而几乎没有姿态旋转，不适合手眼标定。
6. 点击“求解手眼矩阵”，查看一致性指标，并用新姿态独立复核转换结果。

GUI 点动使用增量 `moveJ`，停止按钮不是安全级急停，实体急停必须始终可及。勿与遥操作/回放脚本同时连接控制同一 UR5e。

## 输出、单位及文件隔离

默认输出在程序运行工作目录下：

```text
eye_to_hand_data/
└── eyetohand_YYYYMMDD_HHMMSS_microseconds/
    ├── session.json
    ├── img_001_......png
    ├── depth_001_......png     # 仅在启用深度保存时存在
    ├── result.json
    ├── T_base_cam.npy          # 主要结果：彩色相机光学系 -> UR Base
    ├── T_base_cam.txt
    ├── T_cam_base.npy          # 逆变换：UR Base -> 彩色相机光学系
    ├── T_cam_base.txt
    ├── T_tcp_board.npy         # 辅助结果：标定板 -> UR 当前 TCP
    └── T_tcp_board.txt
```

`T_*.npy` 均为 NumPy 4×4 float64 齐次矩阵，平移单位为米、角度隐含于 3×3 旋转矩阵（无欧拉角）。同名 `.txt` 以文本方式保存矩阵。

之前眼在手上程序默认写入 `handeye_data/eyehand_时间戳/`，主矩阵叫 `T_tcp_cam.npy`。这版默认使用独立根目录、不同的会话文件夹前缀和不同的主矩阵名。如果手动修改 GUI 的保存根目录，仍会生成带 `eyetohand_` 前缀的独立子文件夹。

## 矩阵方向与验证

约定 `T_A_B` 把 B 系下的齐次三维点转换到 A 系。第 i 组采样满足：

```text
T_base_tcp(i) @ T_tcp_board = T_base_cam @ T_cam_board(i)
```

其中：

- `T_base_tcp(i)`：UR5e RTDE 从当前 TCP 位姿得到；
- `T_cam_board(i)`：OpenCV `solvePnP()` 从彩色图角点、相机内参与实测方格边长得到；
- `T_base_cam`：眼在手外标定的主要结果；
- `T_tcp_board`：棋盘格与 TCP 的固定安装关系，辅助计算与检查。

利用 OpenCV `calibrateHandEye` 求解时，把机器人原始位姿**取逆**作为第一组输入，PnP 的 `T_cam_board` 保持不变。结果验证使用：

```text
T_tcp_board(i) = inverse(T_base_tcp(i)) @ T_base_cam @ T_cam_board(i)
```

这些 `T_tcp_board(i)` 应尽量一致。程序输出平移和旋转 RMS 离散度；它们并不等于外参相对于实际世界的绝对误差。

三维点转 Base：

```python
import numpy as np
T_base_cam = np.load(r"eye_to_hand_data\eyetohand_某次采集\T_base_cam.npy")
p_cam = np.array([x_m, y_m, z_m, 1.0], dtype=float)  # RGB 彩色光学坐标系，单位米
p_base = T_base_cam @ p_cam
print(p_base[:3])
```

如果三维点由深度图获得，应先将深度坐标变换到 **RGB 彩色相机光学坐标系**（例如使用 RealSense 对齐与对应内参的反投影）；不能把未对齐的深度光学坐标直接乘 `T_base_cam`。

## 已完成的离线验证与限制

- Python 编译/静态语法检查；
- 模拟多姿态 Eye-to-Hand 求解，验证 `T_base_cam`、`T_tcp_board` 的方向；
- 关节点动参数及软件关节角边界测试；
- 虚拟桌面 GUI 启动、采样记录/保存 14 组、矩阵 `.npy`/`.txt`/JSON 写出、历史记录重新载入测试；
- 检查新程序会拒绝眼在手上旧记录；
- 含轻微测量噪声的手眼求解试验。

**未在真实 UR5e 和 D435i 上联调该新版**，真实机器连接、控制行为和绝对标定精度须现场确认。
