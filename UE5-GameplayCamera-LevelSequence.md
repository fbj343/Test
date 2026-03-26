# 虚幻引擎 GameplayCameraComponent 在 LevelSequence 中的使用与过渡

## 技术分享来源

### 主要演讲
**《The Future of Gameplay Cinematic Authoring in Unreal Engine》 | Unreal Fest 2024**

- 🎬 YouTube 视频：https://www.youtube.com/watch?v=f6-KXQKYrYE
- 💬 官方论坛讨论：https://forums.unrealengine.com/t/talks-and-demos-the-future-of-gameplay-cinematic-authoring-in-unreal-engine-unreal-fest-2024/2343347
- 📚 所有 Unreal Fest Seattle 2024 演讲：https://dev.epicgames.com/community/learning/paths/Mk1/unreal-engine-fortnite-unreal-fest-seattle-2024-talks-and-demos

该演讲聚焦于 UE 5.5 中 Sequencer 的改进，展示了 Gameplay 摄像机与过场动画之间更流畅的过渡方式，涵盖：
- 游戏摄像机与影视摄像机之间的平滑混合
- 动态绑定（根据玩家选择/游戏状态响应的过场动画）
- 多平台缩放支持

### 社区教程
**《UE 5.6 - Gameplay Camera Integration in Sequencer: 2-Camera System Test》**

- 🎬 YouTube：https://www.youtube.com/watch?v=JhWfSVwBelQ
- 📖 Epic Dev 社区：https://dev.epicgames.com/community/learning/tutorials/9X8z/unreal-engine-ue-5-6-gameplay-camera-integration-in-sequencer-2-camera-system-test-subtitled

---

## GameplayCameraComponent 简介

UE 5.3 起，Epic 强化了 Gameplay 摄像机系统，在 UE 5.5/5.6 中成为功能更完善的实验性插件。与传统蓝图摄像机组件相比，它具有：

- **数据驱动**：摄像机行为封装为可复用的资产（Camera Asset）
- **模块化组合**：通过 Camera Rig、Camera Director、Variable Collection、State Tree 等模块灵活组合
- **更好的 Sequencer 集成**：可将 Gameplay Camera Rig 直接用于 Sequencer
- **自洽运行**：每个激活的 GameplayCameraComponent 运行自己的摄像机系统（UE 5.6 弃用了 CameraSystemActor）

### 核心概念

| 概念 | 说明 |
|------|------|
| Camera Asset | 摄像机资产，定义 Rig、过渡等行为 |
| Camera Rig | 摄像机装备，定义具体的摄像机运动/行为逻辑 |
| Camera Director | 摄像机导演，控制哪个摄像机处于激活状态（蓝图或 State Tree） |
| Camera Transition | 摄像机过渡，定义在不同 Rig 之间切换时的混合方式 |
| Variable Collection | 变量集合，供 Rig 节点读取的参数（如目标 Actor、偏移量等） |

---

## 在 LevelSequence 中使用 GameplayCameraComponent

### 方式一：在 Sequencer 中直接使用 Camera Rig Actor

1. 启用 **Gameplay Cameras** 插件（编辑 → 插件 → 搜索 "Gameplay Cameras"）
2. 在关卡中放置 `GameplayCameraRigActor`，或将 `GameplayCameraComponent` 挂载在角色/Actor 上
3. 在 Sequencer 中，将该 Actor 添加为 Track（Track → Actor to Sequencer）
4. 使用 **Camera Cuts Track** 切换到对应的 Camera Rig 视角
5. 在 Camera Cuts Track 中启用 **Can Blend** 选项，并拖动 Blend 标记来设置过渡时间

### 方式二：Camera Cuts Track 的 Can Blend 功能

Sequencer 的 Camera Cuts Track 支持平滑混合过渡（非硬切）：

1. 在 Sequencer 中，右键点击 Camera Cut Track
2. 勾选 **Can Blend**
3. 在时间轴的切入/切出点会出现滑块，拖动以设置过渡帧数
4. 这样可以实现从玩家视角平滑切入影视摄像机，避免视角突变

---

## 从 LevelSequence 过渡回 GameplayCameraComponent

### 核心思路

在 LevelSequence 播放结束时，从 Sequence 中的 CineCamera 提取最后一帧的 Transform/FOV 数据，同步到 GameplayCameraComponent，然后触发过渡。

### 蓝图实现

```
LevelSequence 结束事件
    └──► 读取 Sequence 摄像机的最终 Transform
    └──► Set View Target with Blend（混合到 Gameplay 摄像机）
    └──► 恢复玩家控制输入
```

**关键蓝图节点：**
- `Set View Target with Blend`：平滑切换视角目标，可设置混合时间和插值曲线（Linear / EaseIn / EaseOut / EaseInOut 等）
- `Get Sequence Camera`：获取 Level Sequence 中当前激活的摄像机
- `Activate Camera For Player Controller`：激活 GameplayCameraComponent 中指定的 Camera Rig

### C++ 实现（提取 Sequence 最终帧 Transform）

```cpp
bool UMyFunctionLibrary::ExtractTransformFromSequenceAtSeconds(
    UMovieSceneSequence* MovieSceneSequence,
    float Seconds,
    FName BindingTag,
    FTransform& OutTransform)
{
    if (!MovieSceneSequence) return false;

    UMovieScene* MovieScene = MovieSceneSequence->GetMovieScene();
    // 1. 找到对应 Tag 的 Binding
    // 2. 找到 Transform Track 和 Section
    // 3. 根据 Seconds 计算 Frame Number
    // 4. 使用 FMovieSceneDoubleChannel 读取 Location/Rotation/Scale
    // 5. 写入 OutTransform

    return true;
}
```

完整参考实现：
- 知乎教程：https://zhuanlan.zhihu.com/p/1939842538358940796
- CSDN 教程：https://blog.csdn.net/2504_91981494/article/details/150453986

### 实现流程总结

```
[LevelSequence 播片]
        │
        │（即将结束时，提前 0.5~1 秒）
        ▼
[提取 Sequence 摄像机当前 Transform/FOV]
        │
        ▼
[将参数写入 GameplayCameraComponent 的初始位姿]
        │
        ▼
[Set View Target with Blend → 混合时长 0.5~1 秒]
        │
        ▼
[恢复玩家输入控制]
        │
        ▼
[GameplayCameraComponent 完全接管]
```

---

## 注意事项（UE 5.5/5.6/5.7）

- **UE 5.6 重要变更**：`CameraSystemActor` 已弃用。GameplayCameraComponent 现在各自运行独立的摄像机系统，与现有 Engine 功能（如 Pawn View Target、关卡流式加载）更兼容。
- **UE 5.6 Camera Rig 变更**：Camera Rig 成为独立资产，Actor 自动绑定逻辑简化。
- **API 演进**：GameplayCameraSystem 仍处于"实验性"阶段，建议关注每个版本的发布说明。
- **FOV 和后处理同步**：过渡时需要同步 FOV、景深、曝光等后处理参数，否则切换时视觉会有明显断裂感。
- **动画同步**：角色动画也需要配合过渡。建议使用带 Slot 的动画蓝图，在 Gameplay 和 Sequencer 控制之间做动画混合（参见官方文档：Blending Gameplay and Sequencer Animation）。

---

## 相关资源

### 官方文档
- [Gameplay 摄像机系统概览（中文）](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/gameplay-camera-system)
- [Gameplay Camera System Overview（英文）](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-camera-system-overview)
- [Gameplay Camera System Quick Start](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-camera-system-quick-start)
- [Blending Gameplay and Sequencer Animation](https://dev.epicgames.com/documentation/en-us/unreal-engine/blend-gameplay-animation-to-cinematic-animation-in-unreal-engine)
- [Creating Camera Cuts Using Sequencer](https://dev.epicgames.com/documentation/en-us/unreal-engine/creating-camera-cuts-using-sequencer-in-unreal-engine)
- [Cinematic Workflow Guides and Examples](https://dev.epicgames.com/documentation/en-us/unreal-engine/cinematic-workflow-guides-and-examples-in-unreal-engine)

### 视频教程
- [Unreal Fest 2024 演讲：The Future of Gameplay Cinematic Authoring](https://www.youtube.com/watch?v=f6-KXQKYrYE)
- [UE 5.6 Gameplay Camera Integration in Sequencer](https://www.youtube.com/watch?v=JhWfSVwBelQ)
- [Blend the Player/Gameplay Camera with a Level Sequence](https://www.youtube.com/watch?v=I2-lXl4e-Lc)
- [How To Make Cinematics Transition To Gameplay Seamlessly](https://www.youtube.com/watch?v=SO2SqE_OueU)
- [Transition from GAMEPLAY CAMERA to CINEMATIC](https://www.youtube.com/watch?v=mmv-Wg1BxFo)
- [B 站：UE5 新相机系统讲解与演示](https://www.bilibili.com/video/BV1nYZAYcEXY/)

### 中文技术文章
- [知乎：UE教程 LevelSequence/GameplayCamera 实现播片到游戏的摄像机过渡](https://zhuanlan.zhihu.com/p/1939842538358940796)
- [CSDN：LevelSequence/GameplayCamera 实现播片到游戏的摄像机过渡（含源码）](https://blog.csdn.net/2504_91981494/article/details/150453986)
- [UE5 GamePlayCameras 基础框架（知乎）](https://zhuanlan.zhihu.com/p/24572592190)

### 社区讨论
- [Gameplay Cameras: Intended setup for transitioning between focused actors](https://forums.unrealengine.com/t/gameplay-cameras-intended-setup-for-transitioning-between-focused-actors/2642545)
- [Community Tutorial: How to Transition from GAMEPLAY CAMERA to CINEMATIC](https://forums.unrealengine.com/t/community-tutorial-how-to-transition-from-gameplay-camera-to-cinematic/1351091)
- [About GameplayCamera Design（设计讨论）](https://forums.unrealengine.com/t/about-gameplaycamera-design/2692591)
- [UE5 Gameplay Cameras: Upgrading to 5.6 (英文博客)](https://ludovic.chabant.com/blog/2025/06/06/ue5-gameplay-cameras-upgrading-to-5-6/)
