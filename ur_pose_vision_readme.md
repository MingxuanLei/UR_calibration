# UR5e 视觉引导轨迹补偿 GUI 使用说明

**主程序：`ur_pose_vision.py`**

新版基于原 `ur5e_pose_gui.py` 修改，保留原来的手动 XYZ / Yaw 补偿、夹爪闭合事件判断、名义 UR5e DH 正/逆运动学、渐变补偿、三维轨迹预览、NPZ / PNG 保存；新增读取 `square_pose.py` 的视觉 JSON、绑定原始示教 NPZ 的任务包、结果源数据校验。程序**只做离线计算，不控制 UR5e**。

## 一、放置代码

建议把文件放在原来的轨迹目录旁：

```text
robot_project/
  ur_pose_vision.py               # 本次新增的 GUI
  square_pose.py                  # 原视觉定位 GUI（保持不变）
  GUI_takeover_pv.py              # 原遥操作示教/回放 GUI（保持不变）
  ur_takeover_pv.py               # 原遥操作底层（本次不修改）
  teach_records/
    teach_001.npz                 # GUI_takeover_pv 新录制的轨迹
    visual_tasks/                 # 本程序自动创建
    shift_outputs/                # 本程序自动创建
```

依赖：

```powershell
pip install numpy scipy matplotlib
python ur_pose_vision.py
```

Tkinter 通常随 Windows Python 安装。本程序离线运行，不要求 pyrealsense2 / ur-rtde；视觉拍摄仍由 `square_pose.py` 完成。

## 二、推荐的新示教流程

1. **工件置于新的真实示教位置 A**，确认 80 mm 正方形顶面能从安全观察位姿完整看到，暂时不要移动工件。
2. 启动 `square_pose.py`，连接 RealSense 和只读 UR，加载眼在手上 `T_tcp_cam.npy`；识别绿色顶面四边形，检查出现有效 **Base XYZ + 等价 Yaw**，点击**记录示教参考**。得到 `square_pose_data/reference_时间.json`。
3. 保持工件不动。使用 `GUI_takeover_pv.py` 进行正常遥操作示教，抓取过程录制并保存新的 `teach_records/teach_*.npz`。视觉参考只需在示教前记录一次，不需要逐帧记录 RGB-D。
4. 运行 `ur_pose_vision.py`，在顶部选刚录制的 NPZ；进入下方【视觉绑定 / JSON导入】Tab，选择第 2 步参考 JSON，点击**创建任务包**，在确认两者对应同一工件示教位置后进行绑定。任务包将自动保存到 `teach_records/visual_tasks/task_.../`。
5. 复位到安全观察位姿，再将工件移到新位置 B。在 `square_pose.py` 中点击**加载参考JSON**，加载**任务包目录下的 `reference.json`**；重新识别位置 B，点击**保存当前检测**，得到新目录中的 `pose.json`。
6. 返回 `ur_pose_vision.py`，继续使用该任务包，在视觉 Tab 选择新的 `pose.json`，点击**校验并导入视觉变化量**。上方输入模式自动切到视觉模式，GUI 自动填写 Base ΔX/ΔY/ΔZ（默认忽略 Z）、ΔYaw，**原工件旋转中心 = 视觉参考中心 Base XY**。
7. 保持原有补偿方式（可选“前 N 秒渐变”或“夹爪闭合前完成”），点击**生成轨迹并预览（暂不保存）**。观察预览、关节跳变和速度等指标。确认无错误后，再勾选 NPZ / PNG，点击**保存所选结果**。
8. 最后在 `GUI_takeover_pv.py` 加载生成的新 NPZ。在完成实际机器人运动学、起点、关节/速度/加速度与碰撞检查后，先以低速进行无接触验证；不要跳过实体安全防护。

> **这里的“绑定”是相机参考快照 + 遥操作轨迹的任务级关联，不是硬件级的 RGB/关节逐帧时间同步。** 你当前的离线轨迹补偿只需要示教时物体参考位姿，而不需要示教过程每帧视觉数据。录制示教期间工件必须保持与参考一致的初始位置，直到实际抓取动作开始。

## 三、任务包与输出目录

```text
teach_records/
  visual_tasks/
    task_20261012_212000_123456_teach_001/
      task.json                    # 关联记录、SHA256 校验
      original.npz                # 原始 NPZ 快照
      reference.json             # 与该 NPZ 对应的视觉参考快照
  shift_outputs/
    original_shift_....npz       # 新的补偿关节轨迹，保留夹爪字段
    original_shift_..._original_3d.png
    original_shift_..._shifted_3d.png
```

新 NPZ 继续保留 `ur_q`, `ur_target_q`, `timestamp`, `gripper_open`, `gripper_target_open`, `rtde_control_frequency_hz` 等原程序字段；新增 `visual_*` 元数据记录示教中心、当前中心、视觉差分、相机序列号以及 JSON SHA256，方便以后追溯。**生成的 NPZ 是否完全兼容实际回放，还需结合当前 `ur_takeover_pv.py` 做完整验证**。

## 四、自动校验条件

程序在读取参考和当前检测时检查：

- 两个结果均在 UR5e **Base 坐标系**（而不是仅相机坐标）；
- 两者的 RealSense 序列号一致；
- 顶面尺寸均为 80 mm，且手眼矩阵一致；
- 当前 `pose.json` 内嵌的参考与任务包参考一致，不能混用不同示教任务；
- 当前检测确认 UR 已静止，PnP RMSE 不超过 3 px；
- 当前位姿数据、预存 ΔXYZ / ΔYaw 和离线重新计算结果一致；
- 顶面 Z 变化超过 12 mm 时拒绝（适用于本次高度不变的工件实验）；
- 任务包 NPZ、参考 JSON 文件的 SHA256 与创建时一致；
- 原版补偿程序规定的每轴 XYZ 偏移 ±10 cm、Yaw ±45°、数值 IK 条件仍然适用。

默认勾选“忽略视觉 ΔZ”，因为你的工件是在相同高度的桌面内平移和自转。程序会显示原始 Z 差值，但是生成轨迹时令 ΔZ=0。若实验确实有 Z 改变，需先确认几何条件、安全高度和视觉标定质量，再考虑取消勾选。

## 五、保留的手动模式

上方选择“手动输入（保留原功能）”即可与原来的 `ur5e_pose_gui.py` 一样，直接输入 XYZ、Yaw，选择手动或自动旋转中心、补偿模式、夹爪事件，并离线生成 NPZ / PNG。视觉模式不改写原始 NPZ。

## 六、限制与验证注意事项

1. 本版**不修改** `square_pose.py` 和 `GUI_takeover_pv.py`，也**不同时启动相机与遥操作示教线程**。前者负责采集视觉参考/当前检测，后者负责实机示教与回放，新版负责绑定、计算和保存。
2. 任务包只能证明两份文件在创建时被关联和校验，**不能自动证明你示教时工件没有移动**。请在创建任务时确认实际工件状态。
3. 当前 IK 依旧沿用原代码的**名义 UR5e DH/tool0 模型**，未纳入机器人出厂标定、活动 TCP、夹爪几何和碰撞模型。离线 IK 残差小不代表实机安全，更不代表抓取精度。
4. 模 90° 的等价 Yaw 只适用于四边相同且抓取接触条件等价的正方形；若不同边或周围障碍限制抓取，不能简单使用等价角。
5. “夹爪闭合自动补偿”实际上依据夹爪开度变化识别闭合命令，它不是检测接触；补偿应在接触前完成。如果原轨迹包含固定位置放置工件的阶段，要考虑该放置段是否需要单独规划。
6. 建议先做零扰动回归（原参考位置不变），再做 +10 mm、+20 mm 和 ±10° 等小幅扰动，先空抓验证，确认视觉差分和回放方向后才接触金属工件。
