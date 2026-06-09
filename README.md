# Rehab AI — RK3588 智能康复训练交互系统

基于 Rockchip RK3588 边缘 AI 平台的实时康复训练系统。通过摄像头捕捉人体姿态，利用 NPU 加速的 YOLOv8s-Pose 模型做关键点检测，再结合 DTW 动态时间规整算法评估动作规范性，最后用 TTS 语音实时给出纠正指导。

---

## 功能

- **实时姿态检测** — YOLOv8s-Pose 在 RK3588 NPU（6 TOPS）上跑推理，输出 17 个 COCO 关键点。
- **关节角度计算** — 膝、肘、髋关节角度实时算出并在画面中可视化。
- **动作评估** — DTW 算法把用户膝角时序曲线和标准模板对比，输出 0~1 之间的评分。
- **语音反馈** — TTS 实时播报纠正建议，支持 espeak-ng / festival / gTTS 后端。
- **LLM 评语（可选）** — 可接 Qwen2.5 0.5B（RKLLM 版）生成智能康复指导。
- **分屏 UI** — 左侧标准参考姿态，右侧用户实时姿态，骨骼点和 HUD 叠加显示。

---

## 环境要求

### 桌面调试

- CMake ≥ 3.14
- C++17 编译器
- OpenCV ≥ 4.x（需要 highgui、imgproc、videoio）
- 摄像头（USB 或内置均可）

### 板端部署

- RK3588 开发板（Firefly / Orange Pi 5 / Rock 5 等）
- RKNN Runtime（`librknnrt.so`）
- RKLLM Runtime（`librkllm.so`，可选）
- aarch64 交叉编译工具链

---

## 编译

```bash
# 桌面模式（无 NPU，模拟关键点，适合 UI 调试）
cmake -B build && cmake --build build

# 板端模式（交叉编译）
cmake -B build -DRK3588=ON \
    -DCMAKE_TOOLCHAIN_FILE=/path/to/toolchain.cmake
cmake --build build
```

编译产物：

- `rehab_app` — 主程序
- `test_init` — 模块冒烟测试

---

## 运行

```bash
# 默认动作 m01，摄像头 0
./rehab_app

# 指定动作和摄像头
./rehab_app -a m02 -c 1

# 启用 LLM 评语
./rehab_app -a m01 --llm

# 禁用语音
./rehab_app --no-audio

# 查看全部选项
./rehab_app -h
```

### 键盘快捷键

| 键 | 功能 |
|----|------|
| ESC / Q | 退出程序 |
| 1 ~ 0 | 切换动作（m01 ~ m10）|
| R | 重置关节角度历史 |
| S | 截图保存 |

---

## 架构

三个线程各司其职：

- **主线程**：摄像头采集 → NPU 推理 → 骨骼绘制 → UI 渲染
- **评估线程**：DTW 评分 + 关节偏差分析
- **语音线程**：接收 TTS 反馈队列并播报

---

## 项目结构

```text
├── include/
│   ├── app_config.h       # 集中配置常量
│   ├── cv_pipeline.h      # 视觉管线接口
│   ├── audio_engine.h     # TTS 语音引擎接口
│   └── llm_pipeline.h     # LLM 评语生成接口
├── src/
│   ├── main.cpp           # 入口，三线程调度
│   ├── cv_pipeline.cpp    # NPU 推理 + 后处理 + 骨骼绘制 + DTW
│   ├── audio_engine.cpp   # TTS 后端检测与播放
│   └── llm_pipeline.cpp   # 规则引擎 + RKLLM 评语生成
├── test/test_init.cpp     # 模块冒烟测试（8 项）
├── models/                # RKNN 模型文件
├── templates/             # 动作模板 JSON
├── board/                 # 板端 RGA/DRM 渲染支持
├── 3rdparty/json.hpp      # nlohmann JSON
└── CMakeLists.txt
```

---

## 动作模板格式

以 `templates/m01_golden.json` 为例：

```json
{
  "action": "deep_squat",
  "angle_sequence": [90.0, 88.5, 85.2, ...],
  "description": "标准深蹲膝角序列"
}
```

---

## 测试

```bash
./test_init
```

会跑 8 项冒烟测试：配置常量、角度计算、CV 管线初始化、帧处理、DTW 评分、音频引擎、LLM 管道、参考姿态渲染。如果哪一项挂了，日志会直接报出来。

---

## 团队竞赛 Demo · 康复 AI 系统 v1.0
