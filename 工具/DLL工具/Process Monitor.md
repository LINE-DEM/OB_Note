Process Monitor 是微软 Sysinternals 套件中的一个强大工具，它可以实时监控进程的所有文件访问、注册表操作和网络活动。这是最直接和最准确的验证方法。

首先从微软官网下载并安装 Process Monitor。启动这个工具后，你会看到系统中所有进程的实时活动，数据流会非常快。你需要设置过滤器来只显示与 Unity 相关的活动。

在 Process Monitor 的菜单栏中，点击 Filter 然后选择 Filter。在过滤器对话框中，添加一个新的规则：Process Name 包含 Unity.exe（或者更准确地说是 Unity 编辑器的进程名，可能是类似 Unity.exe 或者你的项目名.exe）。同时添加另一个规则：Path 包含 onnxruntime.dll。这样可以只显示 Unity 进程尝试访问 onnxruntime.dll 的操作。

现在在 Unity 中运行你的场景。观察 Process Monitor 的输出，你会看到一系列的文件操作记录。关注 Operation 列显示为 CreateFile 或 Load Image 的条目，这些表示进程正在尝试打开或加载文件。在 Path 列中，你会看到完整的文件路径，这就是 Unity 实际尝试访问的文件位置。

如果你看到多个针对不同路径的 onnxruntime.dll 的访问尝试，这表明 Windows 加载器正在按照搜索顺序尝试多个位置。查看 Result 列，成功的操作会显示 SUCCESS，而失败的操作会显示 NAME NOT FOUND 或其他错误代码。找到第一个显示 SUCCESS 的条目，它的 Path 就是 Unity 实际成功加载的 DLL 文件的完整路径。