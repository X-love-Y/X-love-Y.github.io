# Unity 无尽跑酷项目开发与代码解析

在新水忍和水武出之前，这里先自制了一个水忍和水武器，同时尝试使用 Unity 复刻忍三的部分玩法，包括无尽奔跑，上下绳索、滑铲、Boss 战、武器、分段忍术等机制。后续进阶可能会加入Lua 热更新脚本、 AI Boss 战、多人联机，如果有时间的话……

本文会尽可能从小白视角讲解代码和设计思路，但不会讲解全部代码，因为那将是一个巨大的工程，而且没有必要。不过依旧会解释很多基础的代码写法及其作用，保证干货满满。如果是同样热爱忍者必须死三或者想要学习Unity ，都可以参考本篇文章，也欢迎提出修改意见和讨论，后续完善了会将demo上传。

---

# 一、开始游戏：场景加载

首先是「开始游戏」。游戏最开始只有一个按钮，点击按钮后加载游戏场景。

代码如下：

```csharp
/// <summary>
/// 挂在 StartMenu 场景的 StartGameButton 对象上。
/// UGUI 的 Button 负责识别点击或键盘提交，本组件负责打开游戏场景。
/// </summary>
[RequireComponent(typeof(Button))]
public sealed class StartGameButton : MonoBehaviour
{
    [SerializeField, Tooltip("点击后加载的场景，必须已加入 Build Settings。")]
    private string gameSceneName = "Game";

    private Button button;
    private bool loading;

    private void Awake()
    {
        button = GetComponent<Button>();
    }

    private void OnEnable()
    {
        button.onClick.AddListener(EnterGame);
    }

    private void OnDisable()
    {
        button.onClick.RemoveListener(EnterGame);
    }

    private void EnterGame()
    {
        if (loading)
            return;

        if (!Application.CanStreamedLevelBeLoaded(gameSceneName))
        {
            Debug.LogError(
                $"无法进入场景 {gameSceneName}，请检查 Build Settings 中的场景名称与启用状态。",
                this);
            return;
        }

        loading = true;
        button.interactable = false;

        SceneManager.LoadSceneAsync(gameSceneName, LoadSceneMode.Single);
    }
}
```

## 1. 基础语法与组件说明

首先看这几个比较常见的写法：

**`<summary>` 注释**

这种 XML 文档注释一般用于说明整个类、函数或者属性的用途。鼠标悬停在代码中的类名或函数名上时，IDE 可以显示对应的注释，方便阅读和维护代码。

**`[RequireComponent(typeof(Button))]`**

顾名思义，挂载了这个脚本的 GameObject 必须拥有 Button 组件。如果没有，Unity 会自动添加，避免忘记添加依赖组件而导致空引用报错。

**`sealed`**

密封类，表示这个类不能被其他类继承。这里的脚本只是一个专用的开始游戏按钮，不需要被继承，所以使用 `sealed`。它主要用于限制继承关系、明确类的设计意图，某些情况下也可能带来微小的优化。

**`[SerializeField]`**

这个特性经常用到。正常情况下，`private` 变量不会显示在 Inspector 面板中，而加上 `[SerializeField]` 后，即使变量是 private，也可以在 Inspector 中查看和修改。

这样既可以让变量在 Inspector 中配置，又不需要将它设置成 public，避免其他脚本随意访问和修改。

这其实就是封装思想：将变量的访问权限控制在合理范围内，需要修改时通过特定的函数或属性进行操作。

**`[Tooltip]`**

鼠标悬停在 Inspector 面板中的变量名称上时，可以显示对应的说明文字，方便配置。

**`Awake()`**

Unity 常见的生命周期函数之一，在脚本实例被加载时执行，通常用于获取组件、初始化变量等。它一般在 `Start()` 之前执行。

## 2. OnEnable、OnDisable 与按钮事件

```csharp
private void Awake()
{
    button = GetComponent<Button>();
}

private void OnEnable()
{
    button.onClick.AddListener(EnterGame);
}

private void OnDisable()
{
    button.onClick.RemoveListener(EnterGame);
}
```

`OnEnable()` 是脚本或 GameObject 变为启用状态时调用的生命周期函数，`OnDisable()` 则是在脚本或 GameObject 被禁用时调用。

`button.onClick` 是 UGUI Button 自带的点击事件。

- `AddListener()`：注册一个监听函数。
- `RemoveListener()`：移除一个监听函数。

这里的意思就是：按钮启用时注册 `EnterGame()`，按钮被禁用时移除监听。

为什么不把注册和移除写在 `Start()` 和 `OnDestroy()` 中？

因为如果按钮所在的 GameObject 会反复隐藏和显示，例如弹窗，那么 `Start()` 只会执行一次，而 `OnEnable()` 和 `OnDisable()` 会随着启用状态反复执行。

使用这两个函数配对管理监听，可以保证每次启用时只注册一次，禁用时及时移除，避免重复注册导致点击一次执行多次的问题。

## 3. EnterGame：场景加载逻辑

```csharp
private void EnterGame()
{
    if (loading)
        return;

    if (!Application.CanStreamedLevelBeLoaded(gameSceneName))
    {
        Debug.LogError("场景无法加载");
        return;
    }

    loading = true;
    button.interactable = false;

    SceneManager.LoadSceneAsync(gameSceneName, LoadSceneMode.Single);
}
```

首先判断 `loading`，这是一个防止重复加载的标记。如果异步加载尚未结束，玩家连续点击按钮也只会发出一个加载请求。

接下来：

```csharp
Application.CanStreamedLevelBeLoaded(gameSceneName)
```

检查指定场景是否可以通过当前加载方式加载。对于普通的项目场景，需要确保它已经加入 Build Settings 中的场景列表并启用。

Unity 顶部菜单栏的 `File -> Build Settings` 可以管理场景列表，新版本 Unity 则可以通过 Build Profiles 管理构建场景。

需要注意，场景直接在编辑器中打开，与运行时通过 `SceneManager.LoadSceneAsync()` 加载是两回事。不能因为编辑器中能够打开场景，就认为运行时一定可以加载。

最后：

```csharp
SceneManager.LoadSceneAsync(gameSceneName, LoadSceneMode.Single);
```

使用异步方式加载场景。

`LoadSceneAsync()` 不会像同步加载那样直接等待整个场景加载完成才继续执行，因此能够减少主线程阻塞。但异步加载不代表完全不会卡顿，资源初始化、场景激活等操作仍可能产生性能开销。

`LoadSceneMode.Single` 表示以单场景模式加载，新场景加载后会卸载原来的场景。

---

# 二、无尽跑酷赛道：EndlessTrack

接下来讲一下管理地图的脚本。

地图的实现原理是按照一定规律组合地面和绳索，在屏幕右边生成，然后整体向左移动，制造出玩家一直向右奔跑的错觉。

由于地图会不断重复出现和消失，这里采用对象池模式进行优化。因为频繁创建和销毁 GameObject 会产生额外的性能开销，还可能造成 GC（垃圾回收）卡顿。

比如这里的地面和绳索，需要反复出现和消失。如果每次都重新 Instantiate 和 Destroy，就会产生大量不必要的开销。

对象池模式就是提前创建一定数量的对象存起来，需要时取出来使用，不需要时放回去，只有对象池容量不足时才创建新对象。这样可以减少频繁创建和销毁对象带来的开销。

## 1. 变量与数据结构

首先看一些会用到的变量：

```csharp
public enum Surface
{
    Ground,
    Rope
}

[SerializeField, Tooltip("赛道表面在世界坐标中的高度。")]
private float surfaceY = -1.5f;

[SerializeField, Tooltip("赛道向左移动的世界单位/秒。")]
private float speed = 6f;

[SerializeField, Tooltip("每个区段的长度。")]
private float sectionLength = 14f;

[SerializeField, Tooltip("角色通常停留的屏幕横坐标。")]
private float runnerX = -2.4f;

private readonly List<Section> sections = new List<Section>();

private readonly Stack<Section> cachedSections = new Stack<Section>();

private const int MaxCachedSections = 12;

private float nextX;
private int sectionNumber;
```

`enum` 是枚举，用来定义一组固定的分类。

这里的 `Surface` 就是将路面分成两种类型：普通地面 Ground 和绳索 Rope。

相比使用字符串判断，枚举更安全，也不容易因为拼写错误而导致判断失败。

`List` 是动态数组，类似于 C++ 中的 `vector`，可以动态添加和删除元素。这里的 `sections` 用来存放当前正在场景中显示的路段。

`readonly` 表示这个字段只能在声明时或者构造时赋值，不能再将它重新指向另一个 List，但仍然可以对 List 内部的元素进行添加和删除。

`Stack` 是栈，遵循后进先出（LIFO），也就是最后放进去的元素最先取出来。这里的 `cachedSections` 用来存放暂时不需要显示的路段。

`MaxCachedSections` 就是对象池最多保存 12 个闲置路段。

## 2. Section：自定义路段类

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

`Section` 是一个自定义类，包含：

- `start`：路段的逻辑起点。
- `end`：路段的逻辑终点。
- `surface`：路段类型，即地面或绳索。
- `view`：负责显示路段的 GameObject。
- `renderer`：这个 GameObject 上的 SpriteRenderer。

这里就是面向对象思想，将一个路段包装成一个拥有自身属性的对象，方便统一管理。

## 3. 无尽跑酷的整体实现原理

先大致讲一下无尽跑酷的实现原理。

首先是开局初始化，在程序规定的最左侧起始位置，预生成一段安全地面，然后继续生成路段，直到覆盖屏幕右侧并留出一定的预生成距离。

为了简便，我让地面和绳索按照 1:7 的规律排列，也就是一段地面后面接七段绳索。

生成的路段会先加入 `sections` 中进行管理，然后路段整体向左移动。当最左侧的路段，也就是 `sections[0]`，超过规定的回收界限后，就将它回收进对象池。右侧则不断生成新的路段，或者从对象池中取出已经创建好的路段重新使用。

对象池有大小限制。刚开始会根据需要创建对象，池子满了以后，如果回收的对象超过容量，就直接销毁多余的对象；否则就反复从对象池中存取，实现重复利用。

而路面的向左移动逻辑，就是在 `Update()` 中更新每个路段在世界坐标中的 X 位置。

每个路段生成时都会记录一个 `start` 和 `end`，其中：

```csharp
end = start + length;
```

它们分别表示这个路段在逻辑世界坐标中的起点和终点。

注意，从路段生成或者从对象池中取出，到再次放回对象池期间，这两个逻辑坐标是不会变化的。只有下一次从对象池取出并重新使用时，才会被赋予新的 `start` 和 `end`。

路段的中心逻辑坐标就是：

```csharp
(start + end) / 2
```

程序另外维护一个 `Distance`，记录赛道整体已经向左滚动的距离。

每帧将路段的中心逻辑坐标减去 `Distance`，得到它当前在世界坐标中的实际显示位置：

```csharp
实际X坐标 = (start + end) / 2 - Distance;
```

随着 `Distance` 不断增加，所有路段的实际 X 坐标就会不断减小，从而形成赛道整体向左移动的效果。

现在来看代码。

## 4. Awake：赛道初始化

```csharp
private void Awake()
{
    Speed = speed;

    if (groundRockSprite == null || inkRopeSprite == null)
    {
        Debug.LogWarning("赛道 Sprite 引用不完整，临时改用纯色线条。", this);

        pixel = Sprite.Create(
            Texture2D.whiteTexture,
            new Rect(0, 0, 1, 1),
            new Vector2(0.5f, 0.5f),
            1f);
    }

    nextX = -16f;

    while (nextX < 30f)
        AddSection();
}
```

这里的 `nextX` 表示下一个待生成路段的起点，也就是当前已生成路段的最右侧边界。

开局将它设置为 `-16f`，然后不断调用 `AddSection()` 生成路段，直到 `nextX` 达到规定的右侧预生成边界。

这里的 `30f` 和 `-16f` 都是人为规定的世界坐标边界，不是自动根据摄像机计算出来的。

另外，如果地面或绳索图片丢失，就创建一个纯白色的 1×1 像素 Sprite，作为临时替代素材，避免赛道完全消失。

## 5. AddSection：生成路段

```csharp
private void AddSection()
{
    Surface kind = sectionNumber % 8 == 0
        ? Surface.Ground
        : Surface.Rope;

    float length = sectionNumber == 0 ? 25f : sectionLength;

    Section section = cachedSections.Count > 0
        ? cachedSections.Pop()
        : CreateSection();

    section.start = nextX;
    section.end = nextX + length;
    section.surface = kind;

    section.view.name = $"{kind}_{sectionNumber:000}";

    section.view.transform.localRotation = Quaternion.identity;

    float y = kind == Surface.Ground
        ? surfaceY - groundTopOffset
        : surfaceY;

    section.view.transform.position = new Vector3(
        (section.start + section.end) * 0.5f - Distance,
        y,
        0f);

    SpriteRenderer line = section.renderer;

    Sprite artwork = kind == Surface.Ground
        ? groundRockSprite
        : inkRopeSprite;

    line.sprite = artwork != null ? artwork : pixel;

    line.color = artwork != null
        ? Color.white
        : new Color(0.055f, 0.065f, 0.09f, 1f);

    line.sortingOrder = 0;

    float width = artwork != null
        ? artwork.bounds.size.x
        : 1f;

    float heightScale = artwork != null
        ? (kind == Surface.Ground ? 1f : ropeVerticalScale)
        : (kind == Surface.Ground ? 0.22f : 0.055f);

    section.view.transform.localScale = new Vector3(
        length / width,
        heightScale,
        1f);

    section.view.SetActive(true);

    sections.Add(section);

    nextX += length;
    sectionNumber++;
}
```

首先看：

```csharp
Surface kind = sectionNumber % 8 == 0
    ? Surface.Ground
    : Surface.Rope;
```

这里使用了取余运算符 `%`。

因为前面说过，地面和绳索按照 1:7 的规律出现，总共八段。

当 `sectionNumber` 除以 8 的余数为 0 时，就生成地面，否则生成绳索。

例如：

- 第 0 段：地面。
- 第 1～7 段：绳索。
- 第 8 段：地面。
- 第 9～15 段：绳索。

另外：

```csharp
float length = sectionNumber == 0 ? 25f : sectionLength;
```

第一段地面长度特殊设置为 25，作为玩家出生时的安全地面，其余路段使用统一长度。

接下来：

```csharp
Section section = cachedSections.Count > 0
    ? cachedSections.Pop()
    : CreateSection();
```

这里使用了三元运算符。

如果对象池中还有闲置路段，就通过 `Pop()` 取出栈顶对象；如果对象池为空，就调用 `CreateSection()` 创建一个新的路段。

随后重新设置这个路段的逻辑起点和终点：

```csharp
section.start = nextX;
section.end = nextX + length;
```

注意，虽然可能是从对象池中取出来的旧对象，但它现在已经被赋予了新的逻辑位置。

然后通过：

```csharp
(section.start + section.end) * 0.5f - Distance
```

计算这个路段当前应该显示的世界坐标 X。

后面则是设置 Sprite、颜色、缩放比例等显示属性。

最后：

```csharp
section.view.SetActive(true);

sections.Add(section);

nextX += length;
sectionNumber++;
```

激活这个 GameObject，将它加入当前显示路段的管理集合，并更新下一个路段的生成起点。

## 6. RecycleSection：回收路段

```csharp
private void RecycleSection(Section section)
{
    section.view.SetActive(false);

    if (cachedSections.Count < MaxCachedSections)
        cachedSections.Push(section);
    else
        Destroy(section.view);
}
```

这是回收路段的函数。

首先将路段设置为不可见，也就是关闭 GameObject。

然后判断对象池是否已经达到最大容量：

- 如果没有达到上限，就使用 `Push()` 将对象放入栈中。
- 如果已经达到上限，就直接 `Destroy()` 销毁这个对象。

这样就避免了对象池无限增长，占用过多内存。

## 7. Update：赛道移动与路段回收

下面是 Update 中的逻辑，每帧执行一次。

前面说过，负责让路段整体左移的 `Distance` 就是在这里更新的，同时也在这里判断左侧路段是否出界，需要回收。

```csharp
private void Update()
{
    if (Failed || Completed)
        return;

    lastDistance = Distance;

    float delta = Speed * Time.deltaTime;
    Distance += delta;

    while (sections.Count > 0 &&
           sections[0].end - Distance < -16f)
    {
        RecycleSection(sections[0]);
        sections.RemoveAt(0);
    }

    while (nextX - Distance < 30f)
    {
        AddSection();

        while (sections.Count > 0 &&
               sections[0].end - Distance < -16f)
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

首先：

```csharp
float delta = Speed * Time.deltaTime;
Distance += delta;
```

`Speed` 是赛道移动速度，`Time.deltaTime` 是上一帧到当前帧经过的时间。

两者相乘得到本帧赛道应该移动的距离，然后累加到 `Distance` 中。

这样就实现了赛道随时间不断向左移动。

接下来：

```csharp
while (sections.Count > 0 &&
       sections[0].end - Distance < -16f)
```

判断最左侧路段是否已经超过回收边界。

这里的：

```csharp
sections[0].end - Distance
```

表示最左侧路段终点当前的实际世界坐标。

如果它小于 `-16f`，就说明整个路段已经超过回收界限，可以回收了。

然后调用 `RecycleSection()` 将它放回对象池，再通过 `RemoveAt(0)` 从当前显示路段集合中移除。

接下来：

```csharp
while (nextX - Distance < 30f)
{
    AddSection();
}
```

判断右侧是否需要补充新的路段。

`nextX - Distance` 表示当前最右侧边界的实际世界坐标。

当它小于 30 时，就说明右侧预生成区域不足，需要继续生成新的路段。

最后：

```csharp
foreach (Section section in sections)
{
    Vector3 p = section.view.transform.position;

    p.x = (section.start + section.end) * 0.5f - Distance;

    section.view.transform.position = p;
}
```

遍历所有当前显示的路段，根据它们各自的逻辑中心位置和当前 `Distance`，重新计算实际世界坐标。

注意，这里只修改 X 坐标，Y 和 Z 坐标保持不变。

这样就完成了整条赛道的滚动。

## 8. 代码优化：使用 Queue 替代 List

这里有一个可以优化的地方。

现在 Vibe Coding 时代，很少有人会完全手动编写所有代码，但是 Code Review 依旧是必不可少的环节。

就像这里的对象管理，当前用于显示的 `sections` 使用的是 `List` 类型。

每次回收最左端路段时：

```csharp
sections.RemoveAt(0);
```

都需要将后面的所有元素向前移动一位。

例如原本：

```text
[0] [1] [2] [3] [4]
```

删除 `[0]` 后：

```text
[1] [2] [3] [4]
```

后面的元素都要移动位置。

虽然本案例中的对象数量不多，性能影响有限，但是如果对象数量增加，这种操作就会产生不必要的开销。

实际上，我们的路段管理具有很明显的先进先出特征：最早生成的路段最先被回收，最新生成的路段位于最右侧。

因此，使用 `Queue`（队列）更加合适。

队列遵循先进先出（FIFO），正好符合这里的使用场景。

首先修改变量声明：

```csharp
private readonly Queue<Section> sections = new Queue<Section>();
```

然后修改 Update 中的回收逻辑：

```csharp
// 回收已经离开左侧安全边界的路段
while (sections.Count > 0 &&
       sections.Peek().end - Distance < -16f)
{
    RecycleSection(sections.Dequeue());
}

// 补充右侧路段
while (nextX - Distance < 30f)
{
    AddSection();

    while (sections.Count > 0 &&
           sections.Peek().end - Distance < -16f)
    {
        RecycleSection(sections.Dequeue());
    }
}
```

最后修改 AddSection 中加入路段的部分：

```csharp
section.view.SetActive(true);

sections.Enqueue(section);

nextX += length;
sectionNumber++;
```

这里有几个小知识点：

| 数据结构 | 放入 | 取出 | 只查看不取出 |
|---|---|---|---|
| Stack | Push | Pop | Peek |
| Queue | Enqueue | Dequeue | Peek |

使用 Queue 后，`Dequeue()` 可以直接移除队首元素，不需要像 List 一样移动后续所有元素。

对于这种不断从队首回收、从队尾添加的场景，Queue 更合适。

---

# 三、玩家角色控制：EndlessRunner

接下来开始 EndlessRunner 脚本的讲解，也就是我们的玩家角色控制器。

依旧先大致讲解一下代码做了什么，心里有数，后面看代码就不会那么烧心。

这个脚本主要负责控制无尽跑酷中的玩家角色行为，核心是通过状态机管理角色的不同动作。

首先在 `Awake()` 中获取角色的 SpriteRenderer、主相机和战斗组件，并将角色放置到赛道规定的初始位置，同时初始化脚印特效管理对象。

随后在 `Update()` 中每帧更新角色状态，首先判断是否处于移动锁定或游戏结束状态，然后根据赛道的加速比例计算角色的目标横向位置，通过 `MoveTowards()` 让角色逐渐向目标位置移动，并调用 EndlessTrack 的 `SurfaceAt()` 判断脚下是普通地面还是绳索，同时检测岩壁碰撞。

如果角色被岩壁阻挡，就根据岩壁的逻辑位置和赛道当前滚动距离计算角色应该停留的位置，直到被挤出屏幕后判定失败。

玩家输入主要通过 `Up()` 和 `Down()` 两个函数处理：

- `Up()`：根据当前状态决定是跳跃、翻回绳索上方还是重新开始游戏。
- `Down()`：根据当前状态决定是滑铲、空中下翻还是翻到绳索下方。

角色跳跃时通过垂直速度和重力计算 Y 坐标，下翻时则通过 Lerp 插值在规定时间内下降到赛道表面，绳索上下翻转则通过修改角色状态和 SpriteRenderer 的 `flipY` 实现。

`Enter()` 是统一的状态切换入口，负责重置动作计时器，并在进入或退出滑铲时通知赛道开启或关闭加速。

最后，`UpdateFrame()` 根据当前状态和计时器切换不同的精灵图片，`RunnerTrails` 则负责管理跑步时的脚印特效。

整体来说，这段代码就是一个以状态机为核心的角色控制系统。`EndlessTrack` 负责提供赛道信息，`EndlessRunner` 根据赛道信息和玩家输入控制角色的移动、动作和动画。

## 1. 变量与状态机

```csharp
[SerializeField, Tooltip("提供脚底高度、路面种类、滚动速度和岩壁位置。")]
private EndlessTrack track;

[SerializeField, Tooltip("跑步循环帧。")]
private Sprite[] runFrames;

[SerializeField, Tooltip("首跳帧。末帧也是越过最高点后的下落姿势。")]
private Sprite[] jumpFrames;

[SerializeField, Tooltip("三张翻滚关键帧，依次收身、倒转、回正。")]
private Sprite[] rollFrames;

[SerializeField, Tooltip("地面滑铲期间保持的贴地姿势。")]
private Sprite slideFrame;

[SerializeField, Tooltip("起跳的世界单位/秒。")]
private float jumpSpeed = 9.6f;

[SerializeField, Tooltip("每秒增加的向下速度。")]
private float gravity = 30f;

[SerializeField, Tooltip("一次滑铲的最长秒数。")]
private float slideSeconds = 0.65f;

private const float DiveSeconds = 0.16f;
private const float RollSeconds = 0.2f;

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
private bool ninjutsuMovementLocked;
```

这里的 `Motion` 是一个枚举，定义了玩家可能处于的不同状态：

```csharp
private enum Motion
{
    Run,
    Slide,
    Air,
    Dive,
    Under,
    Pinned,
    Failed
}
```

这些状态分别代表正常跑步、滑铲、跳跃、空中下翻、绳索下方倒挂、被岩壁卡住以及失败。

再看几个比较有意思的属性：

```csharp
public bool IsUnderRope =>
    state == Motion.Under || state == Motion.Pinned;

public bool IsAirborne =>
    state == Motion.Air || state == Motion.Dive;

public bool IsSliding => state == Motion.Slide;

public int JumpCount => jumpCount;
```

这种 `=>` 写法叫表达式主体成员。

可以理解为没有单独定义一个用于存储 `IsUnderRope` 结果的字段，而是定义了一个只读属性，每次访问时都会执行右侧表达式，并返回计算结果。

例如：

```csharp
public bool IsSliding => state == Motion.Slide;
```

当外部调用：

```csharp
if (runner.IsSliding)
{
    // ...
}
```

实际上就是实时判断当前 `state` 是否等于 `Motion.Slide`。

这样就不需要额外维护一个 `isSliding` 布尔变量，也避免了状态和布尔变量不一致的问题。

## 2. SetNinjutsuMovementLocked：移动锁定

```csharp
public void SetNinjutsuMovementLocked(bool locked)
{
    if (ninjutsuMovementLocked == locked)
        return;

    ninjutsuMovementLocked = locked;

    if (!locked)
        return;

    if (IsSliding)
        Enter(Motion.Run);

    if (state == Motion.Pinned)
        Enter(Motion.Under);

    if (trails != null)
        trails.ClearFootprints();
}
```

这是设置角色移动锁定状态的函数。

例如玩家正在滑铲时，触发大招最后的终结动画，这个期间玩家不能继续移动。

或者角色原本被墙体卡住，现在进入特殊动画，也需要解除卡住状态。

因此，这里在开启锁定时会先清理当前动作：

- 如果正在滑铲，就退出滑铲，关闭赛道加速。
- 如果被岩壁卡住，就恢复到绳索下方状态。
- 清除之前留下的脚印。

之后 Update 会检查 `ninjutsuMovementLocked`，如果为 true，就直接结束本帧更新。

这样就可以保证特殊动画播放期间，普通角色控制逻辑不会干扰它。

## 3. Enter：统一状态切换函数

```csharp
private void Enter(Motion next)
{
    if (IsSliding)
        track.SetSlideBoost(false);

    state = next;
    elapsed = 0f;

    if (IsSliding)
        track.SetSlideBoost(true);
}
```

`Enter()` 是状态切换函数。

假设从滑铲切换到其他状态：

```csharp
if (IsSliding)
    track.SetSlideBoost(false);
```

因为当前状态是 Slide，所以条件成立，先关闭赛道加速。

然后：

```csharp
state = next;
```

将当前状态切换成传入的目标状态。

接着：

```csharp
elapsed = 0f;
```

重置动作计时器，让新状态从零开始计时。

最后再判断一次：

```csharp
if (IsSliding)
    track.SetSlideBoost(true);
```

注意，这里判断的是切换后的新状态。

如果新状态是 Slide，就开启赛道加速；否则不做任何操作。

这样无论通过哪条路径进入或者退出滑铲，都能正确管理赛道加速状态。

## 4. Turn：绳索上下翻转

```csharp
private void Turn(bool below)
{
    Enter(below ? Motion.Under : Motion.Run);

    rollAge = 0f;
    velocityY = 0f;
    jumpCount = 0;

    transform.position = new Vector3(
        transform.position.x,
        track.SurfaceY - (below ? 0.06f : 0f),
        0f);

    trails.TurnFootprints(below);
}
```

`Turn(bool below)` 是切换绳索上下侧的函数。

传入 true，就进入绳索下方；传入 false，就回到绳索上方。

这里使用了三元运算符：

```csharp
below ? Motion.Under : Motion.Run
```

相当于：

```csharp
if (below)
    Enter(Motion.Under);
else
    Enter(Motion.Run);
```

然后重置翻滚动画计时器、垂直速度和跳跃次数。

接着修改角色 Y 坐标：

```csharp
track.SurfaceY - (below ? 0.06f : 0f)
```

如果在绳索下方，就比赛道表面低 0.06 个单位；否则就保持在赛道表面。

最后通知脚印系统切换接触面。

另外，角色真正倒挂的视觉效果是通过 SpriteRenderer 的 `flipY` 实现的，而不是直接旋转整个角色物体。

## 5. UpdateFrame：动画帧切换

```csharp
private void UpdateFrame()
{
    if (rollAge < RollSeconds - 0.000001f)
        figure.sprite = rollFrames[
            Mathf.Min(2, (int)(rollAge * 15f + 0.000001f))];

    else if (state == Motion.Pinned)
        figure.sprite = runFrames[0];

    else if (IsSliding)
        figure.sprite = slideFrame;

    else if (state == Motion.Air && velocityY <= 0f)
        figure.sprite = jumpFrames[jumpFrames.Length - 1];

    else if (state == Motion.Air)
        figure.sprite = jumpFrames[
            Mathf.Min((int)(elapsed * 12f), jumpFrames.Length - 1)];

    else
        figure.sprite = runFrames[
            (int)(elapsed * 15f) % runFrames.Length];
}
```

`UpdateFrame()` 专门根据当前状态和计时变量，选择对应的 Sprite 贴图，是这套角色的动画切换函数。

乍一看比较复杂，其实原理很简单。

这里玩家并没有专门使用 Animator 动画状态机来管理跑步、跳跃、翻滚等动画，而是直接通过代码驱动，将动画帧存在数组里，然后每帧计算数组下标，播放对应帧动画。

例如：

```csharp
Mathf.Min(2, (int)(rollAge * 15f + 0.000001f))
```

这里的 `Mathf.Min()` 是取两个数中的较小值，相当于限制最大值。

因为数组里只有三张动画图片，下标分别是 0、1、2，所以最大不能超过 2，否则就会数组越界。

`rollAge` 是翻滚动画已经播放的时间，乘以 15，就可以计算当前应该播放到第几帧。

例如：

| 播放时间 | 计算结果 | 动画下标 |
|---|---|---|
| 0～0.0667 秒 | 0 | 第 0 帧 |
| 0.0667～0.1333 秒 | 1 | 第 1 帧 |
| 0.1333～0.2 秒 | 2 | 第 2 帧 |

这里的 `RollSeconds` 应设置为：

```csharp
private const float RollSeconds = 0.2f;
```

因为三张图片按照每秒 15 帧播放，总播放时间就是：

```text
3 / 15 = 0.2 秒
```

原代码中如果使用 `0.15f`，就会导致翻滚动画尚未完整播放完三帧，就结束动画计时，因此需要统一修改。

再看最后的跑步动画：

```csharp
runFrames[(int)(elapsed * 15f) % runFrames.Length]
```

这里才是真正使用 `%` 取余。

`elapsed * 15f` 计算已经经过了多少帧，然后对数组长度取余，使下标始终处于合法范围内，从而实现循环播放。

跳跃动画也是类似的原理，只不过使用 12 FPS，并且通过 `Mathf.Min()` 限制最大下标，防止超过数组长度。

---
