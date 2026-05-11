# Crawlergo 项目代码文档

## 一、项目概述

Crawlergo 是一款专为 Web 漏洞扫描器设计的强大浏览器爬虫工具。它利用 Chrome 浏览器的 headless 模式进行 URL 收集，通过 Hook 页面的关键位置实现 DOM 渲染阶段的自动化处理，支持智能表单填充与提交，并能够触发各类 JavaScript 事件，从而尽可能多地收集目标网站暴露的入口点。Crawlergo 内置了高效的 URL 去重模块，能够过滤大量伪静态 URL，同时保持对大型网站的快速解析与爬取速度，最终输出高质量的请求结果集。该项目主要采用 Go 语言开发，依赖 chromedp 库实现与 Chrome 浏览器的通信交互。

### 1.1 核心功能特性

Crawlergo 具备多项实用的爬取功能。在浏览器环境渲染方面，工具通过 Chrome headless 模式完整执行 JavaScript，确保 SPA（单页应用）等动态页面的内容能够被正确解析。在表单处理方面，Crawlergo 支持智能表单填充与自动化提交，能够识别多种类型的输入字段并自动填充合理的测试数据。在事件触发方面，工具能够全面收集页面的 DOM 事件并自动触发执行，包括 inline 事件和 DOM2 级事件。在 URL 去重方面，Crawlergo 提供了 simple、smart、strict 三种过滤模式，能够有效过滤伪静态 URL 和重复请求。在路径探索方面，工具支持智能分析网页内容，收集 JavaScript 文件中的 URL、页面注释、robots.txt 文件，并可对常见路径进行 fuzz 探测。此外，Crawlergo 还支持 Host 绑定、浏览器请求代理、结果推送至被动扫描器等高级功能。

### 1.2 技术栈概述

该项目基于 Go 语言开发，使用 Go 1.16 及以上版本。主要依赖包括：chromedp/chromedp 库用于控制 Chrome 浏览器；chromedp/cdproto 库提供 Chrome DevTools Protocol 的类型定义；gogf/gf 框架提供字符串编码等工具函数；ants 库用于实现协程池管理； sirupsen/logrus 用于日志记录；urfave/cli 用于命令行参数解析。项目整体采用模块化设计，各模块职责清晰，便于维护和扩展。

## 二、项目架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        cmd/crawlergo                                 │
│                     (命令行入口与适配器)                               │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │ CLI 参数解析 │  │ 结果输出处理  │  │ 代理推送功能  │  │ 信号处理  │ │
│  └─────────────┘  └──────────────┘  └──────────────┘  └───────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          pkg/ (核心爬虫包)                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    task_main.go (爬虫任务管理)                │    │
│  │  ┌──────────────┐  ┌───────────────┐  ┌─────────────────┐  │    │
│  │  │ CrawlerTask  │  │ 协程池管理     │  │ 结果聚合与去重   │  │    │
│  │  └──────────────┘  └───────────────┘  └─────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────────────┐  │
│  │  engine/       │ │  filter/      │ │  model/                │  │
│  │  浏览器引擎控制   │ │  URL过滤处理   │ │  数据模型定义          │  │
│  └────────────────┘ └────────────────┘ └────────────────────────┘  │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────────────┐  │
│  │  config/       │ │  logger/       │ │  js/                    │  │
│  │  配置常量定义   │ │  日志管理       │ │  JavaScript脚本          │  │
│  └────────────────┘ └────────────────┘ └────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  tools/ (通用工具函数)  │  domain_collect.go (域名收集)           │ │
│  │  path_expansion.go (路径扩展)                                   │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构说明

```
/workspace/
├── cmd/
│   └── crawlergo/
│       ├── main.go          # 主入口文件，负责命令行适配和结果输出
│       └── flag.go          # 命令行参数定义（可能存在）
├── pkg/                     # 核心代码包
│   ├── config/              # 配置常量和枚举定义
│   │   └── config.go
│   ├── engine/              # 浏览器引擎核心模块
│   │   ├── browser.go       # 浏览器初始化与管理
│   │   ├── tab.go           # 单个标签页的爬取逻辑
│   │   ├── collect_links.go  # 链接收集实现
│   │   ├── after_loaded_tasks.go  # 页面加载后的任务执行
│   │   └── intercept_request.go   # 请求拦截与处理
│   ├── filter/              # URL 过滤模块
│   │   ├── filter.go        # 过滤接口定义
│   │   ├── simple_filter.go # 简单过滤实现
│   │   └── smart_filter.go  # 智能过滤实现
│   ├── model/               # 数据模型
│   │   ├── request.go       # 请求数据结构
│   │   └── url.go           # URL 工具函数
│   ├── js/                  # JavaScript 脚本
│   │   └── javascript.go     # 注入页面的 JS 代码
│   ├── logger/              # 日志模块
│   │   └── logger.go
│   ├── tools/               # 工具函数
│   │   ├── requests/        # HTTP 请求工具
│   │   │   ├── requests.go  # 请求封装
│   │   │   ├── response.go   # 响应封装
│   │   │   └── utils.go      # 请求相关工具
│   │   ├── common.go        # 通用工具
│   │   └── random.go        # 随机字符串生成
│   ├── task_main.go         # 爬虫任务主逻辑
│   ├── taskconfig.go        # 任务配置与选项模式
│   ├── domain_collect.go    # 域名收集功能
│   └── path_expansion.go    # 路径探测与扩展
├── examples/                # Python 调用示例
│   ├── host_binding.py
│   ├── request_with_cookie.py
│   ├── subprocess_call.py
│   └── zombie_clean.py
├── go.mod                   # Go 模块依赖定义
├── go.sum                   # 依赖校验和
├── Makefile                 # 构建脚本
└── README.md               # 项目说明文档
```

### 2.3 数据流向

爬虫任务的数据流向遵循以下主要步骤：首先，用户通过命令行或 API 调用传入目标 URL 及相关配置参数；接着，系统初始化 Chrome 浏览器实例并创建协程池；然后，对于每个待爬取的 URL，系统创建一个独立的浏览器 Tab 进行处理；在 Tab 执行过程中，工具会拦截并收集所有发出的 HTTP 请求，包括导航请求、XHR 请求、静态资源请求等；同时，工具会触发页面的各种 DOM 事件以发现更多 URL；最后，收集到的所有请求经过过滤模块处理，去除重复和伪静态 URL，聚合为最终结果输出。整个过程中，过滤模块贯穿始终，确保只保留有价值的请求结果。

## 三、主要模块职责详解

### 3.1 命令行入口模块（cmd/crawlergo）

命令行入口模块是整个项目的入口点，负责接收用户输入、解析命令行参数、初始化爬虫任务、输出爬取结果。该模块的主要职责包括：使用 urfave/cli 库定义和解析命令行参数，包括 Chrome 路径、并发 Tab 数量、过滤模式、输出格式等；创建 CrawlerTask 实例并执行爬取；处理爬取结果，支持 console、json、none 三种输出模式；实现被动代理推送功能，将结果推送到指定的被动扫描器地址；处理进程信号，实现优雅退出。

在 main.go 文件中，Result 结构体定义了最终输出的 JSON 格式，包含四个主要字段：req_list 字段存储同域名去重后的请求列表，all_req_list 字段存储所有域名下的所有请求，all_domain_list 字段存储发现的所有域名列表，sub_domain_list 字段存储子域名列表。ProxyTask 结构体用于实现代理推送的协程任务，通过协程池并发地将结果请求发送到被动扫描器。

### 3.2 爬虫任务管理模块（pkg/task_main.go）

task_main.go 是爬虫任务的核心管理模块，定义了 CrawlerTask 结构体及其相关方法。CrawlerTask 结构体包含以下关键字段：Browser 字段指向浏览器实例，RootDomain 字段记录当前爬取的根域名，Targets 字段存储输入的目标请求列表，Result 字段指向最终结果，Config 字段存储任务配置，filter 字段指向过滤处理器，Pool 字段管理协程池，crawledCount 字段统计已爬取数量，Start 字段记录任务开始时间。

NewCrawlerTask 函数负责初始化爬虫任务，根据配置选择合适的过滤模式（simple、smart、strict），处理 robots.txt 和 fuzz 路径扩展，自动补充 http/https 协议，初始化 Chrome 浏览器和协程池。Run 方法是任务执行的入口，它依次处理 robots.txt 路径获取、字典 fuzz 路径扩展、初始 URL 过滤、协程任务提交、结果聚合与去重，最终收集所有域名和子域名信息。

tabTask 结构体代表单个 Tab 的爬取任务，包含指向父任务、浏览器实例和请求信息的引用。addTask2Pool 方法负责将任务添加到协程池，该方法在添加前会检查是否达到最大爬取数量限制或任务超时。tabTask.Task 方法是单个 Tab 的执行逻辑，它创建新的 Tab、设置超时、收集结果并处理新发现的 URL。

### 3.3 任务配置模块（pkg/taskconfig.go）

taskconfig.go 实现了任务配置的选项模式（Functional Options Pattern），提供了灵活的配置方式。TaskConfig 结构体定义了所有可配置的参数，包括：MaxCrawlCount 控制最大爬取数量，FilterMode 设置过滤模式（simple、smart、strict），ExtraHeaders 定义额外的 HTTP 头，DomContentLoadedTimeout 设置 DOM 内容加载超时时间，TabRunTimeout 设置单个 Tab 的超时时间，MaxTabsCount 设置最大并发 Tab 数量，ChromiumPath 指定 Chrome 可执行文件路径，ChromiumWSUrl 支持连接已运行的 Chrome 实例，EventTriggerMode 设置事件触发模式（async 或 sync），EventTriggerInterval 设置事件触发间隔，BeforeExitDelay 设置退出前的延迟时间，IgnoreKeywords 定义忽略的关键字列表，Proxy 设置代理地址，CustomFormValues 和 CustomFormKeywordValues 定义表单填充的默认值和关键词匹配值。

选项模式通过 TaskConfigOptFunc 函数类型实现，每个配置项都有对应的 With 函数，如 WithMaxCrawlCount、WithFilterMode、WithChromiumPath 等。这种设计允许调用者只设置需要的配置项，未设置的项使用默认值，极大地提高了 API 的灵活性和易用性。

### 3.4 浏览器引擎模块（pkg/engine）

浏览器引擎模块是 Crawlergo 的核心，负责与 Chrome 浏览器进行交互。该模块包含以下主要文件：

**browser.go** 实现了浏览器管理功能。Browser 结构体包含上下文、取消函数、标签页列表和额外头信息。InitBrowser 函数使用 chromedp 库初始化 Chrome 浏览器，设置多个关键参数：无头模式（可通过配置关闭）、禁用 GPU、不使用沙盒模式、忽略证书错误、禁用图片加载以提升性能、禁用 Web 安全限制、禁用 XSS 审计器、设置窗口大小为 1920x1080 等。ConnectBrowser 函数支持通过 WebSocket URL 连接已运行的 Chrome 实例，适用于需要复用浏览器会话的场景。NewTab 方法创建新的标签页，Close 方法负责关闭浏览器及所有标签页。

**tab.go** 实现了单个标签页的爬取逻辑。Tab 结构体是爬取过程的核心数据结构，包含：Ctx 和 Cancel 字段管理标签页上下文，NavigateReq 字段存储导航请求信息，ResultList 字段存储收集到的请求列表，TopFrameId 和 LoaderID 字段用于识别主框架，PageCharset 字段存储页面编码信息，config 字段存储标签页配置。Tab 通过 ListenTarget 方法监听多种 Chrome 事件，包括网络请求事件（RequestWillBeSent、ResponseReceived）、Fetch 事件（RequestPaused、AuthRequired）、页面事件（DomContentEventFired、LoadEventFired）、对话框事件（JavascriptDialogOpening）和绑定函数事件（BindingCalled）。

Tab.Start 方法是标签页爬取的入口，它执行初始导航后等待所有异步任务完成，然后收集页面中的所有链接。AddResultUrl 方法用于添加收集到的 URL 到结果列表，同时处理 Host 绑定问题。Evaluate 方法用于在页面中执行 JavaScript 代码。

**collect_links.go** 实现了页面链接的收集功能。collectLinks 函数并发执行三个收集任务：collectHrefLinks 收集 src、href、data-url、data-href 属性中的链接，collectObjectLinks 收集 object[data] 标签中的链接，collectCommentLinks 使用正则表达式从 HTML 注释中提取链接。

**after_loaded_tasks.go** 实现了页面加载完成后的各项任务。AfterLoadedRun 是主调度函数，它协调执行表单提交和事件触发。formSubmit 函数负责自动化表单处理，包括设置表单 target 到隐藏 iframe、提交表单、点击 submit 按钮、点击所有按钮。事件触发包括三个方面：triggerInlineEvents 触发内联事件（如 onclick 属性），triggerDom2Events 触发 DOM2 级事件监听器注册的事件，triggerJavascriptProtocol 触发 javascript: 伪协议的链接。

**intercept_request.go** 实现了请求拦截功能。InterceptRequest 函数处理被 Fetch API 暂停的请求，根据请求类型进行不同处理：对于忽略关键字匹配的请求，直接阻断并记录；对于静态资源请求，阻断并添加到结果；对于导航请求，交由 HandleNavigationReq 处理；对于 XHR 请求，正常放行并记录。HandleNavigationReq 函数处理导航请求的各种情况，包括后端重定向响应、302/301 重定向标记、主导航请求、子 frame 导航和前端跳转。HandleHostBinding 函数处理 Host 绑定问题，确保请求头中的 Host 信息被正确传递。ParseResponseURL 函数使用正则表达式从响应内容中提取 URL。

### 3.5 URL 过滤模块（pkg/filter）

URL 过滤模块负责去除重复请求和伪静态 URL，提供三种过滤模式以满足不同场景的需求。

**filter.go** 定义了过滤器的接口 FilterHandler，该接口只包含一个 DoFilter 方法，需要过滤的请求返回 true，否则返回 false。

**simple_filter.go** 实现了简单过滤器 SimpleFilter。SimpleFilter 基于 mapset 数据结构实现了三个核心功能：唯一性过滤（UniqueFilter）通过请求的唯一 ID 进行去重，静态资源过滤（StaticFilter）过滤掉常见的静态文件后缀，域名过滤（DomainFilter）只保留指定域名的请求。SimpleFilter 的 DoFilter 方法依次执行这三个过滤逻辑。

**smart_filter.go** 实现了智能过滤器 SmartFilter，是项目最复杂的模块之一。SmartFilter 在 SimpleFilter 的基础上增加了大量智能去重策略，能够有效识别和过滤伪静态 URL。SmartFilter 实现了多种参数和路径标记机制：参数名标记（markParamName）将过长的参数名替换为标记、替换参数名中的数字；参数值标记（markParamValue）识别并标记各种类型的参数值，包括中文、URL 编码、Unicode、纯数字、混合字母数字、大小写混合、时间格式等；路径标记（MarkPath）对 URL 路径进行类似的标记处理。

SmartFilter 的核心去重逻辑包括：全局数值型参数过滤（globalFilterLocationMark）识别那些填入测试值后会产生变化的参数位置，将该参数标记为逻辑型而非伪静态；重复统计与阈值标记（repeatCountStatistic 和 overCountMark）对重复出现的参数名、参数值、路径进行统计分析，超过阈值的进行额外标记；Fragment ID 计算（calcFragmentID）对 URL 的 fragment 部分（# 后面的内容）进行唯一性计算。

SmartFilter 支持严格模式（StrictMode），在严格模式下会采用更激进的过滤策略，对参数值的组合类型进行分析，识别伪静态模式。多种标记常量定义了不同的标记类型，如 {{Crawlergo}} 表示自定义测试值、{{number}} 表示数字参数、{{chinese}} 表示中文参数、{{fix_param}} 表示需要修复的重复参数等。

### 3.6 数据模型模块（pkg/model）

**request.go** 定义了请求数据模型。Request 结构体包含：URL 字段指向解析后的 URL 对象，Method 字段存储 HTTP 方法，Headers 字段存储请求头信息，PostData 字段存储 POST 请求数据，Filter 字段存储过滤相关信息，Source 字段标识请求来源（Target、Navigation、XHR、DOM、JavaScript、PathFuzz、robots.txt、Comment 等），RedirectionFlag 标记是否为重定向请求，Proxy 字段指定代理地址。

Filter 结构体存储请求的过滤状态，包括：MarkedQueryMap 存储标记后的查询参数，MarkedPostDataMap 存储标记后的 POST 数据，MarkedPath 存储标记后的路径，FragmentID 存储 fragment 唯一标识，UniqueId 存储计算后的唯一 ID 等。

Request 提供了多个实用方法：GetRequest 构造函数创建请求实例，FormatPrint 方法格式化输出完整请求信息，SimplePrint 和 SimpleFormat 方法输出简化格式，NoHeaderId 方法返回不含 Header 的请求 ID（用于请求去重），UniqueId 方法返回请求的唯一标识（考虑重定向标志），PostDataMap 方法解析 POST 数据为 map 结构，支持 application/json 和 application/x-www-form-urlencoded 两种格式。

**url.go** 定义了 URL 扩展类型。URL 类型基于标准库的 url.URL 封装，提供了多个实用方法：GetUrl 解析字符串为 URL 对象，支持相对路径解析和路径修复；parse 方法修复不完整的 URL，处理 javascript: 和 mailto: 等特殊协议；QueryMap 方法返回查询参数的 map 结构；NoQueryUrl 方法返回不含查询参数的 URL；NoFragmentUrl 方法返回不含 fragment 的 URL；RootDomain 方法计算根域名（如 a.b.c.example.com 返回 example.com）；FileName 和 FileExt 方法提取文件名和扩展名；ParentPath 方法获取上一级路径。

### 3.7 配置模块（pkg/config）

config.go 定义了项目使用的所有常量、枚举和默认配置。静态资源后缀列表（StaticSuffix）定义了需要过滤的文件类型，包括图片、音视频、文档、压缩包等。脚本后缀列表（ScriptSuffix）定义了常见的服务器端脚本后缀。InputTextMap 定义了表单填充的默认值，根据输入框的类型和名称智能填充合理的测试数据，如邮箱、验证码、手机号、用户名、密码、QQ 号、身份证号、URL、日期、数字等。

请求来源常量定义了 URL 的各种来源类型，包括：Target（初始目标）、Navigation（页面导航）、XHR（异步请求）、DOM（DOM 解析）、JavaScript（JS 文件）、PathFuzz（路径探测）、robots.txt（robots 文件）、Comment（页面注释）、WebSocket、EventSource、Fetch、HistoryAPI、OpenWindow、HashChange、StaticResource、StaticRegex 等。

配置默认值包括：MaxTabsCount = 10（最大并发 Tab 数）、TabRunTimeout = 20秒、DomContentLoadedTimeout = 5秒、EventTriggerInterval = 100毫秒、BeforeExitDelay = 1秒、MaxCrawlCount = 200（最大爬取数量）、MaxRunTime = 3600秒（最大运行时间）。忽略关键字默认为 ["logout", "quit", "exit"]。

### 3.8 JavaScript 脚本模块（pkg/js）

javascript.go 存储了注入到 Chrome 页面的 JavaScript 代码，这些脚本是爬虫功能实现的关键部分。

**TabInitJS** 是最重要的初始化脚本，在每个新页面加载前执行，主要功能包括：绕过 WebDriver 检测（重写 navigator.webdriver 属性）；伪装插件列表（navigator.plugins）；模拟 Chrome 对象（window.chrome）；处理权限 API（navigator.permissions.query）；修改 User-Agent 和平台信息；Hook History API（pushState、replaceState）以捕获前端路由变化；监听 hashchange 事件；Hook WebSocket 构造函数；Hook EventSource 构造函数；Hook fetch 函数；Hook XMLHttpRequest 的 open 和 send 方法；Hook addEventListener 和 DOM0 级事件处理器；Hook window.open 和 window.close 方法；修改 setInterval 的间隔时间为 60 秒以减轻浏览器压力；定义辅助函数（randArr 用于打乱数组、sleep 用于异步延迟）。

**DeliverResultJS** 用于向页面返回异步调用的结果。**ObserverJS** 注册 DOM 变化监听器，收集动态插入的链接。**RemoveDOMListenerJS** 移除 DOM 监听器。**NewFrameTemplate** 创建一个隐藏的 iframe 用于表单提交。**TriggerInlineEventJS** 触发所有带内联事件处理器（如 onclick）的元素。**TriggerDom2EventJS** 触发通过 addEventListener 注册的 DOM2 级事件。**TriggerJavascriptProtocol** 触发 javascript: 伪协议的链接。**FormNodeClickJS** 使用 JavaScript 的 click 方法点击元素。

### 3.9 工具模块（pkg/tools）

**requests/requests.go** 封装了 HTTP 请求功能。ReqOptions 结构体定义请求选项，包括超时时间、重试次数、SSL 验证、是否跟随重定向、代理设置。doRequest 方法是实际发起请求的函数，设置默认的 User-Agent 和 Content-Type，处理重试逻辑，支持代理设置。Get 和 Request 函数提供简洁的请求接口。ReqInfo 结构体封装了完整的请求信息，支持链式调用。

**common.go** 提供通用工具函数，包括字符串处理、文件读写等功能。**random.go** 提供随机字符串生成功能，用于表单填充值和 iframe 名称的生成。

### 3.10 域名收集模块（pkg/domain_collect.go）

domain_collect.go 提供了两个域名收集函数。SubDomainCollect 函数接收请求列表和根域名，返回所有子域名列表，函数内部使用 mapset 进行去重。AllDomainCollect 函数返回请求列表中出现过的所有唯一域名。

### 3.11 路径扩展模块（pkg/path_expansion.go）

path_expansion.go 实现了路径探测和扩展功能，用于发现更多可爬取的 URL。

GetPathsFromRobots 函数解析 robots.txt 文件，使用正则表达式提取 Disallow 和 Allow 指令中的路径信息。GetPathsByFuzz 函数使用内置的常见路径字典进行 fuzz 探测，内置字典包含约 300 个常见路径，如 admin、api、login、user、upload、backup 等。GetPathsByFuzzDict 函数支持使用自定义字典文件进行 fuzz。doFuzz 函数并发执行 fuzz 请求，使用协程池管理并发度（默认 20），对返回 200 状态码或 301 重定向到同域名的路径进行记录。

## 四、关键类与函数说明

### 4.1 命令行入口层

**main.go 中的核心结构体和函数：**

Result 结构体定义了 JSON 输出格式，包含四个字段：ReqList 存储同域名去重结果、AllReqList 存储所有请求、AllDomainList 存储所有域名、SubDomainList 存储子域名列表。

Request 结构体是输出格式中的请求表示，区别于 model.Request，包含 Url、Method、Headers、Data、Source 字段。

ProxyTask 结构体用于代理推送任务，包含 req 字段指向请求对象、pushProxy 字段存储推送地址。doRequest 方法是协程池执行的任务实现。

run 函数是 CLI 的主动作函数，负责：解析命令行参数、设置日志级别、构建目标请求列表、初始化爬虫任务、执行爬取、处理代理推送、输出结果。

outputResult 函数根据配置的输出模式处理结果，json 模式输出带分隔符的 JSON 字符串，console 模式格式化输出每个请求。

Push2Proxy 函数使用协程池并发推送结果到被动扫描器，通过 ants 库管理并发池。

### 4.2 任务管理层

**CrawlerTask 结构体**（task_main.go）：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| Browser | *engine.Browser | Chrome 浏览器实例 |
| RootDomain | string | 爬取根域名 |
| Targets | []*model.Request | 输入目标列表 |
| Result | *Result | 最终结果 |
| Config | *TaskConfig | 任务配置 |
| filter | filter.FilterHandler | 过滤处理器 |
| Pool | *ants.Pool | 协程池 |
| crawledCount | int | 已爬取数量 |
| Start | time.Time | 开始时间 |

关键方法包括：NewCrawlerTask 创建任务实例并初始化浏览器和协程池；Run 执行完整爬取流程；addTask2Pool 添加任务到协程池（含数量和超时检查）。

**Result 结构体**（task_main.go）：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| ReqList | []*model.Request | 同域名去重结果 |
| AllReqList | []*model.Request | 所有请求 |
| AllDomainList | []string | 所有域名 |
| SubDomainList | []string | 子域名列表 |
| resultLock | sync.Mutex | 结果合并锁 |

**tabTask 结构体**（task_main.go）：封装单个 Tab 的爬取任务，包含 crawlerTask、browser、req 三个字段，Task 方法实现具体的爬取逻辑。

### 4.3 浏览器引擎层

**Browser 结构体**（engine/browser.go）：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| Ctx | *context.Context | 浏览器上下文 |
| Cancel | *context.CancelFunc | 取消函数 |
| tabs | []*context.Context | 标签页上下文列表 |
| tabCancels | []context.CancelFunc | 标签页取消函数列表 |
| ExtraHeaders | map[string]interface{} | 额外 HTTP 头 |
| lock | sync.Mutex | 并发锁 |

关键方法包括：InitBrowser 初始化新浏览器实例，ConnectBrowser 连接已运行的浏览器实例，NewTab 创建新标签页，Close 关闭浏览器。

**Tab 结构体**（engine/tab.go）：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| Ctx | *context.Context | 标签页上下文 |
| Cancel | context.CancelFunc | 取消函数 |
| NavigateReq | model2.Request | 导航请求 |
| ExtraHeaders | map[string]interface{} | 额外头 |
| ResultList | []*model2.Request | 结果列表 |
| TopFrameId | string | 顶层框架 ID |
| LoaderID | string | 加载器 ID |
| PageCharset | string | 页面编码 |
| config | TabConfig | 标签页配置 |

关键方法包括：NewTab 创建标签页并设置事件监听，Start 执行爬取流程，AddResultUrl 添加 URL 到结果，InterceptRequest 处理拦截的请求。

### 4.4 过滤层

**SimpleFilter 结构体**（filter/simple_filter.go）：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| UniqueSet | mapset.Set | 唯一性集合 |
| HostLimit | string | 域名限制 |
| staticSuffixSet | mapset.Set | 静态资源后缀集合 |

**SmartFilter 结构体**（filter/smart_filter.go）：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| SimpleFilter | *SimpleFilter | 内嵌简单过滤器 |
| StrictMode | bool | 严格模式标志 |
| filterLocationSet | mapset.Set | 参数位置集合 |
| filterParamKeyRepeatCount | sync.Map | 参数重复计数 |
| filterParamKeySingleValues | sync.Map | 单 URL 参数值 |
| filterPathParamKeySymbol | sync.Map | 路径参数标记计数 |
| filterParamKeyAllValues | sync.Map | 全局参数值统计 |
| filterPathParamEmptyValues | sync.Map | 空参数值统计 |
| filterParentPathValues | sync.Map | 父路径值统计 |
| uniqueMarkedIds | mapset.Set | 标记后唯一 ID |

关键方法包括：DoFilter 执行智能过滤，getMark 对 GET 请求标记，postMark 对 POST 请求标记，globalFilterLocationMark 全局数值型参数标记，repeatCountStatistic 重复统计，overCountMark 超阈值标记。

### 4.5 数据模型层

**Request 结构体**（model/request.go）：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| URL | *URL | 解析后的 URL |
| Method | string | HTTP 方法 |
| Headers | map[string]interface{} | 请求头 |
| PostData | string | POST 数据 |
| Filter | Filter | 过滤信息 |
| Source | string | 来源标识 |
| RedirectionFlag | bool | 重定向标志 |
| Proxy | string | 代理地址 |

**URL 结构体**（model/url.go）：基于标准库 url.URL 封装，扩展方法包括 GetUrl、parse、QueryMap、NoQueryUrl、NoFragmentUrl、RootDomain、FileName、FileExt、ParentPath。

### 4.6 核心流程函数

**Tab.Start 方法**执行流程：首先初始化 Tab，启用 runtime、network、fetch API，添加初始化脚本，设置额外头，执行页面导航；然后等待所有异步任务完成（DOMContentLoaded、事件处理、请求拦截等）；接着并发收集所有链接（href 链接、object 链接、注释链接）；最后检测页面编码并编码所有 URL。

**CrawlerTask.Run 方法**执行流程：首先处理 robots.txt 路径扩展；然后处理 fuzz 路径扩展（内置字典或自定义字典）；接着对初始目标进行过滤；提交初始任务到协程池；等待所有任务完成；对全部请求进行唯一去重；收集所有域名和子域名信息。

## 五、依赖关系

### 5.1 直接依赖

Crawlergo 的 go.mod 文件定义了以下直接依赖：

| 依赖包 | 版本 | 用途说明 |
|--------|------|----------|
| github.com/chromedp/cdproto | v0.0.0-20220629234738-4cfc9cdeeb92 | Chrome DevTools Protocol 的类型定义和命令接口 |
| github.com/chromedp/chromedp | v0.8.2 | Chrome 浏览器控制库，提供与 Chrome 的通信封装 |
| github.com/deckarep/golang-set | v1.7.1 | 高性能的 Set 数据结构实现，用于去重和集合操作 |
| github.com/gogf/gf | v1.16.6 | GoFrame 框架，提供字符串编码等工具函数 |
| github.com/panjf2000/ants/v2 | v2.2.2 | 高性能协程池库，用于管理并发爬取任务 |
| github.com/pkg/errors | v0.8.1 | 错误处理库，提供错误包装和格式化功能 |
| github.com/sirupsen/logrus | v1.4.2 | 结构化日志库，支持多种日志级别和格式 |
| github.com/stretchr/testify | v1.7.0 | 断言和测试工具库 |
| github.com/urfave/cli/v2 | v2.23.6 | 命令行应用框架，用于解析命令行参数 |
| golang.org/x/net | v0.7.0 | 提供公共后缀列表等功能 |

### 5.2 间接依赖

chromedp 库内部依赖多个 Chromium 协议相关的包，这些包由 chromedp 组织维护，提供 CDP 协议的完整封装。gogf/gf 框架是一个全功能的 Web 框架，Crawlergo 仅使用了其中的编码转换功能。ants 协程池库依赖于标准的 sync 包，提供 goroutine 的复用和管理功能。

### 5.3 系统依赖

运行 Crawlergo 需要安装 Chrome/Chromium 浏览器。项目代码中通过指定 chrome 可执行文件路径或 WebSocket URL 来连接浏览器。建议使用较新版本的 Chromium，以确保所有 CDP 功能（如 Fetch API）正常工作。

在 Linux 系统上，可能需要安装以下系统库以确保 Chrome 正常运行：libasound2、libatk1.0-0、libcairo2、libcups2、libdbus-1-3、libexpat1、libfontconfig1、libgbm1、libglib2.0-0、libgtk-3-0、libnss3、libx11-6 等。

## 六、项目运行方式

### 6.1 编译构建

项目支持多种编译方式。直接编译当前平台版本，执行以下命令：

```bash
make build
```

编译完成后，可执行文件位于 bin/crawlergo 目录。为其他平台交叉编译，执行：

```bash
make build_all
```

直接使用 Go 命令编译：

```bash
go build -o bin/crawlergo ./cmd/crawlergo
```

### 6.2 Docker 运行

项目提供了 Dockerfile，可构建 Docker 镜像：

```bash
docker build . -t crawlergo
docker run crawlergo http://example.com/
```

Docker 方式运行可以避免本地安装 Chrome 和依赖库的问题。

### 6.3 命令行参数

**必需参数：**

- -c, --chromium-path：Chrome/Chromium 可执行文件路径

**基本参数：**

- -m, --max-crawled-count：最大爬取数量，默认 200
- -f, --filter-mode：过滤模式（simple/smart/strict），默认 smart
- -o, --output-mode：输出模式（console/json/none），默认 console
- --output-json：输出 JSON 到指定文件
- --custom-headers：自定义 HTTP 头（JSON 格式）
- -d, --post-data：POST 数据
- --request-proxy：Socks5 代理地址

**路径扩展参数：**

- --fuzz-path：使用内置字典进行路径 fuzz
- --fuzz-path-dict：使用自定义字典文件
- --robots-path：从 robots.txt 解析路径

**表单填充参数：**

- -fv, --form-values：表单填充值，格式为 type=value，如 -fv mail=test@mail.com
- -fkv, --form-keyword-values：关键词匹配表单填充，格式为 keyword=value
- -iuk, --ignore-url-keywords：忽略的 URL 关键字，默认 logout、quit、exit

**高级参数：**

- -t, --max-tab-count：最大并发 Tab 数，默认 8
- --tab-run-timeout：单个 Tab 超时时间，默认 20s
- --event-trigger-interval：事件触发间隔，默认 100ms
- --event-trigger-mode：事件触发模式（async/sync），默认 async
- --before-exit-delay：退出前延迟，默认 1s

**其他参数：**

- --push-to-proxy：推送结果到被动扫描器地址
- --log-level：日志级别（debug/info/warn/error）
- --no-headless：关闭无头模式，可视化观察爬取过程

### 6.4 使用示例

基本爬取：

```bash
./crawlergo -c /path/to/chrome -t 10 http://example.com/
```

使用代理：

```bash
./crawlergo -c /path/to/chrome -t 10 --request-proxy socks5://127.0.0.1:7891 http://example.com/
```

启用路径 fuzz：

```bash
./crawlergo -c /path/to/chrome -t 10 --fuzz-path http://example.com/
```

JSON 输出：

```bash
./crawlergo -c /path/to/chrome -t 10 -o json http://example.com/
```

### 6.5 Python 调用示例

Crawlergo 支持作为子进程被其他语言调用，Python 调用示例如下：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import simplejson
import subprocess

def crawl(url, chrome_path="/path/to/chrome"):
    cmd = ["bin/crawlergo", "-c", chrome_path, "-o", "json", url]
    rsp = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    output, error = rsp.communicate()
    result_text = output.decode().split("--[Mission Complete]--")[1]
    result = simplejson.loads(result_text)
    return result

if __name__ == '__main__':
    result = crawl("http://example.com/")
    for req in result["req_list"]:
        print(req["method"], req["url"])
```

### 6.6 结果输出格式

JSON 输出格式包含四个部分：

```json
{
  "req_list": [
    {
      "url": "http://example.com/page",
      "method": "GET",
      "headers": {"Host": "example.com", "User-Agent": "..."},
      "data": "",
      "source": "DOM"
    }
  ],
  "all_req_list": [...],
  "all_domain_list": ["example.com", "cdn.example.com"],
  "sub_domain_list": ["cdn.example.com"]
}
```

req_list 字段包含同域名、去重后的请求列表，不包含静态资源；all_req_list 包含所有发现的所有请求；all_domain_list 列出所有涉及的域名；sub_domain_list 列出目标域名的所有子域名。

## 七、扩展开发指南

### 7.1 添加新的过滤模式

要添加新的过滤模式，需要在 filter 包中创建新的过滤器实现，实现 FilterHandler 接口。参考 SimpleFilter 和 SmartFilter 的实现方式，在 DoFilter 方法中实现具体的过滤逻辑。创建完成后，在 task_main.go 的 NewCrawlerTask 函数中添加对新过滤模式的支持。

### 7.2 添加新的事件监听

Tab.Init 方法中的 ListenTarget 调用监听各种 Chrome 事件。如果需要处理新的事件类型，在 switch 语句中添加相应的 case 分支，实现事件处理逻辑。新增的事件处理函数应该遵循现有命名规范，如 InterceptRequest、ParseResponseURL 等。

### 7.3 添加新的路径探测方式

在 path_expansion.go 中添加新的探测函数。函数应该返回 []*model2.Request 类型的切片。参考 GetPathsFromRobots 和 GetPathsByFuzz 的实现，使用 requests 包发起探测请求，根据响应状态码判断路径有效性。

### 7.4 集成被动扫描器

使用 --push-to-proxy 参数将结果推送到被动扫描器。推送逻辑在 main.go 的 Push2Proxy 函数中实现。如果需要支持其他推送协议（如 WebSocket、自定义协议），可以修改该函数的实现，或在收到请求后进行协议转换。

## 八、注意事项

### 8.1 性能优化建议

静态资源过滤在客户端完成，可以减少网络传输和结果处理开销。合理设置 --max-tab-count 参数，并发过高可能导致系统资源不足。智能过滤模式会增加内存占用，大规模爬取时可考虑使用 simple 模式。禁用图片加载（已默认启用）可以显著提升爬取速度。

### 8.2 常见问题排查

Navigation timeout 错误通常是由于 Chrome 路径配置错误或 Chrome 版本过低导致。检查 -c 参数指定的路径是否正确，尝试升级 Chrome 版本。

Fetch.enable 错误表示 Chrome 版本过低。Fetch API 是较新的 CDP 功能，需要较新版本的 Chrome 支持。

爬取结果为空可能是因为目标页面需要登录认证。使用 --custom-headers 或 --form-values 参数提供认证信息。某些网站可能检测到 headless 模式并拒绝访问，Crawlergo 已内置部分绕过措施，但并非所有网站都能成功爬取。

### 8.3 安全注意事项

Crawlergo 仅用于授权的安全测试和漏洞扫描。在使用前请阅读并同意 Disclaimer 文件。爬取行为可能对目标网站产生负载，请谨慎使用，避免对生产环境造成影响。使用代理可以隐藏真实 IP 地址，但不应将其用于非法目的。

## 九、总结

Crawlergo 是一款设计精良的浏览器爬虫工具，采用模块化的架构设计，代码结构清晰，易于理解和扩展。项目充分利用了 Go 语言的并发优势，通过协程池高效管理并发爬取任务。智能 URL 过滤算法能够有效识别伪静态 URL，在保证爬取完整性的同时避免了大量重复结果。JavaScript 注入机制使得工具能够完整执行页面的动态内容，发现更多的 URL 和功能入口。

项目的配置系统采用选项模式，提供了灵活的参数定制能力。命令行接口设计简洁直观，JSON 输出格式便于与其他工具集成。完整的注释和文档使得二次开发变得更加容易。如果需要在 Web 漏洞扫描器中使用浏览器爬取功能，Crawlergo 是一个值得考虑的选择。
