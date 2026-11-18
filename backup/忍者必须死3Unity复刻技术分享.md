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
