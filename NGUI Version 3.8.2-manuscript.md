# NGUI Version 3.8.2（Overlay模式）
## 配置 NGUI Camera
- 创建一个摄像机
- 设置`Target Display`和主摄像机一样，以渲染到同一设备
- 设置`Depth`比主摄像机大，使UI元素覆盖在场景之上
- 设置`Projection`为`Orthographic`，因为Overlay模式下不需要透视效果
- 设置`Clear Flags`为`Depth only`，以在空白区域显示主摄像机的渲染画面（即场景）
- 设置`Culling Mask`为UI元素所在的层，使相机只渲染UI元素

## 配置 NGUI Atlas
图集将一个Sprite集合打包进一个纹理中，使得不同Widget可以使用相同的纹理，最好将同一个Panel下的Widget使用的Sprite打包进同一个图集，有利于合批操作。


## 渲染 NGUI Camera
ps: UI元素的顶点数据包括position、uv、color等顶点属性

### UIWidget
UIWidget继承自MonoBehaviour，NGUI创建的UI元素会挂载UIWidget的子类，如UILabel、UISprite，他们决定了要渲染什么。但是UI元素并不会挂载Render组件，因为渲染操作不是由UI元素自己控制的。\
UIWidget缓存了它需要渲染的UI元素的顶点数据\
UIWidget保存了用于确认渲染顺序的depth，depth越小，越先渲染。\
UIWidget定义了抽象方法OnFill()，子类需要实现它以填充UIWidget缓存的顶点数据。\
UIWidget定义了方法UpdateGeometry()，用来触发OnFill()的调用。

### UIDrawCall
UIDrawCall继承自MonoBehaviour，负责批量渲染UI元素。\
UIDrawCall缓存了其需要绘制的UI元素合批后的顶点数据。\
UIDrawCall所在游戏对象会挂载另外两个组件：MeshFilter和MeshRender。\
UIDrawCall定义了方法UpdateGeometry()，将缓存的顶点数据设置到MeshFilter.mesh中，从而批量渲染UI元素。\

### UIPanel
UIPanel继承自MonoBehaviour，负责其下UIWidget的合批操作。\
UIPanel定义了脚本生命周期函数LateUpdate()，行为如下：
1. 调用其下所有的UIWidget的UpdateGeometry()
2. 对其下所有的UIWidget按depth升序进行排序
3. 遍历已排序的UIWidget列表，按照合批规则创建UIDrawCall并将同一批UIWidget缓存的顶点数据填充到对应的UIDrawCall缓存的顶点数据，规则如下：
    - 对第一个UIWidget，创建一个新的UIDrawCall，并将UIWidget缓存的顶点数据填充到这个UIDrawCall缓存的顶点数据中
    - 对第二个UIWidget，判断其使用的材质、着色器和纹理是不是和第一个UIWidget都相同。如果都相同，则将缓存的顶点数据填充到第一个UIWidget使用的UIDrawCall的缓存的顶点数据中。如果不都相同，则创建一个新的UIDrawCall，并将缓存的顶点数据填充到这个新的UIDrawCall缓存的顶点数据中。
    - 对于第三个UIWidget，需要和第二个UIWidget做对比，相同则使用第二个UIWidget使用的UIDrawCall
    - 对于接下来的每个UIWidget，以此类推
4. 当按规则创建好所有的UIDrawCall后，调用这些UIDrawCall的UpdateGeometry()
5. 给所有的UIDrawCall设置RenderQueue

### Unity内置渲染管线
相机深度 -> 排序层 -> 层内顺序 -> 渲染队列

在NGUI创建好合批的mesh后，Unity的内置渲染管线会对场景中启用的每个摄像机进行渲染。摄像机的Depth决定了摄像机的渲染先后顺序，值越小越先渲染，后渲染的图像会覆盖先渲染的图像，因此要确保负责渲染UI的摄像机的Depth要大于渲染场景的摄像机的Depth。
对于Camera的渲染来说，场景中的mesh也有渲染顺序，因为UI Camera渲染的时UI层，我们所有的UI元素都在这个层，所以我们可以控制UI元素的渲染顺序。
- 同一UIDrawCall下的UI元素按UIWidget的Depth从小到大填充到mesh中，因此它们可以正确遮挡。
- 所有的UIPanel可能会创建很多UIDrawCall，影响它们的渲染顺序的配置主要是render的Sorting Layer、render的Sorting Order和材质的Render Queue，UIPanel在创建UIDrawCall时会为它们设置这些属性
    - render的Sorting Layer的优先级最高，值越小越先渲染，但是NGUI没有在Inspector面板值给出编辑方式，所以默认为Default
    - render的Sorting Order优先级次之，值越小越先渲染，使用UIPanel的Inspector面板展示的Sort Order来配置
    - 材质的Render Queue最后判断，值越小越先渲染。使用UIPanel在Inspector面板展示的Render Q来配置，有三种模式可以选择，用来定义由这个Panel创建的UIDrawCall的RenderQueue：
        - Automatic：这个模式的递增值会被所有的UIPanel共享，创建的第1个UIDrawCall的值为3000，创建的第2个UIDrawCall的值为3001，以此类推。可以通过UIPanel的Depth来控制UIPanel创建UIDrawCall的先后顺序，值越小越先创建，RenderQueue就越小，也就越先渲染
        - Start At：和Automatic一样递增分配，只是初始值可以指定
        - Explicit：创建的每个UIDrawCall的值都是一样的，需要指定一个值

## NGUI Tween (NGUI过程动画)
UITweener继承自MonoBehaviour，是一个使用模板方法模式实现的抽象类，定义了根据配置（时长、动画曲线、延迟时间）进行数值采样的算法流程，具体的根据数值进行属性更新的操作由子类实现，如TweenColor的实现为Color.Lerp(from, to, factor);

## Unity静态合批
### 构建时合批
- 在Player项目设置启用静态合批
- 在对象的Static Flag 启用 Batching Static
### 运行时合批
- 调用StaticBatchingUtility.Combine传入游戏对象，为静态合批做好准备，适合程序生成的对象

## Unity动态合批
- 在Player项目设置启用动态合批\
对满足条件的网格，Unity会自动进行动态批处理，动态合批会在CPU上进行顶点转换，开销可能大于DrawCall，可能会对性能产生影响
