# 화면 네비게이션 패턴

> **목표**: WinForms의 `Form.Show()` / `Form.ShowDialog()` 방식에서 벗어나,
> WPF MVVM에서 화면을 전환하고, 데이터를 전달하고, 다이얼로그를 처리하는 방법을 배웁니다.

---

## 목차

1. [WinForms vs WPF 네비게이션 비교](#1-winforms-vs-wpf-네비게이션-비교)
2. [WPF의 세 가지 네비게이션 방식](#2-wpf의-세-가지-네비게이션-방식)
3. [ContentControl + DataTemplate 방식 (권장)](#3-contentcontrol--datatemplate-방식-권장)
4. [네비게이션 서비스 구현](#4-네비게이션-서비스-구현)
5. [화면 간 데이터 전달](#5-화면-간-데이터-전달)
6. [뒤로 가기 (네비게이션 스택)](#6-뒤로-가기-네비게이션-스택)
7. [다이얼로그 처리](#7-다이얼로그-처리)
8. [DI 등록 및 전체 연결](#8-di-등록-및-전체-연결)
9. [전체 코드 예제](#9-전체-코드-예제)

---

## 1. WinForms vs WPF 네비게이션 비교

### WinForms 방식 (익숙한 방식)

```csharp
// WinForms: 새 폼을 직접 생성하고 표시
private void btnEdit_Click(object sender, EventArgs e)
{
    // 1. 새 폼 인스턴스 직접 생성
    var editForm = new EmployeeEditForm();

    // 2. 데이터 전달: public 프로퍼티에 직접 설정
    editForm.EmployeeId = selectedId;

    // 3. 모달 다이얼로그로 표시
    if (editForm.ShowDialog() == DialogResult.OK)
    {
        // 4. 결과 받기: 폼의 프로퍼티에서 직접 읽기
        var updatedEmployee = editForm.UpdatedEmployee;
        RefreshGrid();
    }
}
```

**WinForms 방식의 문제점**:
- Form이 다른 Form을 **직접 알고 있음** (강한 결합)
- `new EmployeeEditForm()`으로 **직접 생성** (DI 불가)
- 테스트하려면 실제 Form을 띄워야 함 (단위 테스트 불가)
- 폼 간 데이터 전달이 public 프로퍼티나 생성자에 의존

### WPF MVVM 방식 (배울 방식)

```csharp
// WPF MVVM: 네비게이션 서비스를 통해 ViewModel만 전환
[RelayCommand]
private void EditEmployee()
{
    // 1. 네비게이션 서비스에 "어떤 ViewModel로 갈지"만 지시
    _navigationService.NavigateTo<EmployeeEditViewModel>(
        parameter: SelectedEmployee.Id);

    // View(화면)는 DataTemplate이 자동으로 결정
    // ViewModel은 View를 전혀 모름!
}
```

---

## 2. WPF의 세 가지 네비게이션 방식

| 방식 | 설명 | 장점 | 단점 | 추천도 |
|------|------|------|------|--------|
| **ContentControl + DataTemplate** | 하나의 Window 안에서 ContentControl의 내용을 교체 | MVVM 친화적, DI 호환 | 초기 설정 필요 | ★★★ |
| **Frame + Page** | WPF의 Frame 컨트롤에 Page를 로드 | 브라우저식 네비게이션 기본 제공 | MVVM과 궁합 나쁨 | ★★ |
| **Window 방식** | WinForms처럼 새 Window를 띄움 | 가장 직관적 | MVVM 위반 가능성 높음 | ★ |

> 이 튜토리얼에서는 **ContentControl + DataTemplate** 방식을 사용합니다.
> 실무에서 가장 많이 쓰이고, MVVM 패턴과 가장 잘 맞기 때문입니다.

### 각 방식의 간단한 비교

#### 방식 1: ContentControl + DataTemplate (권장)

```xml
<!-- MainWindow.xaml -->
<!-- ContentControl 하나로 화면 전환 -->
<ContentControl Content="{Binding CurrentViewModel}"/>
```

```csharp
// ViewModel을 바꾸면 DataTemplate이 자동으로 View를 교체
CurrentViewModel = new EmployeeListViewModel();  // → EmployeeListView 표시
CurrentViewModel = new EmployeeEditViewModel();  // → EmployeeEditView 표시
```

#### 방식 2: Frame + Page

```xml
<!-- MainWindow.xaml -->
<Frame x:Name="MainFrame" NavigationUIVisibility="Hidden"/>
```

```csharp
// 코드 비하인드에서 Page를 직접 네비게이션
MainFrame.Navigate(new EmployeeListPage());
```

#### 방식 3: Window 방식 (WinForms과 유사)

```csharp
// 새 Window를 직접 생성하여 표시
var editWindow = new EmployeeEditWindow();
editWindow.ShowDialog();
```

---

## 3. ContentControl + DataTemplate 방식 (권장)

### 동작 원리

```
┌─ MainWindow ─────────────────────────────────────────┐
│                                                       │
│  ┌─ ContentControl ────────────────────────────────┐ │
│  │                                                   │ │
│  │  Content = "{Binding CurrentViewModel}"           │ │
│  │                                                   │ │
│  │  CurrentViewModel이 EmployeeListViewModel이면    │ │
│  │  → DataTemplate에 의해 EmployeeListView 표시     │ │
│  │                                                   │ │
│  │  CurrentViewModel이 EmployeeEditViewModel이면    │ │
│  │  → DataTemplate에 의해 EmployeeEditView 표시     │ │
│  │                                                   │ │
│  └───────────────────────────────────────────────────┘ │
│                                                       │
└───────────────────────────────────────────────────────┘
```

**핵심 원리**: ContentControl에 ViewModel 객체를 바인딩하면, WPF가 해당 타입에 맞는 DataTemplate을 찾아 자동으로 View를 렌더링합니다.

### MainWindow.xaml

```xml
<!-- 파일: Views/MainWindow.xaml -->
<Window x:Class="MyApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        mc:Ignorable="d"
        d:DataContext="{d:DesignInstance vm:MainViewModel, IsDesignTimeCreatable=False}"
        Title="직원 관리 시스템"
        Width="900" Height="650"
        WindowStartupLocation="CenterScreen">

    <Grid>
        <Grid.RowDefinitions>
            <!-- 행 0: 상단 네비게이션 바 -->
            <RowDefinition Height="Auto"/>
            <!-- 행 1: 메인 콘텐츠 영역 -->
            <RowDefinition Height="*"/>
            <!-- 행 2: 상태 바 -->
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- === 상단 네비게이션 바 === -->
        <Border Grid.Row="0" Background="#2196F3" Padding="12,8">
            <DockPanel>
                <!-- 앱 제목 -->
                <TextBlock Text="직원 관리 시스템"
                           Foreground="White"
                           FontSize="18"
                           FontWeight="Bold"
                           VerticalAlignment="Center"/>

                <!-- 네비게이션 버튼들 (오른쪽 정렬) -->
                <StackPanel DockPanel.Dock="Right"
                            Orientation="Horizontal"
                            HorizontalAlignment="Right">

                    <!-- 뒤로 가기 버튼 -->
                    <Button Content="← 뒤로"
                            Command="{Binding GoBackCommand}"
                            Padding="12,6"
                            Margin="4,0"
                            Foreground="White"
                            Background="Transparent"
                            BorderThickness="1"
                            BorderBrush="White"/>

                    <!-- 직원 목록 버튼 -->
                    <Button Content="직원 목록"
                            Command="{Binding NavigateToEmployeeListCommand}"
                            Padding="12,6"
                            Margin="4,0"
                            Foreground="White"
                            Background="Transparent"
                            BorderThickness="1"
                            BorderBrush="White"/>
                </StackPanel>
            </DockPanel>
        </Border>

        <!-- === 메인 콘텐츠 영역 === -->
        <!-- 이것이 핵심! CurrentViewModel에 따라 View가 자동 전환됨 -->
        <ContentControl Grid.Row="1"
                         Content="{Binding CurrentViewModel}"/>

        <!-- === 상태 바 === -->
        <StatusBar Grid.Row="2">
            <TextBlock Text="{Binding StatusMessage}"/>
        </StatusBar>
    </Grid>
</Window>
```

### DataTemplate 등록 (App.xaml)

```xml
<!-- 파일: App.xaml -->
<Application x:Class="MyApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             xmlns:views="clr-namespace:MyApp.Views">

    <Application.Resources>
        <!-- 공통 컨버터 -->
        <BooleanToVisibilityConverter x:Key="BooleanToVisibilityConverter"/>

        <!-- === ViewModel → View 매핑 === -->
        <!-- WPF는 ContentControl에 바인딩된 객체의 타입을 보고,
             해당 타입에 맞는 DataTemplate을 자동으로 적용합니다. -->

        <!-- EmployeeListViewModel → EmployeeListView -->
        <DataTemplate DataType="{x:Type vm:EmployeeListViewModel}">
            <views:EmployeeListView/>
        </DataTemplate>

        <!-- EmployeeEditViewModel → EmployeeEditView -->
        <DataTemplate DataType="{x:Type vm:EmployeeEditViewModel}">
            <views:EmployeeEditView/>
        </DataTemplate>

        <!-- SettingsViewModel → SettingsView -->
        <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
            <views:SettingsView/>
        </DataTemplate>

        <!-- 화면을 추가할 때마다 여기에 DataTemplate을 등록하면 됩니다 -->
    </Application.Resources>
</Application>
```

### MainViewModel

```csharp
// 파일: ViewModels/MainViewModel.cs
namespace MyApp.ViewModels;

using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using MyApp.Services;
using Serilog;

/// <summary>
/// 메인 윈도우의 ViewModel
/// - 현재 표시 중인 ViewModel(화면)을 관리
/// - NavigationService의 이벤트를 수신하여 CurrentViewModel 갱신
/// </summary>
public partial class MainViewModel : ObservableObject
{
    private readonly INavigationService _navigationService;
    private readonly ILogger _logger;

    /// <summary>
    /// 현재 화면에 표시 중인 ViewModel
    /// ContentControl에 바인딩되어 DataTemplate으로 자동 View 전환
    /// </summary>
    [ObservableProperty]
    private ObservableObject? _currentViewModel;

    /// <summary>상태 바 메시지</summary>
    [ObservableProperty]
    private string _statusMessage = "준비";

    public MainViewModel(
        INavigationService navigationService,
        ILogger logger)
    {
        _navigationService = navigationService;
        _logger = logger;

        // NavigationService에서 화면 전환이 발생하면 CurrentViewModel 갱신
        _navigationService.CurrentViewModelChanged += viewModel =>
        {
            CurrentViewModel = viewModel;
            _logger.Information("화면 전환: {ViewModelType}", viewModel?.GetType().Name);
        };

        // 초기 화면: 직원 목록
        _navigationService.NavigateTo<EmployeeListViewModel>();
    }

    /// <summary>뒤로 가기 커맨드</summary>
    [RelayCommand(CanExecute = nameof(CanGoBack))]
    private void GoBack()
    {
        _navigationService.GoBack();
    }

    /// <summary>뒤로 가기 가능 여부</summary>
    private bool CanGoBack()
    {
        return _navigationService.CanGoBack;
    }

    /// <summary>직원 목록 화면으로 이동</summary>
    [RelayCommand]
    private void NavigateToEmployeeList()
    {
        _navigationService.NavigateTo<EmployeeListViewModel>();
    }

    /// <summary>CurrentViewModel 변경 시 뒤로가기 버튼 상태 갱신</summary>
    partial void OnCurrentViewModelChanged(ObservableObject? value)
    {
        GoBackCommand.NotifyCanExecuteChanged();
    }
}
```

---

## 4. 네비게이션 서비스 구현

### INavigationService 인터페이스

```csharp
// 파일: Services/INavigationService.cs
namespace MyApp.Services;

using CommunityToolkit.Mvvm.ComponentModel;

/// <summary>
/// 화면 네비게이션 서비스 인터페이스
/// ViewModel 계층에서 View를 모르고도 화면을 전환할 수 있게 해줍니다.
/// </summary>
public interface INavigationService
{
    /// <summary>현재 표시 중인 ViewModel</summary>
    ObservableObject? CurrentViewModel { get; }

    /// <summary>뒤로 가기가 가능한지 여부</summary>
    bool CanGoBack { get; }

    /// <summary>
    /// 지정한 ViewModel 타입의 화면으로 이동
    /// DI 컨테이너에서 ViewModel 인스턴스를 자동으로 해결합니다.
    /// </summary>
    /// <typeparam name="TViewModel">이동할 ViewModel 타입</typeparam>
    /// <param name="parameter">전달할 파라미터 (선택)</param>
    void NavigateTo<TViewModel>(object? parameter = null)
        where TViewModel : ObservableObject;

    /// <summary>이전 화면으로 돌아가기</summary>
    void GoBack();

    /// <summary>화면 전환 시 발생하는 이벤트</summary>
    event Action<ObservableObject?> CurrentViewModelChanged;
}
```

### NavigationService 구현

```csharp
// 파일: Services/NavigationService.cs
namespace MyApp.Services;

using CommunityToolkit.Mvvm.ComponentModel;
using Microsoft.Extensions.DependencyInjection;
using MyApp.ViewModels;
using Serilog;

/// <summary>
/// 화면 네비게이션 서비스 구현체
/// - DI 컨테이너를 이용하여 ViewModel 인스턴스를 생성
/// - 네비게이션 스택으로 뒤로 가기 지원
/// - 파라미터 전달 지원
/// </summary>
public class NavigationService : INavigationService
{
    // DI 서비스 프로바이더 (ViewModel 인스턴스 해결용)
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger _logger;

    // 네비게이션 스택 (뒤로 가기용)
    private readonly Stack<ObservableObject> _navigationStack = new();

    /// <summary>현재 ViewModel</summary>
    public ObservableObject? CurrentViewModel { get; private set; }

    /// <summary>뒤로 가기 가능 여부 (스택에 이전 화면이 있으면 true)</summary>
    public bool CanGoBack => _navigationStack.Count > 0;

    /// <summary>화면 전환 이벤트</summary>
    public event Action<ObservableObject?>? CurrentViewModelChanged;

    public NavigationService(IServiceProvider serviceProvider, ILogger logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    /// <summary>
    /// 지정한 ViewModel 타입의 화면으로 이동
    /// </summary>
    public void NavigateTo<TViewModel>(object? parameter = null)
        where TViewModel : ObservableObject
    {
        _logger.Information(
            "네비게이션: {TargetVM}, 파라미터: {Parameter}",
            typeof(TViewModel).Name, parameter);

        // 1. 현재 ViewModel을 스택에 저장 (뒤로 가기용)
        if (CurrentViewModel is not null)
        {
            _navigationStack.Push(CurrentViewModel);
        }

        // 2. DI 컨테이너에서 새 ViewModel 인스턴스 생성
        //    DI가 생성자 파라미터(Repository, Logger 등)를 자동 주입
        var viewModel = _serviceProvider.GetRequiredService<TViewModel>();

        // 3. 파라미터가 있으면 전달
        //    INavigationAware 인터페이스를 구현한 ViewModel에 파라미터 전달
        if (parameter is not null && viewModel is INavigationAware navigationAware)
        {
            navigationAware.OnNavigatedTo(parameter);
        }

        // 4. 현재 ViewModel 업데이트
        CurrentViewModel = viewModel;

        // 5. MainViewModel에 화면 전환 알림
        CurrentViewModelChanged?.Invoke(CurrentViewModel);
    }

    /// <summary>이전 화면으로 돌아가기</summary>
    public void GoBack()
    {
        if (!CanGoBack)
        {
            _logger.Warning("뒤로 가기 불가: 네비게이션 스택이 비어있음");
            return;
        }

        // 스택에서 이전 ViewModel 꺼내기
        var previousViewModel = _navigationStack.Pop();
        CurrentViewModel = previousViewModel;

        _logger.Information("뒤로 가기: {ViewModelType}", previousViewModel.GetType().Name);

        CurrentViewModelChanged?.Invoke(CurrentViewModel);
    }
}
```

### INavigationAware 인터페이스

```csharp
// 파일: Services/INavigationAware.cs
namespace MyApp.Services;

/// <summary>
/// 네비게이션 파라미터를 수신할 수 있는 ViewModel이 구현하는 인터페이스
/// NavigateTo&lt;T&gt;(parameter)로 전달된 파라미터를 이 메서드에서 받습니다.
/// </summary>
public interface INavigationAware
{
    /// <summary>
    /// 이 화면으로 네비게이션되었을 때 호출됨
    /// </summary>
    /// <param name="parameter">전달받은 파라미터</param>
    void OnNavigatedTo(object parameter);
}
```

---

## 5. 화면 간 데이터 전달

화면 간 데이터를 전달하는 세 가지 방법을 알아봅니다.

### 방법 1: NavigationParameter (INavigationAware)

가장 일반적인 방법. 네비게이션 시 파라미터를 함께 전달합니다.

```csharp
// 파일: ViewModels/EmployeeEditViewModel.cs (INavigationAware 구현 버전)
namespace MyApp.ViewModels;

using CommunityToolkit.Mvvm.ComponentModel;
using MyApp.Services;
using Serilog;

/// <summary>
/// INavigationAware를 구현하여 네비게이션 파라미터를 수신합니다.
/// </summary>
public partial class EmployeeEditViewModel : ObservableValidator, INavigationAware
{
    private readonly IEmployeeRepository _employeeRepository;
    private readonly ILogger _logger;

    // ... 생성자, 프로퍼티 등 생략 ...

    /// <summary>
    /// 네비게이션으로 이 화면에 도착했을 때 호출됨
    /// </summary>
    public async void OnNavigatedTo(object parameter)
    {
        if (parameter is int employeeId)
        {
            // 전달받은 직원 ID로 데이터 로드
            _logger.Information("네비게이션 파라미터 수신: EmployeeId={Id}", employeeId);
            await LoadEmployeeAsync(employeeId);
        }
    }
}
```

```csharp
// 호출하는 쪽 (EmployeeListViewModel)
[RelayCommand]
private void EditEmployee()
{
    // 직원 ID를 파라미터로 전달
    _navigationService.NavigateTo<EmployeeEditViewModel>(
        parameter: SelectedEmployee!.Id);
}
```

### 방법 2: Messenger (CommunityToolkit.Mvvm)

ViewModel 간에 느슨하게 결합된 메시지를 주고받습니다. 직접적인 참조 없이 통신할 수 있습니다.

```csharp
// 메시지 정의
// 파일: Messages/EmployeeSelectedMessage.cs
namespace MyApp.Messages;

using CommunityToolkit.Mvvm.Messaging.Messages;
using MyApp.Models;

/// <summary>
/// 직원 선택 메시지
/// ValueChangedMessage<T>를 상속하면 값 변경 메시지로 사용할 수 있습니다.
/// </summary>
public class EmployeeSelectedMessage : ValueChangedMessage<Employee>
{
    public EmployeeSelectedMessage(Employee employee) : base(employee)
    {
    }
}

/// <summary>
/// 직원 저장 완료 메시지 (단순 알림용)
/// </summary>
public class EmployeeSavedMessage
{
    public int EmployeeId { get; init; }
}

/// <summary>
/// 요청-응답 메시지 (응답을 받을 수 있는 메시지)
/// </summary>
public class ConfirmDeleteMessage : AsyncRequestMessage<bool>
{
    public string EmployeeName { get; init; } = string.Empty;
}
```

```csharp
// 메시지 보내기 (EmployeeListViewModel)
// 파일: ViewModels/EmployeeListViewModel.cs
using CommunityToolkit.Mvvm.Messaging;

// 직원 선택 메시지 전송
WeakReferenceMessenger.Default.Send(
    new EmployeeSelectedMessage(SelectedEmployee!));

// 저장 완료 메시지 전송
WeakReferenceMessenger.Default.Send(
    new EmployeeSavedMessage { EmployeeId = 42 });
```

```csharp
// 메시지 받기 (EmployeeEditViewModel)
// 파일: ViewModels/EmployeeEditViewModel.cs
using CommunityToolkit.Mvvm.Messaging;

public partial class EmployeeEditViewModel : ObservableValidator,
    IRecipient<EmployeeSelectedMessage>  // 메시지 수신 인터페이스
{
    public EmployeeEditViewModel(/* ... */)
    {
        // 이 ViewModel을 메시지 수신자로 등록
        WeakReferenceMessenger.Default.RegisterAll(this);
    }

    /// <summary>직원 선택 메시지를 수신했을 때 호출됨</summary>
    public void Receive(EmployeeSelectedMessage message)
    {
        var employee = message.Value;
        // 수신한 직원 데이터로 폼 채우기
        Name = employee.Name;
        Department = employee.Department;
        Email = employee.Email;
    }
}
```

### 방법 3: 공유 서비스 (Shared State)

DI로 주입받는 공유 서비스를 통해 상태를 공유합니다.

```csharp
// 파일: Services/ISharedStateService.cs
namespace MyApp.Services;

/// <summary>
/// 화면 간 공유 상태 서비스
/// Singleton으로 등록하여 여러 ViewModel이 같은 인스턴스를 공유합니다.
/// </summary>
public interface ISharedStateService
{
    /// <summary>현재 선택된 직원 ID</summary>
    int? SelectedEmployeeId { get; set; }

    /// <summary>현재 로그인한 사용자명</summary>
    string CurrentUser { get; set; }
}

public class SharedStateService : ISharedStateService
{
    public int? SelectedEmployeeId { get; set; }
    public string CurrentUser { get; set; } = string.Empty;
}
```

```csharp
// DI 등록 (Singleton으로 등록해야 상태가 공유됨)
services.AddSingleton<ISharedStateService, SharedStateService>();
```

```csharp
// 사용하는 쪽 (ViewModel에서 DI로 주입)
public class EmployeeEditViewModel : ObservableValidator
{
    private readonly ISharedStateService _sharedState;

    public EmployeeEditViewModel(ISharedStateService sharedState)
    {
        _sharedState = sharedState;

        // 공유 상태에서 선택된 직원 ID 읽기
        if (_sharedState.SelectedEmployeeId is int id)
        {
            _ = LoadEmployeeAsync(id);
        }
    }
}
```

### 세 가지 방법 비교

| 방법 | 사용 시점 | 장점 | 단점 |
|------|----------|------|------|
| **NavigationParameter** | 화면 전환 시 직접 전달 | 명확하고 직관적 | 네비게이션에 의존 |
| **Messenger** | ViewModel 간 느슨한 통신 | 완전한 디커플링 | 추적이 어려울 수 있음 |
| **공유 서비스** | 여러 화면에서 상태 공유 | 간단, 상태 중앙 관리 | 상태 관리 복잡해질 수 있음 |

> **권장**: 일반적인 화면 전환에는 **NavigationParameter**, 서로 독립적인 ViewModel 간 알림에는 **Messenger**를 사용하세요.

---

## 6. 뒤로 가기 (네비게이션 스택)

NavigationService에 이미 스택 기반 뒤로 가기가 구현되어 있지만, 좀 더 완성된 형태를 살펴봅니다.

### 향상된 NavigationService (네비게이션 히스토리 포함)

```csharp
// 파일: Services/NavigationService.cs (향상 버전)
namespace MyApp.Services;

using CommunityToolkit.Mvvm.ComponentModel;
using Microsoft.Extensions.DependencyInjection;
using Serilog;

/// <summary>
/// 네비게이션 히스토리를 관리하는 향상된 NavigationService
/// 뒤로 가기뿐만 아니라 전체 히스토리 조회도 지원합니다.
/// </summary>
public class NavigationService : INavigationService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger _logger;

    // 뒤로 가기 스택
    private readonly Stack<NavigationEntry> _backStack = new();

    // 앞으로 가기 스택 (뒤로 간 후 다시 앞으로 가기)
    private readonly Stack<NavigationEntry> _forwardStack = new();

    public ObservableObject? CurrentViewModel { get; private set; }
    public bool CanGoBack => _backStack.Count > 0;
    public bool CanGoForward => _forwardStack.Count > 0;

    public event Action<ObservableObject?>? CurrentViewModelChanged;

    public NavigationService(IServiceProvider serviceProvider, ILogger logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    public void NavigateTo<TViewModel>(object? parameter = null)
        where TViewModel : ObservableObject
    {
        // 현재 화면을 뒤로가기 스택에 저장
        if (CurrentViewModel is not null)
        {
            _backStack.Push(new NavigationEntry(CurrentViewModel, null));
        }

        // 새 화면으로 이동하면 앞으로 가기 스택은 초기화
        _forwardStack.Clear();

        // DI에서 ViewModel 생성
        var viewModel = _serviceProvider.GetRequiredService<TViewModel>();

        // 파라미터 전달
        if (parameter is not null && viewModel is INavigationAware aware)
        {
            aware.OnNavigatedTo(parameter);
        }

        CurrentViewModel = viewModel;
        CurrentViewModelChanged?.Invoke(CurrentViewModel);

        _logger.Information(
            "네비게이션 → {VM} (뒤로: {BackCount}, 앞으로: {ForwardCount})",
            typeof(TViewModel).Name, _backStack.Count, _forwardStack.Count);
    }

    public void GoBack()
    {
        if (!CanGoBack) return;

        // 현재 화면을 앞으로 가기 스택에 저장
        if (CurrentViewModel is not null)
        {
            _forwardStack.Push(new NavigationEntry(CurrentViewModel, null));
        }

        // 이전 화면 복원
        var entry = _backStack.Pop();
        CurrentViewModel = entry.ViewModel;
        CurrentViewModelChanged?.Invoke(CurrentViewModel);

        _logger.Information("뒤로 가기 → {VM}", CurrentViewModel?.GetType().Name);
    }

    /// <summary>앞으로 가기 (뒤로 간 것을 되돌림)</summary>
    public void GoForward()
    {
        if (!CanGoForward) return;

        if (CurrentViewModel is not null)
        {
            _backStack.Push(new NavigationEntry(CurrentViewModel, null));
        }

        var entry = _forwardStack.Pop();
        CurrentViewModel = entry.ViewModel;
        CurrentViewModelChanged?.Invoke(CurrentViewModel);

        _logger.Information("앞으로 가기 → {VM}", CurrentViewModel?.GetType().Name);
    }

    /// <summary>전체 네비게이션 스택 초기화 (예: 로그아웃 시)</summary>
    public void ClearHistory()
    {
        _backStack.Clear();
        _forwardStack.Clear();
        _logger.Information("네비게이션 히스토리 초기화");
    }
}

/// <summary>네비게이션 스택 항목</summary>
/// <param name="ViewModel">저장된 ViewModel</param>
/// <param name="Parameter">저장된 파라미터</param>
public record NavigationEntry(ObservableObject ViewModel, object? Parameter);
```

---

## 7. 다이얼로그 처리

WinForms에서는 `MessageBox.Show()`를 코드 비하인드에서 직접 호출했지만,
MVVM에서는 ViewModel이 View를 모르므로 **IDialogService**를 통해 간접적으로 처리합니다.

### IDialogService 인터페이스

```csharp
// 파일: Services/IDialogService.cs
namespace MyApp.Services;

/// <summary>
/// 다이얼로그 서비스 인터페이스
/// ViewModel에서 MessageBox나 커스텀 다이얼로그를 표시할 때 사용합니다.
///
/// 왜 인터페이스인가?
/// → ViewModel이 MessageBox.Show()를 직접 호출하면 단위 테스트가 불가능합니다.
/// → 인터페이스를 사용하면 테스트 시 Mock으로 교체할 수 있습니다.
/// </summary>
public interface IDialogService
{
    /// <summary>정보 메시지 표시</summary>
    Task ShowInfoAsync(string title, string message);

    /// <summary>에러 메시지 표시</summary>
    Task ShowErrorAsync(string title, string message);

    /// <summary>경고 메시지 표시</summary>
    Task ShowWarningAsync(string title, string message);

    /// <summary>확인/취소 다이얼로그 — true: 확인, false: 취소</summary>
    Task<bool> ShowConfirmAsync(string title, string message);

    /// <summary>
    /// 커스텀 다이얼로그 표시
    /// ViewModel을 전달하면 해당하는 View가 다이얼로그로 표시됩니다.
    /// </summary>
    /// <typeparam name="TResult">다이얼로그 결과 타입</typeparam>
    /// <param name="dialogViewModel">다이얼로그 ViewModel</param>
    /// <returns>다이얼로그 결과 (닫기 시 default)</returns>
    Task<TResult?> ShowDialogAsync<TResult>(object dialogViewModel);
}
```

### DialogService 구현

```csharp
// 파일: Services/DialogService.cs
namespace MyApp.Services;

using System.Windows;
using Serilog;

/// <summary>
/// 다이얼로그 서비스 구현체
/// WPF의 MessageBox를 래핑하여 ViewModel에서 사용할 수 있게 합니다.
/// </summary>
public class DialogService : IDialogService
{
    private readonly ILogger _logger;

    public DialogService(ILogger logger)
    {
        _logger = logger;
    }

    /// <summary>정보 메시지</summary>
    public Task ShowInfoAsync(string title, string message)
    {
        _logger.Information("다이얼로그 [정보]: {Title} - {Message}", title, message);

        // UI 스레드에서 실행 보장
        Application.Current.Dispatcher.Invoke(() =>
        {
            MessageBox.Show(
                Application.Current.MainWindow,  // 부모 윈도우 지정 (모달)
                message,
                title,
                MessageBoxButton.OK,
                MessageBoxImage.Information);
        });

        return Task.CompletedTask;
    }

    /// <summary>에러 메시지</summary>
    public Task ShowErrorAsync(string title, string message)
    {
        _logger.Error("다이얼로그 [에러]: {Title} - {Message}", title, message);

        Application.Current.Dispatcher.Invoke(() =>
        {
            MessageBox.Show(
                Application.Current.MainWindow,
                message,
                title,
                MessageBoxButton.OK,
                MessageBoxImage.Error);
        });

        return Task.CompletedTask;
    }

    /// <summary>경고 메시지</summary>
    public Task ShowWarningAsync(string title, string message)
    {
        _logger.Warning("다이얼로그 [경고]: {Title} - {Message}", title, message);

        Application.Current.Dispatcher.Invoke(() =>
        {
            MessageBox.Show(
                Application.Current.MainWindow,
                message,
                title,
                MessageBoxButton.OK,
                MessageBoxImage.Warning);
        });

        return Task.CompletedTask;
    }

    /// <summary>확인/취소 다이얼로그</summary>
    public Task<bool> ShowConfirmAsync(string title, string message)
    {
        _logger.Information("다이얼로그 [확인]: {Title} - {Message}", title, message);

        var result = Application.Current.Dispatcher.Invoke(() =>
        {
            return MessageBox.Show(
                Application.Current.MainWindow,
                message,
                title,
                MessageBoxButton.YesNo,
                MessageBoxImage.Question) == MessageBoxResult.Yes;
        });

        return Task.FromResult(result);
    }

    /// <summary>
    /// 커스텀 다이얼로그
    /// ViewModel을 DataContext로 가진 Window를 생성하여 ShowDialog()로 표시합니다.
    /// </summary>
    public Task<TResult?> ShowDialogAsync<TResult>(object dialogViewModel)
    {
        _logger.Information("커스텀 다이얼로그 표시: {Type}", dialogViewModel.GetType().Name);

        var result = Application.Current.Dispatcher.Invoke(() =>
        {
            // 커스텀 다이얼로그 윈도우 생성
            var dialogWindow = new Window
            {
                // DataContext를 ViewModel로 설정
                // → App.xaml의 DataTemplate이 적절한 View를 자동 렌더링
                Content = dialogViewModel,
                Owner = Application.Current.MainWindow,
                WindowStartupLocation = WindowStartupLocation.CenterOwner,
                SizeToContent = SizeToContent.WidthAndHeight,
                ResizeMode = ResizeMode.NoResize,
                ShowInTaskbar = false
            };

            // ViewModel이 IDialogViewModel을 구현하면 창 닫기 기능 연결
            if (dialogViewModel is IDialogViewModel<TResult> dialogVm)
            {
                dialogVm.CloseRequested += (sender, dialogResult) =>
                {
                    dialogWindow.Tag = dialogResult;
                    dialogWindow.DialogResult = true;
                };
            }

            dialogWindow.ShowDialog();

            // Tag에 저장된 결과 반환
            return dialogWindow.Tag is TResult typedResult ? typedResult : default;
        });

        return Task.FromResult(result);
    }
}

/// <summary>
/// 커스텀 다이얼로그 ViewModel이 구현하는 인터페이스
/// 다이얼로그를 닫을 때 결과를 전달하는 데 사용됩니다.
/// </summary>
public interface IDialogViewModel<TResult>
{
    /// <summary>다이얼로그 닫기 요청 이벤트</summary>
    event EventHandler<TResult> CloseRequested;
}
```

### 커스텀 다이얼로그 예제

파일 선택이나 상세 입력 같은 복잡한 다이얼로그가 필요할 때의 패턴입니다.

```csharp
// 파일: ViewModels/Dialogs/DepartmentSelectDialogViewModel.cs
namespace MyApp.ViewModels.Dialogs;

using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using MyApp.Services;

/// <summary>
/// 부서 선택 다이얼로그 ViewModel
/// IDialogViewModel<string>을 구현하여 선택된 부서명을 반환합니다.
/// </summary>
public partial class DepartmentSelectDialogViewModel
    : ObservableObject, IDialogViewModel<string>
{
    /// <summary>다이얼로그 닫기 요청 이벤트</summary>
    public event EventHandler<string>? CloseRequested;

    /// <summary>선택 가능한 부서 목록</summary>
    public ObservableCollection<string> Departments { get; } =
    [
        "개발팀", "기획팀", "디자인팀", "인사팀", "영업팀", "마케팅팀"
    ];

    /// <summary>현재 선택된 부서</summary>
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(ConfirmCommand))]
    private string? _selectedDepartment;

    /// <summary>확인 버튼 — 선택된 부서를 결과로 반환하고 다이얼로그 닫기</summary>
    [RelayCommand(CanExecute = nameof(CanConfirm))]
    private void Confirm()
    {
        if (SelectedDepartment is not null)
        {
            // 이벤트를 발생시키면 DialogService가 창을 닫고 결과를 반환
            CloseRequested?.Invoke(this, SelectedDepartment);
        }
    }

    private bool CanConfirm() => SelectedDepartment is not null;

    /// <summary>취소 버튼 — 빈 문자열을 반환하고 다이얼로그 닫기</summary>
    [RelayCommand]
    private void Cancel()
    {
        CloseRequested?.Invoke(this, string.Empty);
    }
}
```

```csharp
// 커스텀 다이얼로그 사용 (ViewModel에서)
[RelayCommand]
private async Task SelectDepartmentAsync()
{
    var dialog = new DepartmentSelectDialogViewModel();

    // 다이얼로그를 모달로 표시하고 결과(부서명)를 받음
    var selectedDepartment = await _dialogService.ShowDialogAsync<string>(dialog);

    if (!string.IsNullOrEmpty(selectedDepartment))
    {
        Department = selectedDepartment;
        _logger.Information("부서 선택: {Department}", selectedDepartment);
    }
}
```

---

## 8. DI 등록 및 전체 연결

### App.xaml.cs DI 구성

```csharp
// 파일: App.xaml.cs (네비게이션 관련 DI 등록 전체)
namespace MyApp;

using Microsoft.Extensions.DependencyInjection;
using MyApp.Services;
using MyApp.ViewModels;
using MyApp.Views;
using Serilog;

public partial class App : Application
{
    public IServiceProvider Services { get; private set; } = null!;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // Serilog 설정 (간략화)
        Log.Logger = new LoggerConfiguration()
            .WriteTo.Console()
            .WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day)
            .CreateLogger();

        var services = new ServiceCollection();

        // 로깅
        services.AddSingleton(Log.Logger);

        // === 서비스 등록 ===
        // NavigationService는 Singleton (앱 전체에서 하나의 인스턴스)
        services.AddSingleton<INavigationService, NavigationService>();
        services.AddSingleton<IDialogService, DialogService>();

        // === ViewModel 등록 ===
        // MainViewModel은 Singleton (앱 수명과 동일)
        services.AddSingleton<MainViewModel>();

        // 화면 ViewModel은 Transient (매번 새 인스턴스)
        // → 화면을 다시 방문할 때 깨끗한 상태로 시작
        services.AddTransient<EmployeeListViewModel>();
        services.AddTransient<EmployeeEditViewModel>();
        services.AddTransient<SettingsViewModel>();

        // === View 등록 (필요 시) ===
        services.AddSingleton<MainWindow>();

        // === Repository 등록 ===
        // ... (01-simple-crud.md 참고)

        Services = services.BuildServiceProvider();

        // MainWindow 생성 및 표시
        var mainWindow = Services.GetRequiredService<MainWindow>();
        mainWindow.DataContext = Services.GetRequiredService<MainViewModel>();
        mainWindow.Show();
    }
}
```

---

## 9. 전체 코드 예제

### 프로젝트 파일 구조

```
MyApp/
├── App.xaml                          ← DataTemplate 매핑
├── App.xaml.cs                       ← DI 컨테이너 구성
├── Models/
│   └── Employee.cs                   ← 직원 엔티티
├── Repositories/
│   ├── IEmployeeRepository.cs        ← Repository 인터페이스
│   └── EmployeeRepository.cs         ← Dapper + Npgsql 구현
├── Services/
│   ├── INavigationService.cs         ← 네비게이션 인터페이스
│   ├── NavigationService.cs          ← 네비게이션 구현
│   ├── INavigationAware.cs           ← 파라미터 수신 인터페이스
│   ├── IDialogService.cs             ← 다이얼로그 인터페이스
│   └── DialogService.cs              ← 다이얼로그 구현
├── Messages/
│   ├── EmployeeSavedMessage.cs       ← Messenger 메시지
│   └── EmployeeSelectedMessage.cs
├── ViewModels/
│   ├── MainViewModel.cs              ← 메인 ViewModel
│   ├── EmployeeListViewModel.cs      ← 목록 ViewModel
│   ├── EmployeeEditViewModel.cs      ← 편집 ViewModel
│   └── Dialogs/
│       └── DepartmentSelectDialogViewModel.cs
└── Views/
    ├── MainWindow.xaml               ← 메인 윈도우 (ContentControl)
    ├── MainWindow.xaml.cs
    ├── EmployeeListView.xaml         ← 목록 화면
    ├── EmployeeListView.xaml.cs
    ├── EmployeeEditView.xaml         ← 편집 화면
    └── EmployeeEditView.xaml.cs
```

### 동작 요약 다이어그램

```
[사용자 클릭]
    │
    ▼
[View의 Button]
    │  Command="{Binding EditEmployeeCommand}"
    ▼
[EmployeeListViewModel.EditEmployee()]
    │  _navigationService.NavigateTo<EmployeeEditViewModel>(parameter: id)
    ▼
[NavigationService]
    │  1. 현재 ViewModel을 스택에 Push
    │  2. DI에서 EmployeeEditViewModel 생성
    │  3. INavigationAware.OnNavigatedTo(id) 호출
    │  4. CurrentViewModelChanged 이벤트 발생
    ▼
[MainViewModel]
    │  CurrentViewModel = editViewModel
    ▼
[MainWindow.xaml의 ContentControl]
    │  Content="{Binding CurrentViewModel}"
    ▼
[WPF DataTemplate 엔진]
    │  EmployeeEditViewModel → EmployeeEditView
    ▼
[EmployeeEditView 렌더링]
    사용자에게 편집 화면이 표시됨
```

---

## WinForms 대비 핵심 차이점

| 개념 | WinForms | WPF MVVM |
|------|----------|----------|
| 화면 전환 | `new Form().Show()` | `NavigationService.NavigateTo<VM>()` |
| 데이터 전달 | public 프로퍼티, 생성자 | NavigationParameter, Messenger |
| 다이얼로그 | `MessageBox.Show()` 직접 호출 | `IDialogService` 통해 간접 호출 |
| 결과 받기 | `DialogResult` + public 프로퍼티 | Messenger 메시지 또는 `Task<T>` |
| 뒤로 가기 | 없음 (폼 닫기) | 네비게이션 스택 (Stack) |
| View-ViewModel 매핑 | 해당 없음 | DataTemplate 자동 매핑 |
| 의존성 관리 | `new`로 직접 생성 | DI 컨테이너 자동 주입 |

> **요약**: WinForms에서는 "Form A가 Form B를 직접 알고 생성"하지만,
> WPF MVVM에서는 "ViewModel A가 NavigationService에 요청하고, Service가 ViewModel B를 DI에서 생성하여 화면에 표시"합니다.
> ViewModel은 View를 전혀 모르며, View는 DataTemplate에 의해 자동으로 결정됩니다.
