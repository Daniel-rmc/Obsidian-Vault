---
created: 2026-01-11
tags:
  - 📝/Research-Idea
  - 🤖/Robotics
  - 🤖/Manipulation
  - 🧠/VLA
  - 🖐️/Tactile
  - 🛠️/LeRobot
  - 📅/IROS
status: 🟢 Planning
project: Visuo-Tactile-Peg-In-Hole
hardware: Realman-RM65, Xsense-Tactile, Realsense
---

# 🚀 Research Plan: Coarse-to-Fine Visuo-Tactile Fusion for Precision Assembly

## 1. 核心理念 (Core Idea)

**目标 (Goal):** 解决精密装配（Peg-in-the-hole）任务中，最后紧密装配时刻，末端执行器和物体遮挡视觉，导致状态估计失效的问题。
**方法 (Method):** 模仿人类操作逻辑，采用 **"Coarse-to-Fine"** (由粗到细) 的分层策略：
1.  **Coarse:** 利用 [[VLA]] (Vision-Language-Action) 模型进行语义理解和长程规划，到达操作空间。
2.  **Fine:** 利用 **视触觉融合 (Visuo-Tactile Fusion)** 的 [[Diffusion Policy]] 处理局部遮挡下的微调与插入。

> [!TIP] Innovation Point (IROS 投稿卖点)
> 传统的 VLA 缺乏几何精度，传统的 BC (Behavior Cloning) 缺乏泛化且受限于遮挡。本工作提出了一种 **多模态切换架构**，首次在 [[LeRobot]] 框架下验证了触觉图像 (Tactile Image) 在 Diffusion Policy 中的显式引入能显著提升遮挡下的装配成功率。


## 2. 系统架构 (System Architecture)

### 2.1 整体控制流

```mermaid
graph TD
    A[User Instruction: 'Insert cylinder to hole'] --> B(Global Planner: VLA Model);
    B -->|Output: Coarse Pose / Hover| C{Distance < Threshold?};
    C -- No --> B;
    C -- Yes/Occluded --> D(Local Controller: Tactile-Diffusion);
    D -->|Input: RGB + Tactile + Force| E[Action Execution];
    E --> F{Contact/Success?};
    F -- No --> D;
    F -- Yes --> G[Task Done
```

### 2.2 模型设计 (Modified Diffusion Policy)

基于 [[LeRobot]] 框架进行魔改，核心在于 **Multi-Modal Encoder** 的设计。

- **输入空间 (Observation Space):**
    
    - $I_{rgb}$: 外部/腕部相机 (Realsense)
        
    - $I_{tactile}$: 视触觉传感器图像 (Xsense style)
        
    - $F_{force}$: 6-DoF 力/力矩数据：来自触觉传感器解算
        
    - $q_{proprio}$: 机械臂关节状态
        
- **网络结构:**
    
    - **Vision Enc:** ResNet-18 / ViT
        
    - **Tactile Enc:** EfficientNet-B0 (专门提取纹理与形变特征)
        
    - **Fusion:** Concat -> MLP -> Diffusion Head (U-Net)
## 3. 实验计划 (Experiments)

### 3.1 任务设定

- **场景:** 桌面级 Peg-in-the-hole (圆柱、方块等不同形状)。
    
- **难点:** 设置强遮挡场景 (手完全挡住孔)，且孔位存在 $\pm 5mm$ 的随机误差。
    

### 3.2 对比基线 (Baselines)

1. **Vision-Only (BC/Diffusion):** 仅使用 RGB。预期结果：在遮挡时刻容易插歪或产生剧烈碰撞。
    
2. **Blind-Tactile:** 仅使用触觉。预期结果：搜索效率极低，难以找到初始孔位。
    
3. **Ours (CTF-VT):** 视觉引导 + 触觉修正。
    

### 3.3 评估指标 (Metrics)

- **Success Rate (SR):** 成功插入的概率。
    
- **Time to Completion:** 完成时间。

## 4. 实施路线图 (Roadmap)

- [ ] **Phase 1: 基础环境搭建**
    
    - [ ] 跑通 [[LeRobot]] 标准流程 (Realsense + Realman)。
        
    - [ ] 验证 Realman 的控制频率是否满足 Diffusion Policy 要求 (10Hz+)。
        
- [ ] **Phase 2: 数据流同步 (关键)**
    
    - [ ] 编写 Python 脚本，同步采集 `RGB` + `Tactile Image` + `Force` + `Robot State`。
        
    - [ ] 解决时间戳对齐问题。
        
- [ ] **Phase 3: 算法开发**
    
    - [ ] 修改 LeRobot `Dataset` 类，增加触觉图像加载器。
        
    - [ ] 修改 LeRobot `Config`，设计多模态 Encoder 融合层。
        
- [ ] **Phase 4: 数据采集与训练**
    
    - [ ] 遥操作采集 50-100 条 demo (包含失败-修正的数据)。
        
    - [ ] 训练与真机部署调试。
        
- [ ] **Phase 5: 论文撰写**
    
    - [ ] 整理实验数据，撰写 IROS 草稿。