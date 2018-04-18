---
sidebar: auto
---

# wepy与mpvue对比分析

## wepy的坑点

### 蛋疼的组件体系
由于wepy诞生之初小程序还没有原生组件, 所以其组件体系是完全自己实现的.
坑点如下:
* 一个页面内声明为同名的组件是同一个实例, 共享一切. 意味着复用组件必须用不同的名字. 在实例对象下可以看到组件的变量名为`$$组件名$$变量名`
* 循环渲染需要使用自造的repeat标签及其语法, 才能勉强做到将data分离, 但computed依然是所有同名组件实例共用的, 其值为最后一次计算结果.
* 因为组件根本就是嵌入父层统一渲染的, 所以css scope什么的, 自己通过类名去实现吧...
* 辣鸡的传参:
  * 传参如果要穿对象只能直接传对象, 而不能传对象的子对象! 不能有任何运算甚至是点操作符和[]! 非循环渲染情况下还可以通过computed处理一下, 循环中这种情况基本无解, 要么再造一个新数组.
  * 蜜汁奇怪的绑定方式, 据我记得不写.sync一样可以父绑定到子...还有组件传参的twoWay参数, 很鸡肋的存在.
  * 这里动态传参语法仿照vue使用:prop, 但由于并没有vue里最常用的:class和:style的实现, 所以其实我常常压根想不起来这里居然是用冒号.

::: tip mpvue的自定义组件
* 也未使用原生自定义组件, 而是template([#305](https://github.com/Meituan-Dianping/mpvue/issues/305)), 所以也无法在组件外层绑定class或事件.
* 没有wepy的传参和computed这些问题. css scope也有.
:::

### 事件体系
事件的模仿感觉比较糟糕, 魔改的痕迹明显.
坑点如下:
* 挺不人性化的事件语法. 组件里emit出来的事件外部在捕获时一定要加.user修饰符. 事件既可以在构造器的events里面声明同名函数, 也可以在标签上绑定methods里声明的处理函数.
* 自定义参数十二分蛋疼. 记得有一次我要额外传个布尔值, 但怎么写都获取不到, 最后成功的语法好像是这样写的`@tap="handleTap(_, {{!0}})"`. 没错, 虽然双大括号里理论上来说能写true和false, 但实践下来只能这样写能拿到正确的值. 只有别问我前面那个参数是个啥玩意, 我只知道那地方写什么都能获取到原事件. 按它官方手册上写的方式...反正我是没成功.
* 作者似乎对原生事件的捕获和阻止冒泡有啥[误解](https://tencent.github.io/wepy/document.html#/?id=%E7%BB%84%E4%BB%B6%E8%87%AA%E5%AE%9A%E4%B9%89%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86%E5%87%BD%E6%95%B0), .stop修饰符绑定catch事件, 作用的确是阻止冒泡, 但却和事件的捕获阶段capture没啥关系.

::: tip mpvue的事件
* 没有wepy的这些坑, 一切都很vue.
* 强制使用@, 如果要使用原生事件, 则无法绑定处理函数, 只能使用诸如capture-catch:touchmove这样不需要处理的事件. 原因应该是, 事件是绑定在vnode上的, 中间有一层代理, 写在vue构造参数对象顶层的自定义方法其实会被丢弃掉, 而不会加入page或component的构造对象.
:::

::: warning mpvue关于事件的一些下个版本会填掉的小坑
* onShareAppMessage是每个page都有的, 写在源码里的无法去除.
* 暂时不支持capture捕获阶段的事件.
:::

### 数据绑定
wepy主打的feature, 初衷是提供类似vue那样直接赋值来修改数据并更新视图, 避免原生提供的setData方式. 但很不幸...在异步流程里直接修改是无效的, 需要再手动调用`this.$apply()`以更新视图, 并且这样做会和setData起冲突, 使之前的修改无效.
据我分析是wepy在实例初始化的时候深拷贝了一份初始data对象, 合并到实例对象下, 使得可以通过`this.abc`直接去访问数据(而非`this.data.abc`), 然后实际维护的是这份data, 在生命周期或事件方法中修改data后会自动setData更新真正被使用的数据及视图. 但在异步流程中修改无效, 可见并未如vue一样采用defineProperty绑定set描述符的方式.

::: tip mpvue的数据绑定
* 理所应当没有问题.
:::

### 问题根源
[核心贡献者数量太少了](https://github.com/Tencent/wepy/graphs/contributors), 几乎就是个从零开始的个人项目. 2月6号至今只更新了一个补丁版本. 应该是在憋大招.

# 除以上对比之外mpvue的可赞之处
* 文档里详细说明了**什么不能做**, 各种地方都有**踩坑注意**, 这点特别暖! 因为是脱胎自vue, 部分功能受限于小程序无法实现, 都会有一一说明. 光从文档这点就比wepy不知道高到哪里去了.
* app或页面参数options, 原生小程序只能从onLoad函数里获取, wepy对此未做工作. mpvue将其集成至根实例, 可以全局访问.
* 现成的ide插件支持, vue全家桶的snippet和提示已经相当成熟. 小程序因为只有中国人才用所以...
* 可以直接写html标签, 编译器会转译成对应小程序标签并带上类名, 这点对于小程序和web复用很重要.
::: tip 标签转化
source | target
---|---
`<div>`|`<view class="_div">`  
`<span>`|`<label class="_span">`  
`<a>`|`<navigator class="_a">`  
`<view>`|`<view class="_view">`
`<custom-tag>`|`<custom-tag class="_custom-tag">`
:::

## 小程序本身的无法避免的坑点

### 开发者工具
**这个模拟器里最良心的可能是它的代码编辑器...**  
这年头还用nwjs的都应该看看隔壁微软vs code...
* 不太稳定, 偶尔黑屏or白屏.
* 特定时间经常连不上wx自家的服务器.
* 莫名的模拟器专属真机没有的bug.

### 明明能做好但就是不做好气死你系列
* 双大括号里只能进行简单运算, 不能调用正常函数, 想用函数? wxs了解一下?  
::: tip wxs
这个东西懒得吐槽. 魔改阉割版es5. 不是跑在正常js环境里的, 目测是运行在原生渲染层, 因此和正常的js代码无法有任何互相作用.
:::
* css选择器支持太孱弱了, 很多地方都有限制, 只有简单的类选择器是保险的.
* image组件并不是html标准的img标签, 它不是图片本身, 它是图片的容器, 并且有默认宽高.
* 不支持PATCH方法, 用PUT替代.
* 标签里的属性明明可以写数组, 动态类名挺有用的, 但文档里只字不提.

### 跨平台统一性并不完善
* ios上还是基于wkwebkit, 因此ios8会是个大坑, css要前缀, js要es5.
* 原生控件的样式不统一, 如textarea的内边距, 如1px宽的线, 视觉差异显著.

### 一些小程序底层的问题
* map, canvas, video, textarea原生控件z-index高于一切, 甚至高于调试器.
* 安卓上图片链接不支持//自适应协议写法.
* 体验版经常进去就无限转菊花, 开启调试模式就会好.

## 迁移

### 迁移工作
* 模板语法改写. 标签就不替换了，主要数据绑定，样式绑定，事件和指令语法. 
* 状态管理大迁移，wepy-redux到vuex. 
* 换用less和框架配套插件，如原生api的promisify(comm-mpvue). 
* 构造器参数对象结构微调. 

### 迁移成本
* 重构周期. 保守预计五天.
* 测试负担. 除了米妮应该都需要测一遍，保守预计测试＋修bug需要三天.
* 后续开发者的学习成本, vue.
