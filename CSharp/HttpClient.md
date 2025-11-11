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
