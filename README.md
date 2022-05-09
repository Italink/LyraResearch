[TOC]

# Lyra

> - 推荐打开方式：Typora （Github主题）> Gitlab >  PDF
> - 推荐阅读方式：略读【工程简析】->跟着【关卡解析】自定义关卡->结合源码，再回头看工程简析。

## 工程简析

把Lyra的大致流程简单过了一遍，架构中比较深刻的是：

- Lrya通过覆盖UE默认的**引擎类**，为射击游戏提供了专门的事件及属性插槽，GameFeature中实现射击游戏的核心逻辑，并以插件的形式制作游戏的扩展，最终挂载到游戏主逻辑上。

### 默认类覆盖

在Editor中，可以在项目设置中看到如下默认选项：

![image-20220509120036310](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509120036310.png)

上述配置对应`Lyra/Config/DefaultEngine.ini`的条目：

![image-20220505115701979](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505115701979.png)

该配置文件将影响引擎工程的构建，UE在执行时会根据配置文件的覆盖原有的引擎类，Lyra中通过覆盖这些引擎默认类来实现自己的项目配置，以GameMode为例：

> **LyraGameMode**的构造函数中绑定了各个状态对应的元类型（StaticClass），之后将根据这些绑定的**StaticClass**使用函数`SpawnActor`创建相应的实例。

![image-20220505121609346](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505121609346.png)

### SubSystem

简单点说，SubSystem就是官方推荐的单例方案，相比传统的C++单例，它主要有以下好处：

- 依附在引擎中的已有的单例上（比如GameInstance，Engine，Editor等），SubSystem的生命周期由其同步和维护。

- 无需修改引擎代码，UE预留了接口，在引擎执行时，会根据反射信息得到上述单例的派生类（**DerivedClass**）的元对象（**StaticClass**），创建所有的SubSystem，而SubSystem通过引擎提供的**事件插槽**进行构造并提供接口。

  > UE中很多结构都体现了这种插件式的架构思路

#### 用法及原理

使用时只需继承自对应的SubSystem即可，其中UE支持Subsystem的类有：

- Engine：`UEngineSubsystem`
- Editor：`UEditorSubsystem`
- GameInstance：`UGameInstanceSubsystem`
- World：`UWorldSubsystem`
- LocalPlayer：`ULocalPlayerSubsystem`

以**UEngineSubsystem**为例：

```C++
UCLASS()
class MySubsystem : public UEngineSubsystem{
    GENERATED_BODY()
}
```

而在类**UEngine**的定义中，拥有成员变量：

```c++
TUniqueObj<FSubsystemCollection<UEngineSubsystem>> EngineSubsystemCollection;
```

在函数`UEngine::Init()`中，将会调用：

```C++
EngineSubsystemCollection->Initialize(this);
```

其中该函数的实现如下：

![image-20220506095125643](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506095125643.png)

##### 解析

对于支持Subsystem的类，都具有成员变量**FSubsystemCollection**，在类初始化时，将调用函数**FSubsystemCollectionBase::Initialize**，该函数会根据反射信息，创建所有Subsytem的子类。

#### 何时使用

官方的说法是：如果你觉得需要一个Manager，那么这就是使用SubSystem的时机。

#### Lyra使用点

- **UCommonSessionSubsystem** : public UGameInstanceSubsystem

  > 处理托管和加入在线游戏的请求。  

- **UCommonUserSubsystem** : public UGameInstanceSubsystem

  > 处理查询和更改用户身份和登录状态。   

- **UGameplayMessageSubsystem** : public UGameInstanceSubsystem

  > 该系统允许事件引发器和侦听器注册消息，而不必直接了解对方，尽管它们必须就消息的格式(作为USTRUCT()类型)达成一致。  

- **ULyraAudioMixEffectsSubsystem** : public UWorldSubsystem

  > 该子系统旨在自动参与默认和用户控制总线混合，以检索以前保存的用户设置，并将它们应用到激活的用户混合。此外，该子系统将根据用户对HDR音频的偏好自动应用HDR/LDR音频Submix效果链覆盖。Submix效果链覆盖在天琴座音频设置中定义。

- **ULyraContextEffectsSubsystem** : public UWorldSubsystem

- **ULyraExperienceManager** : public UEngineSubsystem

  > 主要用于多个PIE会话之间的仲裁

- **ULyraGamePhaseSubsystem** : public UWorldSubsystem

- **ULyraGlobalAbilitySystem** : public UWorldSubsystem

- **ULyraLoadingScreenSubsystem** : public UGameInstanceSubsystem

  > 用于显示和隐藏Loading界面

- **ULyraPerformanceStatSubsystem** : public UGameInstanceSubsystem

  > 子系统允许访问性能统计数据以进行显示

- **ULyraTeamSubsystem** : public UWorldSubsystem

  > 用于方便地访问基于团队的参与者的团队信息(例如角色状态)

- **UGameUIManagerSubsystem** : public UGameInstanceSubsystem

  - **ULyraUIManagerSubsystem** : public UGameUIManagerSubsystem

- **UCommonMessagingSubsystem** : public ULocalPlayerSubsystem

  - **ULyraUIMessaging** : public UCommonMessagingSubsystem

- **UPocketCaptureSubsystem** : public UWorldSubsystem

- **UPocketLevelSubsystem** : public UWorldSubsystem

- **USubtitleDisplaySubsystem** : public UGameInstanceSubsystem

- **UUIExtensionSubsystem** : public UWorldSubsystem

### PrimaryDataAsset

Lyra中大量使用C++派生**UPrimaryDataAsset**并公开特定的Property，然后在编辑器中派生蓝图进行配置，以供内部C++使用。

#### Lyra使用点

- **ULyraAbilitySet**：定义技能集合，供Lyra内置的GameplayAbilitySystem使用。
- **ULyraAimSensitivityData**：定义一组对浮点值的手柄灵敏度
- **ULyraExperienceDefinition**：定义Lyra的GameFeature数据，包括插件列表，Action操作等
- **ULyraExperienceActionSet**：存放具有关联性的Actions
- **ULyraGameData**：定义全局游戏数据
- **ULyraLobbyBackground**：定义Lyra中的加载背景（关卡）
- **ULyraPawnData**：默认的角色数据，其中包括：角色类的指定，技能集合，标签映射，输入配置，相机模式。定义如下：
- **ULyraUserFacingExperienceDefinition**：用于在UI中显示体验并开始新会话的设置描述

> 详细的使用过程请仔细查阅下方的**ULyraExperienceDefinition**

### GameFeature

> 提前阅读：
>
> https://www.bilibili.com/video/BV1dL4y1h7YW?spm_id_from=333.337.search-card.all.click
>
> https://www.zhihu.com/column/insideue4

#### GameFeaturePolicy

Lyra中项目配置中覆盖了GameFeaturePolicy，用于追踪游戏中的内置及外部插件(例如，通过web服务或其他终端)。

![image-20220509121115404](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509121115404.png)

#### ExperienceDefinition

Lyra中覆盖了引擎的**WorldSetting**，并新增了**ULyraExperienceDefinition**属性，用于配置**GameFeature**及相关的行为

![image-20220505142206317](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505142206317.png)

其中**ULyraExperienceDefinition**的定义如下：

![image-20220509140822267](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509140822267.png)

##### 属性说明

- **GameFeaturesToEnable**：需要开启的GameFeature（名称数组）

- **DefaultPawnData**：默认的角色数据，其中包括：角色类的指定，技能集合，标签映射，输入配置，相机模式。定义如下：

  > ![image-20220509141437986](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509141437986.png)

- **Actions**：单个元素可以是**UGameFeatureAction**的子类，在Lyra的目录`Lyra\Source\LyraGame\GameFeatures`中，派生了许多Action

  > ![image-20220509141958466](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509141958466.png)

- **ActionSet**：单个元素为  具有关联性的一组Action（包含GameFeature），Lyra中通过使用蓝图派生**ULyraExperienceActionSet**来进行配置

  > ![image-20220509142443796](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509142443796.png)

##### 配置面板示例

![image-20220509142300804](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509142300804.png)

#### ULyraExperienceManager

该类仅仅是为了在编辑器模式下处理多个PIE会话

#### ULyraExperienceManagerComponent

**ULyraExperienceDefinition**做数据的定义，**ULyraExperienceManagerComponent**才是真正管理GameFeature的角色

##### 入口点

ULyraExperienceManagerComponent的创建及管理位于**ALyraGameState**中

![image-20220509145029083](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509145029083.png)

##### Experience加载时机

由**ALyraGameMode**负责加载

![image-20220509145330702](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509145330702.png)

##### 连锁操作

![image-20220509143328446](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509143328446.png)

###### 开启GameFeature

![image-20220509144043232](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509144043232.png)

###### 执行Actions

![image-20220509144348154](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509144348154.png)

###### Pawn设置

![image-20220509154801020](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509154801020.png)

### GameplayAbilitySystem

> 请务必提前阅读：
>
> - https://docs.unrealengine.com/5.0/en-US/gameplay-ability-system-for-unreal-engine/
> - [[玩转UE4/UE5动画系统＞技能系统（GAS）篇] 二 技能 Gameplay Ability（GA）](https://zhuanlan.zhihu.com/p/425001766)
> - [[玩转UE4/UE5动画系统＞技能系统（GAS）篇] 三 影响 Gameplay Effect（GE）](https://zhuanlan.zhihu.com/p/425165414)

#### ULyraGlobalAbilitySystem

- **作用**：监控所有的**ULyraAbilitySystemComponent**（**下文简称ASC**），并对**全体ASC**的**Ability**或**Effect**进行设置。

​	![image-20220509111822600](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509111822600.png)

- **解析**：上面的代码可以看出**ULyraGlobalAbilitySystem**使用了一种常见的对象监控管理手段：

  > 在对象（创建/初始化/激活）时添加到全局的管理器中（注意添加时会应用当前管理器的设置），在对象（销毁/卸载/休眠）时，从全局管理器中移除，这样可以通过全局管理器对其中的所有对象进行统一操作。

  很明显，**RegisterASC**和**UnregisterASC**将由**ULyraAbilitySystemComponent**在恰当时机调用，这两个函数主要修改的目标是成员变量**RegisteredASCs**

#### ULyraAbilitySystemComponent

> **能力系统组件**( `UAbilitySystemComponent`) 是演员和**游戏能力系统**之间的桥梁。任何打算与 Gameplay 能力系统交互的 Actor 都需要自己的能力系统组件，或访问另一个 Actor 拥有的能力系统组件。
>
> 参阅：https://docs.unrealengine.com/5.0/en-US/gameplay-ability-system-component-and-gameplay-attributes-in-unreal-engine/

Lyra中也是覆盖默认的**UAbilitySystemComponent**做了扩展实现。

在源码中拥有**ULyraAbilitySystemComponent**的类型有：

- **ALyraGameState**

- **ALyraPlayerState**

- **ALyraCharacterWithAbilities**

  > **特别注意**
  >
  > 虽然**ALyraCharacter**包括了**ULyraPawnExtensionComponent**，它里面有**ULyraAbilitySystemComponent***，但是值为`nullptr`，需要调用函数`ULyraPawnExtensionComponent::InitializeAbilitySystem(ULyraAbilitySystemComponent*, AActor*)`对其进行赋值，Lyra中使用的Character蓝图为**B_Hero_ShooterMannequin**：
  >
  > ![image-20220509161305436](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509161305436.png)
  >
  > 它还包括了**ULyraHeroComponent**，其中就包含了以下操作，使用**ALyraPlayerState**中的**ASC**对**PawnExtComp**的**ASC**初始化：
  >
  > ![image-20220509161547272](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509161547272.png)

#### ULyraAbilitySet

> 数据资产

![image-20220509162401672](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220509162401672.png)

#### Waiting...

## 关卡解析

### L_LyraFrontEnd

> 路径为：`Lyra\Content\System\FrontEnd\Maps\L_LyraFrontEnd.umap`

#### Experience

- Lyra覆盖了引擎的**WorldSetting**，并新增了**ULyraExperienceDefinition**属性，用于管理**GameFeature**，在当前关卡，它的值为**B_LyraFrontEnd_Experience**

  > 其路径为`Game/System/FrontEnd/B_LyraFrontEnd_Experience`

  ![image-20220505142206317](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505142206317.png)

- 其中**B_LyraFrontEnd_Experience**继承自C++类**ULyraExperienceDefinition**：

  ![image-20220505142506991](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505142506991.png)

- **B_LyraFrontEnd_Experience**中具有以下的**Actions**，它们会在程序开始时执行对应操作（比如**Add Components**，**Add Widgets**...）

  ![image-20220505142714580](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505142714580.png)

#### 初始菜单

![image-20220505153110723](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505153110723.png)

- 界面中的背景由蓝图`Lyra/Content/Environments/B_LoadRandomLobbyBackground`提供：

  > 其本质是加载关卡作为背景，其中加载的关卡为：`Lyra/Plugins/GameFeatures/ShooterMaps/Content/Maps/L_ShooterFrontendBackground.umap`

  ![image-20220505152250626](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505152250626.png)

- 初始界面的前置菜单由蓝图类`/Game/UI/B_LyraFrontendStateComponent`提供，它继承自`Lyra/Source/LyraGame/UI/Frontend/LyraFrontendStateComponent`

  ![image-20220505144004138](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505144004138.png)

- UI文件位于：

![image-20220505143906787](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505143906787.png)

- 源码中会依次加载UI

  ![image-20220505154916352](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505154916352.png)

- 菜单中的按钮对应如下事件

  ![image-20220505160332174](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505160332174.png)

- 单击按钮**StartGame** 将跳转到界面`Lyra/Content/UI/Menu/Experiences/W_ExperienceSelectionScreen`

![image-20220505161024831](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505161024831.png)

- 点击事件如下：

  ![image-20220505165550940](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505165550940.png)

- 其中加载游戏的主要操作在节点**Quick Play Session**中完成，主要运行步骤如下：

  > 该节点由`Lyra\Plugins\CommonUser\Source\CommonUser\UCommonSessionSubsystem`提供

  - 执行`UCommonSessionSubsystem::QuickPlaySession()`，查找Session。

    ![image-20220505175115001](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505175115001.png)

  - 查找结束将调用`UCommonSessionSubsystem::HandleQuickPlaySearchFinished()`，如果找到Session则加入，否则创建Session：

    ![image-20220505175632834](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505175632834.png)

  - 对于`UCommonSessionSubsystem::HostSession()`，将执行以下逻辑，默认情况下会走**CreateOnlineSessionInternal(LocalPlayer, Request)**的分支

    ![image-20220505180027731](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505180027731.png)

  - 其中`CreateOnlineSessionInternal()`会对PendingTravelURL赋值，并创建Session![image-20220505180226209](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505180226209.png)

    > 当前PendingTravelURL的值是：L"/ShooterMaps/Maps/L_Expanse?listen?Experience=B_ShooterGame_Elimination"

  - Session创建完毕将调用以下函数，通过**GetWorld()->ServerTravel(PendingTravelURL);**切换场景。

    ![image-20220505180506185](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505180506185.png)

#### 加载页面

切换场景时，**ULoadingScreenManager**（public UGameInstanceSubsystem）的Tick函数会验证是否要显示LoadingScreen，上述的逻辑将导致以下分支：

![image-20220505180930881](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505180930881.png)

- 其中bCurrentlyInLoadMap状态设置主要通过以下函数委托：

  ![image-20220505183320017](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505183320017.png)

- 显示加载场景的逻辑如下：

  ![image-20220506094423293](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506094423293.png)

- 最终，切换场景时将显示以下界面：

  ![image-20220505181151231](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220505181151231.png)



###  L_Convolution_Blockout

![image-20220506095742172](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506095742172.png)

> 关卡蓝图中，仅有一个附加粒子的逻辑，且并未生效

<img src="https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506102203001.png" alt="image-20220506102203001" style="zoom: 67%;" />

#### Experience

该关卡的Experience为**B_LyraShooterGame_ControlPoints**

> 其路径为`Lyra/Plugins/GameFeatures/ShooterCore/Content/Experiences/B_LyraShooterGame_ControlPoints`

![image-20220506100821885](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506100821885.png)

![image-20220506110608113](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506110608113.png)

> 可以从上面的Actions看出场景运行时将加载：
>
> - UI
> - ControlPointScoring：控制点计分机制
> - PickRandomCharacter：随机角色（男性模型或女性模型）
> - TeamSetup_TwoTeams：队伍**生成**机制（该组件的作用是分为红蓝两队）
> - TeamSpawningRules：队伍**出生**机制
> - MusicManagerComponent：音频管理组件
> - ShooterBotSpawner：控制机器人的生成

#### 控制点计分(ControlPointScoring)

> 该关卡的游戏方式是：占领控制点，控制点多的队伍会持续加分，当分数累加到一定程度时，该队伍获胜。

蓝图**B_ControlPointScoring**的逻辑如下

> 位于`Lyra/Plugins/GameFeatures/ShooterCore/Content/ControlPoint/B_ControlPointScoring`

![image-20220506113908951](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506113908951.png)

- 事件说明：
  - **事件开始运行**：开启一个名为Scroing的定时器。
  - **GameStart** （游戏开始后执行）：根据游戏人数来确定获胜所需的分数。
  - **CapturePoint**（占领控制点后执行）：根据控制点及当前控制点中的角色(0)获取到对应的**TeamId**，修改**ControlPointOwnerTeams**的值为对应的**ID**，并更新响应的Tag。
  - **RegisterControlPoint**（注册（创建）控制点时执行）：将控制点添加到数组**Control Points**中，并将**ControlPointOwnerTeams**对应**index**的元素置为**-1**表示中立。
  - **Scroing**（由上方的定时器触发）：用于定时去更新当前的比分，并判断游戏的胜利条件，结束时进行结算。

##### UI 

![image-20220506110742078](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506110742078.png)

##### 控制点体积（ControlPointVolume）

> 此Actor蓝图位于`Lyra/Plugins/GameFeatures/ShooterCore/Content/Blueprint/B_ControlPointVolume`，

<img src="https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506135622290.png" alt="image-20220506135622290" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506140012978.png" alt="image-20220506140012978" style="zoom:50%;" />

- 事件说明：
  - **事件开始运行**：触发事件**RegisterControlPoint**
  - **组件开始重叠时（Cube）**：触发事件**RecomputeContest**及**EnterAudio**
  - **组件结束重叠时（Cube）**：触发事件**RecomputeContest**及**ExitAudio**
  - **RecomputeContest**：将当前覆盖控制点体积的所有Actor的TeamID加入（**AddUnique**）到**OverlappingTeams**，如果只有一个Team，则开始占领控制点，此时会开启一个时间轴，并设置材质及Niagara粒子的颜色。

#### 随机角色（PickRandomCharacter）

该模块的作用是：随机生成男性(Manny)或女性(Quinn)角色。

![image-20220506180327848](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506180327848.png)

> 该蓝图位于：Lyra/Content/Characters/Cosmetics/B_PickRandomCharacter

![image-20220506180122117](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506180122117.png)

其中C++类主要在BeginPlay时做如下操作：

![image-20220506181917671](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506181917671.png)

![image-20220506181843267](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506181843267.png)

> 上述代码的作用是在**BeginPlay**时，将当前的Pawn，或者之后生成的Pawn，应用该蓝图的设置

#### 团队生成（TeamSetup_TwoTeams）

Lyra中默认使用**B_TeamSetup_TwoTeams**来定义队伍规模，其蓝图参数为：

> 位于`Lyra/Plugins/GameFeatures/ShooterCore/Content/Game/B_TeamSetup_TwoTeams`

![image-20220506173923752](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506173923752.png)

**B_TeamSetup_TwoTeams**继承自C++类**ULyraTeamCreationComponent**，其主要逻辑如下：

> 位于`Lyra\Source\LyraGame\Teams\ULyraTeamCreationComponent.h`

![image-20220506174332136](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220506174332136.png)

> 可以看出该代码的作用是：当创建Experience或BeginPlay时，在服务器上根据参数**TeamsToCreate**去生成队伍。

#### 出生机制（TeamSpawningRules）

> 该组件用于控制如何在出生点  生成 团队角色

蓝图**B_TeamSpawningRules**的继承关系是

**B_TeamSpawningRules**->

**UTDM_PlayerSpawningManagmentComponent(C++)**->

**ULyraPlayerSpawningManagerComponent(C++)**

其中主要的操作是：

![image-20220507100211731](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507100211731.png)

> 上述代码的逻辑是：
>
> - 加载关卡时，把Level中的所有**ALyraPlayerStart**存起来
> - 在World中生成Actor时，如果是**ALyraPlayerStart**，就存起来
> - 把当前World中的所有**ALyraPlayerStart**存起来

该组件提供了接口**ChoosePlayerStart**，该接口将根据所有的**ALyraPlayerStart**挑选Player的出生点

![image-20220507101139822](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507101139822.png)

其中挑选逻辑的位于函数**OnChoosePlayerStart()**中，该函数为虚函数，其中子类**UTDM_PlayerSpawningManagmentComponent**的实现为：

![image-20220507101614873](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507101614873.png)

需要注意的是接口**ChoosePlayerStart**，将由**ALyraGameMode::ChoosePlayerStart_Implementation()**调用，它又是由**GameModeBase**提供的接口：

![image-20220507101830041](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507101830041.png)

![image-20220507101949081](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507101949081.png)

在Lyra中，它的触发时机主要是C++内部调用**AGameModeBase::RestartPlayer(AController* NewPlayer)**，部分用例如下：

> ![image-20220507104208939](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507104208939.png)
>
> ![image-20220507110024604](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507110024604.png)
>
> ![image-20220507110129090](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507110129090.png)

#### AI生成（ShooterBotSpawner）

蓝图**B_ShooterBotSpawner**有以下参数：

![image-20220507110842131](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507110842131.png)

> 可以看出上面的参数指定了：
>
> - 机器人生成的数量
> - 机器人类
> - 机器人随机名称池

其C++类中的挂载操作为：

![image-20220507111131912](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507111131912.png)

> 可以看出，代码的逻辑是：加载Experience后在服务器上创建对应数量的机器人

### 尝试自定义射击关卡

#### 新增关卡

- 在如下目录新建Level——**L_MyLevel**

  ![image-20220507142823768](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507142823768.png)

- 搭建基础场景

  ![image-20220507111927969](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507111927969.png)

- 用网格简单搭建关卡地形，这里简单加了个地板，中间加了个立方体

  ![image-20220507112825731](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507112825731.png)

- 在地图的四个角落放置**LyraPlayerStart**

  > 注意不是普通的**PlayerStart**，否则Lyra覆盖的WorldSetting将报错：
  >
  > ![image-20220507113706034](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507113706034.png)

  ![image-20220507113831548](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507113831548.png)

#### 测试Experience

- 打开世界场景设置，做如下設置

  ![image-20220507115535741](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507115535741.png)

  > 如果下拉选项没有，可在如下位置找到：
  >
  > ![image-20220507115605886](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507115605886.png)

- 启动游戏，能看到如下画面：

  ![image-20220507115109300](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507115109300.png)

#### 自定义Experience

- 在文件夹**Plugins\ShooterCore\Experiences**下新建**蓝图类**，继承自C++类**LyraExperienceDefinition**，命名为**B_MyExperience**

  ![image-20220507115919189](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507115919189.png)

  ![image-20220507133650264](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507133650264.png)

- 构建**MyExperience**

  - 添加GameFeature—ShooterCore

    ![image-20220507120253124](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507120253124.png)

  - 设置Pawn Data

    ![image-20220507133722695](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507133722695.png)

  - 设置Action Set

    ![image-20220507133819408](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507133819408.png)

  - 设置Experience的加载操作

    - 添加计分板UI

      > 绑定到HUD中

      ![image-20220507134505054](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507134505054.png)

    - 添加技能

      > 包括角色的血量，伤害，复活机制

      ![image-20220507153319244](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507153319244.png)

    - 添加组件

      - 加入音频管理组件

        ![image-20220507135802602](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507135802602.png)

      - 加入控制点计分组件

        ![image-20220507140508845](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507140508845.png)

      - 加入出生点控制组件

        ![image-20220507140530307](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507140530307.png)

      - 加入团队分组组件

        - 新建蓝图类（派生于C++类**LyraTeamCreationComponent**），命名为**B_TeamCreationComponent**

          ![image-20220507141507089](![](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507141507089.png)

        - 打开蓝图进行编辑，添加两个队伍（Lyra的游戏机制只允许有两个队伍），且注意标签为1,2

          ![image-20220507160118344](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507160118344.png)

        - 在MyExperience中添加该组件

          ![image-20220507160228368](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507160228368.png)

      - 自定义角色模型生成组件

        - 新建蓝图类**B_MyCharactorParts**，派生于C++类

          ![image-20220507143804926](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507143804926.png)

        - 添加Part

          ![image-20220507144208864](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507144208864.png)

        - 将该组件加入到Experience中

          ![image-20220507150327968](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507150327968.png)

      - 加入AI生成器—**B_ShooterBotSpawner_ControlPoint**

        ![image-20220507200659364](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507200659364.png)

      - 配置完毕

        ![image-20220507160449263](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507160449263.png)

#### 游戏性道具

- 在场景中加入三个控制点

  ![image-20220507161005418](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507161005418.png)

- 加入补血包

  ![image-20220507164841529](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507164841529.png)

- 加入武器

  ![image-20220507164957163](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507164957163.png)

- 加入跳跃点并设置其高度

  ![image-20220507163300383](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507163300383.png)

- 加入多个传送门，并指定其传送目标

  ![image-20220507163409377](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507163409377.png)

#### AI寻路

加入导航体积，并包裹住场景

![image-20220507165139999](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507165139999.png)

![image-20220507165401681](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507165401681.png)

#### 掉落自毁

![image-20220507181043998](https://cdn.jsdelivr.net/gh/Italink/BlogImage/image-20220507181043998.png)

#### 大功告成

![debug](https://cdn.jsdelivr.net/gh/Italink/BlogImage/debug.gif)

## 游戏机制

> Waiting

## 美术效果

> Waiting













