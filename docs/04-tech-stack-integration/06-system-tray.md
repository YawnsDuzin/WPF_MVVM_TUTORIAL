# 6. 시스템 트레이 아이콘 통합

> **목표**: Hardcodet.NotifyIcon.Wpf 패키지를 사용하여 WPF 앱에 시스템 트레이 아이콘을 추가하고, MVVM 패턴으로 컨텍스트 메뉴, 풍선 팝업 알림, 창 최소화/복원 기능을 구현합니다.

---

## 목차

1. [시스템 트레이란?](#1-시스템-트레이란)
2. [Hardcodet.NotifyIcon.Wpf 패키지 소개](#2-hardcodetnotifyiconwpf-패키지-소개)
3. [트레이 아이콘 설정](#3-트레이-아이콘-설정)
4. [주요 기능](#4-주요-기능)
5. [MVVM 패턴과 통합](#5-mvvm-패턴과-통합)
6. [앱 종료 vs 트레이로 최소화 처리](#6-앱-종료-vs-트레이로-최소화-처리)
7. [전체 코드 예제](#7-전체-코드-예제)

---

## 1. 시스템 트레이란?

**시스템 트레이(System Tray)**는 Windows 작업 표시줄 오른쪽에 있는 **알림 영역(Notification Area)**입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│ [시작] │ 열린 앱들...                    │ △ 🔊 🌐 🛡️ ⏰  │
│        │                                 │    ↑               │
│        │                                 │  시스템 트레이      │
│        │                                 │  (알림 영역)       │
└─────────────────────────────────────────────────────────────────┘
```

### 시스템 트레이의 용도

| 기능 | 설명 | 예시 |
|------|------|------|
| **백그라운드 실행** | 창을 닫아도 앱이 계속 실행 | 메신저, 모니터링 앱 |
| **알림 표시** | 풍선 팝업(BalloonTip)으로 사용자에게 알림 | 새 메시지 도착, 작업 완료 |
| **빠른 액세스** | 우클릭 컨텍스트 메뉴로 주요 기능 접근 | 설정 열기, 종료 |
| **더블 클릭 복원** | 트레이 아이콘 더블 클릭으로 창 복원 | 숨겨진 창 다시 표시 |

> **WinForms 경험자를 위한 참고**: WinForms에서는 `NotifyIcon` 컴포넌트를 도구 상자에서 끌어다 놓으면 됐습니다. WPF에는 기본 제공 트레이 아이콘이 없어서 서드파티 패키지를 사용합니다.

---

## 2. Hardcodet.NotifyIcon.Wpf 패키지 소개

**Hardcodet.NotifyIcon.Wpf**는 WPF에서 시스템 트레이 아이콘을 사용할 수 있게 해주는 가장 인기 있는 패키지입니다.

### 설치

```bash
dotnet add package Hardcodet.NotifyIcon.Wpf --version 4.0.1
```

### 주요 특징

| 특징 | 설명 |
|------|------|
| **XAML 선언적 사용** | XAML에서 직접 `TaskbarIcon`을 선언할 수 있음 |
| **WPF 네이티브** | WPF의 `ContextMenu`, `Popup`, 데이터 바인딩 지원 |
| **Command 바인딩** | MVVM 패턴의 `ICommand` 바인딩 완벽 지원 |
| **커스텀 풍선 팝업** | 기본 BalloonTip 외에 커스텀 XAML 팝업 가능 |
| **이벤트 지원** | 클릭, 더블 클릭, 마우스 오버 등 다양한 이벤트 |

### .csproj 확인

```xml
<ItemGroup>
  <PackageReference Include="Hardcodet.NotifyIcon.Wpf" Version="4.0.1" />
  <PackageReference Include="CommunityToolkit.Mvvm" Version="8.4.0" />
  <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="10.0.0" />
</ItemGroup>
```

---

## 3. 트레이 아이콘 설정

### 3-1. 아이콘 리소스 준비 (.ico 파일)

시스템 트레이에 표시할 `.ico` 파일이 필요합니다.

**아이콘 파일을 프로젝트에 추가하는 방법**:

1. `.ico` 파일을 프로젝트의 `Resources` 폴더에 복사합니다.
2. `.csproj`에 리소스로 등록합니다.

```
MyApp/
├── Resources/
│   └── app-icon.ico    ← 트레이에 표시될 아이콘
├── Views/
├── ViewModels/
└── ...
```

```xml
<!-- .csproj에 아이콘 리소스 추가 -->
<ItemGroup>
  <!-- Resource로 설정하면 어셈블리에 임베드됩니다 -->
  <Resource Include="Resources\app-icon.ico" />
</ItemGroup>
```

> **팁**: `.ico` 파일은 16x16, 32x32, 48x48, 256x256 등 다양한 크기를 포함하는 것이 좋습니다. 시스템 트레이에서는 주로 16x16이 사용됩니다.

### 3-2. XAML에서 TaskbarIcon 선언

**App.xaml**에서 글로벌로 선언하는 방법이 가장 깔끔합니다.

```xml
<!-- App.xaml -->
<Application x:Class="MyApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:tb="http://www.hardcodet.net/taskbar"
             StartupUri="Views/MainWindow.xaml">

    <Application.Resources>
        <ResourceDictionary>
            <!--
                TaskbarIcon을 App.Resources에 선언합니다.
                이렇게 하면 앱 전체에서 접근 가능하고,
                MainWindow가 닫혀도 트레이 아이콘은 유지됩니다.
            -->
            <tb:TaskbarIcon x:Key="TrayIcon"
                            IconSource="/Resources/app-icon.ico"
                            ToolTipText="내 WPF 앱 - 실행 중">

                <!-- 우클릭 컨텍스트 메뉴 -->
                <tb:TaskbarIcon.ContextMenu>
                    <ContextMenu>
                        <MenuItem Header="열기"
                                  Command="{Binding ShowWindowCommand}" />
                        <Separator />
                        <MenuItem Header="설정"
                                  Command="{Binding OpenSettingsCommand}" />
                        <Separator />
                        <MenuItem Header="종료"
                                  Command="{Binding ExitApplicationCommand}" />
                    </ContextMenu>
                </tb:TaskbarIcon.ContextMenu>

                <!-- 마우스 오버 시 보여줄 툴팁 (커스텀 가능) -->
                <tb:TaskbarIcon.TrayToolTip>
                    <Border Background="White"
                            BorderBrush="Gray"
                            BorderThickness="1"
                            CornerRadius="4"
                            Padding="8">
                        <StackPanel>
                            <TextBlock Text="내 WPF 앱"
                                       FontWeight="Bold" />
                            <TextBlock Text="상태: 정상 실행 중"
                                       Foreground="Green" />
                        </StackPanel>
                    </Border>
                </tb:TaskbarIcon.TrayToolTip>
            </tb:TaskbarIcon>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

### 3-3. 코드 비하인드에서 설정

App.xaml.cs에서 트레이 아이콘의 DataContext를 설정합니다.

```csharp
// App.xaml.cs
using Hardcodet.Wpf.TaskbarNotification;
using Microsoft.Extensions.DependencyInjection;
using MyApp.ViewModels;

namespace MyApp;

public partial class App : Application
{
    private TaskbarIcon? _trayIcon;
    private ServiceProvider? _serviceProvider;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // DI 컨테이너 구성
        var services = new ServiceCollection();
        services.AddSingleton<MainViewModel>();
        services.AddTransient<MainWindow>();
        _serviceProvider = services.BuildServiceProvider();

        // 트레이 아이콘 가져오기 (App.xaml에서 선언한 리소스)
        _trayIcon = (TaskbarIcon)FindResource("TrayIcon");

        // 트레이 아이콘의 DataContext를 ViewModel로 설정
        // → 컨텍스트 메뉴의 Command 바인딩이 작동합니다.
        var mainViewModel = _serviceProvider.GetRequiredService<MainViewModel>();
        _trayIcon.DataContext = mainViewModel;

        // 메인 윈도우 표시
        var mainWindow = _serviceProvider.GetRequiredService<MainWindow>();
        mainWindow.DataContext = mainViewModel;
        mainWindow.Show();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        // 트레이 아이콘 정리 (필수! 안 하면 트레이에 아이콘이 남음)
        _trayIcon?.Dispose();
        _serviceProvider?.Dispose();
        base.OnExit(e);
    }
}
```

> **중요**: 앱 종료 시 반드시 `_trayIcon.Dispose()`를 호출하세요. 호출하지 않으면 앱이 종료되어도 트레이 아이콘이 그대로 남아 있습니다 (마우스를 올려야 사라짐).

---

## 4. 주요 기능

### 4-1. 트레이 아이콘 표시/숨기기

```csharp
// 트레이 아이콘의 Visibility를 직접 제어할 수 있습니다.

// 아이콘 숨기기
_trayIcon.Visibility = Visibility.Collapsed;

// 아이콘 표시
_trayIcon.Visibility = Visibility.Visible;
```

XAML에서 바인딩으로 제어할 수도 있습니다:

```xml
<tb:TaskbarIcon Visibility="{Binding TrayIconVisibility}" ... />
```

### 4-2. 컨텍스트 메뉴 (우클릭 메뉴)

트레이 아이콘을 우클릭하면 나타나는 메뉴입니다.

```xml
<!-- 컨텍스트 메뉴 정의 -->
<tb:TaskbarIcon.ContextMenu>
    <ContextMenu>
        <!-- 일반 메뉴 항목 -->
        <MenuItem Header="열기(_O)"
                  Command="{Binding ShowWindowCommand}"
                  InputGestureText="더블 클릭" />

        <Separator />

        <!-- 체크 가능한 메뉴 항목 -->
        <MenuItem Header="알림 활성화"
                  IsCheckable="True"
                  IsChecked="{Binding IsNotificationEnabled}" />

        <!-- 서브 메뉴가 있는 항목 -->
        <MenuItem Header="상태 변경">
            <MenuItem Header="온라인"
                      Command="{Binding SetStatusCommand}"
                      CommandParameter="Online" />
            <MenuItem Header="자리 비움"
                      Command="{Binding SetStatusCommand}"
                      CommandParameter="Away" />
            <MenuItem Header="방해 금지"
                      Command="{Binding SetStatusCommand}"
                      CommandParameter="DND" />
        </MenuItem>

        <Separator />

        <!-- 종료 -->
        <MenuItem Header="종료(_X)"
                  Command="{Binding ExitApplicationCommand}" />
    </ContextMenu>
</tb:TaskbarIcon.ContextMenu>
```

> **참고**: `Header`에서 `(_O)`처럼 밑줄+문자를 쓰면 Alt 키 단축키가 됩니다. WinForms의 `&O`와 동일한 역할입니다.

### 4-3. 풍선 팝업 알림 (BalloonTip)

```csharp
// 기본 풍선 팝업 표시
// 첫 번째 인자: 제목
// 두 번째 인자: 본문
// 세 번째 인자: 아이콘 종류
_trayIcon.ShowBalloonTip(
    "새 알림",                          // 제목
    "장비에서 얼굴 인식 이벤트가 발생했습니다.", // 본문
    BalloonIcon.Info);                  // 아이콘 (Info, Warning, Error, None)
```

**BalloonIcon 종류**:

| 값 | 아이콘 | 용도 |
|----|--------|------|
| `BalloonIcon.Info` | ℹ️ 정보 | 일반 알림 |
| `BalloonIcon.Warning` | ⚠ 경고 | 주의가 필요한 알림 |
| `BalloonIcon.Error` | ❌ 오류 | 에러 알림 |
| `BalloonIcon.None` | 없음 | 아이콘 없는 알림 |

**커스텀 풍선 팝업** (WPF UserControl을 팝업으로 사용):

```csharp
// 커스텀 XAML 컨트롤을 풍선 팝업으로 표시
var customBalloon = new CustomNotificationControl
{
    DataContext = new NotificationViewModel
    {
        Title = "얼굴 인식",
        Message = "홍길동님이 출입했습니다.",
        PhotoUrl = "/images/hong.jpg"
    }
};

// 커스텀 풍선 표시 (애니메이션, 표시 시간 설정 가능)
_trayIcon.ShowCustomBalloon(
    customBalloon,
    PopupAnimation.Slide,  // 슬라이드 애니메이션
    timeout: 4000);        // 4초 후 자동 닫힘 (null이면 수동 닫기)
```

### 4-4. 더블 클릭으로 창 복원

XAML에서 더블 클릭 Command를 바인딩합니다:

```xml
<tb:TaskbarIcon
    DoubleClickCommand="{Binding ShowWindowCommand}"
    IconSource="/Resources/app-icon.ico" />
```

또는 코드 비하인드에서 이벤트를 처리합니다:

```csharp
// 코드 비하인드 방식 (간단한 경우)
_trayIcon.TrayMouseDoubleClick += (sender, args) =>
{
    // 메인 윈도우를 찾아서 복원
    var mainWindow = Application.Current.MainWindow;
    if (mainWindow is not null)
    {
        mainWindow.Show();             // 숨겨진 창 표시
        mainWindow.WindowState = WindowState.Normal; // 최소화 해제
        mainWindow.Activate();         // 포커스 설정
    }
};
```

### 4-5. 최소화 시 트레이로 이동

창을 최소화하거나 닫을 때 트레이로 이동하는 동작입니다.

```csharp
// MainWindow.xaml.cs (코드 비하인드)
namespace MyApp.Views;

public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
    }

    /// <summary>
    /// 창 닫기 버튼(X)을 누르면 실제로 닫지 않고 숨깁니다.
    /// </summary>
    protected override void OnClosing(CancelEventArgs e)
    {
        // 실제 종료가 아닌 경우 (트레이로 최소화)
        if (!_isReallyClosing)
        {
            e.Cancel = true; // 닫기 취소
            Hide();          // 창 숨기기

            // 풍선 팝업으로 트레이 이동 안내
            var trayIcon = (TaskbarIcon)Application.Current
                .FindResource("TrayIcon");
            trayIcon.ShowBalloonTip(
                "최소화됨",
                "앱이 시스템 트레이에서 계속 실행 중입니다.",
                BalloonIcon.Info);
        }

        base.OnClosing(e);
    }

    // 실제 종료 플래그
    private bool _isReallyClosing;

    /// <summary>
    /// 실제로 앱을 종료할 때 호출합니다.
    /// </summary>
    public void ReallyClose()
    {
        _isReallyClosing = true;
        Close();
    }
}
```

---

## 5. MVVM 패턴과 통합

### 5-1. MainViewModel

```csharp
// ViewModels/MainViewModel.cs
using System.Windows;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using CommunityToolkit.Mvvm.Messaging;
using Hardcodet.Wpf.TaskbarNotification;
using Microsoft.Extensions.Logging;

namespace MyApp.ViewModels;

/// <summary>
/// 메인 ViewModel입니다.
/// 시스템 트레이 아이콘의 커맨드와 앱 전체 상태를 관리합니다.
/// </summary>
public partial class MainViewModel : ObservableObject
{
    private readonly ILogger<MainViewModel> _logger;

    public MainViewModel(ILogger<MainViewModel> logger)
    {
        _logger = logger;
    }

    // ──────────────────────────────────────────────
    // Observable 속성
    // ──────────────────────────────────────────────

    /// <summary>앱 상태 텍스트</summary>
    [ObservableProperty]
    private string _statusText = "정상 실행 중";

    /// <summary>알림 활성화 여부</summary>
    [ObservableProperty]
    private bool _isNotificationEnabled = true;

    /// <summary>현재 상태 (Online, Away, DND)</summary>
    [ObservableProperty]
    private string _currentStatus = "Online";

    // ──────────────────────────────────────────────
    // 트레이 아이콘 커맨드
    // ──────────────────────────────────────────────

    /// <summary>
    /// 메인 윈도우를 표시합니다.
    /// 트레이 아이콘 더블 클릭 또는 컨텍스트 메뉴 "열기"에서 호출됩니다.
    /// </summary>
    [RelayCommand]
    private void ShowWindow()
    {
        var mainWindow = Application.Current.MainWindow;
        if (mainWindow is null) return;

        mainWindow.Show();                           // 숨겨진 창 표시
        mainWindow.WindowState = WindowState.Normal; // 최소화 해제
        mainWindow.Activate();                       // 포커스 맞추기
        mainWindow.Topmost = true;                   // 최상위로 올리기
        mainWindow.Topmost = false;                  // 최상위 해제 (트릭)

        _logger.LogDebug("메인 윈도우 복원됨");
    }

    /// <summary>
    /// 설정 창을 엽니다.
    /// </summary>
    [RelayCommand]
    private void OpenSettings()
    {
        _logger.LogInformation("설정 창 열기 요청");
        // 설정 창 열기 로직 (예: 새 Window 생성)
        // var settingsWindow = new SettingsWindow();
        // settingsWindow.Show();
    }

    /// <summary>
    /// 상태를 변경합니다.
    /// </summary>
    [RelayCommand]
    private void SetStatus(string? status)
    {
        if (status is null) return;

        CurrentStatus = status;
        StatusText = status switch
        {
            "Online" => "온라인",
            "Away" => "자리 비움",
            "DND" => "방해 금지",
            _ => status
        };

        _logger.LogInformation("상태 변경: {Status}", status);
    }

    /// <summary>
    /// 앱을 완전히 종료합니다.
    /// 트레이 컨텍스트 메뉴의 "종료"에서 호출됩니다.
    /// </summary>
    [RelayCommand]
    private void ExitApplication()
    {
        _logger.LogInformation("사용자 요청으로 앱 종료");

        // Application.Current.Shutdown()을 호출하면
        // App.OnExit()에서 트레이 아이콘 정리가 수행됩니다.
        Application.Current.Shutdown();
    }

    // ──────────────────────────────────────────────
    // 풍선 팝업 트리거 (ViewModel에서 View로 알림)
    // ──────────────────────────────────────────────

    /// <summary>
    /// ViewModel에서 풍선 팝업을 표시합니다.
    /// 트레이 아이콘에 직접 접근하지 않고, 메시지를 통해 간접적으로 호출합니다.
    /// </summary>
    public void ShowNotification(string title, string message,
        BalloonIcon icon = BalloonIcon.Info)
    {
        if (!IsNotificationEnabled) return;

        // 방법 1: Application 리소스에서 직접 접근 (간단한 방법)
        if (Application.Current.TryFindResource("TrayIcon")
            is TaskbarIcon trayIcon)
        {
            trayIcon.ShowBalloonTip(title, message, icon);
        }
    }
}
```

### 5-2. Messenger를 이용한 느슨한 결합 (권장)

ViewModel에서 트레이 아이콘에 직접 접근하는 대신, **CommunityToolkit.Mvvm의 Messenger**를 사용하면 더 깔끔합니다.

```csharp
// Messages/ShowBalloonMessage.cs
namespace MyApp.Messages;

/// <summary>
/// 풍선 팝업 표시를 요청하는 메시지입니다.
/// ViewModel → View (트레이 아이콘) 방향으로 전달됩니다.
/// </summary>
public sealed record ShowBalloonMessage(
    string Title,                       // 팝업 제목
    string Body,                        // 팝업 본문
    BalloonIcon Icon = BalloonIcon.Info // 아이콘 종류
);
```

```csharp
// ViewModel에서 메시지 발행
using CommunityToolkit.Mvvm.Messaging;

// 다른 ViewModel이나 서비스에서 풍선 팝업을 요청할 때:
WeakReferenceMessenger.Default.Send(
    new ShowBalloonMessage(
        "이벤트 발생",
        "새로운 얼굴인식 이벤트가 수신되었습니다."));
```

```csharp
// App.xaml.cs에서 메시지 수신 등록
protected override void OnStartup(StartupEventArgs e)
{
    base.OnStartup(e);

    // ... (이전 코드) ...

    // 풍선 팝업 메시지 수신 등록
    WeakReferenceMessenger.Default
        .Register<ShowBalloonMessage>(this, (recipient, message) =>
        {
            // UI 스레드에서 실행
            Current.Dispatcher.Invoke(() =>
            {
                _trayIcon?.ShowBalloonTip(
                    message.Title,
                    message.Body,
                    message.Icon);
            });
        });
}
```

```
┌─────────────────┐    ShowBalloonMessage    ┌─────────────────┐
│                 │  ─────────────────────→  │                 │
│   ViewModel     │   (WeakReference-        │    App.xaml.cs   │
│   또는 Service  │    Messenger)            │   (수신 측)     │
│                 │                           │                 │
│   .Send(msg)    │                           │  _trayIcon      │
│                 │                           │  .ShowBalloon() │
└─────────────────┘                           └─────────────────┘
```

---

## 6. 앱 종료 vs 트레이로 최소화 처리

실무에서는 "X 버튼을 눌렀을 때 실제 종료할지, 트레이로 최소화할지"를 결정해야 합니다.

### 전체 흐름

```
사용자가 X 버튼 클릭
       │
       ▼
  OnClosing 이벤트 발생
       │
       ├─── 실제 종료 요청? (트레이 메뉴 → "종료")
       │    └── YES → 그대로 닫기 (e.Cancel = false)
       │
       └── 아니오 → 트레이로 최소화
            ├── e.Cancel = true (닫기 취소)
            ├── Window.Hide()
            └── BalloonTip 표시 (선택사항)
```

### 구현 패턴: Attached Behavior 활용

MVVM에서 코드 비하인드를 최소화하려면 **Attached Behavior**를 사용합니다.

```csharp
// Behaviors/MinimizeToTrayBehavior.cs
using System.ComponentModel;
using System.Windows;
using Hardcodet.Wpf.TaskbarNotification;

namespace MyApp.Behaviors;

/// <summary>
/// Window에 붙이면 닫기 시 트레이로 최소화하는 동작을 추가하는
/// Attached Behavior입니다.
///
/// 사용법: <Window behaviors:MinimizeToTrayBehavior.Enabled="True" />
/// </summary>
public static class MinimizeToTrayBehavior
{
    // ──────────────────────────────────────────────
    // Enabled Attached Property
    // ──────────────────────────────────────────────

    public static readonly DependencyProperty EnabledProperty =
        DependencyProperty.RegisterAttached(
            "Enabled",
            typeof(bool),
            typeof(MinimizeToTrayBehavior),
            new PropertyMetadata(false, OnEnabledChanged));

    public static bool GetEnabled(DependencyObject obj) =>
        (bool)obj.GetValue(EnabledProperty);

    public static void SetEnabled(DependencyObject obj, bool value) =>
        obj.SetValue(EnabledProperty, value);

    // ──────────────────────────────────────────────
    // 실제 종료 플래그
    // ──────────────────────────────────────────────

    /// <summary>
    /// 실제 종료 시 true로 설정하여 트레이 최소화를 건너뜁니다.
    /// </summary>
    public static readonly DependencyProperty IsReallyClosingProperty =
        DependencyProperty.RegisterAttached(
            "IsReallyClosing",
            typeof(bool),
            typeof(MinimizeToTrayBehavior),
            new PropertyMetadata(false));

    public static bool GetIsReallyClosing(DependencyObject obj) =>
        (bool)obj.GetValue(IsReallyClosingProperty);

    public static void SetIsReallyClosing(DependencyObject obj, bool value) =>
        obj.SetValue(IsReallyClosingProperty, value);

    // ──────────────────────────────────────────────
    // 이벤트 핸들러 연결
    // ──────────────────────────────────────────────

    private static void OnEnabledChanged(
        DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is not Window window) return;

        if ((bool)e.NewValue)
        {
            // Closing 이벤트에 핸들러 등록
            window.Closing += OnWindowClosing;
        }
        else
        {
            window.Closing -= OnWindowClosing;
        }
    }

    private static void OnWindowClosing(object? sender, CancelEventArgs e)
    {
        if (sender is not Window window) return;

        // 실제 종료 플래그가 true이면 그대로 닫기
        if (GetIsReallyClosing(window)) return;

        // 닫기 취소 → 트레이로 최소화
        e.Cancel = true;
        window.Hide();

        // 풍선 팝업 안내 (선택사항)
        if (Application.Current.TryFindResource("TrayIcon")
            is TaskbarIcon trayIcon)
        {
            trayIcon.ShowBalloonTip(
                "최소화됨",
                "시스템 트레이에서 계속 실행 중입니다. " +
                "종료하려면 트레이 아이콘을 우클릭하세요.",
                BalloonIcon.Info);
        }
    }
}
```

XAML에서 사용:

```xml
<!-- MainWindow.xaml -->
<Window x:Class="MyApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:behaviors="clr-namespace:MyApp.Behaviors"
        behaviors:MinimizeToTrayBehavior.Enabled="True"
        Title="내 WPF 앱" Width="800" Height="600">

    <!-- 창 내용 -->
</Window>
```

실제 종료 시에는 Attached Property를 설정한 후 닫기:

```csharp
// MainViewModel.cs의 ExitApplication 커맨드
[RelayCommand]
private void ExitApplication()
{
    var mainWindow = Application.Current.MainWindow;
    if (mainWindow is not null)
    {
        // 실제 종료 플래그를 true로 설정
        MinimizeToTrayBehavior.SetIsReallyClosing(mainWindow, true);
    }

    Application.Current.Shutdown();
}
```

---

## 7. 전체 코드 예제

아래는 시스템 트레이 통합의 전체 구조입니다.

### 프로젝트 구조

```
MyApp/
├── Resources/
│   └── app-icon.ico              ← 트레이 아이콘 파일
├── Behaviors/
│   └── MinimizeToTrayBehavior.cs ← 트레이 최소화 동작
├── Messages/
│   └── ShowBalloonMessage.cs     ← 풍선 팝업 메시지
├── ViewModels/
│   └── MainViewModel.cs          ← 트레이 커맨드 포함
├── Views/
│   └── MainWindow.xaml           ← 메인 윈도우
├── App.xaml                       ← TaskbarIcon 선언
└── App.xaml.cs                    ← 트레이 아이콘 초기화/정리
```

### 완성된 App.xaml

```xml
<!-- App.xaml -->
<Application x:Class="MyApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:tb="http://www.hardcodet.net/taskbar">

    <Application.Resources>
        <ResourceDictionary>
            <!-- BoolToVisibility 컨버터 (자주 쓰이는 유틸) -->
            <BooleanToVisibilityConverter x:Key="BoolToVisibilityConverter" />

            <!-- 시스템 트레이 아이콘 -->
            <tb:TaskbarIcon x:Key="TrayIcon"
                            IconSource="/Resources/app-icon.ico"
                            ToolTipText="내 WPF 앱"
                            DoubleClickCommand="{Binding ShowWindowCommand}">

                <!-- 우클릭 컨텍스트 메뉴 -->
                <tb:TaskbarIcon.ContextMenu>
                    <ContextMenu>
                        <MenuItem Header="열기(_O)"
                                  Command="{Binding ShowWindowCommand}" />
                        <Separator />
                        <MenuItem Header="알림 활성화"
                                  IsCheckable="True"
                                  IsChecked="{Binding IsNotificationEnabled}" />
                        <MenuItem Header="상태 변경">
                            <MenuItem Header="온라인"
                                      Command="{Binding SetStatusCommand}"
                                      CommandParameter="Online" />
                            <MenuItem Header="자리 비움"
                                      Command="{Binding SetStatusCommand}"
                                      CommandParameter="Away" />
                            <MenuItem Header="방해 금지"
                                      Command="{Binding SetStatusCommand}"
                                      CommandParameter="DND" />
                        </MenuItem>
                        <Separator />
                        <MenuItem Header="종료(_X)"
                                  Command="{Binding ExitApplicationCommand}" />
                    </ContextMenu>
                </tb:TaskbarIcon.ContextMenu>

                <!-- 커스텀 툴팁 -->
                <tb:TaskbarIcon.TrayToolTip>
                    <Border Background="White"
                            BorderBrush="DarkGray"
                            BorderThickness="1"
                            CornerRadius="4"
                            Padding="10">
                        <StackPanel>
                            <TextBlock Text="내 WPF 앱"
                                       FontWeight="Bold"
                                       FontSize="14" />
                            <TextBlock Text="{Binding StatusText}"
                                       Margin="0,4,0,0" />
                            <TextBlock Margin="0,2,0,0">
                                <Run Text="상태: " />
                                <Run Text="{Binding CurrentStatus}"
                                     FontWeight="SemiBold" />
                            </TextBlock>
                        </StackPanel>
                    </Border>
                </tb:TaskbarIcon.TrayToolTip>
            </tb:TaskbarIcon>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

### 완성된 App.xaml.cs

```csharp
// App.xaml.cs
using System.Windows;
using CommunityToolkit.Mvvm.Messaging;
using Hardcodet.Wpf.TaskbarNotification;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using MyApp.Messages;
using MyApp.ViewModels;
using MyApp.Views;

namespace MyApp;

public partial class App : Application
{
    private TaskbarIcon? _trayIcon;
    private ServiceProvider? _serviceProvider;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // ── DI 컨테이너 구성 ──
        var services = new ServiceCollection();
        services.AddLogging(builder => builder.AddDebug());
        services.AddSingleton<MainViewModel>();
        services.AddTransient<MainWindow>();
        _serviceProvider = services.BuildServiceProvider();

        // ── ViewModel 생성 ──
        var mainViewModel = _serviceProvider
            .GetRequiredService<MainViewModel>();

        // ── 트레이 아이콘 초기화 ──
        _trayIcon = (TaskbarIcon)FindResource("TrayIcon");
        _trayIcon.DataContext = mainViewModel;

        // ── 풍선 팝업 메시지 수신 등록 ──
        WeakReferenceMessenger.Default
            .Register<ShowBalloonMessage>(this, (recipient, message) =>
            {
                Current.Dispatcher.Invoke(() =>
                {
                    _trayIcon?.ShowBalloonTip(
                        message.Title,
                        message.Body,
                        message.Icon);
                });
            });

        // ── 메인 윈도우 표시 ──
        var mainWindow = _serviceProvider
            .GetRequiredService<MainWindow>();
        mainWindow.DataContext = mainViewModel;
        MainWindow = mainWindow; // Application.MainWindow 설정
        mainWindow.Show();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        // Messenger 등록 해제
        WeakReferenceMessenger.Default.UnregisterAll(this);

        // 트레이 아이콘 정리 (반드시!)
        _trayIcon?.Dispose();

        // DI 컨테이너 정리
        _serviceProvider?.Dispose();

        base.OnExit(e);
    }
}
```

### 완성된 MainWindow.xaml

```xml
<!-- Views/MainWindow.xaml -->
<Window x:Class="MyApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:behaviors="clr-namespace:MyApp.Behaviors"
        behaviors:MinimizeToTrayBehavior.Enabled="True"
        Title="내 WPF 앱" Width="800" Height="600"
        WindowStartupLocation="CenterScreen">

    <Grid Margin="16">
        <StackPanel VerticalAlignment="Center"
                    HorizontalAlignment="Center">

            <TextBlock Text="시스템 트레이 통합 예제"
                       FontSize="24" FontWeight="Bold"
                       HorizontalAlignment="Center" />

            <TextBlock Margin="0,16,0,0"
                       HorizontalAlignment="Center"
                       TextAlignment="Center"
                       LineHeight="24">
                <Run Text="이 앱은 X 버튼을 눌러도 종료되지 않고" />
                <LineBreak />
                <Run Text="시스템 트레이에서 계속 실행됩니다." />
                <LineBreak />
                <LineBreak />
                <Run Text="종료하려면 트레이 아이콘을 우클릭 → '종료'를 선택하세요." />
            </TextBlock>

            <StackPanel Orientation="Horizontal"
                        HorizontalAlignment="Center"
                        Margin="0,24,0,0">
                <TextBlock Text="현재 상태: "
                           FontSize="16" />
                <TextBlock Text="{Binding StatusText}"
                           FontSize="16" FontWeight="Bold"
                           Foreground="Green" />
            </StackPanel>
        </StackPanel>
    </Grid>
</Window>
```

### 핵심 포인트 정리

| 항목 | 설명 |
|------|------|
| **패키지** | `Hardcodet.NotifyIcon.Wpf` 4.0.1 |
| **TaskbarIcon 위치** | `App.xaml` Resources에 선언 (창 닫아도 유지) |
| **DataContext** | `App.xaml.cs`에서 트레이 아이콘에 ViewModel 바인딩 |
| **컨텍스트 메뉴** | XAML의 `ContextMenu`로 선언, Command 바인딩 |
| **풍선 팝업** | `ShowBalloonTip()` 또는 Messenger 패턴 |
| **트레이 최소화** | `OnClosing`에서 `Cancel + Hide()` |
| **실제 종료** | `Application.Current.Shutdown()` |
| **정리** | `OnExit`에서 반드시 `_trayIcon.Dispose()` 호출 |

> **다음 단계**: [Hikvision HCNetSDK P/Invoke 연동](./07-hikvision-sdk.md)에서는 비관리 DLL을 C#에서 호출하여 얼굴인식 단말기와 통신하는 방법을 다룹니다.
