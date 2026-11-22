&emsp;&emsp;下个春节要出新水忍和水武，这里玩家自制了一个水忍和水武器，尝试使用Unity复刻无尽跑酷，上下绳索，滑铲，多段跳跃，boss攻击，对战，武器，分段忍术等机制，后续进阶可能会加入AI boss战，Lua热更新脚本，多人联机，如果有时间的话...本文会尽可能从小白视角讲解代码和设计思路，但是不会讲解全部的代码，因为那将是巨大的工程且没有意义，但是依旧会解释很多基础的代码写法作用，保证干货满满，如果是同样热爱忍三或者Unity的入门小白，都可以参阅本篇文章，后续完善了会将项目上传供大家参考指教。
&emsp;&emsp;首先是“开始游戏”，游戏的最开始，一个按钮，加载一个场景，代码如下

```csharp
    /// <summary>
    /// 挂在 StartMenu 场景的 StartGameButton 对象上。
    /// UGUI 的 Button 负责识别点击或键盘提交，本组件负责打开游戏场景。
    /// </summary>
    [RequireComponent(typeof(Button))]
    public sealed class StartGameButton : MonoBehaviour
    {
        // 这里填写 Build Settings 中的场景名称。序列化字段允许直接在 Inspector 修改。
        [SerializeField, Tooltip("点击后加载的场景，必须已加入 Build Settings。")]
        private string gameSceneName = "Game";

        private Button button;
        private bool loading;

        private void Awake()
        {
            // 同一个对象上的 Button 由 RequireComponent 保证存在，只需要取一次。
            button = GetComponent<Button>();
        }

        private void OnEnable()
        {
            // 订阅按钮本身的 onClick，鼠标、触屏和 UGUI 的键盘提交都走同一条路径。
            button.onClick.AddListener(EnterGame);
        }

        private void OnDisable()
        {
            // 和 OnEnable 配对。组件再次启用时，每次点击仍只执行一次。
            button.onClick.RemoveListener(EnterGame);
        }

        private void EnterGame()
        {
            // 异步加载尚未结束时，连续点击也只发出一个加载请求。
            if (loading)
                return;

            // 在编辑器直接打开场景可以正常显示，但运行时切换还要求该场景已加入构建列表。
            if (!Application.CanStreamedLevelBeLoaded(gameSceneName))
            {
                Debug.LogError($"无法进入场景 {gameSceneName}，请检查 Build Settings 中的场景名称与启用状态。", this);
                return;
            }

            loading = true;
            button.interactable = false;
            // Single 会卸载开始界面的 Canvas 和 EventSystem，随后显示 Game 场景中的角色。
            SceneManager.LoadSceneAsync(gameSceneName, LoadSceneMode.Single);
        }
    }
```
&emsp;&emsp;summary的这种注释写法一般用于写整个脚本用途，鼠标悬停Inspector可以显示。
&emsp;&emsp;RequireComponent顾名思义，挂载了这个脚本的GameObject必须有这个组件，没有的话Unity自动帮你添加，可以用来防止遗忘，导致空引用报错。
&emsp;&emsp;sealed是密封类，代表这个类不能被别的类继承。这个脚本是专用按钮，不需要继承，所以加 sealed，性能微小提升、防止被意外继承改逻辑。
&emsp;&emsp;[SerializeField]经常用到，private变量不会在Inspector面板中出现并修改，加上[SerializeField]后就可以了，这样做的目的是既想要变量在Inspector面板中可以修改，又不想代码里外随便访问这个变量，而是通过特定的方式比如void Change(loding){loading = true};，即封装思想。
Tooltip是鼠标悬停可以展示的注释，Awake是Unity常见的生命周期函数，在Start之前，物体加载，脚本激活的时候立刻执行。
&emsp;&emsp;OnEnable是脚本/物体变成激活状态的时候执行，相当于挂载了这个脚本的物体一加载到世界场景中，就执行这个函数，OnDisable就是脚本/物体关闭隐藏销毁的时候执行，button.onClick是UGUI按钮的点击事件，AddListener(注册函数)，按钮点击执行EnterGame()，RemoveListener(移除注册)后，按钮再次点击，不再执行EnterGame()，这是规范的UGUI写法，为什么不把注册和移除注册写在Start和OnDestroy里，因为如果这个按钮物体会反复隐藏、显示（比如弹窗），写在 Start 里只会注册一次，隐藏再显示的时候事件可能错乱、多次叠加监听，点一次执行好几次。
&emsp;&emsp;EnterGame就是核心加载逻辑，CanStreamLevelBeLoaded就是检查场景有没有加到Build Settings列表里，Unity顶部菜单栏，File->Build Settings，是打包、管理场景列表的窗口。只有加到这个列表、并且前面打勾的场景，SceneManager.LoadSceneAsync()才能加载成功，编辑器里直接点 Play 运行的时候，哪怕不在列表里，临时也能切；打包成 exe 之后，不在列表里的场景完全找不到，直接报错，添加方法，直接拖拽或者点击Add Open Scenes。
&emsp;&emsp;LoadSceneAsync异步加载场景，不会卡顿主线程，用LoadScene同步加载，大场景会直接卡死画面。
&emsp;&emsp;接下来讲一下管理地图的脚本，地图的实现是按照一定规律组合地面和绳索，在屏幕右边生成，然后向左移动，制造成玩家在向右跑的错觉，由于地图的重复和需要不断出现消失，这里采用对象池优化，因为频繁的创建和销毁是很浪费性能的，并且可能引起GC（垃圾回收）卡顿，比如这里的地面和绳索，反复出现消失，对象池模式就是提前创建好足够数量的对象存起来，要用就拿出来，不用就放回去，开局创建直到游戏结束才销毁，大大提高性能，下面看看重要代码：
这些是部分会用到的变量，有注释可以看看，就不一一解释，后面用到再说，enum是枚举，就是分类地面有哪些种类，比起字符串容易打错从而找不到索引，用枚举更安全方便。
```csharp
        public enum Surface { Ground, Rope }

        [SerializeField, Tooltip("赛道表面在世界坐标中的高度。")]
        private float surfaceY = -1.5f;
        [SerializeField, Tooltip("赛道向左移动的世界单位/秒。")]
        private float speed = 6f;
        [SerializeField, Tooltip("每个区段的长度。起跑安全地面之后，循环生成 1 段地面和 7 段等长绳索，长度比例为 1:7。")]
        private float sectionLength = 14f;
        [SerializeField, Tooltip("角色通常停留的屏幕横坐标，供路段查询。")]
        private float runnerX = -2.4f;
        [SerializeField, Tooltip("向下延伸的岩块 Sprite；顶部可行走边缘与 Surface Y 对齐。")]
        private Sprite groundRockSprite;
        [SerializeField, Tooltip("透明背景的黑色毛边绳索 Sprite。")]
        private Sprite inkRopeSprite;
        [SerializeField, Tooltip("岩块素材的可行走顶边相对图片中心高约 0.13 世界单位，放置时向下抵消。")]
        private float groundTopOffset = 0.13f;
        [SerializeField, Tooltip("绳索素材画布留有透明区域，缩小竖向比例得到参考图中的细绳宽度。")]
        private float ropeVerticalScale = 0.45f;

        private readonly List<Section> sections = new List<Section>();
        // 路段只有 SpriteRenderer 和 Transform，地面/绳索共用一个池，取出时重设完整外观。
        // 上限防止极端长帧或编辑器测试一次创建大量区段后长期占用场景内存。
        private readonly Stack<Section> cachedSections = new Stack<Section>();
        private const int MaxCachedSections = 12;
        private Sprite pixel;
        private float nextX;
        private int sectionNumber;
```
List是动态数组，类似于C++的vector，用来放出现在场景中的路段，readonly只是不允许把sections这个字段重新赋值给另一个List，stack是栈，先进先出，用来放暂时不用的路段，MaxCachedSections 就是对象池最多保存12个闲置路段
```csharp
        private sealed class Section
        {
            public float start;
            public float end;
            public Surface surface;
            public GameObject view;
            public SpriteRenderer renderer;
        }
```
Section是自定义类，包含路段的起点，终点，路段类型（前面的枚举），显示地面的GameObject，和这个对象上的SpiteRenderer，就是面向对象思想，将路段包装成一个有基础属性的对象。
先大致讲一下无尽跑酷的实现原理，首先是开局初始化，在最左侧边界（程序规定），相机外预生成一段路段，然后一直生成到最右侧边界，这里是“陆地”和“绳索”的拼接，为了简便，我就让陆地和绳索1：7的排列出现，生成了的路段会先加入Sections显示出来，然后路段整体向左移动，超过一定界限，最左侧即Section[0]就被回收会对象池，右边同理重新生成新的路段，或者从对象池中拿出来用，对象池有大小限制，刚开始填充，满了以后就不再生成，反复从对象池中存取，而路面的向左移动逻辑，就是在UpDate里更新在世界坐标中的x位置，每个路段生成会有一个start和end的逻辑位置，end = start + length(路段长度)，就是这个路段刚生成在世界中的两端，注意在从生成或者说从对象池中拿出，到放回对象池期间，这个start和end是不会变的，（start + end) / 2是中心坐标，就是防止这个路段的位置，然后维护一个Distance，每个路段的（start + end) / 2 - Distance赋值给新的坐标，Distance不断增加，就形成了路面整体左移的效果，现在来看代码。
```csharp
        private void Awake()
        {
            Speed = speed;
            // 正常情况下使用 Inspector 中的赛道图；只有素材丢失时才退回旧矩形，避免整条路消失。
            if (groundRockSprite == null || inkRopeSprite == null)
            {
                Debug.LogWarning("赛道 Sprite 引用不完整，临时改用纯色线条。", this);
                pixel = Sprite.Create(Texture2D.whiteTexture, new Rect(0, 0, 1, 1),
                    new Vector2(0.5f, 0.5f), 1f);
            }
            nextX = -16f;
            while (nextX < 30f)
                AddSection();
        }
```
nextX就是路段放进显示的场景中最左侧的位置，section.start = nextX，sction.end = nextX + length，这个nextX也是会不断计算变化的，并且是用来判断是否要添加新路段的判断标准，当nextX < 30f了，就超过左边界了，说明路段已经左移动了很多，可以在右端添加新的路段补充了。
```csharp
        private void AddSection()
        {
            // 第 0 段保留 25 单位安全地面，角色出生时前方有约两秒准备时间。
            // 此后按八段一组生成：第 8、16……段为地面，其余七段均为绳索。
            // 七段绳索彼此连续且共享 SurfaceY，角色上下换面不会在内部接缝被误判撞墙；
            // 地面与绳索都采用 sectionLength，因此常规循环的实际行进长度比例也严格是 1:7。
            Surface kind = sectionNumber % 8 == 0 ? Surface.Ground : Surface.Rope;
            float length = sectionNumber == 0 ? 25f : sectionLength;
            // 取得池中已经创建过 SpriteRenderer 的区段；池空时才实例化一次。
            Section section = cachedSections.Count > 0 ? cachedSections.Pop() : CreateSection();
            section.start = nextX;
            section.end = nextX + length;
            section.surface = kind;
            section.view.name = $"{kind}_{sectionNumber:000}";
            section.view.transform.localRotation = Quaternion.identity;
            // 地面图片上半部分透明，下半部分是岩块；用偏移让真正的草石顶边恰在角色脚底。
            float y = kind == Surface.Ground ? surfaceY - groundTopOffset : surfaceY;
            section.view.transform.position = new Vector3((section.start + section.end) * 0.5f - Distance, y, 0f);

            // 每次借用都重设 Sprite、颜色、排序和缩放。若上一轮是绳索而这一轮是地面，
            // 不能留下旧外观；逻辑 start/end 也与新图形在同一处更新。
            SpriteRenderer line = section.renderer;
            Sprite artwork = kind == Surface.Ground ? groundRockSprite : inkRopeSprite;
            line.sprite = artwork != null ? artwork : pixel;
            line.color = artwork != null ? Color.white : new Color(0.055f, 0.065f, 0.09f, 1f);
            line.sortingOrder = 0;
            float width = artwork != null ? artwork.bounds.size.x : 1f;
            float heightScale = artwork != null
                ? (kind == Surface.Ground ? 1f : ropeVerticalScale)
                : (kind == Surface.Ground ? 0.22f : 0.055f);
            section.view.transform.localScale = new Vector3(length / width, heightScale, 1f);
            section.view.SetActive(true);
            sections.Add(section);
            nextX += length;
            sectionNumber++;
        }
```
%8是因为前面说的陆地和绳索1：7出现，加起来取余，余数0就是陆地。
```csharp
        /// <summary>路段完全离开可见范围时调用；关掉渲染和层级活动状态后可安全复用。</summary>
        private void RecycleSection(Section section)
        {
            section.view.SetActive(false);
            if (cachedSections.Count < MaxCachedSections) cachedSections.Push(section);
            else Destroy(section.view);
        }
```
这是回收路段的函数，先把最左端的路段设置为不可见，再放进对象池（栈）中。
下面是Update里的逻辑，每帧更新实现一遍，前面说的让路段左移的Distance，就是在这里更新的值，还有判断左侧路段是否出界需要回收，也是在这里执行。
```csharp
        private void Update()
        {
            // 每帧统一推进世界距离，路段的逻辑边界与可视对象始终使用同一偏移量。
            if (Failed || Completed) return;
            lastDistance = Distance;
            float delta = Speed * Time.deltaTime;
            Distance += delta;
            // 先收回已离开左侧安全边界的路段，再补齐右侧，可在同一帧借用刚归还的对象。
            while (sections.Count > 0 && sections[0].end - Distance < -16f)
            {
               RecycleSection(sections[0]);
               sections.RemoveAt(0);
            }
            while (nextX - Distance < 30f)
            {
                AddSection();
                // 极端长帧可能一次跨过许多整段；刚补出的段若也已落到屏幕左边，
                // 当帧立即归还，让下一段接着复用，避免短暂生成一串无用 GameObject。
                while (sections.Count > 0 && sections[0].end - Distance < -16f)
                {
                    RecycleSection(sections[0]);
                    sections.RemoveAt(0);
                }
            }
            foreach (Section section in sections)
            {
                Vector3 p = section.view.transform.position;
                p.x = (section.start + section.end) * 0.5f - Distance;
                section.view.transform.position = p;
            }
        }
```
但是这里有些可优化的点，现在vibe coding时代很少人会手动写代码了，但是review是必不可少的环节，就像这里的对象池模式，用于显示的sections用的是List类型，每次回收最左端remove()，本案例对象少没什么，但是一旦对象多起来，remove还是很消耗性能的，因为sections.RemoveAt(0)，后面所有的元素都要前移（开销来源），之前的[1]变成[0]，不如使用更合适的队列Queue，先进先出，所以修改下列几处代码：
```csharp
           // 先收回已离开左侧安全边界的路段，再补齐右侧，可在同一帧借用刚归还的对象。
            while (sections.Count > 0 && sections.Peek().end - Distance < -16f)
            {
                RecycleSection(sections.Dequeue());
            }
            while (nextX - Distance < 30f)
            {
                AddSection();
                // 极端长帧可能一次跨过许多整段；刚补出的段若也已落到屏幕左边，
                // 当帧立即归还，让下一段接着复用，避免短暂生成一串无用 GameObject。
                while (sections.Count > 0 && sections.Peek().end - Distance < -16f)
                {
                    RecycleSection(sections.Dequeue());
                }
            }
```
```csharp
        //AddSection()中
        section.view.SetActive(true);
        sections.Enqueue(section);
        nextX += length;
         ectionNumber++;
```
这里有小知识点，在C#中，Stack的放入和取出是push和pop，Queue是Enqueue和Dequeue，只看不取走都是Peek()。
接下来开始EndlessRunner脚本的讲解，就是我们的玩家，依旧先大致讲解一下代码干了啥，心里有数，代码看着不烧心，这个脚本主要负责控制无尽跑酷中的玩家角色行为，核心是通过状态机管理角色的不同动作。首先在 Awake() 中获取角色的 SpriteRenderer、主相机和战斗组件，并将角色放置到赛道规定的初始位置，同时初始化脚印特效管理对象。随后在 Update() 中每帧更新角色状态，首先判断是否处于移动锁定或游戏结束状态，然后根据赛道的加速比例计算角色的目标横向位置，通过 MoveTowards 让角色逐渐向目标位置移动，并调用 EndlessTrack 的 SurfaceAt() 判断脚下是普通地面还是绳索，同时检测岩壁碰撞。如果角色被岩壁阻挡，就根据岩壁的逻辑位置和赛道当前滚动距离计算角色应该停留的位置，直到被挤出屏幕后判定失败。玩家输入主要通过 Up() 和 Down() 两个函数处理，Up() 根据当前状态决定是跳跃、翻回绳索上方还是重新开始游戏，Down() 则根据当前状态决定是滑铲、空中下翻还是翻到绳索下方。角色跳跃时通过垂直速度和重力计算 Y 坐标，下翻时则通过 Lerp 插值在规定时间内下降到赛道表面，绳索上下翻转则通过修改角色状态和 SpriteRenderer 的 flipY 实现。Enter() 是统一的状态切换入口，负责重置动作计时器，并在进入或退出滑铲时通知赛道开启或关闭加速。最后，UpdateFrame() 根据当前状态和计时器切换不同的精灵图片，RunnerTrails 则负责管理跑步时的脚印特效。整体来说，这段代码就是一个以状态机为核心的角色控制系统，EndlessTrack 负责提供赛道信息，EndlessRunner 根据赛道信息和玩家输入控制角色的移动、动作和动画。
```csharp
        [SerializeField, Tooltip("提供脚底高度、路面种类、滚动速度和岩壁位置。")] private EndlessTrack track;
        [SerializeField, Tooltip("跑步循环帧。")] private Sprite[] runFrames;
        [SerializeField, Tooltip("首跳帧。末帧也是越过最高点后的下落姿势。")] private Sprite[] jumpFrames;
        [SerializeField, Tooltip("三张翻滚关键帧，依次收身、倒转、回正。")] private Sprite[] rollFrames;
        [SerializeField, Tooltip("地面滑铲期间保持的贴地姿势。")] private Sprite slideFrame;
        [SerializeField, Tooltip("起跳的世界单位/秒。")] private float jumpSpeed = 9.6f;
        [SerializeField, Tooltip("每秒增加的向下速度。")] private float gravity = 30f;
        [SerializeField, Tooltip("一次滑铲的最长秒数。")] private float slideSeconds = 0.65f;
        [SerializeField, Tooltip("脚底墨迹图。")] private Sprite runInkPuff;
        [SerializeField, Tooltip("两团墨迹的间隔秒数。")] private float puffInterval = 0.11f;
        [SerializeField, Tooltip("每团墨迹渐隐的秒数。")] private float puffLifetime = 0.42f;

        // 空中下翻用 0.16 秒从按键时的高度落向赛道；三帧翻滚用 0.2 秒播完。
        // 两个计时独立，最后一帧姿势可以在触地后短暂显示，新按键仍能立即接管。
        private const float DiveSeconds = 0.16f, RollSeconds = 0.15f;
        // 角色根节点在脚下，向前约 0.8 单位处是倒挂跑姿的头部。
        private const float FrontReach = 0.8f;
        private Motion state;
        private SpriteRenderer figure;
        private Camera view;
        private WaterBattle battle;
        private RunnerTrails trails;
        private float velocityY, elapsed, wallDistance, diveStartY;
        private float rollAge = RollSeconds;
        private int jumpCount;
        private bool restarting;
        // 仅由 NinjutsuCombat 在终结海浪动画开始/结束时切换；锁定期间保持角色的世界坐标。
        private bool ninjutsuMovementLocked;

        public bool IsUnderRope => state == Motion.Under || state == Motion.Pinned;
        public bool IsAirborne => state == Motion.Air || state == Motion.Dive;
        public bool IsSliding => state == Motion.Slide;
        public int JumpCount => jumpCount;
        // HUD 与 Up 共用此条件。只有下一次按键确实能起跳时，才显示跃起的人形图标。
        public bool CanJumpOnUp => !ninjutsuMovementLocked &&
            (state == Motion.Run || state == Motion.Slide || state == Motion.Air) && jumpCount < 4;
        /// <summary>供 HUD 与编辑器验收读取；锁定时键盘和屏幕按钮都不能改变玩家动作或位置。</summary>
        public bool NinjutsuMovementLocked => ninjutsuMovementLocked;
```
上面是会用到的一些变量，注意public bool IsUnderRope => state == Motion.Under || state == Motion.Pinned;类似于这种写法，=>箭头运算符，可以理解为根本没有为IsUnderRope 变量分配内存，每次调用这个变量，实际上是获得右边式子实时运算获得的返回值。
```csharp
        public void SetNinjutsuMovementLocked(bool locked)
        {
            if (ninjutsuMovementLocked == locked) return;
            ninjutsuMovementLocked = locked;
            if (!locked) return;
            if (IsSliding) Enter(Motion.Run);
            if (state == Motion.Pinned) Enter(Motion.Under);
            if (trails != null) trails.ClearFootprints();
        }
```
这是将忍者状态设置为锁定的函数，比如正在滑铲然后进入大招最后的无敌帧无法行动，下翻到绳子下被墙体卡住无法前进。
```chsrap
       // 每次真正切换动作时重置动画时钟。离开滑铲的所有路径共用同一处减速逻辑。
        private void Enter(Motion next)
        {
            if (IsSliding) track.SetSlideBoost(false);
            state = next;
            elapsed = 0f;
            if (IsSliding) track.SetSlideBoost(true);
        }
```
Enter是状态切换函数，假设从滑铲切换别的状态，if (IsSliding) track.SetSlideBoost(false)为真先把加速设置为假，然后state = next;切换为传入的要切换的状态，elapsed用来重置动画钟，新状态计时从零开始。
```csharp
        private void Turn(bool below)
        {
            Enter(below ? Motion.Under : Motion.Run);
            rollAge = 0f;
            velocityY = 0f;
            jumpCount = 0;
            // 绳上方按 ↓ 时贴到下侧；清掉旧侧墨迹，新墨迹按新接触面重新生成。
            transform.position = new Vector3(transform.position.x, track.SurfaceY - (below ? 0.06f : 0f), 0f);
            trails.TurnFootprints(below);
        }
```
Turn(bool below) 是切换绳索上下侧的函数：在绳索上面 ↔ 绳索下面翻转，同时重置一堆运动参数、切换状态、改位置、通知脚印轨迹切换。
```csharp
        private void UpdateFrame()
        {
            // 15 帧/秒播放三个姿势，总计 0.2 秒。到时交回当前跑步或跳跃姿势。
            if (rollAge < RollSeconds - 0.000001f)
                figure.sprite = rollFrames[Mathf.Min(2, (int)(rollAge * 15f + 0.000001f))];
            else if (state == Motion.Pinned) figure.sprite = runFrames[0];
            else if (IsSliding) figure.sprite = slideFrame;
            else if (state == Motion.Air && velocityY <= 0f) figure.sprite = jumpFrames[jumpFrames.Length - 1];
            else if (state == Motion.Air) figure.sprite = jumpFrames[Mathf.Min((int)(elapsed * 12f), jumpFrames.Length - 1)];
            else figure.sprite = runFrames[(int)(elapsed * 15f) % runFrames.Length];
        }
```
UpdateFrame()`专门根据当前状态 + 计时变量，选对应的 Sprite 贴图，是这套角色的动画切换函数。这个函数很重要，乍一看很复杂，其实很简单，这里玩家并没有专门制作跑步动画跳跃动画等，用动画状态机来管理，而是直接代码驱动，将动画帧存在数组里，然后每帧计算出该数组的下标播放对应帧动画，Mathf.Min(2, (int)(rollAge * 15f + 0.000001f))以这个式子为例，Min括号里的取最小值，即封顶，这里些2，是因为数组里只有三个动画帧，下标0，1，2，超过2就越界了，15帧每秒，播放3帧正好0.2秒，rollAge是已经播放了多久的动画，播放的时间乘上每秒播放的帧数，就是这个时间播放了多少帧，对2取余，就是该播放到那一帧（因为数组里只有三帧）。后面的空中上升播放跳跃动画和循环播放跑步动画也是这个原理。