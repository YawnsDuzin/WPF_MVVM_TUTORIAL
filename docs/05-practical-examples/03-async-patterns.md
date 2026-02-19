# 비동기 프로그래밍 패턴 (async/await와 UI 스레드)

> **목표**: WinForms의 BackgroundWorker/Invoke에서 벗어나,
> WPF MVVM에서 async/await를 올바르게 사용하는 방법, UI 스레드와 Dispatcher의 역할,
> 로딩 상태 관리, 취소 지원 등 비동기 패턴을 체계적으로 배웁니다.

---

## 목차

1. [WinForms vs WPF 비동기 비교](#1-winforms-vs-wpf-비동기-비교)
2. [UI 스레드란?](#2-ui-스레드란)
3. [async/await 기본 패턴](#3-asyncawait-기본-패턴)
4. [Dispatcher 사용법](#4-dispatcher-사용법)
5. [IsBusy / IsLoading 패턴](#5-isbusy--isloading-패턴)
6. [Task.Run과 ConfigureAwait](#6-taskrun과-configureawait)
7. [취소 지원 (CancellationToken)](#7-취소-지원-cancellationtoken)
8. [전체 코드 예제 (다양한 시나리오)](#8-전체-코드-예제-다양한-시나리오)

---

## 1. WinForms vs WPF 비동기 비교

### WinForms의 비동기 처리 (예전 방식)

```csharp
// WinForms: BackgroundWorker를 사용한 비동기 처리
private BackgroundWorker _worker = new BackgroundWorker();

private void Form1_Load(object sender, EventArgs e)
{
    // BackgroundWorker 이벤트 핸들러 설정
    _worker.DoWork += Worker_DoWork;
    _worker.RunWorkerCompleted += Worker_RunWorkerCompleted;
    _worker.ProgressChanged += Worker_ProgressChanged;
    _worker.WorkerReportsProgress = true;
}

private void btnLoad_Click(object sender, EventArgs e)
{
    // 버튼 비활성화 (수동)
    btnLoad.Enabled = false;
    progressBar1.Visible = true;

    // 백그라운드 작업 시작
    _worker.RunWorkerAsync();
}

// 백그라운드 스레드에서 실행 (UI 접근 불가!)
private void Worker_DoWork(object sender, DoWorkEventArgs e)
{
    var data = LoadDataFromDatabase();  // 시간이 오래 걸리는 작업
    e.Result = data;

    // UI를 갱신하려면 Invoke 필요
    this.Invoke(new Action(() =>
    {
        label1.Text = "로딩 중...";
    }));
}

// UI 스레드에서 실행 (완료 후)
private void Worker_RunWorkerCompleted(object sender, RunWorkerCompletedEventArgs e)
{
    if (e.Error != null)
    {
        MessageBox.Show("오류: " + e.Error.Message);
    }
    else
    {
        dataGridView1.DataSource = e.Result;
    }

    btnLoad.Enabled = true;
    progressBar1.Visible = false;
}
```

### WPF MVVM의 비동기 처리 (현대 방식)

```csharp
// WPF MVVM: async/await으로 간결하게 처리
[RelayCommand]
private async Task LoadEmployeesAsync()
{
    try
    {
        // 로딩 상태 시작 (바인딩으로 UI가 자동 반응)
        IsLoading = true;

        // await: UI 스레드를 블로킹하지 않고 비동기 작업 대기
        var employees = await _employeeRepository.GetAllAsync();

        // await 이후: UI 스레드로 자동 복귀! (SynchronizationContext 덕분)
        Employees.Clear();
        foreach (var emp in employees)
        {
            Employees.Add(emp);
        }
    }
    catch (Exception ex)
    {
        // 에러 처리도 같은 메서드 안에서
        _logger.Error(ex, "직원 목록 로드 실패");
        await _dialogService.ShowErrorAsync("오류", ex.Message);
    }
    finally
    {
        // 로딩 상태 종료
        IsLoading = false;
    }
}
```

**비교 요약**:

| 항목 | WinForms (BackgroundWorker) | WPF MVVM (async/await) |
|------|---------------------------|----------------------|
| 코드량 | 이벤트 핸들러 3~4개 필요 | 메서드 1개로 충분 |
| 에러 처리 | RunWorkerCompleted에서 e.Error 확인 | try-catch로 자연스럽게 |
| UI 갱신 | Invoke/BeginInvoke 필요 | await 후 자동 복귀 |
| 진행률 | ReportProgress 이벤트 | IProgress\<T\> 또는 바인딩 |
| 취소 | WorkerSupportsCancellation | CancellationToken |
| 코드 흐름 | 콜백 기반 (순서 파악 어려움) | 위에서 아래로 (동기 코드처럼) |

---

## 2. UI 스레드란?

### STA (Single Thread Apartment) 모델

WPF(그리고 WinForms도)는 **STA 모델**을 사용합니다. 이것은 모든 UI 컨트롤이 **하나의 스레드(UI 스레드)**에서만 접근 가능하다는 뜻입니다.

```
┌─────────────────────────────────────────────────────┐
│                    WPF 애플리케이션                    │
│                                                       │
│  ┌─ UI 스레드 (메인 스레드) ─────────────────────┐   │
│  │                                                 │   │
│  │  1. XAML 렌더링                                 │   │
│  │  2. 사용자 입력 처리 (클릭, 키보드)             │   │
│  │  3. 데이터 바인딩 업데이트                       │   │
│  │  4. 애니메이션 렌더링                            │   │
│  │  5. 컨트롤 프로퍼티 변경                         │   │
│  │                                                 │   │
│  │  → 이 스레드를 블로킹하면 UI가 멈춤! (Not       │   │
│  │    Responding)                                   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                       │
│  ┌─ 백그라운드 스레드들 ────────────────────────┐    │
│  │                                                 │   │
│  │  Task.Run(), ThreadPool, async I/O 등           │   │
│  │  → UI 컨트롤에 직접 접근 불가!                   │   │
│  │  → Dispatcher를 통해야 UI 접근 가능              │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Dispatcher의 역할

```csharp
// Dispatcher는 UI 스레드의 "메시지 큐 관리자"입니다.
// 백그라운드 스레드에서 UI를 변경하려면 Dispatcher에 작업을 위임해야 합니다.

// 비유: 레스토랑의 서빙 담당자
// - 주방(백그라운드 스레드)에서 요리가 완성됨
// - 주방 직원이 직접 손님 테이블에 가져다 줄 수 없음 (스레드 규칙)
// - 서빙 담당자(Dispatcher)에게 "이 요리를 2번 테이블에 가져다줘"라고 요청
// - 서빙 담당자가 적절한 타이밍에 가져다줌
```

### 왜 UI 컨트롤은 생성 스레드에서만 접근 가능한가?

```csharp
// 잘못된 예: 백그라운드 스레드에서 UI 컨트롤 직접 접근
// → System.InvalidOperationException 발생!
Task.Run(() =>
{
    // 이 코드는 ThreadPool 스레드에서 실행됨
    // UI 컨트롤에 직접 접근하면 예외 발생!
    // textBox.Text = "Hello";  // InvalidOperationException!

    // 이유: UI 컨트롤은 스레드 안전(thread-safe)하지 않음
    // 여러 스레드가 동시에 컨트롤을 수정하면 예측 불가능한 상태가 됨
    // 예: 렌더링 도중 텍스트가 바뀌면 화면이 깨질 수 있음
});
```

---

## 3. async/await 기본 패턴

### ViewModel의 async Command

```csharp
// 파일: ViewModels/DataLoadViewModel.cs
namespace MyApp.ViewModels;

using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using MyApp.Models;
using MyApp.Repositories;
using Serilog;

/// <summary>
/// 비동기 데이터 로딩 패턴 예제 ViewModel
/// </summary>
public partial class DataLoadViewModel : ObservableObject
{
    private readonly IEmployeeRepository _employeeRepository;
    private readonly ILogger _logger;

    public DataLoadViewModel(
        IEmployeeRepository employeeRepository,
        ILogger logger)
    {
        _employeeRepository = employeeRepository;
        _logger = logger;
    }

    /// <summary>직원 목록</summary>
    public ObservableCollection<Employee> Employees { get; } = [];

    /// <summary>로딩 상태</summary>
    [ObservableProperty]
    private bool _isLoading;

    /// <summary>에러 메시지 (에러 발생 시 UI에 표시)</summary>
    [ObservableProperty]
    private string? _errorMessage;

    /// <summary>
    /// [핵심] async Task를 반환하는 RelayCommand
    ///
    /// CommunityToolkit.Mvvm의 [RelayCommand]는 async Task 메서드를 지원합니다.
    /// - 자동으로 IAsyncRelayCommand 타입의 프로퍼티 생성
    /// - IsRunning 프로퍼티로 실행 상태 추적 가능
    /// - 중복 실행 방지 (이미 실행 중이면 다시 실행하지 않음)
    /// </summary>
    [RelayCommand]
    private async Task LoadDataAsync()
    {
        try
        {
            // 1. 로딩 시작 → UI의 프로그레스 바 표시
            IsLoading = true;
            ErrorMessage = null;

            _logger.Information("데이터 로딩 시작");

            // 2. 비동기 DB 조회 (UI 스레드를 블로킹하지 않음)
            //    await 키워드 덕분에:
            //    - DB 조회가 진행되는 동안 UI는 정상적으로 반응
            //    - 조회가 완료되면 자동으로 UI 스레드로 복귀
            var employees = await _employeeRepository.GetAllAsync();

            // 3. await 이후: UI 스레드에서 실행됨
            //    ObservableCollection을 안전하게 조작 가능
            Employees.Clear();
            foreach (var emp in employees)
            {
                Employees.Add(emp);
            }

            _logger.Information("데이터 로딩 완료: {Count}건", Employees.Count);
        }
        catch (Exception ex)
        {
            // 4. 에러 처리: try-catch로 자연스럽게
            _logger.Error(ex, "데이터 로딩 실패");
            ErrorMessage = $"데이터를 불러오는 중 오류가 발생했습니다: {ex.Message}";
        }
        finally
        {
            // 5. 로딩 종료 → UI의 프로그레스 바 숨김
            IsLoading = false;
        }
    }
}
```

### XAML에서 비동기 상태 활용

```xml
<!-- 파일: Views/DataLoadView.xaml -->
<UserControl x:Class="MyApp.Views.DataLoadView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- 로드 버튼 (로딩 중에는 비활성화) -->
        <Button Grid.Row="0"
                Content="데이터 로드"
                Command="{Binding LoadDataCommand}"
                IsEnabled="{Binding IsLoading, Converter={StaticResource InverseBoolConverter}}"
                Padding="16,8"
                Margin="0,0,0,12"/>

        <!-- 데이터 표시 영역 -->
        <DataGrid Grid.Row="1"
                  ItemsSource="{Binding Employees}"
                  AutoGenerateColumns="True"
                  IsReadOnly="True"/>

        <!-- 로딩 오버레이 -->
        <Border Grid.Row="1"
                Background="#80FFFFFF"
                Visibility="{Binding IsLoading, Converter={StaticResource BooleanToVisibilityConverter}}">
            <StackPanel HorizontalAlignment="Center"
                        VerticalAlignment="Center">
                <ProgressBar IsIndeterminate="True"
                             Width="200" Height="4"
                             Margin="0,0,0,8"/>
                <TextBlock Text="데이터를 불러오는 중..."
                           Foreground="Gray"/>
            </StackPanel>
        </Border>

        <!-- 에러 메시지 (ErrorMessage가 있을 때만 표시) -->
        <Border Grid.Row="2"
                Background="#FFEBEE"
                Padding="12"
                Margin="0,8,0,0"
                CornerRadius="4"
                Visibility="{Binding ErrorMessage, Converter={StaticResource NullToCollapsedConverter}}">
            <TextBlock Text="{Binding ErrorMessage}"
                       Foreground="#C62828"
                       TextWrapping="Wrap"/>
        </Border>
    </Grid>
</UserControl>
```

### async/await 동작 흐름 시각화

```
[사용자가 버튼 클릭]
    │  UI 스레드
    ▼
LoadDataAsync() 시작
    │  IsLoading = true  ← UI가 프로그레스 바 표시
    │
    │  var employees = await _repository.GetAllAsync();
    │                    ↓
    │         ┌─────────────────────────────┐
    │         │  IO 스레드 (DB 조회 진행)     │
    │         │  UI 스레드는 자유!            │
    │         │  (사용자가 다른 작업 가능)     │
    │         └─────────────────────────────┘
    │                    ↓  조회 완료
    │  UI 스레드로 자동 복귀
    │
    │  Employees.Clear();           ← UI 스레드에서 안전하게 실행
    │  Employees.Add(emp);          ← ObservableCollection 갱신
    │  IsLoading = false;           ← UI가 프로그레스 바 숨김
    ▼
LoadDataAsync() 완료
```

---

## 4. Dispatcher 사용법

### 언제 Dispatcher가 필요한가?

`async/await`를 사용하면 **대부분의 경우** Dispatcher가 필요 없습니다. await 후에 자동으로 UI 스레드로 복귀하기 때문입니다.

하지만 다음과 같은 경우에는 Dispatcher가 **반드시** 필요합니다:

```csharp
// 1. 외부 라이브러리의 콜백 (MQTT, WebSocket 등)
//    → 라이브러리가 자체 스레드에서 콜백을 호출함

// 2. 이벤트 핸들러가 백그라운드 스레드에서 발생하는 경우
//    → 시리얼 포트, 타이머(System.Threading.Timer) 등

// 3. Task.Run() 내부에서 UI를 업데이트해야 하는 경우
//    → 일반적으로는 Task.Run 결과를 await로 받아 처리하는 것이 더 좋음
```

### 실전 예제: MQTT 콜백에서 UI 업데이트

```csharp
// 파일: ViewModels/MqttDashboardViewModel.cs
namespace MyApp.ViewModels;

using System.Collections.ObjectModel;
using System.Windows;
using CommunityToolkit.Mvvm.ComponentModel;
using MQTTnet.Client;
using Serilog;

/// <summary>
/// MQTT 메시지를 수신하여 대시보드에 표시하는 ViewModel
/// MQTT 콜백은 백그라운드 스레드에서 호출되므로 Dispatcher가 필요합니다.
/// </summary>
public partial class MqttDashboardViewModel : ObservableObject
{
    private readonly IMqttClient _mqttClient;
    private readonly ILogger _logger;

    /// <summary>수신한 메시지 목록 (UI에 바인딩)</summary>
    public ObservableCollection<string> Messages { get; } = [];

    /// <summary>마지막 수신 시각</summary>
    [ObservableProperty]
    private string _lastReceivedTime = "-";

    public MqttDashboardViewModel(IMqttClient mqttClient, ILogger logger)
    {
        _mqttClient = mqttClient;
        _logger = logger;

        // MQTT 메시지 수신 이벤트 등록
        _mqttClient.ApplicationMessageReceivedAsync += OnMqttMessageReceivedAsync;
    }

    /// <summary>
    /// MQTT 메시지 수신 콜백
    /// 주의: 이 메서드는 MQTT 라이브러리의 내부 스레드에서 호출됩니다!
    /// UI 스레드가 아니므로 ObservableCollection에 직접 접근하면 예외 발생!
    /// </summary>
    private Task OnMqttMessageReceivedAsync(MqttApplicationMessageReceivedEventArgs e)
    {
        var payload = System.Text.Encoding.UTF8.GetString(
            e.ApplicationMessage.PayloadSegment);

        _logger.Information("MQTT 메시지 수신: {Topic} = {Payload}",
            e.ApplicationMessage.Topic, payload);

        // ★ Dispatcher를 사용하여 UI 스레드에서 실행 ★
        Application.Current.Dispatcher.Invoke(() =>
        {
            // 이제 UI 스레드에서 안전하게 ObservableCollection 조작 가능
            Messages.Add($"[{DateTime.Now:HH:mm:ss}] {e.ApplicationMessage.Topic}: {payload}");
            LastReceivedTime = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss");

            // 메시지가 너무 많아지면 오래된 것 제거
            while (Messages.Count > 100)
            {
                Messages.RemoveAt(0);
            }
        });

        return Task.CompletedTask;
    }
}
```

### DispatcherHelper 유틸리티

자주 사용하는 Dispatcher 호출을 유틸리티로 만들어 두면 편리합니다.

```csharp
// 파일: Helpers/DispatcherHelper.cs
namespace MyApp.Helpers;

using System.Windows;
using System.Windows.Threading;

/// <summary>
/// Dispatcher 사용을 간편하게 만드는 헬퍼 클래스
/// 백그라운드 스레드에서 UI 작업을 안전하게 실행할 수 있습니다.
/// </summary>
public static class DispatcherHelper
{
    /// <summary>
    /// UI 스레드에서 동기적으로 실행
    /// 작업이 완료될 때까지 현재 스레드를 블로킹합니다.
    /// </summary>
    public static void RunOnUiThread(Action action)
    {
        var dispatcher = Application.Current?.Dispatcher;

        if (dispatcher is null)
            return;

        // 이미 UI 스레드라면 직접 실행
        if (dispatcher.CheckAccess())
        {
            action();
        }
        else
        {
            // 백그라운드 스레드라면 Dispatcher를 통해 실행
            dispatcher.Invoke(action);
        }
    }

    /// <summary>
    /// UI 스레드에서 비동기적으로 실행
    /// 현재 스레드를 블로킹하지 않고, UI 스레드의 메시지 큐에 작업을 추가합니다.
    /// </summary>
    public static void BeginRunOnUiThread(
        Action action,
        DispatcherPriority priority = DispatcherPriority.Normal)
    {
        var dispatcher = Application.Current?.Dispatcher;

        if (dispatcher is null)
            return;

        if (dispatcher.CheckAccess())
        {
            action();
        }
        else
        {
            dispatcher.BeginInvoke(action, priority);
        }
    }

    /// <summary>
    /// UI 스레드에서 비동기적으로 실행하고 결과를 await로 받기
    /// </summary>
    public static async Task<T> RunOnUiThreadAsync<T>(Func<T> func)
    {
        var dispatcher = Application.Current?.Dispatcher;

        if (dispatcher is null)
            return default!;

        if (dispatcher.CheckAccess())
        {
            return func();
        }

        return await dispatcher.InvokeAsync(func);
    }
}
```

### Dispatcher 사용 판단 플로차트

```
[UI 업데이트가 필요한 상황]
    │
    ├── async/await 메서드 내부인가?
    │   │
    │   ├── Yes → Dispatcher 불필요 (await 후 자동 복귀)
    │   │
    │   └── No → 아래 확인
    │
    ├── 외부 라이브러리 콜백인가? (MQTT, WebSocket, 시리얼 등)
    │   │
    │   ├── Yes → ★ Dispatcher 필요! ★
    │   │
    │   └── No → 아래 확인
    │
    ├── System.Threading.Timer 콜백인가?
    │   │
    │   ├── Yes → ★ Dispatcher 필요! ★
    │   │         (DispatcherTimer를 대신 사용하는 것이 더 좋음)
    │   │
    │   └── No → 아래 확인
    │
    └── Task.Run() 내부인가?
        │
        ├── Yes → Dispatcher 필요 (하지만 결과를 await로 받는 것이 더 좋음)
        │
        └── No → 대부분 Dispatcher 불필요
```

---

## 5. IsBusy / IsLoading 패턴

비동기 작업 중 사용자에게 로딩 상태를 보여주고, 중복 실행을 방지하는 패턴입니다.

### 기본 패턴: 버튼 비활성화 + 프로그레스 바

```csharp
// 파일: ViewModels/LoadingPatternViewModel.cs
namespace MyApp.ViewModels;

using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Serilog;

/// <summary>
/// IsLoading 패턴 예제
/// 비동기 작업 중 UI 상태를 관리하는 표준 패턴입니다.
/// </summary>
public partial class LoadingPatternViewModel : ObservableObject
{
    private readonly ILogger _logger;

    /// <summary>로딩 중 여부 — true이면 UI에 로딩 인디케이터 표시</summary>
    [ObservableProperty]
    private bool _isLoading;

    /// <summary>진행률 (0~100) — 프로그레스 바에 바인딩</summary>
    [ObservableProperty]
    private int _progressValue;

    /// <summary>진행 상태 메시지</summary>
    [ObservableProperty]
    private string _progressMessage = string.Empty;

    public LoadingPatternViewModel(ILogger logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// 데이터 가져오기 커맨드
    /// IsLoading이 true인 동안에는 다시 실행할 수 없음 (중복 실행 방지)
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanFetchData))]
    private async Task FetchDataAsync()
    {
        try
        {
            IsLoading = true;
            ProgressValue = 0;
            ProgressMessage = "데이터 준비 중...";

            // 1단계: 데이터 조회
            ProgressMessage = "서버에서 데이터를 가져오는 중...";
            var rawData = await FetchFromServerAsync();
            ProgressValue = 33;

            // 2단계: 데이터 가공
            ProgressMessage = "데이터를 가공하는 중...";
            var processedData = await ProcessDataAsync(rawData);
            ProgressValue = 66;

            // 3단계: UI에 표시
            ProgressMessage = "화면에 표시하는 중...";
            DisplayData(processedData);
            ProgressValue = 100;

            ProgressMessage = "완료!";
            _logger.Information("데이터 처리 완료");
        }
        catch (Exception ex)
        {
            ProgressMessage = $"오류: {ex.Message}";
            _logger.Error(ex, "데이터 처리 실패");
        }
        finally
        {
            IsLoading = false;
            // CanExecute 재평가 (버튼 다시 활성화)
            FetchDataCommand.NotifyCanExecuteChanged();
        }
    }

    /// <summary>로딩 중이 아닐 때만 실행 가능</summary>
    private bool CanFetchData() => !IsLoading;

    // 헬퍼 메서드들 (실제로는 Repository나 Service 호출)
    private async Task<byte[]> FetchFromServerAsync()
    {
        await Task.Delay(1500); // 시뮬레이션
        return [1, 2, 3];
    }

    private async Task<string[]> ProcessDataAsync(byte[] data)
    {
        await Task.Delay(1000); // 시뮬레이션
        return ["처리된 데이터"];
    }

    private void DisplayData(string[] data)
    {
        // ObservableCollection에 추가 등
    }
}
```

### XAML: 로딩 오버레이 패턴

```xml
<!-- 파일: Views/LoadingPatternView.xaml -->
<UserControl x:Class="MyApp.Views.LoadingPatternView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <Grid>
        <!-- 메인 콘텐츠 -->
        <StackPanel Margin="16">
            <!-- 실행 버튼 -->
            <Button Content="데이터 가져오기"
                    Command="{Binding FetchDataCommand}"
                    Padding="16,10"
                    FontSize="14"
                    Margin="0,0,0,16"/>

            <!-- 진행률 표시 (IsLoading이 true일 때만 보임) -->
            <StackPanel Visibility="{Binding IsLoading,
                                     Converter={StaticResource BooleanToVisibilityConverter}}">

                <!-- 진행률 바 -->
                <ProgressBar Value="{Binding ProgressValue}"
                             Maximum="100"
                             Height="20"
                             Margin="0,0,0,8"/>

                <!-- 진행 상태 메시지 -->
                <TextBlock Text="{Binding ProgressMessage}"
                           Foreground="Gray"
                           FontSize="13"/>
            </StackPanel>

            <!-- 데이터 표시 영역 -->
            <DataGrid ItemsSource="{Binding Employees}"
                      Margin="0,16,0,0"/>
        </StackPanel>

        <!-- 로딩 오버레이 (전체 화면을 덮는 반투명 레이어) -->
        <Grid Visibility="{Binding IsLoading,
                           Converter={StaticResource BooleanToVisibilityConverter}}">
            <!-- 반투명 배경 -->
            <Border Background="#60000000"/>

            <!-- 중앙 로딩 카드 -->
            <Border Background="White"
                    CornerRadius="8"
                    Padding="32,24"
                    HorizontalAlignment="Center"
                    VerticalAlignment="Center"
                    MinWidth="300">
                <Border.Effect>
                    <DropShadowEffect BlurRadius="20"
                                      ShadowDepth="2"
                                      Opacity="0.3"/>
                </Border.Effect>

                <StackPanel>
                    <!-- 프로그레스 링 -->
                    <ProgressBar IsIndeterminate="True"
                                 Height="4"
                                 Margin="0,0,0,16"/>

                    <!-- 진행률 바 (확정적 진행률) -->
                    <ProgressBar Value="{Binding ProgressValue}"
                                 Maximum="100"
                                 Height="8"
                                 Margin="0,0,0,12"/>

                    <!-- 상태 메시지 -->
                    <TextBlock Text="{Binding ProgressMessage}"
                               HorizontalAlignment="Center"
                               FontSize="14"/>
                </StackPanel>
            </Border>
        </Grid>
    </Grid>
</UserControl>
```

---

## 6. Task.Run과 ConfigureAwait

### CPU 바운드 vs IO 바운드 작업

```csharp
// CPU 바운드 작업: 계산, 데이터 가공, 이미지 처리 등
// → Task.Run()을 사용하여 백그라운드 스레드에서 실행

// IO 바운드 작업: DB 조회, HTTP 요청, 파일 읽기 등
// → async/await만으로 충분 (Task.Run 불필요)
```

```csharp
// 파일: ViewModels/ComputeViewModel.cs
namespace MyApp.ViewModels;

using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class ComputeViewModel : ObservableObject
{
    [ObservableProperty]
    private bool _isLoading;

    [ObservableProperty]
    private string _result = string.Empty;

    /// <summary>
    /// IO 바운드 작업 — Task.Run 불필요
    /// DB 조회, HTTP 요청 등은 이미 비동기적이므로 await만 하면 됨
    /// </summary>
    [RelayCommand]
    private async Task LoadFromDatabaseAsync()
    {
        IsLoading = true;

        // ★ Task.Run 없이 직접 await ★
        // Dapper의 QueryAsync는 이미 비동기 IO를 사용하므로
        // 추가적인 스레드를 점유하지 않음
        var data = await _repository.GetAllAsync();

        // await 후: UI 스레드로 자동 복귀
        Result = $"조회 결과: {data.Count()}건";
        IsLoading = false;
    }

    /// <summary>
    /// CPU 바운드 작업 — Task.Run 사용
    /// 무거운 계산은 UI 스레드를 블로킹하므로 백그라운드로 넘겨야 함
    /// </summary>
    [RelayCommand]
    private async Task CalculateAsync()
    {
        IsLoading = true;

        // ★ Task.Run으로 CPU 작업을 백그라운드 스레드에서 실행 ★
        var result = await Task.Run(() =>
        {
            // 이 코드는 ThreadPool 스레드에서 실행됨
            // UI 스레드를 블로킹하지 않음
            var sum = 0L;
            for (var i = 0; i < 1_000_000_000; i++)
            {
                sum += i;  // 시간이 오래 걸리는 CPU 계산
            }
            return sum;
        });

        // await 후: UI 스레드로 자동 복귀
        Result = $"계산 결과: {result:N0}";
        IsLoading = false;
    }
}
```

### ConfigureAwait(false) vs ConfigureAwait(true)

```csharp
// ConfigureAwait는 "await 이후 어떤 스레드에서 계속 실행할지"를 제어합니다.

// ┌─────────────────────────────────────────────────────────────┐
// │ ConfigureAwait(true)  — 기본값                               │
// │ await 이후 원래 컨텍스트(UI 스레드)로 복귀                     │
// │ → ViewModel, View 관련 코드에서 사용                          │
// │                                                               │
// │ ConfigureAwait(false)                                         │
// │ await 이후 아무 스레드에서나 계속 실행                          │
// │ → 라이브러리 코드, Repository, Service에서 사용                │
// │ → UI와 관련 없는 코드의 성능을 약간 향상                       │
// └─────────────────────────────────────────────────────────────┘
```

```csharp
// 파일: Repositories/EmployeeRepository.cs

/// <summary>
/// Repository 계층에서의 ConfigureAwait(false) 사용 예시
/// Repository는 UI와 무관하므로 ConfigureAwait(false) 사용이 적절합니다.
/// </summary>
public async Task<IEnumerable<Employee>> GetAllAsync()
{
    const string sql = "SELECT * FROM employees ORDER BY name";

    await using var connection = CreateConnection();

    // ConfigureAwait(false): UI 스레드로 돌아갈 필요 없음
    // → ThreadPool 스레드에서 계속 실행 (미세한 성능 향상)
    var employees = await connection.QueryAsync<Employee>(sql)
        .ConfigureAwait(false);

    return employees;
}
```

```csharp
// 파일: ViewModels/EmployeeListViewModel.cs

/// <summary>
/// ViewModel에서는 ConfigureAwait(false)를 사용하지 마세요!
/// await 이후 UI 스레드에서 ObservableCollection을 조작해야 하기 때문입니다.
/// </summary>
[RelayCommand]
private async Task LoadEmployeesAsync()
{
    IsLoading = true;

    // ★ ViewModel에서는 ConfigureAwait 생략 (= ConfigureAwait(true)) ★
    // await 이후 UI 스레드로 복귀해야 Employees.Add()가 안전
    var employees = await _employeeRepository.GetAllAsync();

    // 이 코드는 UI 스레드에서 실행됨 (ConfigureAwait(true) 덕분)
    Employees.Clear();
    foreach (var emp in employees)
    {
        Employees.Add(emp);  // UI 스레드에서만 안전
    }

    IsLoading = false;
}
```

### ConfigureAwait 사용 가이드

| 계층 | ConfigureAwait | 이유 |
|------|---------------|------|
| **ViewModel** | 생략 (= true) | await 후 UI 스레드에서 바인딩 프로퍼티 갱신 필요 |
| **Repository** | `false` | UI와 무관한 데이터 접근 계층 |
| **Service** | `false` | UI와 무관한 비즈니스 로직 |
| **Helper/Utility** | `false` | 라이브러리 성격의 코드 |

---

## 7. 취소 지원 (CancellationToken)

### 사용자가 긴 작업을 취소하는 패턴

```csharp
// 파일: ViewModels/CancellableOperationViewModel.cs
namespace MyApp.ViewModels;

using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using MyApp.Models;
using Serilog;

/// <summary>
/// CancellationToken을 사용한 작업 취소 패턴 예제
/// 사용자가 "취소" 버튼을 누르면 진행 중인 작업을 안전하게 중단합니다.
/// </summary>
public partial class CancellableOperationViewModel : ObservableObject
{
    private readonly ILogger _logger;

    // 취소 토큰 소스: 취소 신호를 발생시키는 객체
    private CancellationTokenSource? _cts;

    /// <summary>로딩 상태</summary>
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(SearchCommand))]
    [NotifyCanExecuteChangedFor(nameof(CancelSearchCommand))]
    private bool _isSearching;

    /// <summary>검색어</summary>
    [ObservableProperty]
    private string _searchText = string.Empty;

    /// <summary>진행 상태 메시지</summary>
    [ObservableProperty]
    private string _statusMessage = "준비";

    /// <summary>검색 결과</summary>
    public ObservableCollection<Employee> Results { get; } = [];

    public CancellableOperationViewModel(ILogger logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// 검색 실행 커맨드 — 취소 가능한 비동기 작업
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanSearch))]
    private async Task SearchAsync()
    {
        // 1. 이전 검색이 진행 중이면 취소
        _cts?.Cancel();
        _cts?.Dispose();

        // 2. 새 CancellationTokenSource 생성
        _cts = new CancellationTokenSource();
        var token = _cts.Token;

        try
        {
            IsSearching = true;
            StatusMessage = "검색 중...";
            Results.Clear();

            _logger.Information("검색 시작: {Query}", SearchText);

            // 3. 시간이 걸리는 검색 작업 시뮬레이션
            //    각 단계에서 취소 토큰을 확인합니다.
            for (var page = 1; page <= 10; page++)
            {
                // ★ 취소 확인: 사용자가 취소 버튼을 눌렀으면 OperationCanceledException 발생 ★
                token.ThrowIfCancellationRequested();

                StatusMessage = $"검색 중... (페이지 {page}/10)";

                // DB 조회 시 CancellationToken 전달
                // Dapper/Npgsql도 CancellationToken을 지원합니다.
                var pageResults = await SearchPageAsync(SearchText, page, token);

                // 취소되지 않았으면 결과 추가
                foreach (var result in pageResults)
                {
                    Results.Add(result);
                }
            }

            StatusMessage = $"검색 완료: {Results.Count}건";
            _logger.Information("검색 완료: {Count}건", Results.Count);
        }
        catch (OperationCanceledException)
        {
            // 4. 취소된 경우 — 에러가 아닌 정상적인 취소 처리
            StatusMessage = $"검색 취소됨 (중간 결과: {Results.Count}건)";
            _logger.Information("검색 취소됨");
        }
        catch (Exception ex)
        {
            StatusMessage = $"오류: {ex.Message}";
            _logger.Error(ex, "검색 실패");
        }
        finally
        {
            IsSearching = false;
        }
    }

    private bool CanSearch() => !IsSearching;

    /// <summary>
    /// 검색 취소 커맨드
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanCancelSearch))]
    private void CancelSearch()
    {
        if (_cts is not null && !_cts.IsCancellationRequested)
        {
            _logger.Information("사용자가 검색 취소 요청");
            _cts.Cancel();  // 취소 신호 발생
        }
    }

    private bool CanCancelSearch() => IsSearching;

    /// <summary>
    /// 페이지별 검색 시뮬레이션 (실제로는 Repository 호출)
    /// CancellationToken을 매개변수로 받아 DB 조회도 취소 가능하게 합니다.
    /// </summary>
    private async Task<List<Employee>> SearchPageAsync(
        string query, int page, CancellationToken cancellationToken)
    {
        // Dapper에 CancellationToken을 전달하는 실제 예시:
        // var result = await connection.QueryAsync<Employee>(
        //     new CommandDefinition(sql, parameters, cancellationToken: cancellationToken));

        await Task.Delay(500, cancellationToken);  // 시뮬레이션
        return
        [
            new Employee { Name = $"검색결과 {page}-1", Department = "개발팀" },
            new Employee { Name = $"검색결과 {page}-2", Department = "기획팀" }
        ];
    }

    /// <summary>
    /// ViewModel이 해제될 때 CancellationTokenSource도 정리
    /// </summary>
    public void Cleanup()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

### XAML: 취소 버튼 포함 UI

```xml
<!-- 파일: Views/CancellableOperationView.xaml (검색 영역) -->
<StackPanel Orientation="Horizontal" Margin="0,0,0,12">

    <!-- 검색어 입력 -->
    <TextBox Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
             Width="300"
             Padding="8,6"
             FontSize="14"
             Margin="0,0,8,0"/>

    <!-- 검색 버튼 (검색 중에는 비활성화) -->
    <Button Content="검색"
            Command="{Binding SearchCommand}"
            Padding="16,6"
            FontSize="14"
            Margin="0,0,8,0"/>

    <!-- 취소 버튼 (검색 중에만 활성화) -->
    <Button Content="취소"
            Command="{Binding CancelSearchCommand}"
            Padding="16,6"
            FontSize="14"
            Background="#F44336"
            Foreground="White"/>

    <!-- 상태 메시지 -->
    <TextBlock Text="{Binding StatusMessage}"
               VerticalAlignment="Center"
               Foreground="Gray"
               Margin="16,0,0,0"/>
</StackPanel>
```

### 타임아웃과 결합

```csharp
// 타임아웃 + 사용자 취소를 동시에 지원하는 패턴
[RelayCommand]
private async Task FetchWithTimeoutAsync()
{
    // 30초 타임아웃 설정
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

    // 사용자 취소 토큰과 타임아웃 토큰을 연결
    // 둘 중 하나라도 취소되면 linkedToken이 취소됨
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        timeoutCts.Token, _userCts.Token);

    try
    {
        IsLoading = true;
        var data = await _repository.GetAllAsync(linkedCts.Token);
        // ... 결과 처리
    }
    catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested)
    {
        // 타임아웃으로 취소된 경우
        StatusMessage = "요청 시간이 초과되었습니다. (30초)";
        _logger.Warning("요청 타임아웃");
    }
    catch (OperationCanceledException)
    {
        // 사용자가 직접 취소한 경우
        StatusMessage = "사용자에 의해 취소되었습니다.";
        _logger.Information("사용자 취소");
    }
    finally
    {
        IsLoading = false;
    }
}
```

---

## 8. 전체 코드 예제 (다양한 시나리오)

### 시나리오 1: 파일 업로드 (진행률 + 취소)

```csharp
// 파일: ViewModels/FileUploadViewModel.cs
namespace MyApp.ViewModels;

using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Serilog;

/// <summary>
/// 파일 업로드 예제 — 진행률 보고 + 취소 지원
/// IProgress<T>를 사용하여 백그라운드 작업의 진행률을 UI에 안전하게 보고합니다.
/// </summary>
public partial class FileUploadViewModel : ObservableObject
{
    private readonly ILogger _logger;
    private CancellationTokenSource? _uploadCts;

    [ObservableProperty]
    private bool _isUploading;

    [ObservableProperty]
    private int _uploadProgress;  // 0~100

    [ObservableProperty]
    private string _uploadStatus = "대기 중";

    [ObservableProperty]
    private string _fileName = string.Empty;

    public FileUploadViewModel(ILogger logger)
    {
        _logger = logger;
    }

    [RelayCommand]
    private async Task UploadFileAsync()
    {
        if (string.IsNullOrEmpty(FileName)) return;

        _uploadCts = new CancellationTokenSource();

        try
        {
            IsUploading = true;
            UploadProgress = 0;
            UploadStatus = "업로드 준비 중...";

            // IProgress<int>: 백그라운드 작업에서 UI 스레드로
            // 진행률을 안전하게 보고하는 인터페이스
            var progress = new Progress<int>(percent =>
            {
                // 이 콜백은 UI 스레드에서 실행됨 (SynchronizationContext 덕분)
                UploadProgress = percent;
                UploadStatus = $"업로드 중... {percent}%";
            });

            // 실제 업로드 (Task.Run + IProgress)
            await Task.Run(async () =>
            {
                // 파일을 청크(조각) 단위로 업로드하며 진행률 보고
                var totalChunks = 100;
                for (var i = 1; i <= totalChunks; i++)
                {
                    _uploadCts.Token.ThrowIfCancellationRequested();

                    // 실제로는 S3나 HTTP로 청크 업로드
                    await Task.Delay(50, _uploadCts.Token);

                    // 진행률 보고 (UI 스레드의 콜백이 자동 호출됨)
                    ((IProgress<int>)progress).Report(i);
                }
            }, _uploadCts.Token);

            UploadStatus = "업로드 완료!";
            _logger.Information("파일 업로드 완료: {FileName}", FileName);
        }
        catch (OperationCanceledException)
        {
            UploadStatus = "업로드 취소됨";
            _logger.Information("파일 업로드 취소: {FileName}", FileName);
        }
        catch (Exception ex)
        {
            UploadStatus = $"업로드 실패: {ex.Message}";
            _logger.Error(ex, "파일 업로드 실패: {FileName}", FileName);
        }
        finally
        {
            IsUploading = false;
            _uploadCts?.Dispose();
            _uploadCts = null;
        }
    }

    [RelayCommand]
    private void CancelUpload()
    {
        _uploadCts?.Cancel();
    }
}
```

### 시나리오 2: 여러 작업 동시 실행 (Task.WhenAll)

```csharp
// 파일: ViewModels/DashboardViewModel.cs
namespace MyApp.ViewModels;

using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Serilog;

/// <summary>
/// 대시보드 초기 로딩 — 여러 API를 동시에 호출하여 시간 단축
/// </summary>
public partial class DashboardViewModel : ObservableObject
{
    private readonly IEmployeeRepository _employeeRepo;
    private readonly ILogger _logger;

    [ObservableProperty]
    private bool _isLoading;

    [ObservableProperty]
    private int _totalEmployees;

    [ObservableProperty]
    private int _todayAttendance;

    [ObservableProperty]
    private string _latestNotice = string.Empty;

    public DashboardViewModel(
        IEmployeeRepository employeeRepo,
        ILogger logger)
    {
        _employeeRepo = employeeRepo;
        _logger = logger;
    }

    /// <summary>
    /// 대시보드 초기화 — 여러 비동기 작업을 병렬로 실행
    /// Task.WhenAll을 사용하면 모든 작업이 동시에 시작됩니다.
    /// </summary>
    [RelayCommand]
    private async Task InitializeDashboardAsync()
    {
        try
        {
            IsLoading = true;

            // ★ Task.WhenAll: 모든 Task가 완료될 때까지 대기 ★
            // 순차 실행: 1초 + 1초 + 1초 = 3초
            // 병렬 실행: max(1초, 1초, 1초) = 1초  ← 훨씬 빠름!
            var employeesTask = _employeeRepo.GetAllAsync();
            var attendanceTask = GetTodayAttendanceAsync();
            var noticeTask = GetLatestNoticeAsync();

            // 세 작업이 모두 완료될 때까지 비동기 대기
            await Task.WhenAll(employeesTask, attendanceTask, noticeTask);

            // 각 결과 가져오기
            TotalEmployees = (await employeesTask).Count();
            TodayAttendance = await attendanceTask;
            LatestNotice = await noticeTask;

            _logger.Information(
                "대시보드 초기화 완료: 직원 {Total}명, 출근 {Attendance}명",
                TotalEmployees, TodayAttendance);
        }
        catch (Exception ex)
        {
            _logger.Error(ex, "대시보드 초기화 실패");
        }
        finally
        {
            IsLoading = false;
        }
    }

    private async Task<int> GetTodayAttendanceAsync()
    {
        await Task.Delay(800); // 시뮬레이션
        return 42;
    }

    private async Task<string> GetLatestNoticeAsync()
    {
        await Task.Delay(600); // 시뮬레이션
        return "내일은 전체 회의가 있습니다.";
    }
}
```

### 시나리오 3: DispatcherTimer (주기적 작업)

```csharp
// 파일: ViewModels/ClockViewModel.cs
namespace MyApp.ViewModels;

using System.Windows.Threading;
using CommunityToolkit.Mvvm.ComponentModel;
using Serilog;

/// <summary>
/// DispatcherTimer를 사용한 주기적 UI 갱신
///
/// WinForms의 System.Windows.Forms.Timer에 해당합니다.
/// System.Threading.Timer와 달리 UI 스레드에서 콜백이 호출되므로
/// Dispatcher.Invoke 없이 바로 UI를 갱신할 수 있습니다.
/// </summary>
public partial class ClockViewModel : ObservableObject
{
    private readonly DispatcherTimer _timer;
    private readonly ILogger _logger;

    [ObservableProperty]
    private string _currentTime = string.Empty;

    [ObservableProperty]
    private string _serverStatus = "확인 중...";

    public ClockViewModel(ILogger logger)
    {
        _logger = logger;

        // DispatcherTimer: UI 스레드에서 Tick 이벤트가 발생
        // → Dispatcher.Invoke 없이 바로 프로퍼티 변경 가능
        _timer = new DispatcherTimer
        {
            Interval = TimeSpan.FromSeconds(1)
        };
        _timer.Tick += Timer_Tick;
        _timer.Start();
    }

    /// <summary>
    /// 매초 호출되는 타이머 콜백 (UI 스레드에서 실행)
    /// </summary>
    private void Timer_Tick(object? sender, EventArgs e)
    {
        // DispatcherTimer이므로 UI 스레드에서 실행됨 → 바로 프로퍼티 변경 OK
        CurrentTime = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss");
    }

    /// <summary>ViewModel 해제 시 타이머 중지</summary>
    public void Cleanup()
    {
        _timer.Stop();
        _timer.Tick -= Timer_Tick;
    }
}
```

### 시나리오 4: Debounce 패턴 (검색어 입력 시 자동 검색)

```csharp
// 파일: ViewModels/AutoSearchViewModel.cs
namespace MyApp.ViewModels;

using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using MyApp.Models;
using MyApp.Repositories;
using Serilog;

/// <summary>
/// Debounce(디바운스) 패턴 예제
/// 사용자가 검색어를 입력할 때마다 즉시 검색하지 않고,
/// 마지막 입력 후 300ms 동안 추가 입력이 없으면 검색을 실행합니다.
///
/// Python의 time.sleep() 후 실행하는 것과 비슷하지만,
/// 중간에 새 입력이 들어오면 이전 대기를 취소하는 것이 핵심입니다.
/// </summary>
public partial class AutoSearchViewModel : ObservableObject
{
    private readonly IEmployeeRepository _employeeRepository;
    private readonly ILogger _logger;

    // 디바운스용 CancellationTokenSource
    private CancellationTokenSource? _debounceCts;

    // 디바운스 대기 시간 (밀리초)
    private const int DebounceDelayMs = 300;

    /// <summary>검색어 — TextBox에 TwoWay 바인딩</summary>
    [ObservableProperty]
    private string _searchText = string.Empty;

    /// <summary>로딩 상태</summary>
    [ObservableProperty]
    private bool _isSearching;

    /// <summary>검색 결과</summary>
    public ObservableCollection<Employee> SearchResults { get; } = [];

    public AutoSearchViewModel(
        IEmployeeRepository employeeRepository,
        ILogger logger)
    {
        _employeeRepository = employeeRepository;
        _logger = logger;
    }

    /// <summary>
    /// SearchText 프로퍼티가 변경될 때 자동 호출되는 콜백
    /// [ObservableProperty]가 생성하는 partial 메서드입니다.
    /// </summary>
    partial void OnSearchTextChanged(string value)
    {
        // 검색어가 변경될 때마다 디바운스 검색 실행
        _ = DebounceSearchAsync(value);
    }

    /// <summary>
    /// 디바운스 검색 구현
    /// 마지막 호출 후 300ms 동안 추가 호출이 없으면 실제 검색 실행
    /// </summary>
    private async Task DebounceSearchAsync(string query)
    {
        // 1. 이전 대기를 취소 (새 입력이 들어왔으므로)
        _debounceCts?.Cancel();
        _debounceCts?.Dispose();
        _debounceCts = new CancellationTokenSource();

        var token = _debounceCts.Token;

        try
        {
            // 2. 300ms 대기 (이 동안 새 입력이 들어오면 취소됨)
            await Task.Delay(DebounceDelayMs, token);

            // 3. 300ms 동안 새 입력이 없었으면 실제 검색 실행
            if (string.IsNullOrWhiteSpace(query))
            {
                SearchResults.Clear();
                return;
            }

            IsSearching = true;

            _logger.Information("자동 검색 실행: {Query}", query);

            var results = await _employeeRepository.SearchAsync(query);

            SearchResults.Clear();
            foreach (var result in results)
            {
                SearchResults.Add(result);
            }

            _logger.Information("자동 검색 완료: {Count}건", SearchResults.Count);
        }
        catch (OperationCanceledException)
        {
            // 새 입력으로 인한 취소 — 정상적인 동작이므로 무시
        }
        catch (Exception ex)
        {
            _logger.Error(ex, "자동 검색 실패");
        }
        finally
        {
            IsSearching = false;
        }
    }
}
```

```xml
<!-- XAML: 실시간 검색 UI -->
<!-- UpdateSourceTrigger=PropertyChanged로 매 키 입력마다 SearchText 갱신 -->
<TextBox Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
         Padding="8,6"
         FontSize="14">
    <!-- 워터마크(플레이스홀더) 효과 -->
    <TextBox.Style>
        <Style TargetType="TextBox">
            <Style.Triggers>
                <Trigger Property="Text" Value="">
                    <Setter Property="Background">
                        <Setter.Value>
                            <VisualBrush Stretch="None" AlignmentX="Left">
                                <VisualBrush.Visual>
                                    <TextBlock Text="검색어를 입력하세요..."
                                               Foreground="LightGray"
                                               Margin="5,0,0,0"/>
                                </VisualBrush.Visual>
                            </VisualBrush>
                        </Setter.Value>
                    </Setter>
                </Trigger>
            </Style.Triggers>
        </Style>
    </TextBox.Style>
</TextBox>
```

---

## 비동기 패턴 요약 치트시트

| 상황 | 사용할 패턴 | 핵심 |
|------|-----------|------|
| DB 조회/HTTP 요청 | `await` (Task.Run 불필요) | IO 바운드는 await만 |
| 무거운 계산 | `await Task.Run(() => ...)` | CPU 바운드는 Task.Run |
| MQTT/WebSocket 콜백 | `Dispatcher.Invoke` | 외부 스레드에서 UI 갱신 |
| 주기적 UI 갱신 | `DispatcherTimer` | UI 스레드에서 Tick |
| 로딩 표시 | `[ObservableProperty] bool IsLoading` | 바인딩으로 자동 표시/숨김 |
| 작업 취소 | `CancellationToken` | cts.Cancel()으로 취소 신호 |
| 여러 작업 동시 | `Task.WhenAll(t1, t2, t3)` | 병렬 실행으로 시간 단축 |
| 입력 디바운스 | `Task.Delay` + `CancellationToken` | 마지막 입력 후 대기 |
| Repository/Service 계층 | `ConfigureAwait(false)` | UI 컨텍스트 복귀 불필요 |
| ViewModel 계층 | `ConfigureAwait(true)` (기본) | UI 스레드 복귀 필요 |

> **핵심**: WPF MVVM에서 async/await를 사용하면 "비동기인데 동기 코드처럼 읽히는" 깔끔한 코드를 작성할 수 있습니다. BackgroundWorker의 콜백 지옥에서 벗어나세요!
