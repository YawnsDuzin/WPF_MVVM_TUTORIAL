# Serilog 구조적 로깅

> **목표**: WPF MVVM 프로젝트에서 Serilog를 사용하여 구조적 로깅(Structured Logging)을 설정하고, ViewModel과 서비스에서 활용하는 방법을 익힙니다.
> **대상 독자**: `Debug.WriteLine()`이나 파일에 직접 로그를 쓰던 WinForms 개발자

---

## 목차

1. [WinForms 로깅 vs 구조적 로깅](#1-winforms-로깅-vs-구조적-로깅)
2. [Serilog란?](#2-serilog란)
3. [NuGet 패키지 설치](#3-nuget-패키지-설치)
4. [Serilog 설정](#4-serilog-설정)
5. [DI에 등록하기](#5-di에-등록하기)
6. [로그 레벨 사용 가이드](#6-로그-레벨-사용-가이드)
7. [ViewModel에서 로깅하기](#7-viewmodel에서-로깅하기)
8. [Service에서 로깅하기](#8-service에서-로깅하기)
9. [구조적 로깅의 장점](#9-구조적-로깅의-장점)
10. [전체 코드 예제](#10-전체-코드-예제)

---

## 1. WinForms 로깅 vs 구조적 로깅

### WinForms에서 흔히 사용하던 방식

```csharp
// ❌ 방법 1: Debug.WriteLine — 디버그 모드에서만 작동, 배포하면 볼 수 없음
Debug.WriteLine("직원 저장 시작: " + employee.Name);
Debug.WriteLine($"오류 발생: {ex.Message}");

// ❌ 방법 2: 파일에 직접 쓰기 — 동시 접근 문제, 파일 관리 어려움
File.AppendAllText("app.log",
    $"{DateTime.Now}: 직원 {employee.Name} 저장 완료\n");

// ❌ 방법 3: Console.WriteLine — WPF에는 콘솔이 없음
Console.WriteLine("데이터 로드 중...");

// ❌ 방법 4: MessageBox — 사용자를 방해함, 자동화 불가
MessageBox.Show($"오류: {ex.Message}");
```

**이런 방식의 문제점:**

| 문제 | 설명 |
|------|------|
| 구조 없음 | 로그가 단순 문자열이라 검색/필터링이 어려움 |
| 레벨 구분 없음 | 중요한 오류와 디버그 정보가 섞여 있음 |
| 파일 관리 없음 | 로그 파일이 무한정 커짐 (롤링 없음) |
| 성능 문제 | 동기 파일 쓰기로 UI가 느려질 수 있음 |
| 출력 대상 고정 | 파일에만 쓰거나, 콘솔에만 출력하거나 |

### 구조적 로깅이란?

구조적 로깅(Structured Logging)은 로그 메시지를 **단순 텍스트가 아닌 구조화된 데이터**로 기록하는 방식입니다.

```csharp
// 전통적 로깅 — 단순 문자열
"직원 홍길동(ID: 42)이 개발팀에서 인사팀으로 이동했습니다. 처리 시간: 150ms"

// 구조적 로깅 — 의미 있는 속성이 포함된 데이터
{
  "Timestamp": "2025-01-15T10:30:00",
  "Level": "Information",
  "MessageTemplate": "직원 {EmployeeName}(ID: {EmployeeId})이 {FromDept}에서 {ToDept}으로 이동했습니다. 처리 시간: {ElapsedMs}ms",
  "Properties": {
    "EmployeeName": "홍길동",
    "EmployeeId": 42,
    "FromDept": "개발팀",
    "ToDept": "인사팀",
    "ElapsedMs": 150
  }
}
```

구조적으로 기록하면 나중에 **"처리 시간이 500ms를 초과한 로그만 조회"**, **"특정 직원 ID에 대한 모든 로그 검색"** 같은 작업이 가능해집니다.

---

## 2. Serilog란?

Serilog는 .NET 생태계에서 가장 널리 사용되는 **구조적 로깅 라이브러리**입니다.

### 핵심 특징

| 특징 | 설명 |
|------|------|
| **구조적 로깅** | 로그 메시지에 속성(Property)을 자연스럽게 포함 |
| **Sink 시스템** | 콘솔, 파일, Seq, Elasticsearch 등 다양한 출력 대상 |
| **높은 성능** | 비동기 쓰기, 버퍼링 지원 |
| **풍부한 생태계** | 200개 이상의 Sink 패키지 |
| **Microsoft.Extensions.Logging 통합** | ILogger<T>와 완벽 호환 |

### Sink(싱크)란?

Sink는 로그를 **어디에 출력할지** 결정하는 플러그인입니다.

```
로그 메시지 → Serilog 엔진 → ┬─ Console Sink   → 콘솔 창
                              ├─ File Sink      → 로그 파일
                              ├─ Seq Sink       → Seq 서버
                              └─ Debug Sink     → Debug 출력 창
```

---

## 3. NuGet 패키지 설치

```bash
# 핵심 패키지
dotnet add package Serilog --version 4.2.0

# Sink 패키지 — 출력 대상
dotnet add package Serilog.Sinks.Console --version 6.0.0    # 콘솔 출력
dotnet add package Serilog.Sinks.File --version 6.0.0       # 파일 출력

# Microsoft.Extensions 통합
dotnet add package Serilog.Extensions.Hosting --version 9.0.0  # Host 통합

# (선택) appsettings.json 기반 설정
dotnet add package Serilog.Settings.Configuration --version 9.0.0
```

### 패키지 역할 정리

| 패키지 | 역할 |
|--------|------|
| `Serilog` | 핵심 로깅 엔진 |
| `Serilog.Sinks.Console` | 콘솔(Output 창)에 로그 출력 |
| `Serilog.Sinks.File` | 로그 파일 생성 및 롤링 |
| `Serilog.Extensions.Hosting` | `IHost` 빌더에 Serilog 통합 |
| `Serilog.Settings.Configuration` | `appsettings.json`에서 설정 읽기 |

---

## 4. Serilog 설정

### 4.1 코드 기반 설정

코드에서 직접 Serilog를 구성하는 방식입니다. 설정이 컴파일 타임에 확정되므로 오타 위험이 적습니다.

```csharp
// App.xaml.cs 또는 Program.cs에서 설정

using Serilog;
using Serilog.Events;

// Serilog 로거 생성 — 앱 시작 시 가장 먼저 설정해야 함
Log.Logger = new LoggerConfiguration()
    // ── 최소 로그 레벨 ──
    // Information 이상만 기록 (Verbose, Debug는 무시)
    .MinimumLevel.Information()

    // ── 특정 네임스페이스의 레벨 재정의 ──
    // Microsoft 관련 로그는 Warning 이상만 (너무 많은 로그 방지)
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    // System 관련 로그도 Warning 이상만
    .MinimumLevel.Override("System", LogEventLevel.Warning)

    // ── 모든 로그에 공통 속성 추가 ──
    .Enrich.FromLogContext()         // LogContext에서 동적 속성 추가
    .Enrich.WithMachineName()        // 컴퓨터 이름
    .Enrich.WithThreadId()           // 스레드 ID

    // ── 콘솔 Sink ──
    .WriteTo.Console(
        outputTemplate:
            "[{Timestamp:HH:mm:ss} {Level:u3}] {SourceContext}{NewLine}" +
            "  {Message:lj}{NewLine}" +
            "{Exception}")

    // ── 파일 Sink ──
    .WriteTo.File(
        path: "logs/app-.log",          // 로그 파일 경로
        rollingInterval: RollingInterval.Day,  // 매일 새 파일 생성 (app-20250115.log)
        retainedFileCountLimit: 30,     // 최근 30일치 파일만 유지
        fileSizeLimitBytes: 50_000_000, // 파일당 최대 50MB
        rollOnFileSizeLimit: true,      // 크기 초과 시 새 파일로 롤링
        outputTemplate:
            "[{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} {Level:u3}] " +
            "[{SourceContext}] {Message:lj}{NewLine}{Exception}",
        encoding: System.Text.Encoding.UTF8)  // UTF-8 인코딩 (한글 지원)

    // 로거 생성
    .CreateLogger();
```

### 4.2 appsettings.json 기반 설정

앱을 재컴파일하지 않고도 로그 레벨이나 출력 대상을 변경할 수 있습니다.

```jsonc
// appsettings.json
{
  "Serilog": {
    // 최소 로그 레벨
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning",
        "MyApp.Repositories": "Debug"
      }
    },

    // 로그 출력 대상 (Sink)
    "WriteTo": [
      {
        // 콘솔 Sink
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {SourceContext} | {Message:lj}{NewLine}{Exception}"
        }
      },
      {
        // 파일 Sink
        "Name": "File",
        "Args": {
          "path": "logs/app-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30,
          "fileSizeLimitBytes": 50000000,
          "rollOnFileSizeLimit": true,
          "outputTemplate": "[{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} {Level:u3}] [{SourceContext}] {Message:lj}{NewLine}{Exception}",
          "encoding": "System.Text.Encoding.UTF8"
        }
      }
    ],

    // 모든 로그에 공통 속성 추가
    "Enrich": [
      "FromLogContext",
      "WithMachineName",
      "WithThreadId"
    ]
  }
}
```

```csharp
// appsettings.json에서 Serilog 설정을 읽어 적용
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(configuration)  // IConfiguration에서 설정 읽기
    .CreateLogger();
```

### 4.3 출력 템플릿 토큰 설명

| 토큰 | 설명 | 예시 |
|------|------|------|
| `{Timestamp:HH:mm:ss}` | 시간 (포맷 지정) | `14:30:15` |
| `{Level:u3}` | 로그 레벨 (대문자 3글자) | `INF`, `ERR`, `WRN` |
| `{SourceContext}` | 로거를 생성한 클래스명 | `MyApp.ViewModels.EmployeeViewModel` |
| `{Message:lj}` | 로그 메시지 (lj = 리터럴 JSON) | `직원 홍길동 생성 완료` |
| `{Exception}` | 예외 정보 (있는 경우만) | 스택 트레이스 |
| `{NewLine}` | 줄바꿈 | |
| `{Properties:j}` | 모든 속성 (JSON) | `{"EmployeeId": 42, ...}` |

### 4.4 파일 롤링 옵션

| 옵션 | 설명 |
|------|------|
| `RollingInterval.Day` | 매일 새 파일 (app-20250115.log) |
| `RollingInterval.Hour` | 매시간 새 파일 (app-2025011514.log) |
| `retainedFileCountLimit` | 유지할 파일 수 (넘으면 오래된 파일 삭제) |
| `fileSizeLimitBytes` | 파일당 최대 크기 |
| `rollOnFileSizeLimit` | 크기 초과 시 새 파일 생성 (app-20250115_001.log) |

---

## 5. DI에 등록하기

### 방법 1: Microsoft.Extensions.Logging과 통합 (권장)

이 방법을 사용하면 `ILogger<T>` 인터페이스를 통해 Serilog를 사용합니다. Microsoft의 표준 로깅 인터페이스를 따르므로, 나중에 Serilog를 다른 로깅 라이브러리로 교체해도 코드를 수정할 필요가 없습니다.

```csharp
// App.xaml.cs

using Microsoft.Extensions.Hosting;
using Serilog;

namespace MyApp;

public partial class App : Application
{
    private readonly IHost _host;

    public App()
    {
        // ── 1단계: Serilog 로거 생성 (가장 먼저) ──
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Information()
            .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
            .Enrich.FromLogContext()
            .WriteTo.Console()
            .WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day)
            .CreateLogger();

        try
        {
            Log.Information("=== 애플리케이션 시작 ===");

            _host = Host.CreateDefaultBuilder()
                // ── 2단계: Serilog를 호스트에 등록 ──
                // UseSerilog()는 Serilog.Extensions.Hosting 패키지가 제공
                .UseSerilog()
                .ConfigureServices((context, services) =>
                {
                    // Repository, ViewModel, View 등록 (기존과 동일)
                    services.AddTransient<IEmployeeRepository, EmployeeRepository>();
                    services.AddTransient<EmployeeViewModel>();
                    services.AddTransient<EmployeeWindow>();
                })
                .Build();
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "애플리케이션 시작 중 치명적 오류 발생");
            throw;
        }
    }

    protected override async void OnStartup(StartupEventArgs e)
    {
        await _host.StartAsync();
        var mainWindow = _host.Services.GetRequiredService<EmployeeWindow>();
        mainWindow.Show();
        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        Log.Information("=== 애플리케이션 종료 ===");

        await _host.StopAsync();
        _host.Dispose();

        // ── 3단계: 앱 종료 시 로그 버퍼를 모두 기록 ──
        // 이 줄이 없으면 마지막 몇 개의 로그가 파일에 기록되지 않을 수 있음
        await Log.CloseAndFlushAsync();

        base.OnExit(e);
    }
}
```

### 방법 2: Serilog.ILogger 직접 주입

Serilog의 자체 `ILogger` 인터페이스를 직접 사용하는 방법입니다. Serilog에 특화된 기능(Enrichment, ForContext 등)을 더 직접적으로 사용할 수 있습니다.

```csharp
// DI에 Serilog.ILogger를 직접 등록
services.AddSingleton<Serilog.ILogger>(Log.Logger);

// ViewModel에서 사용
public partial class EmployeeViewModel : ObservableObject
{
    private readonly Serilog.ILogger _logger;

    public EmployeeViewModel(Serilog.ILogger logger)
    {
        // ForContext로 이 클래스의 이름을 SourceContext 속성에 추가
        _logger = logger.ForContext<EmployeeViewModel>();
    }
}
```

### 방법 1 vs 방법 2 비교

| 항목 | ILogger<T> (Microsoft) | Serilog.ILogger (직접) |
|------|------------------------|------------------------|
| 인터페이스 | `Microsoft.Extensions.Logging.ILogger<T>` | `Serilog.ILogger` |
| 라이브러리 종속성 | 없음 (표준 인터페이스) | Serilog에 종속 |
| 교체 용이성 | 높음 | 낮음 |
| Serilog 전용 기능 | 제한적 | 모두 사용 가능 |
| **권장 여부** | **권장** | 특수한 경우에 사용 |

> **권장**: 방법 1(`ILogger<T>`)을 사용하세요. 표준 인터페이스를 따르면서도 Serilog의 구조적 로깅 기능을 그대로 활용할 수 있습니다.

---

## 6. 로그 레벨 사용 가이드

Serilog는 6가지 로그 레벨을 제공합니다. 적절한 레벨을 사용하는 것이 중요합니다.

```
Verbose → Debug → Information → Warning → Error → Fatal
 (가장 낮음)                                    (가장 높음)
```

최소 레벨을 `Information`으로 설정하면, `Verbose`와 `Debug`는 무시되고 `Information` 이상만 기록됩니다.

### 각 레벨의 사용 기준

#### Verbose (가장 상세)

```csharp
// 개발 중에만 필요한 매우 상세한 정보
// 운영 환경에서는 절대 활성화하지 않음
_logger.LogTrace("바인딩 값 변경: Name={OldValue} → {NewValue}", oldName, newName);
_logger.LogTrace("SQL 파라미터: @Id={Id}, @Name={Name}", id, name);
```

> **참고**: `ILogger<T>`에서는 `LogTrace()`가 Serilog의 `Verbose`에 해당합니다.

#### Debug (디버그)

```csharp
// 개발/디버깅 시 유용한 정보
// 운영 환경에서는 보통 비활성화
_logger.LogDebug("직원 조회 쿼리 실행: Id={EmployeeId}", id);
_logger.LogDebug("캐시 히트: Key={CacheKey}", cacheKey);
```

#### Information (정보)

```csharp
// 앱의 정상적인 동작을 기록
// 운영 환경에서도 활성화하는 기본 레벨
_logger.LogInformation("직원 생성 완료: {EmployeeName}, Id={EmployeeId}", name, newId);
_logger.LogInformation("사용자 {UserId} 로그인 성공", userId);
_logger.LogInformation("직원 목록 로드: {Count}건, 소요 시간: {ElapsedMs}ms", count, elapsed);
```

#### Warning (경고)

```csharp
// 예상치 못한 상황이지만 앱은 계속 동작
// 주의가 필요한 상황
_logger.LogWarning("직원을 찾을 수 없음: Id={EmployeeId}", id);
_logger.LogWarning("DB 연결 재시도 중: 시도 {Attempt}/3", attempt);
_logger.LogWarning("API 응답 지연: {ElapsedMs}ms (임계값: 2000ms)", elapsed);
```

#### Error (오류)

```csharp
// 특정 작업이 실패했지만 앱 전체가 중단되지는 않음
// 반드시 예외 객체를 함께 기록
_logger.LogError(ex, "직원 저장 실패: {EmployeeName}", name);
_logger.LogError(ex, "DB 연결 실패: {ConnectionString}", maskedConnStr);
_logger.LogError(ex, "API 호출 실패: {Url}, 상태 코드: {StatusCode}", url, statusCode);
```

#### Fatal (치명적)

```csharp
// 앱이 중단되어야 할 정도의 심각한 오류
// 매우 드물게 사용
_logger.LogCritical(ex, "데이터베이스 연결이 완전히 불가능합니다");
_logger.LogCritical(ex, "필수 설정 파일을 찾을 수 없습니다: {Path}", configPath);
```

> **참고**: `ILogger<T>`에서는 `LogCritical()`이 Serilog의 `Fatal`에 해당합니다.

### 레벨 선택 요약표

| 레벨 | 언제 사용 | 예시 |
|------|----------|------|
| **Verbose** | 변수 값 추적, 실행 흐름 추적 | SQL 파라미터 값 |
| **Debug** | 개발 중 디버깅 정보 | 쿼리 실행 시작 |
| **Information** | 정상 동작의 중요 이벤트 | CRUD 완료, 로그인 |
| **Warning** | 예상 밖 상황이나 앱은 계속 동작 | 데이터 못 찾음, 재시도 |
| **Error** | 작업 실패 (예외 포함) | 저장 실패, 연결 오류 |
| **Fatal** | 앱 중단이 필요한 심각한 오류 | DB 완전 불통 |

---

## 7. ViewModel에서 로깅하기

### 기본 패턴

```csharp
// ViewModels/EmployeeViewModel.cs

using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.Logging;
using System.Collections.ObjectModel;
using System.Diagnostics;

namespace MyApp.ViewModels;

/// <summary>
/// 직원 관리 ViewModel — 로깅 패턴 예제
/// </summary>
public partial class EmployeeViewModel : ObservableObject
{
    private readonly IEmployeeRepository _repository;
    private readonly ILogger<EmployeeViewModel> _logger;

    public EmployeeViewModel(
        IEmployeeRepository repository,
        ILogger<EmployeeViewModel> logger)  // DI에서 ILogger<T> 주입
    {
        _repository = repository;
        _logger = logger;

        // 생성자에서 ViewModel 초기화 로그
        _logger.LogDebug("EmployeeViewModel 인스턴스 생성");
    }

    public ObservableCollection<Employee> Employees { get; } = [];

    [ObservableProperty]
    private bool _isLoading;

    /// <summary>
    /// 직원 목록 로드 — 로깅이 포함된 완전한 예제
    /// </summary>
    [RelayCommand]
    private async Task LoadEmployeesAsync()
    {
        // Stopwatch로 소요 시간 측정 (구조적 로깅에 포함)
        var stopwatch = Stopwatch.StartNew();

        _logger.LogInformation("직원 목록 로드 시작");

        try
        {
            IsLoading = true;

            var employees = await _repository.GetAllAsync();

            Employees.Clear();
            foreach (var emp in employees)
            {
                Employees.Add(emp);
            }

            stopwatch.Stop();

            // 구조적 로깅: {Count}와 {ElapsedMs}는 속성으로 기록됨
            _logger.LogInformation(
                "직원 목록 로드 완료: {Count}건, 소요 시간: {ElapsedMs}ms",
                Employees.Count,
                stopwatch.ElapsedMilliseconds);

            // 성능 경고 — 로드가 느리면 Warning 레벨로 기록
            if (stopwatch.ElapsedMilliseconds > 2000)
            {
                _logger.LogWarning(
                    "직원 목록 로드가 느림: {ElapsedMs}ms (임계값: 2000ms)",
                    stopwatch.ElapsedMilliseconds);
            }
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            // Error 레벨: 예외 객체를 첫 번째 인자로 전달
            // Serilog가 스택 트레이스를 자동으로 기록
            _logger.LogError(ex,
                "직원 목록 로드 실패. 소요 시간: {ElapsedMs}ms",
                stopwatch.ElapsedMilliseconds);
        }
        finally
        {
            IsLoading = false;
        }
    }

    /// <summary>
    /// 직원 생성 — 성공/실패 모두 로깅
    /// </summary>
    [RelayCommand]
    private async Task CreateEmployeeAsync(Employee employee)
    {
        // 입력 검증 단계에서도 로깅
        if (string.IsNullOrWhiteSpace(employee.Name))
        {
            _logger.LogWarning("직원 생성 시도 — 이름이 비어 있음");
            return;
        }

        _logger.LogInformation("직원 생성 시작: {EmployeeName}, 부서: {Department}",
            employee.Name, employee.Department);

        try
        {
            var newId = await _repository.CreateAsync(employee);

            // 성공 로그 — 생성된 ID 포함
            _logger.LogInformation("직원 생성 완료: {EmployeeName}, Id={EmployeeId}",
                employee.Name, newId);

            await LoadEmployeesAsync();
        }
        catch (Exception ex)
        {
            // 실패 로그 — 예외와 입력 데이터 포함
            _logger.LogError(ex, "직원 생성 실패: {EmployeeName}, Email={Email}",
                employee.Name, employee.Email);
        }
    }
}
```

### 로깅 시 주의사항

```csharp
// ❌ 문자열 보간 사용 — 구조적 로깅의 장점이 사라짐
_logger.LogInformation($"직원 {employee.Name} 생성 완료, ID: {newId}");
// → 결과: 단순 문자열로 기록됨, 속성 검색 불가

// ✅ 메시지 템플릿 사용 — 구조적 로깅
_logger.LogInformation("직원 {EmployeeName} 생성 완료, Id={EmployeeId}",
    employee.Name, newId);
// → 결과: EmployeeName="홍길동", EmployeeId=42 속성이 구조적으로 기록됨

// ❌ 불필요한 ToString() 호출 — 성능 낭비
_logger.LogDebug("직원 수: " + employees.Count.ToString());

// ✅ 메시지 템플릿에 직접 전달
_logger.LogDebug("직원 수: {Count}", employees.Count);

// ❌ 민감 정보를 로그에 포함 — 보안 위험
_logger.LogInformation("DB 연결: {ConnectionString}", connectionString);

// ✅ 민감 정보는 마스킹
_logger.LogInformation("DB 연결: Host={Host}, Database={Database}",
    settings.Host, settings.Database);
```

---

## 8. Service에서 로깅하기

Repository나 서비스 클래스에서도 동일한 패턴으로 로깅합니다.

```csharp
// Repositories/EmployeeRepository.cs

using Dapper;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using Npgsql;
using System.Diagnostics;

namespace MyApp.Repositories;

/// <summary>
/// 직원 Repository — 서비스 계층의 로깅 예제
/// </summary>
public sealed class EmployeeRepository : IEmployeeRepository
{
    private readonly string _connectionString;
    private readonly ILogger<EmployeeRepository> _logger;

    public EmployeeRepository(
        IOptions<DatabaseSettings> options,
        ILogger<EmployeeRepository> logger)
    {
        _connectionString = options.Value.ConnectionString;
        _logger = logger;
    }

    public async Task<IEnumerable<Employee>> GetAllAsync(
        CancellationToken cancellationToken = default)
    {
        const string sql = "SELECT * FROM employees ORDER BY name";

        var stopwatch = Stopwatch.StartNew();

        // Debug 레벨로 SQL 실행 시작을 기록
        _logger.LogDebug("SQL 실행 시작: {SqlQuery}", sql);

        try
        {
            await using var connection = new NpgsqlConnection(_connectionString);

            var result = await connection.QueryAsync<Employee>(
                new CommandDefinition(sql, cancellationToken: cancellationToken));

            var employees = result.AsList();
            stopwatch.Stop();

            // Information 레벨로 결과를 기록
            _logger.LogInformation(
                "직원 전체 조회 완료: {Count}건, SQL 실행 시간: {ElapsedMs}ms",
                employees.Count,
                stopwatch.ElapsedMilliseconds);

            return employees;
        }
        catch (NpgsqlException ex)
        {
            stopwatch.Stop();

            // PostgreSQL 관련 오류는 상세하게 기록
            _logger.LogError(ex,
                "PostgreSQL 오류 발생. " +
                "SqlState={SqlState}, " +
                "SQL 실행 시간: {ElapsedMs}ms",
                ex.SqlState,
                stopwatch.ElapsedMilliseconds);

            throw; // 예외를 다시 던져 상위에서 처리하게 함
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            _logger.LogError(ex,
                "직원 전체 조회 중 예기치 않은 오류. SQL 실행 시간: {ElapsedMs}ms",
                stopwatch.ElapsedMilliseconds);

            throw;
        }
    }

    public async Task<int> CreateAsync(
        Employee employee,
        CancellationToken cancellationToken = default)
    {
        const string sql = """
            INSERT INTO employees (name, department, email, hire_date, is_active, created_at)
            VALUES (@Name, @Department, @Email, @HireDate, @IsActive, NOW())
            RETURNING id
            """;

        _logger.LogDebug(
            "직원 생성 SQL 실행: Name={EmployeeName}, Department={Department}",
            employee.Name, employee.Department);

        try
        {
            await using var connection = new NpgsqlConnection(_connectionString);

            var newId = await connection.ExecuteScalarAsync<int>(
                new CommandDefinition(sql, employee, cancellationToken: cancellationToken));

            _logger.LogInformation(
                "직원 INSERT 완료: Id={EmployeeId}, Name={EmployeeName}",
                newId, employee.Name);

            return newId;
        }
        catch (NpgsqlException ex) when (ex.SqlState == "23505")
        {
            // PostgreSQL 에러 코드 23505 = 유니크 제약 조건 위반
            _logger.LogWarning(
                "직원 생성 실패 — 중복 데이터: Name={EmployeeName}, Email={Email}. " +
                "SqlState={SqlState}",
                employee.Name, employee.Email, ex.SqlState);

            throw; // 또는 적절한 비즈니스 예외로 변환
        }
    }
}
```

### 서비스 로깅 패턴 요약

```
서비스 메서드 진입
    │
    ├─ Debug: "메서드 시작" + 입력 파라미터
    │
    ├─ (작업 수행)
    │
    ├─ 성공 → Information: "완료" + 결과 + 소요 시간
    │
    └─ 실패 → Error: 예외 + 입력 파라미터 + 소요 시간
              Warning: 비즈니스 규칙 위반 (중복 등)
```

---

## 9. 구조적 로깅의 장점

### 속성(Property)을 활용한 검색

구조적 로깅의 가장 큰 장점은 로그 메시지의 각 값을 **독립적인 속성**으로 검색하고 필터링할 수 있다는 것입니다.

```csharp
// 이 로그 메시지는 단순 텍스트가 아닌 구조화된 데이터로 저장됨
_logger.LogInformation(
    "직원 {EmployeeName}(ID: {EmployeeId})이 {Department} 부서에서 작업 완료. " +
    "소요 시간: {ElapsedMs}ms",
    "홍길동", 42, "개발팀", 150);
```

저장되는 구조:

```json
{
  "Timestamp": "2025-01-15T10:30:00.123+09:00",
  "Level": "Information",
  "MessageTemplate": "직원 {EmployeeName}(ID: {EmployeeId})이 {Department} 부서에서 작업 완료. 소요 시간: {ElapsedMs}ms",
  "RenderedMessage": "직원 \"홍길동\"(ID: 42)이 \"개발팀\" 부서에서 작업 완료. 소요 시간: 150ms",
  "Properties": {
    "EmployeeName": "홍길동",
    "EmployeeId": 42,
    "Department": "개발팀",
    "ElapsedMs": 150,
    "SourceContext": "MyApp.ViewModels.EmployeeViewModel",
    "MachineName": "OFFICE-PC01",
    "ThreadId": 1
  }
}
```

### 객체 구조 분해 (`@` 연산자)

`@` 연산자를 사용하면 객체의 속성을 자동으로 펼쳐서 기록합니다.

```csharp
// 일반 로깅 — 객체의 ToString() 결과만 기록
_logger.LogInformation("직원 정보: {Employee}", employee);
// → "직원 정보: MyApp.Models.Employee"  (쓸모없음!)

// 구조 분해 로깅 — 객체의 속성을 JSON으로 기록
_logger.LogInformation("직원 정보: {@Employee}", employee);
// → "직원 정보: {"Id": 42, "Name": "홍길동", "Department": "개발팀", ...}"
```

```csharp
// 실전 예제: 작업 요청 전체를 구조적으로 기록
var request = new
{
    Action = "TransferDepartment",
    EmployeeId = 42,
    FromDepartment = "개발팀",
    ToDepartment = "인사팀",
    RequestedBy = "admin"
};

_logger.LogInformation("부서 이동 요청: {@TransferRequest}", request);
// 결과: 모든 속성이 개별 필드로 기록되어 검색 가능
```

### 의미 있는 속성 이름 사용

```csharp
// ❌ 의미 없는 이름
_logger.LogInformation("처리 완료: {A}, {B}, {C}", name, count, elapsed);

// ✅ 의미 있는 이름 — 나중에 검색할 때 유용
_logger.LogInformation(
    "처리 완료: {EmployeeName}, 건수: {ProcessedCount}, 소요: {ElapsedMs}ms",
    name, count, elapsed);
```

### LogContext로 범위 정보 추가

```csharp
// LogContext를 사용하면 특정 범위 내의 모든 로그에 공통 속성을 추가할 수 있음
using (LogContext.PushProperty("OperationId", Guid.NewGuid()))
using (LogContext.PushProperty("UserId", currentUserId))
{
    // 이 블록 안의 모든 로그에 OperationId와 UserId가 자동으로 포함됨
    _logger.LogInformation("작업 시작");
    await _repository.CreateAsync(employee);
    _logger.LogInformation("작업 완료");
}
// 블록을 벗어나면 속성이 제거됨
```

---

## 10. 전체 코드 예제

### 10.1 프로젝트 구조

```
MyApp/
├── App.xaml
├── App.xaml.cs              ← Serilog 설정 + DI 등록
├── appsettings.json         ← 로그 설정
├── Models/
│   ├── Employee.cs
│   └── Settings/
│       └── DatabaseSettings.cs
├── Repositories/
│   ├── IEmployeeRepository.cs
│   └── EmployeeRepository.cs   ← 서비스 로깅 예제
├── ViewModels/
│   └── EmployeeViewModel.cs    ← ViewModel 로깅 예제
└── Views/
    └── EmployeeWindow.xaml
```

### 10.2 App.xaml.cs — 전체 설정

```csharp
// App.xaml.cs — Serilog 설정의 전체 예제

using System.Windows;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;
using Serilog.Events;

namespace MyApp;

public partial class App : Application
{
    private readonly IHost _host;

    public App()
    {
        // ╔══════════════════════════════════════════════════╗
        // ║  1단계: Serilog 로거 생성 (앱 시작 시 가장 먼저) ║
        // ╚══════════════════════════════════════════════════╝
        Log.Logger = new LoggerConfiguration()
            // 기본 최소 레벨: Information
            .MinimumLevel.Information()
            // Microsoft/System 네임스페이스는 Warning 이상만
            .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
            .MinimumLevel.Override("System", LogEventLevel.Warning)
            // LogContext에서 동적 속성 추가 지원
            .Enrich.FromLogContext()
            // 콘솔(Output 창) 출력
            .WriteTo.Console(
                outputTemplate:
                    "[{Timestamp:HH:mm:ss} {Level:u3}] " +
                    "{SourceContext}{NewLine}" +
                    "  {Message:lj}{NewLine}" +
                    "{Exception}")
            // 파일 출력 (매일 롤링, 30일 보관)
            .WriteTo.File(
                path: "logs/myapp-.log",
                rollingInterval: RollingInterval.Day,
                retainedFileCountLimit: 30,
                fileSizeLimitBytes: 50_000_000,
                rollOnFileSizeLimit: true,
                outputTemplate:
                    "[{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} {Level:u3}] " +
                    "[{SourceContext}] " +
                    "{Message:lj}{NewLine}{Exception}",
                encoding: System.Text.Encoding.UTF8)
            .CreateLogger();

        // ╔══════════════════════════════════════════════════╗
        // ║  2단계: Host 빌드 + Serilog 통합                 ║
        // ╚══════════════════════════════════════════════════╝
        try
        {
            Log.Information("=== 애플리케이션 시작 ===");

            _host = Host.CreateDefaultBuilder()
                .ConfigureAppConfiguration((context, config) =>
                {
                    config.SetBasePath(AppContext.BaseDirectory);
                    config.AddJsonFile("appsettings.json",
                        optional: false, reloadOnChange: true);
                })
                // Serilog를 Host의 로깅 시스템으로 등록
                // 이후 ILogger<T>를 주입받으면 Serilog가 처리함
                .UseSerilog()
                .ConfigureServices((context, services) =>
                {
                    // 설정
                    services.Configure<DatabaseSettings>(
                        context.Configuration.GetSection(DatabaseSettings.SectionName));

                    // Repository
                    services.AddTransient<IEmployeeRepository, EmployeeRepository>();

                    // ViewModel
                    services.AddTransient<EmployeeViewModel>();

                    // View
                    services.AddTransient<EmployeeWindow>();
                })
                .Build();
        }
        catch (Exception ex)
        {
            // 초기화 중 오류 발생 시 Fatal 레벨로 기록 후 종료
            Log.Fatal(ex, "애플리케이션 초기화 중 치명적 오류");
            Log.CloseAndFlush();
            throw;
        }
    }

    protected override async void OnStartup(StartupEventArgs e)
    {
        await _host.StartAsync();

        var mainWindow = _host.Services.GetRequiredService<EmployeeWindow>();
        mainWindow.Show();

        Log.Information("메인 윈도우 표시 완료");
        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        Log.Information("=== 애플리케이션 종료 시작 ===");

        await _host.StopAsync();
        _host.Dispose();

        // ╔══════════════════════════════════════════════════╗
        // ║  3단계: 로그 버퍼 flush (반드시 호출!)           ║
        // ╚══════════════════════════════════════════════════╝
        // Serilog는 성능을 위해 로그를 버퍼에 모아서 한꺼번에 씁니다.
        // 앱 종료 시 이 메서드를 호출하지 않으면 마지막 로그가 누락될 수 있습니다.
        await Log.CloseAndFlushAsync();

        base.OnExit(e);
    }
}
```

### 10.3 ViewModel — 로깅이 포함된 전체 코드

```csharp
// ViewModels/EmployeeViewModel.cs — 로깅이 포함된 전체 예제

using System.Collections.ObjectModel;
using System.Diagnostics;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.Logging;

namespace MyApp.ViewModels;

public partial class EmployeeViewModel : ObservableObject
{
    private readonly IEmployeeRepository _repository;
    private readonly ILogger<EmployeeViewModel> _logger;

    public EmployeeViewModel(
        IEmployeeRepository repository,
        ILogger<EmployeeViewModel> logger)
    {
        _repository = repository;
        _logger = logger;
        _logger.LogDebug("EmployeeViewModel 인스턴스 생성");
    }

    // ── 속성 ──
    public ObservableCollection<Employee> Employees { get; } = [];

    [ObservableProperty]
    private Employee? _selectedEmployee;

    [ObservableProperty]
    private string _name = string.Empty;

    [ObservableProperty]
    private string _department = string.Empty;

    [ObservableProperty]
    private string _email = string.Empty;

    [ObservableProperty]
    private bool _isLoading;

    [ObservableProperty]
    private string _statusMessage = string.Empty;

    // ── 명령 ──

    [RelayCommand]
    private async Task LoadEmployeesAsync()
    {
        var sw = Stopwatch.StartNew();
        _logger.LogInformation("직원 목록 로드 시작");

        try
        {
            IsLoading = true;
            StatusMessage = "로딩 중...";

            var employees = await _repository.GetAllAsync();

            Employees.Clear();
            foreach (var emp in employees)
            {
                Employees.Add(emp);
            }

            sw.Stop();
            StatusMessage = $"{Employees.Count}명 로드 완료 ({sw.ElapsedMilliseconds}ms)";

            _logger.LogInformation(
                "직원 목록 로드 완료: {Count}건, 소요: {ElapsedMs}ms",
                Employees.Count, sw.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            sw.Stop();
            StatusMessage = $"로드 실패: {ex.Message}";

            _logger.LogError(ex,
                "직원 목록 로드 실패. 소요: {ElapsedMs}ms",
                sw.ElapsedMilliseconds);
        }
        finally
        {
            IsLoading = false;
        }
    }

    [RelayCommand]
    private async Task CreateEmployeeAsync()
    {
        if (string.IsNullOrWhiteSpace(Name))
        {
            _logger.LogWarning("직원 생성 시도 — 이름이 비어 있음");
            StatusMessage = "이름을 입력하세요.";
            return;
        }

        _logger.LogInformation(
            "직원 생성 시작: Name={EmployeeName}, Dept={Department}",
            Name, Department);

        try
        {
            IsLoading = true;

            var employee = new Employee
            {
                Name = Name,
                Department = Department,
                Email = Email,
                HireDate = DateTime.Today,
                IsActive = true
            };

            var newId = await _repository.CreateAsync(employee);

            _logger.LogInformation(
                "직원 생성 완료: Id={EmployeeId}, Name={EmployeeName}",
                newId, Name);

            StatusMessage = $"직원 '{Name}' 생성 완료 (ID: {newId})";

            // 입력 초기화 및 목록 갱신
            Name = string.Empty;
            Department = string.Empty;
            Email = string.Empty;

            await LoadEmployeesAsync();
        }
        catch (Exception ex)
        {
            StatusMessage = $"생성 실패: {ex.Message}";
            _logger.LogError(ex,
                "직원 생성 실패: Name={EmployeeName}, Email={Email}",
                Name, Email);
        }
        finally
        {
            IsLoading = false;
        }
    }

    [RelayCommand]
    private async Task DeleteEmployeeAsync()
    {
        if (SelectedEmployee is null)
        {
            _logger.LogWarning("삭제 시도 — 선택된 직원 없음");
            StatusMessage = "삭제할 직원을 선택하세요.";
            return;
        }

        var targetId = SelectedEmployee.Id;
        var targetName = SelectedEmployee.Name;

        _logger.LogInformation(
            "직원 삭제 시작: Id={EmployeeId}, Name={EmployeeName}",
            targetId, targetName);

        try
        {
            IsLoading = true;

            var success = await _repository.DeleteAsync(targetId);

            if (success)
            {
                _logger.LogInformation(
                    "직원 삭제 완료: Id={EmployeeId}, Name={EmployeeName}",
                    targetId, targetName);

                StatusMessage = $"직원 '{targetName}' 삭제 완료";
                SelectedEmployee = null;
                await LoadEmployeesAsync();
            }
            else
            {
                _logger.LogWarning(
                    "직원 삭제 실패 — 존재하지 않음: Id={EmployeeId}",
                    targetId);

                StatusMessage = "해당 직원을 찾을 수 없습니다.";
            }
        }
        catch (Exception ex)
        {
            StatusMessage = $"삭제 실패: {ex.Message}";
            _logger.LogError(ex,
                "직원 삭제 실패: Id={EmployeeId}, Name={EmployeeName}",
                targetId, targetName);
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

### 10.4 로그 출력 예시

**콘솔 출력 (개발 중 Output 창):**

```
[10:30:00 INF] MyApp.App
  === 애플리케이션 시작 ===
[10:30:01 INF] MyApp.App
  메인 윈도우 표시 완료
[10:30:01 DBG] MyApp.ViewModels.EmployeeViewModel
  EmployeeViewModel 인스턴스 생성
[10:30:01 INF] MyApp.ViewModels.EmployeeViewModel
  직원 목록 로드 시작
[10:30:01 DBG] MyApp.Repositories.EmployeeRepository
  SQL 실행 시작: SELECT * FROM employees ORDER BY name
[10:30:01 INF] MyApp.Repositories.EmployeeRepository
  직원 전체 조회 완료: 15건, SQL 실행 시간: 23ms
[10:30:01 INF] MyApp.ViewModels.EmployeeViewModel
  직원 목록 로드 완료: 15건, 소요: 45ms
[10:30:15 INF] MyApp.ViewModels.EmployeeViewModel
  직원 생성 시작: Name=김철수, Dept=개발팀
[10:30:15 INF] MyApp.Repositories.EmployeeRepository
  직원 INSERT 완료: Id=16, Name=김철수
[10:30:15 INF] MyApp.ViewModels.EmployeeViewModel
  직원 생성 완료: Id=16, Name=김철수
```

**파일 출력 (logs/myapp-20250115.log):**

```
[2025-01-15 10:30:00.123 +09:00 INF] [MyApp.App] === 애플리케이션 시작 ===
[2025-01-15 10:30:01.456 +09:00 INF] [MyApp.App] 메인 윈도우 표시 완료
[2025-01-15 10:30:01.789 +09:00 INF] [MyApp.ViewModels.EmployeeViewModel] 직원 목록 로드 시작
[2025-01-15 10:30:01.812 +09:00 INF] [MyApp.Repositories.EmployeeRepository] 직원 전체 조회 완료: 15건, SQL 실행 시간: 23ms
[2025-01-15 10:30:01.834 +09:00 INF] [MyApp.ViewModels.EmployeeViewModel] 직원 목록 로드 완료: 15건, 소요: 45ms
```

---

## 요약

| 항목 | 핵심 |
|------|------|
| **Serilog** | .NET 최고의 구조적 로깅 라이브러리 |
| **Sink** | 콘솔, 파일 등 로그 출력 대상을 플러그인으로 추가 |
| **DI 통합** | `.UseSerilog()` + `ILogger<T>` 주입 (권장) |
| **구조적 로깅** | `{EmployeeName}` 형태의 메시지 템플릿으로 속성을 기록 |
| **@ 연산자** | `{@Employee}`로 객체를 구조적으로 분해하여 기록 |
| **로그 레벨** | 상황에 맞는 레벨 사용 (Debug, Information, Warning, Error, Fatal) |
| **파일 롤링** | 매일/크기별 자동 롤링, 오래된 파일 자동 삭제 |
| **CloseAndFlush** | 앱 종료 시 반드시 호출하여 버퍼의 로그를 모두 기록 |
| **주의** | 문자열 보간(`$""`) 대신 메시지 템플릿 사용, 민감 정보 마스킹 |

---

> **다음 문서**: [Polly를 이용한 HTTP 복원력 패턴](./03-http-polly.md) — 네트워크 호출 시 재시도, 서킷 브레이커, 타임아웃 전략을 구현합니다.
