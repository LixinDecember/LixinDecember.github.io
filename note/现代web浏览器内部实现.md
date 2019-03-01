# 现代web浏览器内部实现

原英文链接：

1. https://developers.google.com/web/updates/2018/09/inside-browser-part1
2. https://developers.google.com/web/updates/2018/09/inside-browser-part2
3. https://developers.google.com/web/updates/2018/09/inside-browser-part3
4. https://developers.google.com/web/updates/2018/09/inside-browser-part4

## Part1：不同进程和线程何如处理浏览器的不同部分

1. 计算机核心：CPU和GPU
    * CPU：Central Processing Unit。一个一个按序处理不同task，可以处理任何math和art。过去CPU只有单核，现代可以有多核。
    * ![CPU内核](../../img/web-browser-1.png)
    * GPU：Graphics Processing Unit。擅长同时在多核上处理简单的任务。擅长处理图形。使用GPU渲染更加快速，交互更平滑。
    * ![GPU](../../img/web-browser-2.png)
    * CPU VS GPU
    * ![](../../img/cpu-gpu.png)
    * ![](../../img/cpu-gpu-1.png)
    * 一个应用由CPU和GPU驱动，在硬件上由操作系统提供的机制运行在CPU和GPU上。
    * ![](../../img/web-browser-3.png)
2. 进程和线程上的运行的程序
    * 一个进程可以理解为一个应用的执行程序。一个线程是进程执行程序的一部分。
    * 当启动一个应用时，一个进程被创建，一个程序可以创建一个或多个线程。操作系统给进程分配一个内存空间，应用的状态等都被存储在这块内存空间内。当你关闭应用时进程被关闭，操作系统会释放这块内存。
    ![进程和线程](../../img/web-browser-4.png)
    * ![Figure 5: Diagram of a process using memory space and storing application data](https://developers.google.com/web/updates/images/inside-browser/part1/memory.svg)
    * 一个进程可以要求操作系统调用另一个进程来运行不同的task，操作系统会为新进程分配新的内存块。如果两个进程需要通信，他们可以通过IPC（Inter Process Communication）。许多应用都会采用这样的方式运行，如果一个进程挂掉了不会影响其他进程的工作。
    * ![Figure 6: Diagram of separate processes communicating over IPC ](https://developers.google.com/web/updates/images/inside-browser/part1/workerprocess.svg)
3. 浏览器架构
    * 不同浏览器架构不同：可能是一个进程多个线程，或者是多进程多线程，多进程之间使用IPC通信
    * ![](../../img/web-browser-5.png)
    * 以下介绍以chrome为例
    * 所有进程顶部是浏览器进程，其协调浏览器的其他进程。
    * 对于渲染进程而言，chrome会给每个便签页创建一个渲染进程，现在尝试给每一个网站一个自己的进程，包括iframes（`site Isolation`）。
    * ![](../../img/web-browser-6.png)
4. chrome各进程工作：
    * Browser进程：
        * 控制包括地址栏，书签，网页回退和前进按钮；
        * 不可见部分包括：网络请求，文件访问等
    * renderer进程：控制标签页内部展现的所有东西
    * plugin进程：控制网站内使用的插件如flash等
    * GPU：独立于其他进程处理GPU任务，它被分成多个不同的进程，因为GPU处理来自多个不同的应用的请求并且将它们绘制在一个同一个屏幕上
    * ![](../../img/web-browser-7.png)
    * 在chrome上可以通过设置菜单选择更多工具下的任务管理器查看chrome的进程运行状态以及CPU和内存占用情况。
5. chrome内多进程的优缺点：
    * 优点：
        * 一个进程挂掉不会影响其他进程![Figure 10: Diagram showing multiple processes running each tab ](https://developers.google.com/web/updates/images/inside-browser/part1/tabs.svg)。
        * 安全和沙箱操作：操作系统通过沙箱来限制进程的权限。如：chrome浏览器限制任意文件访问处理用户输入的进程类似renderer进程。
    * 缺点：每个进程有自己独立的存储空间，包括复制一些基础设施类似：V8引擎。因此需要消耗更多的存储空间。为了节约存储空间，chrome限制了能创建的最大进程数。该限制根据设备的性能（内存大小以及CPU性能）而各不同。达到限制进程数量后，后面新开的标签页会与上一个标签页共用一个进程。

6. 节约内存消耗-chrome多服务分配
    * chrome将浏览器不同的程序看做不同的服务，这些服务可以分为不同的进程来处理或者合并为一个进程进行处理。chrome根据硬件设施的不同配置进行不同的分配：强大硬件上会将服务分为多个进程来处理，资源贫瘠的硬件设备上会使用一个进程来运行。
    * ![Figure 11: Diagram of Chrome’s servicification moving different services into multiple processes and a single browser process ](https://developers.google.com/web/updates/images/inside-browser/part1/servicfication.svg)

7. 每个frame一个renderer进程-site Isolation
    * `Site Isolation`是chrome近期的一个功能，对每一个跨域frame使用独立的进程渲染。（详见https://zhuanlan.zhihu.com/p/37861033）
    * 网站隔离是chrome近期推出的每个跨域iframe都使用独立renderer进程。目前我们每个标签页允许跨域iframe运行在同一个renderer进程，并且在跨域之间共享同一块内存。在同一个renderer进程上运行两个不同域名的页面看起来没问题。同源策略是web安全的核心，它使得一个网站在没经过同意时不能访问另一个网站的数据。安全攻击的核心就是绕过这个政策。进程隔离是分隔网站最有效的办法。在崩溃以及幽灵攻击上我们非常需要使用隔离进程来处理。该机制在chrome67版本上默认开启。每个跨域iframe标签都被一个独立的renderer进程渲染。
    * ![](../../img/web-browser-8.png)
    * 网站隔离机制不是类似于简单的多renderer进程渲染，它在底层上改变了iframes互相通信的方法。
    * `Ctrl+F`查找操作也以为这跨多renderer进程查找


## Part2：每个进程和线程之间如何通信来展现一个网页

在浏览器输入一个URL，浏览器从网络获取数据然后展现页面。以下部分主要讲：一个用户请求一个网站的过程，浏览器准备渲染一个页面-导航。

1. 由浏览器进程开始：
    * 标签页之外的工作都是由browser进程分配。browser进程分配UI线程负责浏览器内按钮绘制以及文件输入；network线程去从网络上请求数据，storage线程控制文件的访问。当在地址栏输入URL时，由browser进程的UI线程处理输入。
    * ![](../../img/web-browser-9.png)
2. 一个简单导航：
    * 处理输入：
        * 当用户在地址栏输入，UI线程首先判断输入的是query还是URL？在chrome内，地址栏也是一个文案搜索，所以UI线程需要解析并且决定是去搜索引擎搜索还是直接请求URL网页。
        * ![](../../img/web-browser-10.png)
    * 开始导航：
        * 当用户回车，UI线程启动一个network线程来获取网站内容。
        * 展现loading加载状态
        * network线程进行DNS查找，为请求建立TLS连接
        * ![](../../img/web-browser-11.png)
        * network线程可能收到重定向请求头如301。network线程通知UI线程服务器发起了重定向。然后触发一个新的URL请求。
    * 读取响应
        * 响应body一旦接收，network线程依据响应的Content-Type头判断响应内容的格式。不同浏览器对不同格式有不同处理。
        * ![](../../img/web-browser-12.png)
        * 如果响应是HTML文件，下一步将会将数据传递给renderer进程，但是如果是zip文件或者其他文件意味着这是一个下载请求，需要将数据传递给下载管理器。
        * ![](../../img/web-browser-13.png)
        * 同时还会触发SafaBrowsing安全检查，如果响应数据匹配到已标记的恶意网站，network线程会提示展现一个警告页面。另外，CORB（Cross Origin Read Blocking）检查也会触发来确保跨域内容的敏感数据没有进入renderer进程。
    * 查找renderer进程
        * 所有的检查一旦完成，network线程就确认浏览器将导航到请求的网站，network线程通知UI线程数据已经ready，UI线程则查找到一个renderer进程来渲染web页面。
        * ![](../../img/web-browser-14.png)
        * network线程请求可能会需要几百毫秒才能得到相应，有一个加速进程处理的优化：当UI线程发送URL请求给network线程，它已经知道将要导航到哪一个网站，这时UI线程可以并行的去查找并启动一个renderer进程，当network线程接收到数据时一个renderer进程已经准备好渲染网站了。**当返回一个重定向请求时这个备用进程可能会用不上，导致可能会需要另一个新的进程。不能复用前面ready的进程？还是说可能进程类型不同？**
    * 提交导航
        * 现在数据以及renderer进程都已准备好了，browser进程发送IPC消息给renderer进程去提交导航。renderer进程通过数据流格式接收HTML数据，browser进程接收到renderer进程的确认信息那么导航就完成了，文档加载阶段开始。
        * 同时地址栏会更新，显示安全提示，以及刷新按钮等。
        * 标签页的history也会更新，网页的前进后退按钮可用
        * 当关闭一个标签页或者整个窗口，history被存储在磁盘里（以便于页面恢复）。
        * ![](../../img/web-browser-15.png)
    * 初始加载完成。
        * 导航一旦提交renderer进程开始下载资源并渲染页面。一旦renderer进程完成渲染，它将发送IPC信息给browser进程（这里假设该页面的所有frame的所有onload事件已经触发并已经执行）。同时UI线程停止加载标签页的loading状态
        * 以上表示网页首帧渲染完成，客户端脚本后面可能还会加载另外的资源同时更新渲染
        * ![](../../img/web-browser-16.png)

3. 导航到一个新的网站
    * 如果用户在地址栏重新输入不同URL会怎样呢？
    * browser进程会通过以上相同的步骤去导航到一个不同的网站页面。
    * 但是在此之前它需要检查当前的网站是否还有beforeunload事件
    * 在离开或者关闭当前标签页时，beforeunload事件可能会提示离开当前网站
    * 标签页内的所有事情包括js代码都是通过renderer进程处理，因此browser进程在发生新页面请求时必须检查当前renderer进程
    * ![](../../img/web-browser-17.png)
    * 如果renderer进程发起新请求（例如点击链接或者使用js代码类似：`window.location = "https://xxx.com"`），renderer进程首先会检查beforeunload事件处理器，然后再由renderer进程发送导航请求给browser进程。
    * 当新导航请求是与当前渲染页不同的网站则会创建一个新的renderer进程来处理新网站，当前renderer进程会处理旧页面的unload事件等。
    * 页面声明周期更多详见：[an overview of page lifecycle states ](https://developers.google.com/web/updates/2018/07/page-lifecycle-api#overview_of_page_lifecycle_states_and_events)
    * [the page lifecycle API ](https://developers.google.com/web/updates/2018/07/page-lifecycle-api)
    * ![](../../img/web-browser-18.png)

4. Service Worker
    * Service worker是应用代码写网络代理的一种方式。允许web开发者能更多的控制 从本地缓存或者网络上获取数据。如果service worker被设置为从缓存加载页面，就不需要从网络上请求数据了。
    * 重要的是：service worker是运行在renderer进程上的js代码。但是当导航请求来临时browser进程怎么知道这个网站是否有一个service worker呢？
        * 当一个service worker被注册，该worker的作用域被标记下来（[The Service Worker Lifecycle ](https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle)）
        * 当一个导航发生时network线程检查域名与当前Service Worker的注册的作用域，若果在当前Service worker作用域内，UI线程将会找到一个renderer进程来执行service worker代码。Service worker可能从缓存上或者网络上下载数据。（基于此原理能得知：Service worker工作前提是首次注册该worker时js文件等静态资源都已缓存完成）
    * ![](../../img/web-browser-19.png)
    * ![](../../img/web-browser-20.png)
    * 导航预加载
        * 如果service worker最终选择从网络上加载请求数据，browser进程和renderer进程的交互会有请求时间差之间的延迟。导航预加载是一种与service worker并行加载资源来提升速度的机制。它标记这些请求头，允许服务器决定发送不同的内容给不同的请求，例如：只发送更新的数据而不是整个文档。详见：[navigation preload ](https://developers.google.com/web/updates/2017/02/navigation-preload)
       *  ![](../../img/web-browser-21.png)




## Part3：renderer进程内部实现

1. [web性能 ](https://developers.google.com/web/fundamentals/performance/why-performance-matters/)

2. renderer进程处理web内容
    * renderer进程处理标签页内发生的所有事情。在一个renderer进程内，主线程处理大多数的js代码。如果使用web worker或者service worker ，js代码可能由worker线程处理。compositor和raster线程也是在renderer进程内运行，以渲染一个更流畅的页面。
    * renderer进程的核心工作是将HTML、CSS和JS文件转化为web页面与用户进行交互
    * ![](../../img/web-browser-22.png)
    * 解析
        * 创建DOM树：
            * 当renderer进程接收到页面的HTML文件，主线程开始解析字符串将其转换为一个DOM树（Document Object Model）
            * DOM是浏览器页面的内部显示的数据结构和API，web开发者可以通过js进行交互
            * 解析一个HTML文件到一个DOM树是有HTML标准定义的（[HTML Standard ](https://html.spec.whatwg.org/)） 
        * 子资源加载：
            * 一个网站通常使用内部资源，如：image，css以及js文件。这些文件需要从网络或者缓存中加载。当解析DOM树时主线程会挨个请求他们，为了加速，引入预扫描：在HTML文档中如果有img或link标签预扫描器得到一个由HTML解析器生成的一个token，并将请求发送给browser进程的network线程
            * ![](../../img/web-browser-23.png)
            * js可能会阻塞解析
                * 当HTML解析器发现一个script标签，它会暂停当前的HTML文档解析，然后去加载以及解析执行该js代码。因为js代码可能会使用DOM API改变DOM结构。因此HTML解析器必须等到js代码执行完成后再继续解析HTML文档。[V8 team ](https://mathiasbynens.be/notes/shapes-ics)
            * 标签可以标记自己该如何加载资源
                * 如果js没有使用类似`document.write()`的接口可以使用`async`或者`defer`属性在script标签内告诉浏览器是否需要暂停解析来同步加载资源。
                * `<link rel="preload">`表示想要尽快加载资源
                * [资源优先级 ](https://developers.google.com/web/fundamentals/performance/resource-prioritization)
        * 样式计算
            * 仅仅有DOM树还不能展示页面因为我们需要CSS来决定元素样式。主线程解析CSS文件并结合CSS选择器为每一个DOM节点计算样式。可以通过devtools查看计算样式。
            * ![](../../img/web-browser-24.png)
            * 如果你没有css文件，浏览器也会为每个DOM节点设置一个默认的样式。[default css source code ](https://cs.chromium.org/chromium/src/third_party/blink/renderer/core/html/resources/html.css)
        * 布局
            * 现在renderer进程知道文档的结构以及每个节点的样式但是也不足够渲染页面。
            * ![](../../img/web-browser-25.png)
            * 布局是一个查找元素几何位置的过程。主线程通过DOM树以及计算样式创建一个包含了xy坐标以及尺寸的布局树。布局树类似于DOM树，但是它只包含了页面可见的信息。如：`display:none;`会出现在DOM树中但是不会在布局树中；以及伪元素不会出现在DOM树中但是会在布局树中。
            * ![](../../img/web-browser-26.png)
            * 决定页面的布局的是一个具有挑战性的任务。最简单的页面布局如一块从上至下的流也必须考虑字体的大小以及文案可能会被截断导致影响下段落的显示。
            * CSS可以使元素漂浮在一边，标记溢出项，改变文档方向等。在chrome内，有一整个布局团队负责。更多了解[few talks form BlinkOn Conference ](https://www.youtube.com/watch?v=Y5Xa4H2wtVA)
                * https://developers.google.com/web/updates/images/inside-browser/part3/layout.mp4
        * 绘制：
            * 有了DOM，样式，以及布局依旧不能渲染页面。
            * ![](../../img/web-browser-27.png)
            * 例如：`z-index`可能被设置在部分元素上，那按照HTML文档写的顺序来渲染元素则会渲染错误。
            * ![](../../img/web-browser-28.png)
            * 在绘制阶段，主线程会遍历布局树，生成绘制记录。绘制记录记录绘制程序如：“background first, then text, then rectangle” 
            * ![](../../img/web-browser-29.png)
        * 更新渲染是昂贵的
            * 渲染阶段最重要的是每一次前一次操作的结果都会产生新数据。如果布局树被改变了绘制顺序需要重新生成。
            * https://developers.google.com/web/updates/images/inside-browser/part3/trees.mp4
            * 如果是动画元素，浏览器必须在每一帧之间刷新渲染。人眼能感知的最流畅的屏幕刷新频率是60fps（即每秒刷新60帧，每隔16.7ms绘制一帧）。然而如果动画刷新频率低于了屏幕刷新频率则会出现堵塞。
            * ![](../../img/web-browser-30.png)
            * 如果你的渲染操作能跟上屏幕刷新频率，但是如果在主线程上有js执行复杂的计算，也可能会导致渲染阻塞
            * ![](../../img/web-browser-31.png)
            * 你可以将js操作分为多个小块，并使用web提供的`requestAnimationFrame()`接口在每帧渲染之前执行该js小块。更多见：[Optimize javascript Execution ](https://developers.google.com/web/fundamentals/performance/rendering/optimize-javascript-execution)  也可以使用web worker来避免主线程阻塞。[js in web workers ](https://www.youtube.com/watch?v=X57mh8tKkgE)
            * ![](../../img/web-browser-32.png)
        * h. 合成
            * 怎么绘制页面
                * 现在浏览器知道文档的结构以及每个元素的样式，页面的几何结构，绘制顺序，那么它要怎么绘制这个页面呢。将这些信息转换成屏幕上的像素点 这个过程被称为栅格化。
                * native的处理这些的方式可能是在视口内栅格化每部分元素。如果用户滚动页面则移动栅格化的frame，然后栅格化新的部分来填充滑动走的部分。chrome第一次发布的时候就是使用的这种机制。然而，现代浏览器使用更复杂的过程-合成。
                * [Figure 14: Animation of naive rastering process ](https://developers.google.com/web/updates/images/inside-browser/part3/naive_rastering.mp4)
            * 什么是合成
                * 合成是一项将页面分为不同的层然后各自栅格化最后在一个单独的compositor线程来合成页面。如果发生了滚动，已经栅格化的层只需要合成一个新的frame，动画也是一样的过程（移动层并合成新的frame）
                * 通过devtools的layers面板能看见我们的网站页面是被分为了多个层的[layers panel ](https://blog.logrocket.com/eliminate-content-repaints-with-the-new-layers-panel-in-chrome-e2c306d4d752)
                * [Figure 15: Animation of compositing process ](https://developers.google.com/web/updates/images/inside-browser/part3/composit.mp4)
            * 分层
                * 为了找到哪个元素应该在哪一个层，主线程会遍历布局树创建一个层树（这部分在devtools的performance面板上被成为“更新层树”）。可以通过css的`will-change`属性提醒浏览器要创建一个层。
                * ![](../../img/web-browser-33.png)
                * 可以给每一个元素创建一个层，但是层数太多会导致页面性能下降，浪费资源。（每一层都需要内存和管理，每一层的纹理都需要上传至GPU，CPU与GPU之间带宽，以及GPU可用于处理纹理的内存也会受限）[Stick to Compositor-Only Properties and Manage Layer Count. ](https://developers.google.com/web/fundamentals/performance/rendering/stick-to-compositor-only-properties-and-manage-layer-count)
            * 主线程与栅格化及合成的交互
                * 一旦层树被创建以及绘制顺序确定好之后，主线程则传递这些信息给compositor线程。compositor线程栅格化每一层。有些层可能会有整个页面那么大，因此compositor线程会将它们分为多tiles，并将这些tile传递给raster线程。raster线程会光栅化每个tile并将它们存储在GPU内存。
                * ![](../../img/web-browser-34.png)
                * compositor线程会优先处理视野内以及视野周围的raster线程。如果有缩放等操作那么一个层可能会有不同多个tile。
                * 一旦tile被光栅化，compositor线程会得到tile的位置等信息来创建一个合成帧。
                    * `Draw quads`：包括一个tile的在内存中的位置以及在页面上合成的位置
                    * `Compositor frame`：一个页面的所有frame的draw quads的集成
                * 接着compositor frame（合成帧）通过IPC被提交给browser进程。同时还有从UI线程传递给浏览器的UI变化的或者别的拓展renderer进程也可以添加合成帧。这些合成帧被传递给GPU来展示在屏幕上。如果发生滚动事件，compositor线程创建新的合成帧来传递给GPU。
                * ![](../../img/web-browser-35.png)
                * 合成的优点是：没有占用主线程，compositor线程不需要等待样式的计算以及js执行。这是合成层动画流畅的原因。如果层或绘制需要重新计算那么就会用到主线程。



## Part4：合成线程怎样使得用户输入如：鼠标移动和点击等交互流畅

1. 从浏览器view获取输入事件
    * 当你接收到输入事件你可能只想到了输入框或者鼠标点击，但是在浏览器视角：输入意味着用户的所有交互。包括：鼠标滑轮滚动，touch或者鼠标悬浮等都是输入事件
    * 当用户交互类似touch等事件发生时browser进程会首先接收到手势信息，只有browser进程知道手势信息，标签页内的所有内容都是renderer进程负责。所以browser进程发送事件类型以及它的坐标信息给renderer进程。renderer进程处理绑定的事件处理。
    * ![](../../img/web-browser-36.png)

2. 合成器接收输入事件
    * 前文讲到合成器可以通过合成光栅层处理滚动。如果页面没有绑定监听，compositor线程可以独立于主线程创建一个新的合成帧。但是如果页面绑定了事件监听呢？compositor线程怎样判断是否有事件需要处理呢？
    * [Figure 2: Viewport hovering over page layers ](https://developers.google.com/web/updates/images/inside-browser/part3/composit.mp4)

3. 理解non-fast 滚动区域
    * 主线程执行js代码，当一个页面是合成的，那么compositor线程会标记这个页面是"Non-Fast Scrollable Region"的部分。有了这个机制，如果在此区域触发了滚动事件，compositor线程可以发送事件信息给主线程。如果是此区域之外发生了滚动事件就不需要发送事件信息给主线程了。
    * ![](../../img/web-browser-37.png)
    * 了解事件处理机制
        * web开发时通常使用的事件模型是事件委托，基于事件冒泡，我们常在最顶层元素绑定事件委托。这样对所有元素只需要写一个事件处理函数。然而此时整个页面则被标记为了"non-fast scollable region"，这意味着如果你的应用不关心事件来自页面哪部分都需要compositor线程通知主线程然后每次都要等待事件处理。这样合成器的滑动优化则失效了。
        * ![](../../img/web-browser-38.png)
        * 为了优化这种情况，可以对监听绑定`passive: true`属性，提醒浏览器你仍然在主线程监听这个事件但是合成器可以不用等待主线程直接继续合成新的frame。
        * 用法：`element.addEvenListener('touchevent', callback, {passive: true})`

4. 检查事件是否为可取消
    * 想像如果页面内有一的区域只允许水平滚动。在事件绑定时使用`passive:true`选项意味着页面滚动可以很流畅，但是在触发水平滚动之前可能已触发了垂直滚动。可以使用`event.cancelable`方法取消事件
    * ![](../../img/web-browser-39.png)
    * 也可以使用css的`touch-action`来消除事件：`#area{touch-action: pan-x;}`。

5. 找到事件对象
    * 当compositor线程发送一个事件给主线程时，主线程会首先进行hit test来找到对应的事件目标，hit test基于renderer进程生成的绘制记录来找到事件发生的位置坐标对应的元素
    * ![](../../img/web-browser-40.png)

6. 主线程优化事件的分发
    * 前面说过人眼最流畅的帧更新频率是一分钟60次。但我们可能发生一分钟触发了60-120次事件，典型的是鼠标滚动事件一分钟分发100次，事件分发频率远高于我们的帧更新频率。
    * 如果类似`touchmove`事件在主线程一分钟触发120次，可能会引发大量的hit tests以及js执行，阻塞屏幕刷新。
    * ![](../../img/web-browser-41.png)
    * chrome为了优化减少主线程的事件调用，合并一些连续事件（如：`wheel, mousewheel, mousemove, pointermove, touchmove`）在下一帧渲染之前延迟事件分发。
    * ![](../../img/web-browser-42.png)
    * 其他的不连续事件如：`keydown, keyup, mouseup, mousedown, touchstart, and touchend`会被立即执行。

7. 使用`getCoalescedEvents `接口获取连续事件
    * 大多数web应用合并事件是为了优化用户体验。然而如果你需要通过touchmove的点坐标来构建平滑的路线，你可能会失去一些坐标点。这种情况你可以使用`getCoalescedEvents`接口来获取该合并事件。
    * ![](../../img/web-browser-43.png)


