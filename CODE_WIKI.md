# Crawlergo Code Wiki

## 项目概述

**crawlergo** 是一个基于 Chrome headless 模式的强大浏览器爬虫，专为 Web 漏洞扫描器设计。它通过 hooks 整个网页的关键位置进行 DOM 渲染阶段的爬取，自动填充和提交表单，智能触发 JS 事件，尽可能多地收集网站暴露的入口点。

### 核心特性

- Chrome 浏览器环境渲染
- 智能表单填充与自动提交
- 全 DOM 事件收集与自动触发
- 智能 URL 去重（过滤伪静态 URL）
- 智能网页分析：JS 文件内容、页面注释、robots.txt、常用路径 Fuzz
- Host 绑定支持，自动修正 Referer
- 浏览器请求代理支持
- 结果推送至被动 Web 漏洞扫描器

---

## 项目架构

```
crawlergo/
├── cmd/crawlergo/          # 命令行入口
│   ├── main.go            # 主程序，命令行适配器
│   └── flag.go            # CLI 参数定义
├── pkg/                   # 核心业务逻辑
│   ├── config/            # 配置常量
│   ├── engine/            # 浏览器引擎
│   ├── filter/            # URL 过滤去重
│   ├── js/                # JavaScript 脚本
│   ├── logger/            # 日志模块
│   ├── model/             # 数据模型
│   ├── tools/             # 工具函数
│   ├── task_main.go       # 爬虫任务主逻辑
│   ├── taskconfig.go      # 任务配置
│   ├── domain_collect.go  # 域名收集
│   └── path_expansion.go  # 路径扩展
├── examples/              # Python 调用示例
├── Makefile               # 构建脚本
└── go.mod                 # 依赖管理
```

---

## 模块职责

### 1. 命令行入口 (cmd/crawlergo)

#### main.go
- **职责**: 程序主入口，解析命令行参数，初始化爬虫任务，输出结果
- **关键类型**:
  - `Result`: JSON 输出结构
  - `Request`: 请求数据结构
  - `ProxyTask`: 被动代理推送任务
- **关键函数**:
  - `run()`: 执行爬虫任务
  - `Push2Proxy()`: 推送结果到被动扫描器
  - `outputResult()`: 输出结果

#### flag.go
- **职责**: 定义所有 CLI 命令行参数
- **参数列表**:
  | 参数 | 说明 | 默认值 |
  |------|------|--------|
  | `-c, --chromium-path` | Chrome 可执行文件路径 | 必填 |
  | `-t, --max-tab-count` | 最大并发标签页数 | 8 |
  | `-m, --max-crawled-count` | 最大爬取 URL 数量 | 200 |
  | `-f, --filter-mode` | 过滤模式 (simple/smart/strict) | smart |
  | `-o, --output-mode` | 输出模式 (console/json/none) | console |
  | `--post-data` | POST 数据 | - |
  | `--custom-headers` | 自定义 HTTP 头 | - |
  | `--fuzz-path` | 启用路径 fuzz | false |
  | `--robots-path` | 解析 robots.txt | false |
  | `--request-proxy` | Socks5 代理地址 | - |
  | `--push-to-proxy` | 推送结果到被动扫描器 | - |

### 2. 配置模块 (pkg/config)

**config.go** 定义了所有常量配置：

- **过滤模式常量**:
  - `SimpleFilterMode`: 仅过滤静态资源和重复请求
  - `SmartFilterMode`: 智能过滤伪静态
  - `StrictFilterMode`: 严格伪静态过滤

- **事件触发模式**:
  - `EventTriggerAsync`: 异步触发
  - `EventTriggerSync`: 同步顺序触发

- **请求来源类型**:
  - `FromTarget`: 初始目标
  - `FromNavigation`: 页面导航
  - `FromXHR`: Ajax 请求
  - `FromDOM`: DOM 解析
  - `FromJSFile`: JS 文件
  - `FromFuzz`: 路径 fuzz
  - `FromRobots`: robots.txt

- **表单填充配置**:
  - 支持类型: `mail`, `code`, `phone`, `username`, `password`, `qq`, `id_card`, `url`, `date`, `number`
  - 默认填充值: "Crawlergo"

### 3. 数据模型 (pkg/model)

#### Request 结构
```go
type Request struct {
    URL             *URL              // 解析后的 URL
    Method          string            // HTTP 方法
    Headers         map[string]interface{} // 请求头
    PostData        string            // POST 数据
    Filter          Filter            // 过滤相关信息
    Source          string            // 请求来源
    RedirectionFlag bool              // 重定向标记
    Proxy           string            // 代理地址
}
```

#### URL 结构
- `GetUrl()`: 解析 URL，支持父 URL 相对路径解析
- `RootDomain()`: 获取根域名
- `FileExt()`: 获取文件扩展名
- `ParentPath()`: 获取上级路径
- `NoQueryUrl()`: 去除查询参数的 URL
- `NoFragmentUrl()`: 去除锚点的 URL

### 4. 任务配置 (pkg/taskconfig)

**TaskConfig** 结构包含所有任务配置项：
- `MaxCrawlCount`: 最大爬取数量
- `FilterMode`: 过滤模式
- `MaxTabsCount`: 最大标签页数
- `TabRunTimeout`: 单标签页超时时间
- `EventTriggerMode/Interval`: 事件触发配置
- `CustomFormValues/CustomFormKeywordValues`: 表单填充配置
- `IgnoreKeywords`: 忽略的关键字

**TaskConfigOptFunc**: 函数式配置选项模式

### 5. 爬虫任务主逻辑 (pkg/task_main.go)

**CrawlerTask** 核心结构：
```go
type CrawlerTask struct {
    Browser       *engine.Browser    // 浏览器实例
    RootDomain    string             // 根域名
    Targets       []*model.Request   // 目标列表
    Result        *Result            // 爬取结果
    Config        *TaskConfig        // 配置信息
    filter        filter.FilterHandler // 过滤处理器
    Pool          *ants.Pool         // 协程池
    crawledCount  int                // 已爬取计数
}
```

**Result** 结果结构：
```go
type Result struct {
    ReqList       []*model.Request  // 同域名结果
    AllReqList    []*model.Request  // 所有域名请求
    AllDomainList []string          // 所有域名
    SubDomainList []string          // 子域名
}
```

**关键流程**:
1. `NewCrawlerTask()`: 初始化任务，创建浏览器实例
2. `Run()`: 执行爬取，包括 robots.txt 解析、路径 fuzz
3. `generateTabTask()`: 为每个 URL 生成标签页任务
4. `addTask2Pool()`: 添加任务到协程池

### 6. 浏览器引擎 (pkg/engine)

#### Browser (browser.go)
- **职责**: 管理 Chrome 浏览器生命周期
- **关键方法**:
  - `InitBrowser()`: 初始化浏览器（支持代理、无头模式）
  - `ConnectBrowser()`: 连接已运行的 Chrome（WebSocket 模式）
  - `NewTab()`: 创建新标签页
  - `Close()`: 关闭浏览器

**Chrome 启动参数**:
- `--headless`: 无头模式
- `--disable-gpu`: 禁用 GPU
- `--no-sandbox`: 禁用沙箱
- `--ignore-certificate-errors`: 忽略证书错误
- `--disable-images`: 禁用图片加载

#### Tab (tab.go)
- **职责**: 管理单个标签页的爬取逻辑
- **核心方法**:
  - `Start()`: 启动标签页爬取
  - `AddResultUrl()`: 添加 URL 结果
  - `InterceptRequest()`: 拦截处理请求

**TabConfig**: 标签页配置
```go
type TabConfig struct {
    TabRunTimeout           time.Duration
    DomContentLoadedTimeout time.Duration
    EventTriggerMode        string
    EventTriggerInterval    time.Duration
    BeforeExitDelay         time.Duration
    IgnoreKeywords          []string
    CustomFormValues        map[string]string
    CustomFormKeywordValues map[string]string
}
```

#### 任务模块 (after_dom_tasks.go)
- **fillForm()**: 自动化表单填充
  - `fillInput()`: 填充 input 标签
  - `fillTextarea()`: 填充文本域
  - `fillMultiSelect()`: 填充下拉选择框
- **setObserverJS()**: 设置 DOM 节点变化观察

#### 事件触发 (after_loaded_tasks.go)
- **formSubmit()**: 表单自动提交
- **triggerInlineEvents()**: 触发内联事件 (onclick 等)
- **triggerDom2Events()**: 触发 DOM2 级事件
- **triggerJavascriptProtocol()**: 触发 JavaScript 伪协议链接
- **RemoveDOMListener()**: 移除 DOM 监听

#### 请求拦截 (intercept_request.go)
- **InterceptRequest()**: 处理每个被拦截的 HTTP 请求
- **HandleNavigationReq()**: 处理导航请求
- **HandleHostBinding()**: 处理 Host 绑定
- **ParseResponseURL()**: 从响应内容正则匹配 URL
- **HandleRedirectionResp()**: 处理重定向响应

#### 链接收集 (collect_links.go)
- **collectHrefLinks()**: 收集 href/src/data-url 属性链接
- **collectObjectLinks()**: 收集 object[data] 链接
- **collectCommentLinks()**: 收集 HTML 注释中的链接

### 7. 过滤去重 (pkg/filter)

#### Filter 接口
```go
type FilterHandler interface {
    DoFilter(req *model.Request) bool
}
```

#### SimpleFilter (simple_filter.go)
基础过滤器，实现：
- **UniqueFilter()**: 基于 Method + URL + PostData 的 MD5 去重
- **StaticFilter()**: 过滤静态资源（图片、视频、音频等）
- **DomainFilter()**: 域名过滤

#### SmartFilter (smart_filter.go)
智能过滤器，在 SimpleFilter 基础上：
- **参数值标记**: 对参数值进行模式标记
  - `{{number}}`: 纯数字
  - `{{chinese}}`: 包含中文
  - `{{urlencode}}`: URL 编码
  - `{{MixAlphaNum}}`: 字母数字混合
  - 等多种标记模式

- **路径标记**: 对路径进行智能标记
- **全局统计**: 统计参数名/值的出现次数
- **超过阈值标记**: 对重复次数超过阈值的参数打标记

### 8. 域名收集 (pkg/domain_collect.go)

- **SubDomainCollect()**: 收集子域名
- **AllDomainCollect()**: 收集所有域名

### 9. 路径扩展 (pkg/path_expansion.go)

- **GetPathsFromRobots()**: 从 robots.txt 获取路径
- **GetPathsByFuzz()**: 使用内置字典进行路径 fuzz
- **GetPathsByFuzzDict()**: 使用自定义字典进行 fuzz

### 10. JavaScript 模块 (pkg/js/javascript.go)

内置多段 JavaScript 代码用于：

- **TabInitJS**: 页面初始化脚本
  - 绕过 WebDriver 检测
  - Hook history API (pushState/replaceState)
  - Hook WebSocket/EventSource/fetch
  - Hook XMLHttpRequest
  - Hook addEventListener
  - Hook window.open/close

- **ObserverJS**: DOM 变化观察脚本
- **TriggerInlineEventJS**: 触发内联事件
- **TriggerDom2EventJS**: 触发 DOM2 级事件
- **TriggerJavascriptProtocol**: 触发 JavaScript 伪协议

### 11. 工具模块 (pkg/tools)

#### common.go
- `StrMd5()`: MD5 哈希
- `ConvertHeaders()`: 转换 Header 类型
- `WriteFile()`/`ReadFile()`: 文件读写
- `MapStringFormat()`: Map 格式化

#### random.go
- `RandSeq()`: 生成随机字符串

### 12. HTTP 请求模块 (pkg/tools/requests)

- **Request()**: 通用 HTTP 请求
- **Get()**: GET 请求
- **ReqOptions**: 请求配置（超时、重试、代理、SSL）
- **Response**: 响应封装

### 13. 日志模块 (pkg/logger)

基于 logrus 的日志封装，默认级别为 Warn。

---

## 关键流程图

### 爬虫执行流程

```
1. 解析命令行参数
   ↓
2. 创建 CrawlerTask
   ├── 初始化 Browser (Chrome headless)
   ├── 创建协程池 (ants.Pool)
   └── 初始化 Filter (Simple/Smart)
   ↓
3. Run() 执行
   ├── 解析 robots.txt (可选)
   ├── 路径 fuzz (可选)
   ├── 过滤初始 URL
   └── 分配任务到协程池
   ↓
4. 每个 TabTask
   ├── 创建新 Tab
   ├── 导航到目标 URL
   ├── 拦截请求 (Fetch API)
   ├── DOMContentLoaded 后:
   │   ├── 表单自动填充
   │   └── 设置 DOM 观察器
   ├── Page Load 后:
   │   ├── 表单自动提交
   │   ├── 触发 JS 事件
   │   └── 收集链接
   └── 收集结果返回
   ↓
5. 结果聚合
   ├── 过滤去重
   ├── 收集域名/子域名
   └── 输出结果
```

### 表单填充流程

```
DOMContentLoaded 事件
   ↓
获取 <body> NodeID
   ↓
┌─────────────────┬─────────────────┬─────────────────┐
│   fillInput()   │ fillTextarea()  │ fillMultiSelect()│
│  - type=text    │                 │                 │
│  - type=email   │                 │                 │
│  - type=password│                 │                 │
│  - type=radio   │                 │                 │
│  - type=checkbox│                 │                 │
└─────────────────┴─────────────────┴─────────────────┘
   ↓
根据 name/id/class/type 匹配填充值
```

---

## 依赖关系

```
chromedp/chromedp        # Chrome DevTools Protocol Go 实现
chromedp/cdproto         # Chrome DevTools Protocol 协议定义
gogf/gf                  # Go Frame 框架 (编码处理)
deckarep/golang-set      # Go Set 实现
panjf2000/ants/v2        # 高性能协程池
sirupsen/logrus          # 日志库
urfave/cli/v2            # CLI 参数解析
golang.org/x/net         # 网络工具
pkg/errors               # 错误处理
stretchr/testify         # 测试框架
```

---

## 运行方式

### 编译

```shell
# 当前平台
make build

# 所有平台
make build_all
```

### 基本使用

```shell
# 基础爬取
./bin/crawlergo -c /path/to/chrome -t 10 http://target.com/

# JSON 输出
./bin/crawlergo -c /path/to/chrome -o json http://target.com/

# 使用代理
./bin/crawlergo -c /path/to/chrome --request-proxy socks5://127.0.0.1:7891 http://target.com/

# 路径 fuzz
./bin/crawlergo -c /path/to/chrome --fuzz-path http://target.com/

# robots.txt
./bin/crawlergo -c /path/to/chrome --robots-path http://target.com/

# 自定义表单填充
./bin/crawlergo -c /path/to/chrome -fv username=admin -fv password=123456 http://target.com/

# 推送结果到被动扫描器
./bin/crawlergo -c /path/to/chrome --push-to-proxy http://127.0.0.1:1234/ http://target.com/
```

### Docker 运行

```shell
docker build . -t crawlergo
docker run crawlergo http://target.com/
```

### Python 调用

```python
import simplejson
import subprocess

cmd = ["bin/crawlergo", "-c", "/tmp/chromium/chrome", "-o", "json", target]
rsp = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
output, error = rsp.communicate()
result = simplejson.loads(output.decode().split("--[Mission Complete]--")[1])
```

---

## 返回结果格式

```json
{
  "req_list": [           // 同域名去重后的结果
    {
      "url": "http://target.com/path",
      "method": "GET",
      "headers": {},
      "data": "",
      "source": "DOM"
    }
  ],
  "all_req_list": [],     // 所有域名的请求
  "all_domain_list": [],  // 所有发现的域名
  "sub_domain_list": []   // 子域名列表
}
```
