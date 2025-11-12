关键点一句话：  
**Handler 必须继续抛异常**（否则 HTTP 调用链会以为“请求成功”），  
**只要 UI 层（或全局）接住它，WPF 就不会崩溃**。

下面给你“三选一”的落地做法，按自己口味挑。

--------------------------------
### 方案 A：ViewModel 里就地接住（最常用）

```csharp
[RelayCommand]
private async Task LoadLogsAsync()
{
    try
    {
        var logs = await _http.GetFromJsonAsync<Log[]>("api/logs");
        Logs = new ObservableCollection<Log>(logs);
    }
    catch (HttpRequestException ex)   // Handler 抛出的异常
    {
        // 友好提示 + 不写日志可再 throw
        MessageBox.Show(ex.Message, "连接失败", MessageBoxButton.OK, MessageBoxImage.Warning);
    }
}
```
异常在此被吞掉，**再也上不了 UI 线程**，程序稳如老狗。

--------------------------------
### 方案 B：让异常飞，但给 App.xaml.cs 加全局兜底

**App.xaml.cs**

```csharp
protected override void OnStartup(StartupEventArgs e)
{
    base.OnStartup(e);

    // 1. UI 线程
    DispatcherUnhandledException += (s, args) =>
    {
        args.Handled = true;   // 阻止崩溃
        MessageBox.Show(args.Exception.Message, "错误", MessageBoxButton.OK, MessageBoxImage.Error);
    };

    // 2. 后台 Task
    TaskScheduler.UnobservedTaskException += (s, args) =>
    {
        args.SetObserved();    // 阻止崩溃
        MessageBox.Show(args.Exception.InnerException?.Message ?? args.Exception.Message,
                        "错误", MessageBoxButton.OK, MessageBoxImage.Warning);
    };
}
```
此时 ViewModel **完全不用写 try-catch**，用户只会看到统一弹窗，进程依旧存活。

--------------------------------
### 方案 C：又想抛又想不崩溃——在命令层用 `AsyncCommand` 的异常回调

CommunityToolkit.Mvvm 的 `AsyncRelayCommand` 支持 `onException`：

```csharp
LoadLogsCommand = new AsyncRelayCommand(LoadLogsAsync, ex =>
{
    MessageBox.Show(ex.Message, "连接失败", MessageBoxButton.OK, MessageBoxImage.Warning);
    return Task.CompletedTask;
});
```
异常被命令内部捕获并回调，**同样不会冒到 Dispatcher**。

--------------------------------
### 小结

| 做法 | 是否需 try-catch | 是否会崩溃 | 适用场景 |
|---|---|---|---|
| ViewModel 就地 catch | ✅ | ❌ | 最常见，可精细提示 |
| 全局 Dispatcher & Task 事件 | ❌ | ❌ | 图省事，统一提示 |
| AsyncCommand 异常回调 | ❌ | ❌ | MVVM 洁癖，代码简洁 |

**结论**：  
Handler 继续 `throw` 没关系，**只要在 UI 边界（或全局）把它接住**，WPF 就不会崩溃。