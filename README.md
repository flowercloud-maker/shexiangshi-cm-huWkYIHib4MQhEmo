
本文介绍如何将窗口置于最顶层，以及解决在顶层显示时对锁屏登录界面的影响等问题。用于实现类似Launcher、系统工具等应用需要窗口层级比Windows开始菜单以及置顶任务栏还要高的场景


**一般情况下的窗口置顶**，可以设置WPF窗口属性Topmost\=true


也可以使用WIN32\-SetWindowPos函数[SetWindowPos 函数 (winuser.h) \- Win32 apps \| Microsoft Learn](https://github.com)，设置窗口层级：




```
 1     /// 设置窗口位置
 2     /// 窗口句柄
 3     /// 跟随的窗口句柄
 4     /// X轴坐标
 5     /// Y轴坐标
 6     /// 宽
 7     /// 高
 8     /// 标志位
 9     /// 
10     [DllImport("user32.dll", SetLastError = true)]
11     public static extern bool SetWindowPos(IntPtr hwnd, IntPtr hWndInsertAfter, int x, int y, int width, int height, uint uFlags);
```


hWndInsertAfter，需要置顶可以传入参数HWND\_TOPMOST(\-1\)。设置后会在任务栏上方显示（注意：不是开始菜单显示时的任务栏，开始菜单显示后任务栏层级是超级高的，置顶层级需要再次提升，下面会讲到）


如果你软件的置顶需求是常驻，需要解决与其它置顶窗口的层级冲突、抢他们的层级，可以加个定时器：




```
 1     private nint _handle;
 2     private void MainWindow_Loaded(object sender, RoutedEventArgs e)
 3     {
 4         _handle = new WindowInteropHelper(this).Handle;
 5         SetWindowPos(_handle, -1, 0, 0, 0, 0, 1);
 6         //定时器置顶
 7         var timer = new Timer();
 8         timer.Interval = 100;
 9         timer.Elapsed += Timer_Elapsed;
10         timer.Start();
11     }
12     private void Timer_Elapsed(object? sender, System.Timers.ElapsedEventArgs e)
13     {
14         SetWindowPos(_handle, -1, 0, 0, 0, 0, 1);
15     }
```


当然，这种窗口置顶方案，遇上比你更流氓的软件就GG了，会抢来抢去。


**最上层置顶（比Windows开始菜单以及置顶任务栏还要高）**，根据我们MVP毅仔提供的方案 [让你的程序置顶到比系统界面都更上层，就像任务管理器/放大镜一样绝对置顶 \- walterlv](https://github.com)，我们简单补充整理：


1\. 添加app.manifest，并修改requestedExecutionLevel为管理员启动权限、添加UI置顶权限，详细的可以了解 [/MANIFESTUAC（将 UAC 信息嵌入到清单中） \| Microsoft Learn](https://github.com):[cmespeed楚门加速器](https://77yingba.com)



这里的窗口置顶可以设置比系统界面更高的置顶，也就是说可以比一些系统级别的置顶还要高，效果同任务管理器的绝对置顶。UiAccess可以帮应用程序绕过用户界面保护级别、并将输入引导到桌面上的更高权限窗口


2\. 给Windows设置属性ShowInTaskbar\="True"、Topmost\="True",


3\. 添加程序签名


![](https://img2024.cnblogs.com/blog/685541/202501/685541-20250107172128269-50492474.png)


4\. 将程序放在安装目录下C:\\Program Files、C:\\Program Files (x86\)。确保应用程序是从受信任的位置启动的，因为 Windows 对 UIAccess 应用程序的启动位置有严格限制。


启动后，窗口层级就比Windows开始菜单以及设置置顶的任务管理器，都要高。窗口层级关系如下，桌面\<一般应用窗口
![](https://img2024.cnblogs.com/blog/685541/202501/685541-20250107174328318-735055864.png)


层级设置没问题。我们来说下这个方案的几个问题


**1\. Windows锁屏/登录界面，置顶窗口也会显示，影响了用户操作**


解决：监听锁屏/解锁屏的事件，添加窗口Topmost修改




```
 1     public MainWindow()
 2     {
 3         InitializeComponent();
 4         //当前登录的用户变化
 5         SystemEvents.SessionSwitch += SystemEvents_SessionSwitch;
 6     }
 7     private void SystemEvents_SessionSwitch(object sender, SessionSwitchEventArgs e)
 8     {
 9         switch (e.Reason)
10         {
11             //解锁屏
12             case SessionSwitchReason.SessionUnlock:
13                 Topmost = true;
14                 break;
15             //锁屏
16             case SessionSwitchReason.SessionLock:
17                 Topmost = false;
18                 break;
19         }
20     }
```


锁屏后窗口层级隐藏效果：


![](https://img2024.cnblogs.com/blog/685541/202501/685541-20250107173749606-1714571771.gif)


**2\. 任务栏图标如果有需求需要隐藏的话，设置窗口ShowInTaskbar\=false无法隐藏图标**


这种情况下，我磨了下代码，可以这么操作：




```
1     int exStyle = GetWindowLong(hWnd, GWL_EXSTYLE);
2     // 设置窗口样式为工具窗口, 不在任务栏显示
3     exStyle |= WS_EX_TOOLWINDOW;
4     SetWindowLong(hWnd, GWL_EXSTYLE, exStyle);
5     //二次设置任务栏不显示
6     ShowInTaskbar = false;
```


使用SetWindowLong来设置窗口为工具窗口样式，然后更新窗口属性ShowInTaskbar\=false：


WS\_EX\_TOOLWINDOW 样式 \- 此扩展窗口样式用于创建工具窗口。Windows 不将此类工具窗口视为常规应用程序窗口，因此默认情况下不在任务栏中显示


ShowInTaskbar \= false \- WPF是通过设置ShowInTaskbar来实现不在任务栏中显示。


注意：需要俩个同时设置，有uiAccess的置顶应用才能隐藏任务栏


我猜测，是uiAccess会影响窗口样式的应用方式，而WS\_EX\_TOOLWINDOW结合ShowInTaskbar，明确告诉 WPF 不要在任务栏中显示此窗口，进一步确保了图标不显示出来。


此类场景置顶代码如下：




```
 1     public void SetTopmost()
 2     {
 3         IntPtr hWnd = _hWnd;
 4         // 将窗口设置为顶层窗口
 5         Topmost = true;
 6 
 7         int exStyle = GetWindowLong(hWnd, GWL_EXSTYLE);
 8         // 设置窗口样式为工具窗口, 不在任务栏显示
 9         exStyle |= WS_EX_TOOLWINDOW;
10         SetWindowLong(hWnd, GWL_EXSTYLE, exStyle);
11         //二次设置任务栏不显示
12         ShowInTaskbar = false;
13     }
```


也可以看github仓库完整代码  [kybs00/WindowsShowTopDemo](https://github.com)，需要快速验证置顶可以用这个[WindowShowTopDemo.exe](https://github.com)：


**3\. 根据小伙伴反馈，应用设置了uiAccess后，在此进程打开其它软件，其它软件内部调用SetParent实现相关功能时会失败**


这个我验证了下确实如此。目前暂无解决方案、需要具体分析，不过保底可以通过创建子进程、以进程通信去启动相关应用，来规避。


 




