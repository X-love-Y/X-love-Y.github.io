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
&emsp;&emsp;<summary>的这种注释写法一般用于写整个脚本用途，鼠标悬停Inspector可以显示。
&emsp;&emsp;RequireComponent顾名思义，挂载了这个脚本的GameObject必须有这个组件，没有的话Unity自动帮你添加，可以用来防止遗忘，导致空引用报错。
&emsp;&emsp;sealed是密封类，代表这个类不能被别的类继承。这个脚本是专用按钮，不需要继承，所以加 sealed，性能微小提升、防止被意外继承改逻辑。
&emsp;&emsp;[SerializeField]经常用到，private变量不会在Inspector面板中出现并修改，加上[SerializeField]后就可以了，这样做的目的是既想要变量在Inspector面板中可以修改，又不想代码里外随便访问这个变量，而是通过特定的方式比如void Change(loding){loading = true};，即封装思想。
Tooltip是鼠标悬停可以展示的注释，Awake是Unity常见的生命周期函数，在Start之前，物体加载，脚本激活的时候立刻执行。
&emsp;&emsp;OnEnable是脚本/物体变成激活状态的时候执行，相当于挂载了这个脚本的物体一加载到世界场景中，就执行这个函数，OnDisable就是脚本/物体关闭隐藏销毁的时候执行，button.onClick是UGUI按钮的点击事件，AddListener(注册函数)，按钮点击执行EnterGame()，RemoveListener(移除注册)后，按钮再次点击，不再执行EnterGame()，这是规范的UGUI写法，为什么不把注册和移除注册写在Start和OnDestroy里，因为如果这个按钮物体会反复隐藏、显示（比如弹窗），写在 Start 里只会注册一次，隐藏再显示的时候事件可能错乱、多次叠加监听，点一次执行好几次。
&emsp;&emsp;EnterGame就是核心加载逻辑，CanStreamLevelBeLoaded就是检查场景有没有加到Build Settings列表里，Unity顶部菜单栏，File->Build Settings，是打包、管理场景列表的窗口。只有加到这个列表、并且前面打勾的场景，SceneManager.LoadSceneAsync()才能加载成功，编辑器里直接点 Play 运行的时候，哪怕不在列表里，临时也能切；打包成 exe 之后，不在列表里的场景完全找不到，直接报错，添加方法，直接拖拽或者点击Add Open Scenes。
    LoadSceneAsync异步加载场景，不会卡顿主线程，用LoadScene同步加载，大场景会直接卡死画面。