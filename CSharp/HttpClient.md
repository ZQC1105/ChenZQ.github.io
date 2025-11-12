## [HttpClient 类](https://learn.microsoft.com/zh-cn/dotnet/api/system.net.http.httpclient?view=net-9.0)
## [使用 IHttpClientFactory 的最佳实践（.NET）](https://learn.microsoft.com/zh-cn/dotnet/core/extensions/httpclient-factory)

下面给出两段最常用、可直接复制到 VS 跑起来的 WPF 最小实例，任选其一即可。

--------------------------------------------------
方案 A：「DI 容器 + 构造函数注入」——推荐做法  
--------------------------------------------------
1. 装包  
   ```bash
   Install-Package Microsoft.Extensions.Http
   Install-Package Microsoft.Extensions.DependencyInjection
   ```

2. App.xaml.cs  
   ```csharp
   public partial class App : Application
   {
       public static IServiceProvider Services { get; private set; }

       protected override void OnStartup(StartupEventArgs e)
       {
           var services = new ServiceCollection();
           ConfigureServices(services);
           Services = services.BuildServiceProvider();

           // 把主窗口交给 DI 解析
           var mainWindow = Services.GetRequiredService<MainWindow>();
           mainWindow.Show();

           base.OnStartup(e);
       }

       private void ConfigureServices(IServiceCollection services)
       {
           // 1. 注册 IHttpClientFactory
           services.AddHttpClient("GitHub", c =>
           {
               c.BaseAddress = new Uri("https://api.github.com/");
               c.DefaultRequestHeaders.UserAgent.ParseAdd("WpfDemo");
           });

           // 2. 注册自己需要的服务 / VM / 窗口
           services.AddTransient<MainViewModel>();
           services.AddTransient<MainWindow>();
       }
   }
   ```

3. MainViewModel.cs  
   ```csharp
   public class MainViewModel
   {
       private readonly IHttpClientFactory _httpClientFactory;
       public string Result { get; set; } = "等待点击…";

       public ICommand LoadCmd { get; }

       public MainViewModel(IHttpClientFactory httpClientFactory)
       {
           _httpClientFactory = httpClientFactory;
           LoadCmd = new RelayCommand(async () => await LoadAsync());
       }

       private async Task LoadAsync()
       {
           var http = _httpClientFactory.CreateClient("GitHub");
           Result = await http.GetStringAsync("repos/microsoft/dotnet");
       }
   }
   ```

4. MainWindow.xaml（仅放核心）  
   ```xml
   <StackPanel>
       <Button Content="获取 GitHub 仓库信息"
               Command="{Binding LoadCmd}"
               HorizontalAlignment="Center" Padding="20,5"/>
       <TextBox Text="{Binding Result}" TextWrapping="Wrap"
                IsReadOnly="True" Margin="10"/>
   </StackPanel>
   ```

5. MainWindow.xaml.cs  
   ```csharp
   public partial class MainWindow : Window
   {
       public MainWindow(MainViewModel vm)
       {
           InitializeComponent();
           DataContext = vm;   // 通过构造函数注入 VM
       }
   }
   ```

运行效果：点击按钮 → 用 IHttpClientFactory 创建的客户端去调 GitHub API → 结果直接显示在 TextBox。


--------------------------------------------------
小结  
--------------------------------------------------
- WPF 没有内置的通用主机，但只要手动 `new ServiceCollection()` → `AddHttpClient()` → `BuildServiceProvider()`，就能完整享用 ASP.NET Core 那套 IHttpClientFactory 的全部能力。  
- 官方文档同样适用于桌面程序，无需任何额外适配。
  
# IHttpClientFactory 优点一览

&gt; 适用于 WPF、WinForms、MAUI 等桌面客户端，亦适用于 ASP.NET Core。

| 优点类别                          | 具体说明                                                                                                | 对 WPF/桌面开发的意义                                     |
| --------------------------------- | ------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| **1. 连接池与套接字复用**         | 每个命名/类型客户端共享 `HttpMessageHandler`，自动复用 TCP 连接，避免“套接字耗尽”异常。                 | 长生命周期的桌面应用反复调用 API 时，不会把本地端口占光。 |
| **2. 生命周期自动管理**           | 工厂按策略（默认 2 min）回收 Handler，解决 DNS 刷新与连接老化问题；开发者无需手动 `Dispose` 客户端。    | 避免“改完 DNS 必须重启程序”的尴尬。                       |
| **3. 出站策略集中配置**           | 在 `AddHttpClient("name")` 里统一写 BaseAddress、Header、超时、代理、证书等；换环境只需改一处。         | 桌面应用同样有测试/生产多环境需求，配置集中好维护。       |
| **4. Polly 弹性策略一键集成**     | 与 `Microsoft.Extensions.Http.Polly` 结合，可链式声明重试、熔断、超时、回退： `AddPolicyHandler(...)`。 | 网络抖动常见，桌面端也能“自动重试 3 次再弹错误框”。       |
| **5. 日志与遥测开箱即用**         | 内置 `ILogger` 事件（请求、耗时、状态码），与 Serilog/NLog 等对接即可在本地文件或调试窗口看日志。       | 排查用户现场问题时有现成的 HTTP 轨迹。                    |
| **6. 模拟与单元测试友好**         | 注册 `HttpMessageHandler` 的 Mock 实现即可伪造响应，无需改业务代码；支持 Typed Client 接口注入。        | ViewModel 单元测试可脱离真实网络，CI 跑得更快。           |
| **7. Typed Client 强类型**        | 把 HttpClient 封装到业务类构造函数，DI 直接注入 `MyTypedClient`，告别到处写原始 URL。                   | 桌面 MVVM 里 Service/Repository 层同样享受强类型。        |
| **8. 性能更高**                   | 官方基准：复用 Handler 比每次 `new HttpClient()` 减少约 60% 内存分配，提升 2~3 倍吞吐。                 | 后台同步任务多、UI 要保持流畅时尤其明显。                 |
| **9. 与 ASP.NET Core 同一套生态** | 学习一次，全平台通用；代码可从 Web 项目几乎零改动搬到 WPF/WinForms/Xamarin/MAUI。                       | 团队技能栈统一，减少上下文切换。                          |

---

## 一句话总结
**IHttpClientFactory = “会复用、易配置、能重试、好测试、省端口、写一次全平台通用”**，  
在桌面端同样值得第一时间用上。

| 概念                   | 一句话定义                                                       | 在 `AddHttpClient` 里的常见写法（片段）                                | 桌面/WPF 场景举例                                                                |
| ---------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **地址 (BaseAddress)** | 把“共同前缀”抽出来，业务代码只写剩余路径。                       | `c.BaseAddress = new Uri("https://api.github.com/");`                  | 测试环境 `http://localhost:5000`，生产 `https://api.company.com`——只改配置文件。 |
| **Header**             | 每次请求都要带的“元数据”：User-Agent、Authorization、Accept 等。 | `c.DefaultRequestHeaders.UserAgent.ParseAdd("WpfDemo");`               | 调用 GitHub 必须带 User-Agent；调用自家 API 带 `Authorization: Bearer <token>`。 |
| **超时 (Timeout)**     | 从发起请求到读完响应的最大等待时间，防“卡死”。                   | `c.Timeout = TimeSpan.FromSeconds(30);`                                | 公司内网 API 设 10 秒，外网服务设 30 秒；用户可接受时再弹“网络异常”。            |
| **代理 (Proxy)**       | 请求先经过的中间服务器；常用于抓包、翻墙、公司防火墙。           | `c.Proxy = new WebProxy("http://127.0.0.1:8888");`                     | Fiddler/Charles 抓包调试；企业内网需通过代理出网。                               |
| **证书 (Certificate)** | TLS 客户端证书或自定义服务器证书验证逻辑。                       | `c.ClientCertificates.Add(new X509Certificate2("client.pfx", "pwd"));` | 银行 UKey、政府网关要求双向 TLS；测试环境用自签证书需自定义验证回调。            |
| **Polly 策略**         | 失败自动重试、熔断、超时、回退等“弹性”组合。                     | `services.AddHttpClient("demo").AddPolicyHandler(retryPolicy);`        | 笔记本休眠恢复后网络瞬断，自动重试 3 次再弹错误框，避免用户立刻看到“请求失败”。  |

在 .NET 中，`HttpClient` 的工厂模式（通常指 `IHttpClientFactory`）的核心作用是解决传统 HttpClient 使用方式带来的资源浪费、Socket 耗尽、DNS 更新不生效等问题，同时提供集中配置、日志集成、生命周期管理、策略扩展（如重试、熔断）等能力。

---

✅ 传统 HttpClient 的痛点

```csharp
// 常见错误用法
var client = new HttpClient();
```

- Socket 耗尽：`HttpClient` 实现了 `IDisposable`，但频繁创建/销毁会导致端口占用不释放（TIME_WAIT）。
- DNS 不更新：`HttpClient` 默认复用连接，DNS 变更后不会刷新（长期运行的服务）。
- 无法集中配置：每个实例需重复设置 BaseAddress、Header、超时等。
- 难以单元测试：直接依赖具体实现，无法 mock。

---

✅ 工厂模式（IHttpClientFactory）的解决方案

1. 池化 Handler，避免 Socket 耗尽
- 工厂内部复用 `HttpMessageHandler`（如 `SocketsHttpHandler`），共享连接池，避免频繁创建连接。
- 生命周期由工厂管理，不会无限增长。

2. DNS 定期刷新
- 通过 `HandlerLifetime`（默认 2 分钟）定期轮换 Handler，强制重新解析 DNS。

3. 集中配置命名客户端

```csharp
services.AddHttpClient("GitHub", client =>
{
    client.BaseAddress = new Uri("https://api.github.com/");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp");
});
```

- 通过名称区分不同配置（如 GitHub API、微信 API），避免重复代码。

4. 集成 Polly 策略（重试、熔断、超时）

```csharp
services.AddHttpClient("GitHub")
    .AddPolicyHandler(PollyPolicies.RetryAsync());
```

- 通过扩展方法链式配置策略，与业务代码解耦。

5. 日志与诊断
- 自动集成 `ILogger`，记录请求耗时、状态码、异常等（通过 `HttpClient` 的 `LoggingScope`）。

6. 支持 Typed Client（强类型）

```csharp
public class GitHubService
{
    private readonly HttpClient _client;
    public GitHubService(HttpClient client) => _client = client;
}

services.AddHttpClient<GitHubService>();
```

- 通过依赖注入直接注入 `HttpClient`，无需关心工厂，便于单元测试（mock `HttpClient`）。

---

✅ 总结：工厂模式的 5 大作用

问题工厂模式解决方案
Socket 耗尽池化 Handler，共享连接
DNS 不更新定期轮换 Handler
重复配置命名客户端集中配置
策略扩展集成 Polly（重试、熔断）
测试困难Typed Client + 依赖注入

---

❗注意
- 不要在 `AddHttpClient` 中直接 `new HttpClient()`，否则会失去工厂的所有优势。
- 不要手动管理 `HttpClient` 的生命周期（由工厂自动处理）。

一句话区别：  
| 写法 | 拿到的是什么 | 生命周期 | 给谁用 | 典型场景 |
|---|---|---|---|---|
| `services.AddHttpClient();` | `IHttpClientFactory` | 单例 | 你自己 `factory.CreateClient()` | 手动创建、随意命名 |
| `services.AddHttpClient("WPF_API", …)` | 命名实例 | 单例 | 你自己 `factory.CreateClient("WPF_API")` | 一个业务一个配置 |
| `services.AddHttpClient<LogViewModel>()` | **Typed Client** | **每次 new LogViewModel 时注入专属 HttpClient** | 直接注入到 `LogViewModel` 构造函数 | MVVM、一个类一个客户端 |

---

### 1. 第一种：`AddHttpClient()`
- 只把 **IHttpClientFactory** 注册到容器  
- 你在构造函数里注入 `IHttpClientFactory`，然后  
  ```csharp
  var client = _factory.CreateClient();          // 默认配置
  var named = _factory.CreateClient("WPF_API");  // 用命名配置
  ```
- **手动管理** client 的名字、配置、BaseAddress  
- 适合“我要写很多不同地址/不同超时”的场景

---

### 2. 第二种：`AddHttpClient("WPF_API", …)`
- 仍然是 **IHttpClientFactory** 模式，只是提前帮你放好一个“命名配置”  
- 使用时：
  ```csharp
  var client = _factory.CreateClient("WPF_API");
  ```
- 一个名字对应一套 `BaseAddress`、超时、Header、Handler 等  
- 适合“同一个后端多个业务模块”时给每个模块起个名字

---

### 3. 第三种：`AddHttpClient<LogViewModel>()`
- 叫做 **Typed Client**（强类型客户端）  
- **不会给你 IHttpClientFactory**，而是直接在你 new `LogViewModel` 时，**把已经配置好的 HttpClient 注入到它的构造函数**  
  ```csharp
  public LogViewModel(HttpClient http)   // 这里收到的就是专属实例
  {
      _http = http;
  }
  ```
- 生命周期跟随 `LogViewModel`（默认 Transient）  
- 配置写法：
  ```csharp
  services.AddHttpClient<LogViewModel>(c =>
  {
      c.BaseAddress = new Uri("https://localhost:7058/");
      c.Timeout = TimeSpan.FromSeconds(10);
  })
  .AddHttpMessageHandler<GlobalHttpExceptionHandler>();
  ```
- **最适合 MVVM**：一个 ViewModel 对应一个 HttpClient，配置、异常处理、重试策略全部封装好，**不需要管名字，也不需要手动 CreateClient**

---

### 一张表记住

| 调用方式 | 得到的对象 | 需不需要名字 | 推荐场景 |
|---|---|---|---|
| `AddHttpClient()` | `IHttpClientFactory` | 需要 | 手动灵活创建 |
| `AddHttpClient("xxx")` | 同上，只是预配了命名实例 | 需要 | 多后端/多配置 |
| `AddHttpClient<T>()` | 直接注入 `HttpClient` 到 T 的构造函数 | 不需要 | 一个 ViewModel/Service 对应一个客户端，最干净 |

---

### 你该选哪个？
- **WPF + MVVM** → 直接 `AddHttpClient<LogViewModel>()`，** typed client 最省心**  
- 还想共享同一个配置给多个 VM → 先 `AddHttpClient("WPF_API",…)`，再在 VM 里 `factory.CreateClient("WPF_API")`  
- 不需要 DI，只想手动创建 → 第一种

---

把 `AddHttpClient<LogViewModel>()` 换成你原来的命名写法也行，只是要多写一行 `CreateClient("WPF_API")`，看你喜欢哪种风格。