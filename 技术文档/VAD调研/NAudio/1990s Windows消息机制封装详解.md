

## 第一部分：Windows消息机制基础

### 1.1 什么是Windows消息机制

Windows消息机制是Windows操作系统的核心，基于**事件驱动模型**。

```
用户操作 → 产生消息 → 消息队列 → 应用程序处理
```

**核心概念：**

- **消息（Message）** - 系统或应用程序事件的通知
- **消息队列** - 每个线程有自己的消息队列
- **窗口过程** - 处理消息的函数
- **消息循环** - 不断从队列中取出消息并分发

### 1.2 原始的Win32 API方式（1990s标准）

#### 最基础的Windows程序

```c
#include <windows.h>

// 窗口过程函数（Window Procedure）
LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    switch(uMsg)
    {
        case WM_DESTROY:
            PostQuitMessage(0);
            return 0;
        
        case WM_PAINT:
        {
            PAINTSTRUCT ps;
            HDC hdc = BeginPaint(hwnd, &ps);
            TextOut(hdc, 10, 10, "Hello Windows!", 14);
            EndPaint(hwnd, &ps);
            return 0;
        }
        
        case WM_LBUTTONDOWN:
        {
            MessageBox(hwnd, "You clicked!", "Info", MB_OK);
            return 0;
        }
        
        default:
            return DefWindowProc(hwnd, uMsg, wParam, lParam);
    }
}

// WinMain - Windows程序的入口点
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance,
                   LPSTR lpCmdLine, int nCmdShow)
{
    // 1. 注册窗口类
    WNDCLASS wc = {0};
    wc.lpfnWndProc = WindowProc;          // 窗口过程函数
    wc.hInstance = hInstance;
    wc.lpszClassName = "MyWindowClass";
    wc.hCursor = LoadCursor(NULL, IDC_ARROW);
    wc.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1);
    
    RegisterClass(&wc);
    
    // 2. 创建窗口
    HWND hwnd = CreateWindowEx(
        0,
        "MyWindowClass",
        "My First Window",
        WS_OVERLAPPEDWINDOW,
        CW_USEDEFAULT, CW_USEDEFAULT,
        640, 480,
        NULL, NULL, hInstance, NULL
    );
    
    if (hwnd == NULL) return 0;
    
    ShowWindow(hwnd, nCmdShow);
    UpdateWindow(hwnd);
    
    // 3. 消息循环
    MSG msg = {0};
    while(GetMessage(&msg, NULL, 0, 0))
    {
        TranslateMessage(&msg);  // 转换键盘消息
        DispatchMessage(&msg);   // 分发消息到窗口过程
    }
    
    return msg.wParam;
}
```

### 1.3 消息流程图

```
┌─────────────────────────────────────────────────────────────┐
│                     用户操作/系统事件                         │
│              (鼠标点击、键盘输入、定时器等)                   │
└───────────────────────┬─────────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────────────┐
│                  Windows 系统                              │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  将事件转换为消息 (Message)                         │  │
│  │  - uMsg: 消息类型 (WM_PAINT, WM_LBUTTONDOWN...)   │  │
│  │  - wParam: 消息参数1                               │  │
│  │  - lParam: 消息参数2                               │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────┬───────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────────────┐
│              线程消息队列 (Message Queue)                  │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                      │
│  │ MSG │→ │ MSG │→ │ MSG │→ │ MSG │→ ...                 │
│  └─────┘  └─────┘  └─────┘  └─────┘                      │
└───────────────────────┬───────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────────────┐
│                 消息循环 (Message Loop)                    │
│                                                            │
│  while(GetMessage(&msg, NULL, 0, 0))                      │
│  {                                                         │
│      TranslateMessage(&msg);  ← 转换虚拟键消息            │
│      DispatchMessage(&msg);   ← 分发到窗口过程            │
│  }                                                         │
└───────────────────────┬───────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────────────┐
│           窗口过程 (Window Procedure)                      │
│                                                            │
│  LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg,       │
│                               WPARAM wParam, LPARAM lParam)│
│  {                                                         │
│      switch(uMsg) {                                       │
│          case WM_PAINT:                                   │
│              // 处理绘制                                  │
│              break;                                        │
│          case WM_LBUTTONDOWN:                             │
│              // 处理鼠标点击                              │
│              break;                                        │
│          default:                                          │
│              return DefWindowProc(...);                   │
│      }                                                     │
│  }                                                         │
└───────────────────────────────────────────────────────────┘
```

---

## 第二部分：为什么需要封装

### 2.1 原始Win32 API的痛点

#### 痛点1：巨大的switch语句

```c
LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    switch(uMsg)
    {
        case WM_CREATE: // ...
        case WM_DESTROY: // ...
        case WM_PAINT: // ...
        case WM_SIZE: // ...
        case WM_MOVE: // ...
        case WM_LBUTTONDOWN: // ...
        case WM_RBUTTONDOWN: // ...
        case WM_KEYDOWN: // ...
        case WM_KEYUP: // ...
        case WM_COMMAND: // ...
        case WM_NOTIFY: // ...
        // ... 可能有几十甚至上百个case
        default:
            return DefWindowProc(hwnd, uMsg, wParam, lParam);
    }
}
```

**问题：**

- 代码冗长难以维护
- 一个函数处理所有消息，职责不清
- 无法复用消息处理逻辑

#### 痛点2：C语言无面向对象特性

```c
// 如何关联窗口句柄(HWND)和应用程序数据？
// 只能用全局变量或复杂的映射表

// 全局变量方式（不推荐）
MyAppData g_appData;

// 映射表方式（复杂）
std::map<HWND, MyAppData*> g_windowMap;
```

#### 痛点3：代码重复

每个窗口都要写：

- 注册窗口类
- 创建窗口
- 消息循环
- 窗口过程

#### 痛点4：wParam和lParam的类型不安全

```c
// wParam 和 lParam 都是整数类型，需要手动转换
case WM_COMMAND:
{
    WORD wNotifyCode = HIWORD(wParam);  // 手动拆分
    WORD wID = LOWORD(wParam);
    HWND hwndCtl = (HWND)lParam;
    // 容易出错！
}
```

### 2.2 封装的目标

```
原始Win32 API          封装后
┌─────────────┐        ┌──────────────────┐
│ C语言风格    │   →   │ C++面向对象      │
│ 过程式编程   │        │ 消息映射宏       │
│ 全局函数     │        │ 虚函数重载       │
│ switch/case  │        │ 类型安全         │
│ 手动管理资源 │        │ RAII自动管理     │
└─────────────┘        └──────────────────┘
```

---

## 第三部分：MFC的消息映射机制（经典封装）

### 3.1 MFC简介

**MFC (Microsoft Foundation Classes)** 是微软在1992年推出的C++类库，是1990s最流行的Windows编程框架。

### 3.2 MFC的核心思想

#### 思想1：窗口类 = C++类

```cpp
// MFC方式
class CMyWindow : public CWnd
{
    DECLARE_MESSAGE_MAP()
    
public:
    afx_msg void OnPaint();
    afx_msg void OnLButtonDown(UINT nFlags, CPoint point);
    afx_msg void OnDestroy();
};
```

#### 思想2：消息映射表

```cpp
BEGIN_MESSAGE_MAP(CMyWindow, CWnd)
    ON_WM_PAINT()
    ON_WM_LBUTTONDOWN()
    ON_WM_DESTROY()
END_MESSAGE_MAP()
```

**工作原理：** 宏展开后生成一个静态表，将消息ID映射到成员函数指针。

### 3.3 完整的MFC示例

```cpp
// MyWindow.h
class CMyWindow : public CFrameWnd
{
    DECLARE_MESSAGE_MAP()
    
public:
    CMyWindow();
    
protected:
    afx_msg void OnPaint();
    afx_msg void OnLButtonDown(UINT nFlags, CPoint point);
    afx_msg void OnDestroy();
    
private:
    CString m_strText;
};

// MyWindow.cpp
CMyWindow::CMyWindow()
{
    m_strText = "Hello MFC!";
    Create(NULL, "MFC Window Example");
}

BEGIN_MESSAGE_MAP(CMyWindow, CFrameWnd)
    ON_WM_PAINT()
    ON_WM_LBUTTONDOWN()
    ON_WM_DESTROY()
END_MESSAGE_MAP()

void CMyWindow::OnPaint()
{
    CPaintDC dc(this);  // RAII方式管理DC
    dc.TextOut(10, 10, m_strText);
}

void CMyWindow::OnLButtonDown(UINT nFlags, CPoint point)
{
    m_strText.Format("Clicked at (%d, %d)", point.x, point.y);
    Invalidate();  // 触发重绘
}

void CMyWindow::OnDestroy()
{
    PostQuitMessage(0);
}

// App.cpp
class CMyApp : public CWinApp
{
public:
    BOOL InitInstance() override
    {
        m_pMainWnd = new CMyWindow();
        m_pMainWnd->ShowWindow(m_nCmdShow);
        m_pMainWnd->UpdateWindow();
        return TRUE;
    }
};

CMyApp theApp;  // 全局应用程序对象
```

### 3.4 MFC消息映射的实现原理

#### 宏展开过程

```cpp
// 源代码
BEGIN_MESSAGE_MAP(CMyWindow, CFrameWnd)
    ON_WM_PAINT()
    ON_WM_LBUTTONDOWN()
END_MESSAGE_MAP()

// 宏展开后（简化版）
const AFX_MSGMAP_ENTRY CMyWindow::_messageEntries[] = 
{
    { WM_PAINT, 0, 0, 0, AfxSig_vv,
      (AFX_PMSG)(void (CWnd::*)())&CMyWindow::OnPaint },
    { WM_LBUTTONDOWN, 0, 0, 0, AfxSig_vwp,
      (AFX_PMSG)(void (CWnd::*)(UINT, CPoint))&CMyWindow::OnLButtonDown },
    { 0, 0, 0, 0, AfxSig_end, (AFX_PMSG)0 }
};

const AFX_MSGMAP CMyWindow::messageMap =
{
    &CFrameWnd::GetThisMessageMap,  // 基类的消息映射
    &CMyWindow::_messageEntries[0]
};

const AFX_MSGMAP* CMyWindow::GetMessageMap() const
{
    return &CMyWindow::messageMap;
}
```

#### 消息分发流程

```cpp
// MFC内部的WindowProc（简化版）
LRESULT CALLBACK AfxWndProc(HWND hWnd, UINT nMsg, WPARAM wParam, LPARAM lParam)
{
    // 1. 从HWND获取C++对象指针
    CWnd* pWnd = CWnd::FromHandlePermanent(hWnd);
    
    if (pWnd == NULL)
        return DefWindowProc(hWnd, nMsg, wParam, lParam);
    
    // 2. 调用对象的WindowProc
    return pWnd->WindowProc(nMsg, wParam, lParam);
}

LRESULT CWnd::WindowProc(UINT message, WPARAM wParam, LPARAM lParam)
{
    // 3. 查找消息映射表
    const AFX_MSGMAP* pMessageMap = GetMessageMap();
    
    for (const AFX_MSGMAP_ENTRY* lpEntry = pMessageMap->lpEntries;
         lpEntry->nSig != AfxSig_end; lpEntry++)
    {
        if (lpEntry->nMessage == message)
        {
            // 4. 找到匹配的消息，调用成员函数
            union MessageMapFunctions mmf;
            mmf.pfn = lpEntry->pfn;
            
            switch(lpEntry->nSig)
            {
                case AfxSig_vv:  // void Func()
                    (this->*mmf.pfn_vv)();
                    return 0;
                    
                case AfxSig_vwp:  // void Func(UINT, CPoint)
                    {
                        CPoint pt(lParam);
                        (this->*mmf.pfn_vwp)(wParam, pt);
                        return 0;
                    }
                // ... 更多签名类型
            }
        }
    }
    
    // 5. 没找到，调用默认处理
    return DefWindowProc(m_hWnd, message, wParam, lParam);
}
```

### 3.5 MFC的优势

```cpp
// 对比：Win32 vs MFC

// Win32方式 - 需要70+行代码
LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    switch(uMsg) {
        case WM_PAINT: /* 10行代码 */ break;
        case WM_LBUTTONDOWN: /* 5行代码 */ break;
        // ... 更多消息
    }
}

// MFC方式 - 更清晰
class CMyWindow : public CFrameWnd
{
protected:
    afx_msg void OnPaint() { /* 10行代码 */ }
    afx_msg void OnLButtonDown(UINT nFlags, CPoint point) { /* 5行代码 */ }
};
```

**优势：**

1. ✅ **面向对象** - 每个窗口是一个类
2. ✅ **代码复用** - 继承基类获得大量功能
3. ✅ **类型安全** - 参数类型明确（CPoint而非LPARAM）
4. ✅ **职责分离** - 每个消息一个函数
5. ✅ **RAII管理** - 资源自动释放（CPaintDC, CString等）

---

## 第四部分：ATL/WTL的轻量级封装

### 4.1 ATL简介

**ATL (Active Template Library)** 是微软推出的基于模板的轻量级框架，比MFC小很多。

### 4.2 ATL的消息映射

```cpp
#include <atlbase.h>
#include <atlwin.h>

class CMyWindow : public CWindowImpl<CMyWindow>
{
public:
    DECLARE_WND_CLASS(NULL)
    
    BEGIN_MSG_MAP(CMyWindow)
        MESSAGE_HANDLER(WM_PAINT, OnPaint)
        MESSAGE_HANDLER(WM_LBUTTONDOWN, OnLButtonDown)
        MESSAGE_HANDLER(WM_DESTROY, OnDestroy)
    END_MSG_MAP()
    
    LRESULT OnPaint(UINT /*uMsg*/, WPARAM /*wParam*/, LPARAM /*lParam*/, BOOL& /*bHandled*/)
    {
        CPaintDC dc(m_hWnd);
        dc.TextOut(10, 10, _T("Hello ATL!"));
        return 0;
    }
    
    LRESULT OnLButtonDown(UINT /*uMsg*/, WPARAM /*wParam*/, LPARAM lParam, BOOL& /*bHandled*/)
    {
        int x = GET_X_LPARAM(lParam);
        int y = GET_Y_LPARAM(lParam);
        MessageBox(_T("Clicked!"), _T("Info"), MB_OK);
        return 0;
    }
    
    LRESULT OnDestroy(UINT /*uMsg*/, WPARAM /*wParam*/, LPARAM /*lParam*/, BOOL& /*bHandled*/)
    {
        PostQuitMessage(0);
        return 0;
    }
};

int WINAPI _tWinMain(HINSTANCE hInstance, HINSTANCE /*hPrevInstance*/,
                     LPTSTR /*lpCmdLine*/, int nCmdShow)
{
    CMyWindow wnd;
    wnd.Create(NULL, CWindow::rcDefault, _T("ATL Window"));
    wnd.ShowWindow(nCmdShow);
    
    MSG msg;
    while(GetMessage(&msg, NULL, 0, 0))
    {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
    return msg.wParam;
}
```

### 4.3 ATL vs MFC

|特性|MFC|ATL|
|---|---|---|
|大小|~1-2MB|~100-200KB|
|学习曲线|陡峭|中等|
|模板使用|少|大量使用|
|编译速度|慢|较快|
|适用场景|完整应用|COM组件、控件|

---

## 第五部分：自定义简化封装

### 5.1 最小化的消息映射封装

```cpp
// SimpleWindow.h
#include <windows.h>
#include <map>
#include <functional>

class SimpleWindow
{
public:
    SimpleWindow() : m_hwnd(NULL) {}
    virtual ~SimpleWindow() { if (m_hwnd) DestroyWindow(m_hwnd); }
    
    bool Create(const char* title, int width, int height)
    {
        // 注册窗口类
        WNDCLASS wc = {0};
        wc.lpfnWndProc = StaticWindowProc;
        wc.hInstance = GetModuleHandle(NULL);
        wc.lpszClassName = "SimpleWindowClass";
        wc.hCursor = LoadCursor(NULL, IDC_ARROW);
        wc.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1);
        RegisterClass(&wc);
        
        // 创建窗口
        m_hwnd = CreateWindowEx(
            0, "SimpleWindowClass", title,
            WS_OVERLAPPEDWINDOW,
            CW_USEDEFAULT, CW_USEDEFAULT, width, height,
            NULL, NULL, GetModuleHandle(NULL), this  // 传递this指针
        );
        
        return m_hwnd != NULL;
    }
    
    void Show(int nCmdShow)
    {
        ShowWindow(m_hwnd, nCmdShow);
        UpdateWindow(m_hwnd);
    }
    
    static int MessageLoop()
    {
        MSG msg;
        while(GetMessage(&msg, NULL, 0, 0))
        {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
        }
        return (int)msg.wParam;
    }
    
protected:
    // 消息处理器类型
    using MessageHandler = std::function<LRESULT(WPARAM, LPARAM)>;
    
    // 注册消息处理器
    void RegisterMessage(UINT msg, MessageHandler handler)
    {
        m_messageMap[msg] = handler;
    }
    
    // 虚函数：子类可以重载
    virtual void OnCreate() {}
    virtual void OnDestroy() { PostQuitMessage(0); }
    virtual void OnPaint() {}
    
    HWND m_hwnd;
    
private:
    // 静态窗口过程
    static LRESULT CALLBACK StaticWindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
    {
        SimpleWindow* pThis = NULL;
        
        if (uMsg == WM_NCCREATE)
        {
            // 从创建参数中获取this指针
            CREATESTRUCT* pCreate = (CREATESTRUCT*)lParam;
            pThis = (SimpleWindow*)pCreate->lpCreateParams;
            SetWindowLongPtr(hwnd, GWLP_USERDATA, (LONG_PTR)pThis);
            pThis->m_hwnd = hwnd;
        }
        else
        {
            pThis = (SimpleWindow*)GetWindowLongPtr(hwnd, GWLP_USERDATA);
        }
        
        if (pThis)
            return pThis->WindowProc(uMsg, wParam, lParam);
        
        return DefWindowProc(hwnd, uMsg, wParam, lParam);
    }
    
    // 实例窗口过程
    LRESULT WindowProc(UINT uMsg, WPARAM wParam, LPARAM lParam)
    {
        // 先查找自定义消息处理器
        auto it = m_messageMap.find(uMsg);
        if (it != m_messageMap.end())
        {
            return it->second(wParam, lParam);
        }
        
        // 默认消息处理
        switch(uMsg)
        {
            case WM_CREATE:
                OnCreate();
                return 0;
            case WM_DESTROY:
                OnDestroy();
                return 0;
            case WM_PAINT:
                OnPaint();
                return 0;
        }
        
        return DefWindowProc(m_hwnd, uMsg, wParam, lParam);
    }
    
    std::map<UINT, MessageHandler> m_messageMap;
};

// 使用示例
class MyWindow : public SimpleWindow
{
protected:
    void OnCreate() override
    {
        // 注册自定义消息处理器
        RegisterMessage(WM_LBUTTONDOWN, [this](WPARAM w, LPARAM l) {
            int x = LOWORD(l);
            int y = HIWORD(l);
            MessageBox(m_hwnd, "Clicked!", "Info", MB_OK);
            return 0;
        });
    }
    
    void OnPaint() override
    {
        PAINTSTRUCT ps;
        HDC hdc = BeginPaint(m_hwnd, &ps);
        TextOut(hdc, 10, 10, "Simple Window Framework", 23);
        EndPaint(m_hwnd, &ps);
    }
};

// 主程序
int WINAPI WinMain(HINSTANCE, HINSTANCE, LPSTR, int nCmdShow)
{
    MyWindow window;
    window.Create("Simple Window", 640, 480);
    window.Show(nCmdShow);
    return SimpleWindow::MessageLoop();
}
```

### 5.2 设计要点

**关键技巧1：HWND → C++对象的映射**

```cpp
// 在WM_NCCREATE时保存this指针
SetWindowLongPtr(hwnd, GWLP_USERDATA, (LONG_PTR)pThis);

// 后续消息中取回
pThis = (SimpleWindow*)GetWindowLongPtr(hwnd, GWLP_USERDATA);
```

**关键技巧2：静态窗口过程 + 实例窗口过程**

```cpp
静态WindowProc（全局函数）
    ↓
获取C++对象指针
    ↓
调用对象的WindowProc（成员函数）
```

**关键技巧3：Lambda表达式注册消息**

```cpp
RegisterMessage(WM_LBUTTONDOWN, [this](WPARAM w, LPARAM l) {
    // 处理消息，可以访问成员变量
    return 0;
});
```

---

## 第六部分：消息类型详解

### 6.1 常见Windows消息分类

#### 窗口管理消息

```cpp
WM_CREATE      // 窗口创建
WM_DESTROY     // 窗口销毁
WM_CLOSE       // 关闭请求
WM_SIZE        // 窗口大小改变
WM_MOVE        // 窗口移动
WM_SHOWWINDOW  // 显示/隐藏窗口
```

#### 绘制消息

```cpp
WM_PAINT       // 需要重绘
WM_ERASEBKGND  // 擦除背景
WM_NCPAINT     // 非客户区绘制
```

#### 输入消息

```cpp
// 鼠标
WM_LBUTTONDOWN  // 左键按下
WM_LBUTTONUP    // 左键释放
WM_RBUTTONDOWN  // 右键按下
WM_MOUSEMOVE    // 鼠标移动
WM_MOUSEWHEEL   // 鼠标滚轮

// 键盘
WM_KEYDOWN      // 键按下
WM_KEYUP        // 键释放
WM_CHAR         // 字符输入
```

#### 系统消息

```cpp
WM_TIMER        // 定时器
WM_COMMAND      // 菜单/工具栏命令
WM_NOTIFY       // 通用通知
WM_SYSCOMMAND   // 系统命令
```

### 6.2 消息参数解析

```cpp
// WM_LBUTTONDOWN
LRESULT OnLButtonDown(WPARAM wParam, LPARAM lParam)
{
    // wParam: 键盘修饰键状态
    bool ctrlPressed = (wParam & MK_CONTROL);
    bool shiftPressed = (wParam & MK_SHIFT);
    
    // lParam: 鼠标坐标
    int x = LOWORD(lParam);  // 低16位
    int y = HIWORD(lParam);  // 高16位
}

// WM_COMMAND
LRESULT OnCommand(WPARAM wParam, LPARAM lParam)
{
    WORD notifyCode = HIWORD(wParam);  // 通知代码
    WORD controlID = LOWORD(wParam);   // 控件ID
    HWND hwndControl = (HWND)lParam;   // 控件句柄
    
    if (controlID == IDC_BUTTON1 && notifyCode == BN_CLICKED)
    {
        // 按钮被点击
    }
}

// WM_SIZE
LRESULT OnSize(WPARAM wParam, LPARAM lParam)
{
    UINT type = wParam;  // SIZE_RESTORED, SIZE_MINIMIZED, SIZE_MAXIMIZED
    int width = LOWORD(lParam);
    int height = HIWORD(lParam);
}
```

---

## 第七部分：高级话题

### 7.1 子类化（Subclassing）

```cpp
// 替换已有窗口的窗口过程
WNDPROC g_oldButtonProc = NULL;

LRESULT CALLBACK NewButtonProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    if (uMsg == WM_LBUTTONDOWN)
    {
        // 自定义处理
        MessageBox(hwnd, "Custom handler!", "Info", MB_OK);
        return 0;
    }
    
    // 调用原窗口过程
    return CallWindowProc(g_oldButtonProc, hwnd, uMsg, wParam, lParam);
}

// 子类化一个按钮
HWND hButton = GetDlgItem(hwnd, IDC_BUTTON1);
g_oldButtonProc = (WNDPROC)SetWindowLongPtr(hButton, GWLP_WNDPROC, 
                                             (LONG_PTR)NewButtonProc);
```

### 7.2 消息反射（Message Reflection）

MFC的一个巧妙设计：将父窗口收到的控件通知消息反射回控件本身。

```cpp
// 传统方式：父窗口处理子控件消息
LRESULT CParentWnd::OnCommand(WPARAM wParam, LPARAM lParam)
{
    if (LOWORD(wParam) == IDC_MYBUTTON)
    {
        // 处理按钮消息
    }
}

// MFC反射方式：控件自己处理
BEGIN_MESSAGE_MAP(CMyButton, CButton)
    ON_CONTROL_REFLECT(BN_CLICKED, OnClicked)
END_MESSAGE_MAP()

void CMyButton::OnClicked()
{
    // 按钮自己处理点击事件
}
```

### 7.3 消息过滤和钩子（Hook）

```cpp
// 全局鼠标钩子
HHOOK g_hMouseHook = NULL;

LRESULT CALLBACK MouseHookProc(int nCode, WPARAM wParam, LPARAM lParam)
{
    if (nCode == HC_ACTION)
    {
        if (wParam == WM_LBUTTONDOWN)
        {
            MOUSEHOOKSTRUCT* pMouse = (MOUSEHOOKSTRUCT*)lParam;
            // 记录所有鼠标点击
            LogMouseClick(pMouse->pt.x, pMouse->pt.y);
        }
    }
    
    return CallNextHookEx(g_hMouseHook, nCode, wParam, lParam);
}

// 安装钩子
g_hMouseHook = SetWindowsHookEx(WH_MOUSE, MouseHookProc, 
                                 GetModuleHandle(NULL), 0);
```

---

## 第八部分：与现代UI框架对比

### 8.1 消息机制的演变

```
1990s                2000s               2010s               2020s
┌─────────┐         ┌─────────┐        ┌──────────┐       ┌──────────┐
│ Win32   │    →    │  .NET   │   →    │   WPF    │  →    │  WinUI3  │
│ MFC/ATL │         │ WinForms│        │  XAML    │       │  React   │
│消息循环  │         │ 事件驱动 │        │  MVVM    │       │  虚拟DOM │
└─────────┘         └─────────┘        └──────────┘       └──────────┘
```

### 8.2 代码对比

#### Win32/MFC方式

```cpp
// 1990s方式：直接处理消息
void CMyWindow::OnLButtonDown(UINT nFlags, CPoint point)
{
    m_data.x = point.x;
    m_data.y = point.y;
    Invalidate();  // 手动触发重绘
}
```

#### WinForms方式

```csharp
// 2000s方式：事件驱动
private void button1_Click(object sender, EventArgs e)
{
    label1.Text = "Button clicked!";  // 自动更新UI
}
```

#### WPF方式

```xml
<!-- 2010s方式：数据绑定 -->
<Button Content="Click Me" Command="{Binding ClickCommand}"/>
<TextBlock Text="{Binding Message}"/>
```

```csharp
public class ViewModel : INotifyPropertyChanged
{
    private string _message;
    public string Message
    {
        get => _message;
        set
        {
            _message = value;
            OnPropertyChanged(nameof(Message));  // 自动更新UI
        }
    }
    
    public ICommand ClickCommand { get; }
}
```

#### React方式

```jsx
// 2020s方式：声明式UI
function MyComponent() {
    const [message, setMessage] = useState("Hello");
    
    return (
        <div>
            <button onClick={() => setMessage("Clicked!")}>
                Click Me
            </button>
            <p>{message}</p>
        </div>
    );
}
```

### 8.3 优缺点对比

|方面|Win32/MFC|现代框架|
|---|---|---|
|性能|极高|中等-高|
|开发效率|低|高|
|学习曲线|陡峭|平缓|
|代码量|多|少|
|灵活性|极高|中等|
|跨平台|否|部分支持|

---

## 第九部分：实战案例

### 9.1 简单的绘图程序

```cpp
#include <windows.h>
#include <vector>

struct Point
{
    int x, y;
};

class DrawingWindow : public SimpleWindow
{
private:
    std::vector<Point> m_points;
    bool m_isDrawing;
    
protected:
    void OnCreate() override
    {
        m_isDrawing = false;
        
        RegisterMessage(WM_LBUTTONDOWN, [this](WPARAM w, LPARAM l) {
            m_isDrawing = true;
            m_points.clear();
            m_points.push_back({LOWORD(l), HIWORD(l)});
            return 0;
        });
        
        RegisterMessage(WM_MOUSEMOVE, [this](WPARAM w, LPARAM l) {
            if (m_isDrawing && (w & MK_LBUTTON))
            {
                m_points.push_back({LOWORD(l), HIWORD(l)});
                InvalidateRect(m_hwnd, NULL, FALSE);
            }
            return 0;
        });
        
        RegisterMessage(WM_LBUTTONUP, [this](WPARAM w, LPARAM l) {
            m_isDrawing = false;
            return 0;
        });
    }
    
    void OnPaint() override
    {
        PAINTSTRUCT ps;
        HDC hdc = BeginPaint(m_hwnd, &ps);
        
        if (m_points.size() > 1)
        {
            MoveToEx(hdc, m_points[0].x, m_points[0].y, NULL);
            for (size_t i = 1; i < m_points.size(); i++)
            {
                LineTo(hdc, m_points[i].x, m_points[i].y);
            }
        }
        
        EndPaint(m_hwnd, &ps);
    }
};
```

### 9.2 定时器动画

```cpp
class AnimationWindow : public SimpleWindow
{
private:
    int m_x, m_y;
    int m_dx, m_dy;
    
protected:
    void OnCreate() override
    {
        m_x = 100; m_y = 100;
        m_dx = 5; m_dy = 3;
        
        // 设置定时器，每30ms触发一次
        SetTimer(m_hwnd, 1, 30, NULL);
        
        RegisterMessage(WM_TIMER, [this](WPARAM w, LPARAM l) {
            RECT rect;
            GetClientRect(m_hwnd, &rect);
            
            // 更新位置
            m_x += m_dx;
            m_y += m_dy;
            
            // 边界检测
            if (m_x < 0 || m_x > rect.right - 50) m_dx = -m_dx;
            if (m_y < 0 || m_y > rect.bottom - 50) m_dy = -m_dy;
            
            InvalidateRect(m_hwnd, NULL, TRUE);
            return 0;
        });
    }
    
    void OnPaint() override
    {
        PAINTSTRUCT ps;
        HDC hdc = BeginPaint(m_hwnd, &ps);
        
        // 绘制移动的矩形
        RECT ballRect = {m_x, m_y, m_x + 50, m_y + 50};
        HBRUSH hBrush = CreateSolidBrush(RGB(255, 0, 0));
        FillRect(hdc, &ballRect, hBrush);
        DeleteObject(hBrush);
        
        EndPaint(m_hwnd, &ps);
    }
    
    void OnDestroy() override
    {
        KillTimer(m_hwnd, 1);
        SimpleWindow::OnDestroy();
    }
};
```

---

## 第十部分：最佳实践和注意事项

### 10.1 性能优化

#### 1. 减少不必要的重绘

```cpp
// ❌ 错误：整个窗口重绘
Invalidate();

// ✅ 正确：只重绘改变的区域
RECT updateRect = {x, y, x + width, y + height};
InvalidateRect(m_hwnd, &updateRect, FALSE);
```

#### 2. 双缓冲消除闪烁

```cpp
void OnPaint() override
{
    PAINTSTRUCT ps;
    HDC hdc = BeginPaint(m_hwnd, &ps);
    
    // 创建内存DC
    HDC hdcMem = CreateCompatibleDC(hdc);
    HBITMAP hbmMem = CreateCompatibleBitmap(hdc, width, height);
    HBITMAP hbmOld = (HBITMAP)SelectObject(hdcMem, hbmMem);
    
    // 在内存DC上绘制
    // ... 绘制操作 ...
    
    // 一次性复制到屏幕
    BitBlt(hdc, 0, 0, width, height, hdcMem, 0, 0, SRCCOPY);
    
    // 清理
    SelectObject(hdcMem, hbmOld);
    DeleteObject(hbmMem);
    DeleteDC(hdcMem);
    
    EndPaint(m_hwnd, &ps);
}
```

### 10.2 资源管理

#### RAII包装器

```cpp
class DCHandle
{
public:
    DCHandle(HWND hwnd) : m_hwnd(hwnd), m_hdc(NULL) {}
    ~DCHandle() { if (m_hdc) ReleaseDC(m_hwnd, m_hdc); }
    
    HDC Get()
    {
        if (!m_hdc) m_hdc = GetDC(m_hwnd);
        return m_hdc;
    }
    
private:
    HWND m_hwnd;
    HDC m_hdc;
};

// 使用
{
    DCHandle dc(m_hwnd);
    TextOut(dc.Get(), 10, 10, "Text", 4);
}  // 自动释放
```

### 10.3 线程安全

```cpp
// ❌ 错误：从工作线程直接更新UI
void WorkerThread()
{
    // ... 耗时操作 ...
    SetWindowText(hwnd, "Done!");  // 危险！
}

// ✅ 正确：使用PostMessage
void WorkerThread()
{
    // ... 耗时操作 ...
    PostMessage(hwnd, WM_USER_WORK_DONE, 0, 0);
}

// UI线程处理
LRESULT OnWorkDone(WPARAM, LPARAM)
{
    SetWindowText(m_hwnd, "Done!");
    return 0;
}
```

---

## 总结

### 1990s Windows消息机制封装的演进

```
原始Win32 API (1985)
    ↓
MFC (1992) - 宏消息映射
    ↓
ATL (1997) - 模板消息映射
    ↓
WTL (1999) - 轻量级封装
```

### 核心设计模式

1. **HWND ↔ C++对象映射** - 通过GWLP_USERDATA
2. **静态分发 → 实例分发** - StaticWndProc → WindowProc
3. **消息映射表** - 宏或模板生成
4. **RAII资源管理** - 自动释放GDI对象
5. **虚函数重载** - 提供默认实现

### 为什么现在还要学？

尽管有现代框架，Windows消息机制仍然重要：

1. **底层理解** - 所有Windows GUI框架都建立在此之上
2. **性能优化** - 游戏引擎、高性能应用仍使用Win32
3. **系统编程** - 钩子、窗口子类化、COM组件
4. **遗留代码** - 大量现存代码需要维护
5. **跨平台参考** - Linux X11、macOS AppKit类似原理

### 学习路径建议

```
1. 理解消息循环原理
   ↓
2. 手写Win32程序（不用框架）
   ↓
3. 实现简单的消息映射封装
   ↓
4. 学习MFC源码（消息映射实现）
   ↓
5. 对比现代框架（WPF、React等）
```

1990s的Windows消息机制封装虽然古老，但设计思想极其精妙，是理解事件驱动编程和GUI框架设计的绝佳范例。