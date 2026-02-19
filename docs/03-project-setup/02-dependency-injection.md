# 의존성 주입 (Dependency Injection) 설정

> **목표**: Microsoft.Extensions.DependencyInjection을 사용하여 WPF 앱에 DI 컨테이너를 설정하고, ViewModel과 서비스를 자동으로 연결하는 방법을 익힙니다.

---

## 목차

1. [의존성 주입이란?](#1-의존성-주입이란)
2. [WPF에서 DI 설정하기](#2-wpf에서-di-설정하기)
3. [서비스 등록 방법](#3-서비스-등록-방법)
4. [ViewModel에서 서비스 주입받기](#4-viewmodel에서-서비스-주입받기)
5. [View에서 ViewModel 연결하기](#5-view에서-viewmodel-연결하기)
6. [전체 동작 흐름](#6-전체-동작-흐름)

---

## 1. 의존성 주입이란?

### 1.1 WinForms에서는 왜 안 썼는가?

WinForms 개발에서는 보통 이런 식으로 코드를 작성합니다:

```csharp
// WinForms에서 흔히 보는 패턴
// Form 안에서 필요한 객체를 직접 생성 (new 키워드)
public partial class EmployeeForm : Form
{
    // Form이 직접 DB 연결을 생성하고 관리
    private readonly NpgsqlConnection _connection;

    public EmployeeForm()
    {
        InitializeComponent();

        // 여기서 직접 new로 객체 생성!
        // 이것이 "강한 결합(Tight Coupling)"
        _connection = new NpgsqlConnection("Host=localhost;Database=mydb;...");
    }

    private void btnLoad_Click(object sender, EventArgs e)
    {
        // Form이 DB에 직접 접근
        var employees = _connection.Query<Employee>("SELECT * FROM employees");
        dataGridView1.DataSource = employees.ToList();
    }
}
```

이 방식의 **문제점**:
- `EmployeeForm`이 `NpgsqlConnection`에 **강하게 결합**되어 있음
- 연결 문자열이 코드에 하드코딩됨
- 단위 테스트 시 실제 DB가 필요함 (테스트가 느리고 불안정)
- DB를 PostgreSQL에서 MySQL로 바꾸려면 모든 Form을 수정해야 함

### 1.2 비유로 이해하기

```
[의존성 주입 = 레스토랑의 식재료 공급 시스템]

❌ DI 없이 (WinForms 스타일):
   셰프(ViewModel)가 직접 시장에 가서 재료를 구매한다.
   → 셰프가 시장 위치, 가격, 재료 품질을 모두 알아야 함
   → 시장이 바뀌면 셰프의 루틴 전체를 바꿔야 함

✅ DI 사용 (MVVM 스타일):
   식재료 공급업체(DI 컨테이너)가 셰프에게 재료를 배달해 준다.
   → 셰프는 "토마토가 필요해요"라고만 말하면 됨
   → 공급업체가 바뀌어도 셰프는 동일하게 요리만 하면 됨
```

### 1.3 의존성 주입의 핵심 이점

| 이점 | 설명 | 예시 |
|------|------|------|
| **느슨한 결합** | 클래스가 구현체가 아닌 인터페이스에 의존 | `IEmployeeService` vs `EmployeeService` |
| **테스트 용이** | 가짜(Mock) 객체로 대체하여 단위 테스트 가능 | DB 없이 ViewModel 테스트 |
| **유지보수** | 구현체 교체 시 한 곳(DI 등록부)만 수정 | PostgreSQL → MySQL 전환 |
| **수명 관리** | 객체의 생성과 소멸을 컨테이너가 자동 관리 | Singleton, Transient 등 |

### 1.4 Python 개발자를 위한 비유

```python
# Python에서의 "의존성 주입" 개념 (수동)
class EmployeeViewModel:
    def __init__(self, employee_service):  # 외부에서 주입받음
        self.service = employee_service

# 사용 시
service = EmployeeService(connection_string)
vm = EmployeeViewModel(service)  # 주입!
```

```csharp
// C#에서의 의존성 주입 (DI 컨테이너가 자동으로 해줌)
public class EmployeeViewModel
{
    private readonly IEmployeeService _service;

    // 생성자에 인터페이스를 넣으면, DI 컨테이너가 자동으로 구현체를 넣어줌!
    public EmployeeViewModel(IEmployeeService service)
    {
        _service = service;
    }
}
```

Python에서는 개발자가 직접 객체를 생성해서 전달하지만, C#의 DI 컨테이너는 이 과정을 **자동으로** 처리합니다.

---

## 2. WPF에서 DI 설정하기

### 2.1 App.xaml 수정 (StartupUri 제거)

먼저, 기본 생성된 `App.xaml`에서 `StartupUri`를 제거합니다. DI 컨테이너가 MainWindow를 직접 생성할 것이므로, WPF가 자동으로 윈도우를 띄우지 않도록 합니다.

```xml
<!-- App.xaml -->
<!-- 수정 전 (기본 생성된 상태) -->
<Application x:Class="MyApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             StartupUri="MainWindow.xaml">
    <!--                  ↑ 이것을 제거해야 함! -->
    <Application.Resources />
</Application>
```

```xml
<!-- App.xaml -->
<!-- 수정 후 (StartupUri 제거) -->
<Application x:Class="MyApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <!-- StartupUri 제거됨 → OnStartup에서 수동으로 윈도우를 띄움 -->
    <Application.Resources />
</Application>
```

> **왜 StartupUri를 제거하는가?**
> `StartupUri="MainWindow.xaml"`이 있으면 WPF가 `MainWindow`를 직접 `new MainWindow()`로 생성합니다. 이때 DI 컨테이너를 거치지 않으므로, MainWindow의 생성자에 주입할 수 없습니다. 우리는 DI 컨테이너가 모든 객체를 생성하기를 원하므로 `StartupUri`를 제거하고, `OnStartup` 메서드에서 직접 윈도우를 가져옵니다.

### 2.2 전체 App.xaml.cs 코드

이것이 WPF + MVVM 프로젝트의 **핵심 진입점**입니다. 모든 서비스, ViewModel, View가 여기서 등록되고 연결됩니다.

```csharp
// App.xaml.cs
// 앱의 시작점 - DI 컨테이너를 구성하고, 앱을 초기화합니다.

using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using MyApp.Services.Interfaces;
using MyApp.Services.Implementations;
using MyApp.ViewModels;
using MyApp.Views;
using Serilog;
using System.IO;
using System.Windows;

namespace MyApp;

/// <summary>
/// 앱의 진입점.
/// DI 컨테이너 구성, 설정 로드, 로깅 설정을 담당합니다.
/// </summary>
public partial class App : Application
{
    /// <summary>
    /// DI 컨테이너. 앱 전체에서 서비스를 가져올 수 있는 "서비스 제공자".
    /// Python의 의존성 컨테이너와 비슷한 역할.
    /// </summary>
    private IServiceProvider _serviceProvider = null!;

    /// <summary>
    /// 설정 객체. appsettings.json의 내용을 담고 있음.
    /// </summary>
    private IConfiguration _configuration = null!;

    /// <summary>
    /// 앱이 시작될 때 호출되는 메서드.
    /// WinForms의 Program.Main()에서 하던 초기화 작업을 여기서 수행합니다.
    /// </summary>
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // 1단계: 설정 파일 로드 (appsettings.json)
        _configuration = BuildConfiguration();

        // 2단계: Serilog 로깅 초기화
        ConfigureSerilog();

        // 3단계: DI 컨테이너 구성
        _serviceProvider = ConfigureServices();

        // 4단계: 메인 윈도우를 DI 컨테이너에서 가져와서 표시
        var mainWindow = _serviceProvider.GetRequiredService<MainWindow>();
        mainWindow.Show();

        Log.Information("앱이 시작되었습니다.");
    }

    /// <summary>
    /// 앱이 종료될 때 호출되는 메서드.
    /// 리소스 정리 (Serilog 버퍼 플러시, DB 연결 닫기 등)
    /// </summary>
    protected override void OnExit(ExitEventArgs e)
    {
        Log.Information("앱이 종료됩니다.");

        // Serilog의 남은 로그 버퍼를 파일에 기록
        Log.CloseAndFlush();

        // DI 컨테이너가 IDisposable인 경우 정리
        if (_serviceProvider is IDisposable disposable)
        {
            disposable.Dispose();
        }

        base.OnExit(e);
    }

    /// <summary>
    /// appsettings.json에서 설정을 읽어오는 메서드.
    /// 환경별 설정 파일도 함께 로드합니다 (있는 경우).
    /// </summary>
    private static IConfiguration BuildConfiguration()
    {
        return new ConfigurationBuilder()
            // 앱 실행 파일이 있는 디렉토리를 기준으로 설정 파일을 찾음
            .SetBasePath(Directory.GetCurrentDirectory())
            // 기본 설정 파일 (필수, 없으면 에러)
            .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
            // 개발 환경 전용 설정 파일 (선택, 없어도 에러 안 남)
            .AddJsonFile("appsettings.Development.json", optional: true, reloadOnChange: true)
            .Build();
    }

    /// <summary>
    /// Serilog 로깅을 설정하는 메서드.
    /// 콘솔과 파일에 동시에 로그를 기록합니다.
    /// </summary>
    private void ConfigureSerilog()
    {
        Log.Logger = new LoggerConfiguration()
            // appsettings.json의 Serilog 섹션에서 최소 로그 레벨 등을 읽을 수도 있음
            .MinimumLevel.Debug()
            // 콘솔에 로그 출력 (개발 시 디버깅용)
            .WriteTo.Console()
            // 파일에 로그 출력 (운영 환경용)
            // 날짜별로 자동 분리 (예: log-20260219.txt)
            .WriteTo.File(
                path: "logs/log-.txt",
                rollingInterval: RollingInterval.Day,
                retainedFileCountLimit: 30,    // 최근 30일치만 보관
                outputTemplate: "[{Timestamp:yyyy-MM-dd HH:mm:ss.fff} {Level:u3}] {Message:lj}{NewLine}{Exception}")
            .CreateLogger();
    }

    /// <summary>
    /// DI 컨테이너에 모든 서비스를 등록하는 메서드.
    /// 이곳이 앱의 "설계도" 역할을 합니다.
    /// </summary>
    private IServiceProvider ConfigureServices()
    {
        // ServiceCollection = 서비스를 등록하는 "목록"
        var services = new ServiceCollection();

        // ─────────────────────────────────────
        // 설정 (Configuration) 등록
        // ─────────────────────────────────────
        // IConfiguration을 DI에 등록하여 어디서든 설정값에 접근 가능
        services.AddSingleton(_configuration);

        // ─────────────────────────────────────
        // 로깅 (Serilog) 등록
        // ─────────────────────────────────────
        // Serilog.ILogger를 DI에 등록
        services.AddSingleton(Log.Logger);

        // ─────────────────────────────────────
        // 서비스 등록 (비즈니스 로직)
        // ─────────────────────────────────────
        // 인터페이스 → 구현체 매핑
        // 아래에서 Singleton/Scoped/Transient 차이를 자세히 설명합니다.
        services.AddSingleton<IDatabaseService, DatabaseService>();
        services.AddTransient<IEmployeeService, EmployeeService>();
        services.AddSingleton<IMqttService, MqttService>();
        services.AddSingleton<IAwsS3Service, AwsS3Service>();

        // ─────────────────────────────────────
        // ViewModel 등록
        // ─────────────────────────────────────
        // ViewModel은 보통 Transient로 등록
        // (화면이 열릴 때마다 새 인스턴스 생성)
        services.AddTransient<MainViewModel>();
        services.AddTransient<EmployeeListViewModel>();
        services.AddTransient<EmployeeDetailViewModel>();

        // ─────────────────────────────────────
        // View(윈도우/유저컨트롤) 등록
        // ─────────────────────────────────────
        // MainWindow는 앱에서 하나만 필요하므로 Singleton
        services.AddSingleton<MainWindow>();

        // ServiceCollection을 ServiceProvider로 빌드
        // 이 시점부터 등록된 서비스를 꺼내 쓸 수 있음
        return services.BuildServiceProvider();
    }
}
```

### 2.3 단계별 동작 설명

```
앱 시작 시 실행 순서:

1. App.OnStartup() 호출
       │
       ▼
2. BuildConfiguration()
   → appsettings.json 로드
   → IConfiguration 객체 생성
       │
       ▼
3. ConfigureSerilog()
   → 콘솔 + 파일 로깅 설정
   → Log.Logger 전역 로거 생성
       │
       ▼
4. ConfigureServices()
   → ServiceCollection 생성 (빈 목록)
   → 설정, 로거, 서비스, ViewModel, View 등록
   → BuildServiceProvider() → IServiceProvider 생성
       │
       ▼
5. GetRequiredService<MainWindow>()
   → DI 컨테이너가 MainWindow를 생성
   → MainWindow의 생성자에 필요한 MainViewModel을 자동 주입
   → MainViewModel의 생성자에 필요한 서비스들을 자동 주입
       │
       ▼
6. mainWindow.Show()
   → 화면 표시!
```

---

## 3. 서비스 등록 방법

### 3.1 Singleton vs Scoped vs Transient

DI 컨테이너에 서비스를 등록할 때, 객체의 **수명(Lifetime)** 을 지정해야 합니다.

| 수명 | 의미 | 비유 | 예시 |
|------|------|------|------|
| **Singleton** | 앱 전체에서 **단 하나**의 인스턴스만 존재 | 회사에 **하나뿐인** 사장실 열쇠 | DB 연결 풀, MQTT 클라이언트, 로거 |
| **Scoped** | 특정 범위(Scope) 안에서 하나의 인스턴스 | 식당에서 **한 테이블**에 하나인 메뉴판 | ASP.NET에서 주로 사용 (WPF에서는 드물게 사용) |
| **Transient** | 요청할 때마다 **새로운** 인스턴스 생성 | 카페에서 주문할 때마다 **새 컵** | ViewModel, 일회용 서비스 |

```csharp
// Singleton: 앱 전체에서 딱 하나
// DB 커넥션 풀은 하나만 있어야 함
services.AddSingleton<IDatabaseService, DatabaseService>();

// Transient: 매번 새로 생성
// ViewModel은 화면마다 새로 만들어야 함
services.AddTransient<EmployeeListViewModel>();

// Scoped: WPF에서는 잘 안 쓰지만, 알아두면 좋음
// (ASP.NET에서 HTTP 요청당 하나의 인스턴스)
// services.AddScoped<SomeService>();
```

### 3.2 실제 사용 예시: 어떤 수명을 선택할까?

```csharp
/// <summary>
/// DI 서비스 등록 예시 - 실제 프로젝트에서 사용하는 패턴
/// </summary>
private IServiceProvider ConfigureServices()
{
    var services = new ServiceCollection();

    // ─── Singleton (앱 전체에서 하나) ───

    // 설정 - 앱 실행 중 하나만 있으면 됨
    services.AddSingleton(_configuration);

    // 로거 - 하나의 로거가 모든 로그를 관리
    services.AddSingleton(Log.Logger);

    // DB 서비스 - 커넥션 풀을 공유해야 하므로 Singleton
    services.AddSingleton<IDatabaseService, DatabaseService>();

    // MQTT 클라이언트 - 하나의 연결을 유지해야 함
    services.AddSingleton<IMqttService, MqttService>();

    // AWS S3 클라이언트 - 인증 정보를 공유하므로 Singleton
    services.AddSingleton<IAwsS3Service, AwsS3Service>();

    // ─── Transient (매번 새로 생성) ───

    // 직원 서비스 - 상태를 가지지 않으므로 Transient도 OK
    services.AddTransient<IEmployeeService, EmployeeService>();

    // ViewModel - 화면이 열릴 때마다 새로운 상태로 시작
    services.AddTransient<MainViewModel>();
    services.AddTransient<EmployeeListViewModel>();
    services.AddTransient<EmployeeDetailViewModel>();

    // ─── View 등록 ───

    // MainWindow - 앱에서 하나만 존재
    services.AddSingleton<MainWindow>();

    return services.BuildServiceProvider();
}
```

### 3.3 인터페이스 기반 등록의 중요성

```csharp
// ❌ 나쁜 예: 구현체를 직접 등록
// 나중에 EmployeeService를 MockEmployeeService로 바꾸려면
// 등록 코드와 사용하는 코드를 모두 수정해야 함
services.AddTransient<EmployeeService>();

// ✅ 좋은 예: 인터페이스 → 구현체로 등록
// MockEmployeeService로 바꾸고 싶으면 이 줄만 수정!
services.AddTransient<IEmployeeService, EmployeeService>();

// 테스트할 때는 이렇게 바꾸면 됨:
// services.AddTransient<IEmployeeService, MockEmployeeService>();
```

### 3.4 인터페이스 정의 예시

```csharp
// Services/Interfaces/IEmployeeService.cs
// 서비스가 "무엇을 할 수 있는지"만 정의 (HOW는 구현체에서)

namespace MyApp.Services.Interfaces;

/// <summary>
/// 직원 데이터를 관리하는 서비스의 계약(인터페이스).
/// 실제 DB 접근 방식은 구현체에서 결정됨.
/// </summary>
public interface IEmployeeService
{
    /// <summary>
    /// 모든 직원 목록을 가져옴
    /// </summary>
    Task<IEnumerable<Employee>> GetAllAsync();

    /// <summary>
    /// ID로 특정 직원을 가져옴
    /// </summary>
    Task<Employee?> GetByIdAsync(int id);

    /// <summary>
    /// 새 직원을 추가
    /// </summary>
    Task<int> AddAsync(Employee employee);

    /// <summary>
    /// 기존 직원 정보를 수정
    /// </summary>
    Task UpdateAsync(Employee employee);

    /// <summary>
    /// 직원을 삭제
    /// </summary>
    Task DeleteAsync(int id);
}
```

```csharp
// Services/Implementations/EmployeeService.cs
// 인터페이스의 실제 구현 (PostgreSQL + Dapper 사용)

using Dapper;
using MyApp.Models;
using MyApp.Services.Interfaces;
using Npgsql;
using Serilog;

namespace MyApp.Services.Implementations;

/// <summary>
/// IEmployeeService의 실제 구현체.
/// PostgreSQL + Dapper를 사용하여 직원 데이터를 관리합니다.
/// </summary>
public class EmployeeService : IEmployeeService
{
    // DI 컨테이너가 자동으로 주입해주는 서비스들
    private readonly IDatabaseService _db;
    private readonly ILogger _logger;

    /// <summary>
    /// 생성자 주입: DI 컨테이너가 자동으로 필요한 서비스를 넣어줌
    /// </summary>
    /// <param name="db">DB 연결을 관리하는 서비스</param>
    /// <param name="logger">Serilog 로거</param>
    public EmployeeService(IDatabaseService db, ILogger logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task<IEnumerable<Employee>> GetAllAsync()
    {
        _logger.Information("전체 직원 목록을 조회합니다.");

        using var connection = _db.CreateConnection();
        return await connection.QueryAsync<Employee>(
            "SELECT id, name, email, department FROM employees ORDER BY name");
    }

    public async Task<Employee?> GetByIdAsync(int id)
    {
        _logger.Information("직원 조회: ID={EmployeeId}", id);

        using var connection = _db.CreateConnection();
        return await connection.QuerySingleOrDefaultAsync<Employee>(
            "SELECT id, name, email, department FROM employees WHERE id = @Id",
            new { Id = id });
    }

    public async Task<int> AddAsync(Employee employee)
    {
        _logger.Information("새 직원 추가: {Name}", employee.Name);

        using var connection = _db.CreateConnection();
        return await connection.ExecuteScalarAsync<int>(
            @"INSERT INTO employees (name, email, department)
              VALUES (@Name, @Email, @Department)
              RETURNING id",
            employee);
    }

    public async Task UpdateAsync(Employee employee)
    {
        _logger.Information("직원 정보 수정: ID={EmployeeId}", employee.Id);

        using var connection = _db.CreateConnection();
        await connection.ExecuteAsync(
            @"UPDATE employees
              SET name = @Name, email = @Email, department = @Department
              WHERE id = @Id",
            employee);
    }

    public async Task DeleteAsync(int id)
    {
        _logger.Warning("직원 삭제: ID={EmployeeId}", id);

        using var connection = _db.CreateConnection();
        await connection.ExecuteAsync(
            "DELETE FROM employees WHERE id = @Id",
            new { Id = id });
    }
}
```

---

## 4. ViewModel에서 서비스 주입받기

### 4.1 생성자 주입 (Constructor Injection)

DI에서 가장 흔하고 권장되는 주입 방식입니다. 필요한 서비스를 생성자의 매개변수로 선언하면, DI 컨테이너가 자동으로 넣어줍니다.

```csharp
// ViewModels/EmployeeListViewModel.cs
// 직원 목록 화면의 ViewModel

using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using MyApp.Models;
using MyApp.Services.Interfaces;
using Serilog;
using System.Collections.ObjectModel;

namespace MyApp.ViewModels;

/// <summary>
/// 직원 목록 화면의 ViewModel.
/// 이 클래스는 DI 컨테이너에 의해 자동 생성되며,
/// 생성자의 매개변수로 필요한 서비스를 자동 주입받습니다.
/// </summary>
public partial class EmployeeListViewModel : ObservableObject
{
    // ────────────────────────────────
    // DI로 주입받는 서비스들
    // ────────────────────────────────

    // readonly: 한 번 설정하면 변경 불가 (안전성 확보)
    private readonly IEmployeeService _employeeService;
    private readonly ILogger _logger;

    /// <summary>
    /// 생성자 주입 (Constructor Injection)
    ///
    /// 이 생성자를 직접 호출하는 코드는 없습니다!
    /// DI 컨테이너가 자동으로:
    ///   1. IEmployeeService에 등록된 EmployeeService를 찾고
    ///   2. ILogger에 등록된 Serilog Logger를 찾고
    ///   3. 이 생성자에 넣어줍니다.
    /// </summary>
    public EmployeeListViewModel(
        IEmployeeService employeeService,
        ILogger logger)
    {
        _employeeService = employeeService;
        _logger = logger;
    }

    // ────────────────────────────────
    // 바인딩 속성 (View와 연결)
    // ────────────────────────────────

    /// <summary>
    /// 직원 목록 (DataGrid에 바인딩됨)
    /// ObservableCollection: 항목이 추가/삭제되면 UI가 자동 업데이트됨
    /// </summary>
    public ObservableCollection<Employee> Employees { get; } = [];

    /// <summary>
    /// 현재 선택된 직원 (ListView/DataGrid의 SelectedItem에 바인딩됨)
    /// [ObservableProperty]: CommunityToolkit이 자동으로
    ///   - public SelectedEmployee 속성 생성
    ///   - PropertyChanged 이벤트 발생 코드 생성
    /// </summary>
    [ObservableProperty]
    private Employee? _selectedEmployee;

    /// <summary>
    /// 로딩 중 여부 (ProgressBar 표시에 사용)
    /// </summary>
    [ObservableProperty]
    private bool _isLoading;

    // ────────────────────────────────
    // 커맨드 (View의 버튼과 연결)
    // ────────────────────────────────

    /// <summary>
    /// 직원 목록을 DB에서 다시 불러오는 커맨드.
    /// [RelayCommand]: CommunityToolkit이 자동으로
    ///   - public LoadEmployeesCommand 속성 생성
    ///   - async/await 처리
    ///   - 에러 핸들링 지원
    /// </summary>
    [RelayCommand]
    private async Task LoadEmployeesAsync()
    {
        try
        {
            IsLoading = true;
            _logger.Information("직원 목록을 불러옵니다.");

            // 서비스를 통해 DB에서 데이터 가져오기
            var employees = await _employeeService.GetAllAsync();

            // ObservableCollection 업데이트
            Employees.Clear();
            foreach (var emp in employees)
            {
                Employees.Add(emp);
            }

            _logger.Information("직원 {Count}명을 불러왔습니다.", Employees.Count);
        }
        catch (Exception ex)
        {
            _logger.Error(ex, "직원 목록 조회 중 오류가 발생했습니다.");
            // 사용자에게 에러 메시지를 보여주는 로직 (나중에 추가)
        }
        finally
        {
            IsLoading = false;
        }
    }

    /// <summary>
    /// 선택한 직원을 삭제하는 커맨드.
    /// CanExecute: SelectedEmployee가 null이 아닐 때만 버튼 활성화
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanDeleteEmployee))]
    private async Task DeleteEmployeeAsync()
    {
        if (SelectedEmployee is null) return;

        _logger.Information("직원 삭제 요청: {Name}", SelectedEmployee.Name);

        await _employeeService.DeleteAsync(SelectedEmployee.Id);

        // 목록 새로고침
        await LoadEmployeesAsync();
    }

    /// <summary>
    /// 삭제 버튼의 활성화 조건: 직원이 선택되어 있을 때만 삭제 가능
    /// </summary>
    private bool CanDeleteEmployee() => SelectedEmployee is not null;

    /// <summary>
    /// SelectedEmployee가 변경될 때 자동 호출됨.
    /// CommunityToolkit이 자동으로 생성하는 partial 메서드.
    /// 삭제 버튼의 활성화 상태를 업데이트합니다.
    /// </summary>
    partial void OnSelectedEmployeeChanged(Employee? value)
    {
        // SelectedEmployee가 바뀌면 삭제 버튼의 CanExecute를 다시 평가
        DeleteEmployeeCommand.NotifyCanExecuteChanged();
    }
}
```

### 4.2 주입 흐름 시각화

```
DI 컨테이너가 EmployeeListViewModel을 생성하는 과정:

1. 누군가 EmployeeListViewModel을 요청
   (예: MainWindow에서 DataContext로 사용)
       │
       ▼
2. DI 컨테이너가 생성자를 분석
   "IEmployeeService와 ILogger가 필요하구나!"
       │
       ├──► IEmployeeService가 필요!
       │    → 등록된 구현체: EmployeeService (Transient)
       │    → EmployeeService의 생성자를 분석
       │       → IDatabaseService와 ILogger가 필요!
       │       → IDatabaseService → DatabaseService (Singleton, 이미 있으면 재사용)
       │       → ILogger → Log.Logger (Singleton, 이미 있으면 재사용)
       │    → EmployeeService 생성 완료!
       │
       ├──► ILogger가 필요!
       │    → 등록된 인스턴스: Log.Logger (Singleton)
       │    → 바로 반환!
       │
       ▼
3. EmployeeListViewModel(employeeService, logger) 생성 완료!
```

> **핵심 포인트**: 개발자는 `new EmployeeListViewModel(...)`을 직접 호출하지 않습니다. DI 컨테이너가 생성자를 보고 필요한 의존성을 **알아서** 찾아서 넣어줍니다. 이를 **자동 와이어링(Auto-Wiring)** 이라고 합니다.

---

## 5. View에서 ViewModel 연결하기

View(XAML)와 ViewModel을 연결하는 방법은 여러 가지가 있습니다. 여기서는 가장 실용적인 두 가지 방법을 소개합니다.

### 5.1 방법 1: 코드 비하인드에서 DataContext 설정 (권장, 간단)

이 방법은 View의 코드 비하인드(`.xaml.cs`)에서 생성자 주입을 통해 ViewModel을 받아 DataContext에 설정하는 방식입니다.

```csharp
// Views/MainWindow.xaml.cs
// 코드 비하인드에서 ViewModel을 주입받아 연결

using MyApp.ViewModels;
using System.Windows;

namespace MyApp.Views;

/// <summary>
/// MainWindow의 코드 비하인드.
/// DI 컨테이너가 MainViewModel을 자동으로 주입해줍니다.
/// </summary>
public partial class MainWindow : Window
{
    /// <summary>
    /// 생성자 주입으로 ViewModel을 받음.
    /// DI 컨테이너가 MainWindow를 생성할 때 자동으로 MainViewModel도 함께 주입.
    /// </summary>
    public MainWindow(MainViewModel viewModel)
    {
        InitializeComponent();

        // DataContext = ViewModel
        // 이 한 줄로 XAML의 모든 {Binding ...}이 ViewModel과 연결됨!
        DataContext = viewModel;
    }
}
```

```xml
<!-- Views/MainWindow.xaml -->
<!-- ViewModel의 속성에 바인딩 -->
<Window x:Class="MyApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="MyApp" Height="600" Width="800">

    <Grid Margin="20">
        <!-- ViewModel의 속성에 자동으로 바인딩됨 -->
        <TextBlock Text="{Binding Title}" FontSize="24" />
    </Grid>
</Window>
```

**장점:**
- 가장 간단하고 직관적
- DI의 생성자 주입을 그대로 활용
- 코드가 명확하고 디버깅이 쉬움

**단점:**
- XAML 디자이너에서 바인딩 미리보기가 안 됨 (실행해봐야 확인 가능)

### 5.2 방법 2: ViewModelLocator 패턴 (대규모 프로젝트에 적합)

ViewModelLocator는 모든 ViewModel을 한 곳에서 관리하는 중앙 집중식 패턴입니다. XAML에서 디자인 타임 바인딩도 지원됩니다.

```csharp
// ViewModels/ViewModelLocator.cs
// 모든 ViewModel을 한 곳에서 제공하는 서비스 로케이터

using Microsoft.Extensions.DependencyInjection;

namespace MyApp.ViewModels;

/// <summary>
/// ViewModel 로케이터.
/// XAML에서 ViewModel을 쉽게 참조할 수 있게 해주는 클래스.
///
/// 사용법:
///   - App.xaml에 리소스로 등록
///   - 각 View의 DataContext에서 참조
/// </summary>
public class ViewModelLocator
{
    /// <summary>
    /// DI 컨테이너 (IServiceProvider)
    /// App.xaml.cs에서 설정됨
    /// </summary>
    public static IServiceProvider? ServiceProvider { get; set; }

    // ─── 각 ViewModel을 속성으로 노출 ───

    /// <summary>
    /// MainWindow의 ViewModel.
    /// DI 컨테이너에서 가져옴 (자동으로 의존성 주입됨).
    /// </summary>
    public MainViewModel MainViewModel
        => ServiceProvider!.GetRequiredService<MainViewModel>();

    /// <summary>
    /// 직원 목록 화면의 ViewModel
    /// </summary>
    public EmployeeListViewModel EmployeeListViewModel
        => ServiceProvider!.GetRequiredService<EmployeeListViewModel>();

    /// <summary>
    /// 직원 상세 화면의 ViewModel
    /// </summary>
    public EmployeeDetailViewModel EmployeeDetailViewModel
        => ServiceProvider!.GetRequiredService<EmployeeDetailViewModel>();
}
```

**App.xaml에 ViewModelLocator 등록:**

```xml
<!-- App.xaml -->
<Application x:Class="MyApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="clr-namespace:MyApp.ViewModels">

    <Application.Resources>
        <!-- ViewModelLocator를 앱 전역 리소스로 등록 -->
        <!-- Key="Locator"로 XAML 어디서든 접근 가능 -->
        <vm:ViewModelLocator x:Key="Locator" />
    </Application.Resources>
</Application>
```

**App.xaml.cs에서 ServiceProvider 설정:**

```csharp
// App.xaml.cs (OnStartup 메서드 내)
protected override void OnStartup(StartupEventArgs e)
{
    base.OnStartup(e);

    _configuration = BuildConfiguration();
    ConfigureSerilog();
    _serviceProvider = ConfigureServices();

    // ViewModelLocator에 ServiceProvider 설정
    ViewModelLocator.ServiceProvider = _serviceProvider;

    var mainWindow = _serviceProvider.GetRequiredService<MainWindow>();
    mainWindow.Show();
}
```

**View에서 ViewModelLocator 사용:**

```xml
<!-- Views/MainWindow.xaml -->
<!-- ViewModelLocator를 통해 DataContext 설정 -->
<Window x:Class="MyApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        DataContext="{Binding MainViewModel, Source={StaticResource Locator}}"
        Title="MyApp" Height="600" Width="800">
    <!--     ↑ Locator에서 MainViewModel을 가져와 DataContext에 설정 -->

    <Grid Margin="20">
        <TextBlock Text="{Binding Title}" FontSize="24" />
    </Grid>
</Window>
```

### 5.3 어떤 방법을 선택할까?

| 기준 | 코드 비하인드 (방법 1) | ViewModelLocator (방법 2) |
|------|----------------------|--------------------------|
| 복잡도 | 간단 | 보통 |
| DI 활용 | 생성자 주입 (표준) | ServiceLocator (안티패턴 논란) |
| 디자인 타임 | 지원 안 됨 | 제한적 지원 |
| 대규모 프로젝트 | 충분 | 유용 |
| **추천** | **입문자에게 추천** | 필요 시 도입 |

> **이 튜토리얼의 권장**: 처음에는 **방법 1 (코드 비하인드)** 을 사용하세요. 프로젝트가 커지고 필요성을 느낄 때 방법 2를 도입해도 늦지 않습니다.

---

## 6. 전체 동작 흐름

### 6.1 앱 시작부터 화면 표시까지

```
┌──────────────────────────────────────────────────────────────────┐
│                        앱 시작 흐름                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. App.OnStartup() 호출                                         │
│     │                                                            │
│     ├─► appsettings.json 로드 → IConfiguration 생성               │
│     │                                                            │
│     ├─► Serilog 초기화 (콘솔 + 파일)                               │
│     │                                                            │
│     ├─► ServiceCollection 생성                                    │
│     │     │                                                      │
│     │     ├─► IConfiguration 등록 (Singleton)                     │
│     │     ├─► ILogger 등록 (Singleton)                            │
│     │     ├─► IDatabaseService → DatabaseService (Singleton)      │
│     │     ├─► IEmployeeService → EmployeeService (Transient)      │
│     │     ├─► IMqttService → MqttService (Singleton)              │
│     │     ├─► MainViewModel (Transient)                           │
│     │     ├─► EmployeeListViewModel (Transient)                   │
│     │     └─► MainWindow (Singleton)                              │
│     │                                                            │
│     ├─► ServiceProvider 빌드                                      │
│     │                                                            │
│     └─► MainWindow 요청                                          │
│           │                                                      │
│           └─► DI 체인 자동 해결:                                   │
│                                                                  │
│               MainWindow(MainViewModel viewModel)                │
│                    │                                             │
│                    └─► MainViewModel(                             │
│                           IEmployeeService service,              │
│                           IMqttService mqtt,                     │
│                           ILogger logger)                        │
│                              │                                   │
│                              ├─► EmployeeService(                │
│                              │       IDatabaseService db,        │
│                              │       ILogger logger)             │
│                              │          │                        │
│                              │          └─► DatabaseService(     │
│                              │                 IConfiguration)   │
│                              │                                   │
│                              ├─► MqttService(                    │
│                              │       IConfiguration, ILogger)    │
│                              │                                   │
│                              └─► Log.Logger (이미 생성됨)         │
│                                                                  │
│  2. MainWindow.DataContext = MainViewModel                        │
│                                                                  │
│  3. MainWindow.Show() → 화면 표시!                                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 6.2 사용자 상호작용 흐름 (예: "직원 목록 불러오기" 버튼 클릭)

```
사용자가 "불러오기" 버튼 클릭
         │
         ▼
[View] Button의 Command="{Binding LoadEmployeesCommand}"
         │  WPF가 바인딩된 커맨드를 실행
         ▼
[ViewModel] LoadEmployeesAsync() 메서드 실행
         │  _employeeService.GetAllAsync() 호출
         ▼
[Service] EmployeeService.GetAllAsync()
         │  Dapper로 SQL 실행, 결과를 Employee 객체 리스트로 변환
         ▼
[DB] PostgreSQL에서 데이터 조회
         │
         ▼ (결과 반환)
[ViewModel] Employees 컬렉션 업데이트 (ObservableCollection)
         │  컬렉션이 변경되면 PropertyChanged 이벤트 자동 발생
         ▼
[View] DataGrid에 자동으로 목록이 표시됨!
```

### 6.3 WinForms와의 흐름 비교

```
WinForms:
  버튼 클릭 → btnLoad_Click 이벤트 핸들러
            → 핸들러 안에서 DB 직접 접근
            → dataGridView1.DataSource = 결과

WPF + MVVM + DI:
  버튼 클릭 → Command → ViewModel 메서드
            → Service 호출 (DI로 주입된)
            → Service가 DB 접근
            → ViewModel이 컬렉션 업데이트
            → View에 자동 반영 (데이터 바인딩)
```

WPF + MVVM 방식이 단계가 더 많아 보이지만, 각 단계가 **독립적**이므로:
- 어디서 버그가 났는지 찾기 쉬움
- DB를 바꿔도 ViewModel은 수정할 필요 없음
- ViewModel을 UI 없이 테스트 가능

---

## 다음 단계

DI 컨테이너가 설정되었으면, 다음은 `appsettings.json`을 이용한 **설정 관리** 방법을 알아봅니다. DB 연결 문자열, MQTT 설정, AWS 인증 정보 등을 체계적으로 관리하는 방법입니다.

> [다음 문서: 설정 관리 (Configuration) →](./03-configuration.md)
