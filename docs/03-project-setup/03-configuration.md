# 설정 관리 (Configuration)

> **목표**: `appsettings.json`을 사용하여 DB 연결 문자열, MQTT 설정, AWS 인증, Serilog 설정 등을 체계적으로 관리하고, DI 컨테이너를 통해 앱 전체에서 타입 안전하게 사용하는 방법을 익힙니다.

---

## 목차

1. [WinForms의 Settings.settings vs .NET의 Configuration 시스템](#1-winforms의-settingssettings-vs-net의-configuration-시스템)
2. [appsettings.json 파일 생성](#2-appsettingsjson-파일-생성)
3. [appsettings.json을 프로젝트에 포함시키기](#3-appsettingsjson을-프로젝트에-포함시키기)
4. [IConfiguration 바인딩 - Options 패턴](#4-iconfiguration-바인딩---options-패턴)
5. [환경별 설정 (appsettings.Development.json)](#5-환경별-설정-appsettingsdevelopmentjson)
6. [민감 정보 관리 (User Secrets)](#6-민감-정보-관리-user-secrets)
7. [전체 코드 예제](#7-전체-코드-예제)

---

## 1. WinForms의 Settings.settings vs .NET의 Configuration 시스템

### 1.1 WinForms 방식 (예전 방식)

WinForms에서는 프로젝트 속성의 **Settings** 탭에서 설정을 관리했습니다:

```csharp
// WinForms: Properties.Settings를 사용하는 방식
// Visual Studio의 프로젝트 속성 > Settings 탭에서 UI로 관리

// 설정 읽기
string connStr = Properties.Settings.Default.ConnectionString;
int timeout = Properties.Settings.Default.Timeout;

// 설정 쓰기 (User 범위만 가능)
Properties.Settings.Default.LastOpenedFile = "report.xlsx";
Properties.Settings.Default.Save();
```

**WinForms Settings의 한계:**
- Visual Studio에 종속적인 도구 (다른 IDE에서는 불편)
- XML 기반 (`app.config`) → 사람이 읽기 어려움
- 환경별 설정(개발/운영) 전환이 번거로움
- 복잡한 구조(중첩 객체) 표현이 어려움

### 1.2 .NET Configuration 시스템 (현대적 방식)

.NET의 Configuration 시스템은 **JSON 파일** 기반으로, 훨씬 유연하고 강력합니다.

```csharp
// .NET Configuration: appsettings.json 사용
// JSON 파일에서 설정을 읽어 타입 안전하게 사용

// 방법 1: IConfiguration에서 직접 읽기
string connStr = _configuration["ConnectionStrings:DefaultConnection"]!;

// 방법 2: Options 패턴으로 타입 안전하게 사용 (권장!)
// 설정 클래스에 자동으로 매핑됨
public class DatabaseSettings
{
    public string ConnectionString { get; set; } = string.Empty;
    public int CommandTimeout { get; set; } = 30;
}
```

### 1.3 비교 요약

| 항목 | WinForms (Settings.settings) | .NET (appsettings.json) |
|------|------------------------------|------------------------|
| 파일 형식 | XML (`app.config`) | JSON (`appsettings.json`) |
| 관리 도구 | Visual Studio Settings 편집기 | 아무 텍스트 에디터 |
| 가독성 | 낮음 (XML) | 높음 (JSON) |
| 환경별 설정 | 변환 도구 필요 | `appsettings.{환경}.json` |
| 중첩 구조 | 제한적 | 자유로움 (JSON 구조) |
| DI 통합 | 없음 | `IOptions<T>` 패턴으로 자연스럽게 통합 |
| 비밀 관리 | 없음 | User Secrets 지원 |
| Python 비유 | - | `.env` + `python-dotenv` 또는 `config.yaml` |

---

## 2. appsettings.json 파일 생성

프로젝트 루트(`MyApp/`)에 `appsettings.json` 파일을 생성합니다. 이 파일에 앱에서 사용하는 모든 설정을 JSON 형태로 정의합니다.

### 2.1 전체 appsettings.json

```json
{
  // ──────────────────────────────────────
  // 데이터베이스 연결 문자열
  // ──────────────────────────────────────
  "ConnectionStrings": {
    // PostgreSQL 연결 문자열
    // Host: DB 서버 주소
    // Port: PostgreSQL 기본 포트 (5432)
    // Database: 데이터베이스 이름
    // Username/Password: 인증 정보 (운영 환경에서는 User Secrets 사용!)
    "DefaultConnection": "Host=localhost;Port=5432;Database=myapp_db;Username=myapp_user;Password=dev_password_123"
  },

  // ──────────────────────────────────────
  // 데이터베이스 상세 설정
  // ──────────────────────────────────────
  "Database": {
    // SQL 명령 실행 제한 시간 (초)
    "CommandTimeout": 30,
    // 최대 동시 연결 수 (커넥션 풀)
    "MaxPoolSize": 20,
    // 재시도 횟수 (Polly와 연동)
    "RetryCount": 3,
    // 재시도 간격 (초)
    "RetryDelaySeconds": 2
  },

  // ──────────────────────────────────────
  // MQTT 설정
  // ──────────────────────────────────────
  "Mqtt": {
    // MQTT 브로커 서버 주소
    "BrokerHost": "localhost",
    // MQTT 브로커 포트 (1883: 비암호화, 8883: TLS)
    "BrokerPort": 1883,
    // 클라이언트 고유 ID (브로커에서 이 ID로 식별)
    "ClientId": "MyApp-Client-001",
    // 인증 정보 (필요한 경우)
    "Username": "",
    "Password": "",
    // 구독할 토픽 목록
    "Topics": [
      "devices/+/status",
      "devices/+/events",
      "notifications/#"
    ],
    // 연결 끊김 시 자동 재연결 여부
    "AutoReconnect": true,
    // 재연결 간격 (초)
    "ReconnectDelaySeconds": 5,
    // Keep Alive 간격 (초) - 연결 유지 확인 주기
    "KeepAliveSeconds": 60
  },

  // ──────────────────────────────────────
  // AWS S3 설정
  // ──────────────────────────────────────
  "Aws": {
    // AWS 리전 (서울: ap-northeast-2)
    "Region": "ap-northeast-2",
    // S3 버킷 이름
    "S3BucketName": "myapp-photos",
    // AWS 액세스 키 (운영 환경에서는 IAM 역할 또는 User Secrets 사용!)
    "AccessKeyId": "YOUR_ACCESS_KEY_HERE",
    "SecretAccessKey": "YOUR_SECRET_KEY_HERE",
    // 업로드 설정
    "MaxUploadSizeMb": 10,
    // 사전 서명 URL 만료 시간 (분)
    "PresignedUrlExpirationMinutes": 60
  },

  // ──────────────────────────────────────
  // Serilog 로깅 설정
  // ──────────────────────────────────────
  "Serilog": {
    // 최소 로그 레벨
    // Verbose < Debug < Information < Warning < Error < Fatal
    "MinimumLevel": "Information",
    // 로그 파일 설정
    "LogFilePath": "logs/myapp-.log",
    // 로그 파일 보관 일수
    "RetainedFileCountLimit": 30,
    // 단일 로그 파일 최대 크기 (바이트)
    "FileSizeLimitBytes": 10485760
  },

  // ──────────────────────────────────────
  // 앱 자체 설정
  // ──────────────────────────────────────
  "AppSettings": {
    // 앱 이름 (타이틀바 등에 표시)
    "AppName": "MyApp 관리 시스템",
    // 앱 버전
    "Version": "1.0.0",
    // 자동 새로고침 간격 (초) - 데이터를 주기적으로 갱신
    "AutoRefreshIntervalSeconds": 30,
    // 시스템 트레이로 최소화 여부
    "MinimizeToTray": true,
    // 시작 시 자동으로 데이터 로드할지 여부
    "LoadDataOnStartup": true,
    // 페이지당 표시 항목 수
    "PageSize": 50,
    // 날짜 표시 형식
    "DateFormat": "yyyy-MM-dd",
    // 시간 표시 형식
    "TimeFormat": "HH:mm:ss"
  }
}
```

> **주의**: JSON 표준에서는 주석(`//`)이 허용되지 않지만, `Microsoft.Extensions.Configuration.Json`은 주석을 지원합니다. 그래도 운영 환경에서는 주석을 제거하는 것이 좋습니다.

> **Python 개발자 참고**: 이 파일은 Python에서 `config.yaml`이나 `.env` 파일과 비슷한 역할입니다. 구조화된 설정을 JSON 형태로 관리하는 것이죠.

---

## 3. appsettings.json을 프로젝트에 포함시키기

`appsettings.json` 파일은 빌드 시 출력 디렉토리(`bin/Debug/net10.0-windows/`)에 복사되어야 앱이 읽을 수 있습니다.

### 3.1 .csproj에 CopyToOutputDirectory 설정

```xml
<!-- MyApp.csproj의 ItemGroup에 추가 -->
<ItemGroup>
  <!-- 기본 설정 파일: 항상 출력 디렉토리에 복사 -->
  <!-- PreserveNewest: 원본이 더 새로운 경우에만 복사 (빌드 속도 최적화) -->
  <None Update="appsettings.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>

  <!-- 개발 환경 전용 설정 파일: 있으면 복사, 없으면 무시 -->
  <None Update="appsettings.Development.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
</ItemGroup>
```

### 3.2 CopyToOutputDirectory 옵션 설명

| 옵션 | 동작 | 사용 시기 |
|------|------|-----------|
| `Never` | 복사하지 않음 (기본값) | 빌드에 필요 없는 참고 파일 |
| `Always` | 빌드할 때마다 항상 복사 | 외부 도구가 파일을 수정하는 경우 |
| `PreserveNewest` | 원본이 더 새로우면 복사 | **설정 파일에 권장** (빌드 최적화) |

### 3.3 빌드 후 확인

```bash
# 빌드
dotnet build

# 출력 디렉토리 확인
ls bin/Debug/net10.0-windows/

# appsettings.json이 있어야 함!
# 출력 예시:
#   MyApp.exe
#   MyApp.dll
#   appsettings.json       ← 여기에 복사됨!
#   appsettings.Development.json  ← 있으면 여기에도 복사됨
#   ...
```

> **흔한 실수**: `appsettings.json`이 출력 디렉토리에 복사되지 않으면 앱 실행 시 `FileNotFoundException`이 발생합니다. 반드시 `CopyToOutputDirectory`를 설정하세요!

---

## 4. IConfiguration 바인딩 - Options 패턴

### 4.1 Options 패턴이란?

`IConfiguration`에서 문자열 키로 값을 직접 읽는 것은 오타에 취약하고 타입 안전하지 않습니다. **Options 패턴**은 JSON의 각 섹션을 C# 클래스에 자동으로 매핑하여 타입 안전하게 사용할 수 있게 해줍니다.

```csharp
// ❌ 나쁜 예: 문자열 키로 직접 접근
// 오타가 나도 컴파일러가 못 잡음! 런타임에 에러 발생
string host = _configuration["Mqtt:BrokerHost"]!;
int port = int.Parse(_configuration["Mqtt:BrokerPort"]!);

// ✅ 좋은 예: Options 패턴으로 타입 안전하게 접근
// 컴파일 시점에 오타를 잡을 수 있음!
string host = _mqttSettings.BrokerHost;   // IDE 자동완성 지원!
int port = _mqttSettings.BrokerPort;       // 이미 int 타입!
```

### 4.2 설정 클래스 정의

`appsettings.json`의 각 섹션에 대응하는 C# 클래스를 만듭니다. 속성 이름이 JSON 키와 **정확히 일치**해야 합니다.

```csharp
// Models/Settings/DatabaseSettings.cs
// appsettings.json의 "Database" 섹션에 매핑

namespace MyApp.Models.Settings;

/// <summary>
/// 데이터베이스 관련 설정.
/// appsettings.json의 "Database" 섹션과 자동 매핑됩니다.
/// </summary>
public class DatabaseSettings
{
    /// <summary>
    /// 설정 파일의 섹션 이름 (바인딩 시 사용)
    /// </summary>
    public const string SectionName = "Database";

    /// <summary>
    /// SQL 명령 실행 제한 시간 (초)
    /// </summary>
    public int CommandTimeout { get; set; } = 30;

    /// <summary>
    /// 커넥션 풀 최대 연결 수
    /// </summary>
    public int MaxPoolSize { get; set; } = 20;

    /// <summary>
    /// 실패 시 재시도 횟수
    /// </summary>
    public int RetryCount { get; set; } = 3;

    /// <summary>
    /// 재시도 간격 (초)
    /// </summary>
    public int RetryDelaySeconds { get; set; } = 2;
}
```

```csharp
// Models/Settings/MqttSettings.cs
// appsettings.json의 "Mqtt" 섹션에 매핑

namespace MyApp.Models.Settings;

/// <summary>
/// MQTT 브로커 연결 설정.
/// appsettings.json의 "Mqtt" 섹션과 자동 매핑됩니다.
/// </summary>
public class MqttSettings
{
    public const string SectionName = "Mqtt";

    /// <summary>
    /// MQTT 브로커 서버 주소
    /// </summary>
    public string BrokerHost { get; set; } = "localhost";

    /// <summary>
    /// MQTT 브로커 포트
    /// </summary>
    public int BrokerPort { get; set; } = 1883;

    /// <summary>
    /// 클라이언트 고유 식별자
    /// </summary>
    public string ClientId { get; set; } = string.Empty;

    /// <summary>
    /// 인증 사용자 이름 (필요한 경우)
    /// </summary>
    public string Username { get; set; } = string.Empty;

    /// <summary>
    /// 인증 비밀번호 (필요한 경우)
    /// </summary>
    public string Password { get; set; } = string.Empty;

    /// <summary>
    /// 구독할 MQTT 토픽 목록
    /// </summary>
    public List<string> Topics { get; set; } = [];

    /// <summary>
    /// 연결 끊김 시 자동 재연결 여부
    /// </summary>
    public bool AutoReconnect { get; set; } = true;

    /// <summary>
    /// 재연결 대기 시간 (초)
    /// </summary>
    public int ReconnectDelaySeconds { get; set; } = 5;

    /// <summary>
    /// Keep Alive 간격 (초)
    /// </summary>
    public int KeepAliveSeconds { get; set; } = 60;
}
```

```csharp
// Models/Settings/AwsSettings.cs
// appsettings.json의 "Aws" 섹션에 매핑

namespace MyApp.Models.Settings;

/// <summary>
/// AWS S3 관련 설정.
/// appsettings.json의 "Aws" 섹션과 자동 매핑됩니다.
/// </summary>
public class AwsSettings
{
    public const string SectionName = "Aws";

    /// <summary>
    /// AWS 리전 (예: "ap-northeast-2" = 서울)
    /// </summary>
    public string Region { get; set; } = "ap-northeast-2";

    /// <summary>
    /// S3 버킷 이름
    /// </summary>
    public string S3BucketName { get; set; } = string.Empty;

    /// <summary>
    /// AWS 액세스 키 ID
    /// </summary>
    public string AccessKeyId { get; set; } = string.Empty;

    /// <summary>
    /// AWS 비밀 액세스 키
    /// </summary>
    public string SecretAccessKey { get; set; } = string.Empty;

    /// <summary>
    /// 최대 업로드 파일 크기 (MB)
    /// </summary>
    public int MaxUploadSizeMb { get; set; } = 10;

    /// <summary>
    /// 사전 서명 URL 만료 시간 (분)
    /// </summary>
    public int PresignedUrlExpirationMinutes { get; set; } = 60;
}
```

```csharp
// Models/Settings/SerilogSettings.cs
// appsettings.json의 "Serilog" 섹션에 매핑

namespace MyApp.Models.Settings;

/// <summary>
/// Serilog 로깅 관련 설정.
/// appsettings.json의 "Serilog" 섹션과 자동 매핑됩니다.
/// </summary>
public class SerilogSettings
{
    public const string SectionName = "Serilog";

    /// <summary>
    /// 최소 로그 레벨 (Verbose, Debug, Information, Warning, Error, Fatal)
    /// </summary>
    public string MinimumLevel { get; set; } = "Information";

    /// <summary>
    /// 로그 파일 경로 (롤링 패턴 포함)
    /// </summary>
    public string LogFilePath { get; set; } = "logs/myapp-.log";

    /// <summary>
    /// 보관할 최대 로그 파일 수
    /// </summary>
    public int RetainedFileCountLimit { get; set; } = 30;

    /// <summary>
    /// 단일 로그 파일 최대 크기 (바이트). 기본값: 10MB
    /// </summary>
    public long FileSizeLimitBytes { get; set; } = 10_485_760;
}
```

```csharp
// Models/Settings/AppSettings.cs
// appsettings.json의 "AppSettings" 섹션에 매핑

namespace MyApp.Models.Settings;

/// <summary>
/// 앱 자체 설정.
/// appsettings.json의 "AppSettings" 섹션과 자동 매핑됩니다.
/// </summary>
public class AppSettings
{
    public const string SectionName = "AppSettings";

    /// <summary>
    /// 앱 표시 이름
    /// </summary>
    public string AppName { get; set; } = "MyApp";

    /// <summary>
    /// 앱 버전
    /// </summary>
    public string Version { get; set; } = "1.0.0";

    /// <summary>
    /// 자동 새로고침 간격 (초)
    /// </summary>
    public int AutoRefreshIntervalSeconds { get; set; } = 30;

    /// <summary>
    /// 시스템 트레이로 최소화 여부
    /// </summary>
    public bool MinimizeToTray { get; set; } = true;

    /// <summary>
    /// 시작 시 데이터 자동 로드 여부
    /// </summary>
    public bool LoadDataOnStartup { get; set; } = true;

    /// <summary>
    /// 페이지당 항목 수
    /// </summary>
    public int PageSize { get; set; } = 50;

    /// <summary>
    /// 날짜 표시 형식
    /// </summary>
    public string DateFormat { get; set; } = "yyyy-MM-dd";

    /// <summary>
    /// 시간 표시 형식
    /// </summary>
    public string TimeFormat { get; set; } = "HH:mm:ss";
}
```

### 4.3 DI에 Options 패턴으로 등록하기

설정 클래스를 DI 컨테이너에 등록하면, 어디서든 생성자 주입으로 사용할 수 있습니다.

**먼저 추가 NuGet 패키지가 필요합니다:**

```bash
# Options 패턴을 사용하려면 이 패키지가 필요
dotnet add package Microsoft.Extensions.Options.ConfigurationExtensions
```

```csharp
// App.xaml.cs의 ConfigureServices() 메서드 내

using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;
using MyApp.Models.Settings;

private IServiceProvider ConfigureServices()
{
    var services = new ServiceCollection();

    // ─── 설정 등록 ───
    services.AddSingleton(_configuration);

    // ─── Options 패턴으로 설정 클래스 바인딩 ───

    // Configure<T>(): JSON 섹션을 C# 클래스에 자동 매핑
    // GetSection(): appsettings.json에서 지정된 섹션을 가져옴

    // "Database" 섹션 → DatabaseSettings 클래스
    services.Configure<DatabaseSettings>(
        _configuration.GetSection(DatabaseSettings.SectionName));

    // "Mqtt" 섹션 → MqttSettings 클래스
    services.Configure<MqttSettings>(
        _configuration.GetSection(MqttSettings.SectionName));

    // "Aws" 섹션 → AwsSettings 클래스
    services.Configure<AwsSettings>(
        _configuration.GetSection(AwsSettings.SectionName));

    // "Serilog" 섹션 → SerilogSettings 클래스
    services.Configure<SerilogSettings>(
        _configuration.GetSection(SerilogSettings.SectionName));

    // "AppSettings" 섹션 → AppSettings 클래스
    services.Configure<AppSettings>(
        _configuration.GetSection(AppSettings.SectionName));

    // 연결 문자열은 별도로 처리
    // "ConnectionStrings:DefaultConnection"에서 직접 읽어서 등록
    string connectionString = _configuration.GetConnectionString("DefaultConnection")
        ?? throw new InvalidOperationException(
            "ConnectionStrings:DefaultConnection이 appsettings.json에 설정되지 않았습니다.");

    // ─── 로깅 ───
    services.AddSingleton(Log.Logger);

    // ─── 서비스 등록 (연결 문자열을 직접 전달하는 경우) ───
    services.AddSingleton<IDatabaseService>(provider =>
    {
        // IOptions<DatabaseSettings>에서 설정 값을 가져옴
        var dbSettings = provider.GetRequiredService<IOptions<DatabaseSettings>>().Value;
        var logger = provider.GetRequiredService<Serilog.ILogger>();

        return new DatabaseService(connectionString, dbSettings, logger);
    });

    services.AddTransient<IEmployeeService, EmployeeService>();
    services.AddSingleton<IMqttService, MqttService>();
    services.AddSingleton<IAwsS3Service, AwsS3Service>();

    // ─── ViewModel 등록 ───
    services.AddTransient<MainViewModel>();
    services.AddTransient<EmployeeListViewModel>();

    // ─── View 등록 ───
    services.AddSingleton<MainWindow>();

    return services.BuildServiceProvider();
}
```

### 4.4 서비스에서 설정 사용하기 (IOptions<T>)

등록된 설정은 `IOptions<T>`를 통해 생성자 주입으로 사용합니다.

```csharp
// Services/Implementations/MqttService.cs
// IOptions<MqttSettings>를 주입받아 MQTT 설정을 사용하는 예시

using Microsoft.Extensions.Options;
using MQTTnet;
using MQTTnet.Client;
using MyApp.Models.Settings;
using MyApp.Services.Interfaces;
using Serilog;

namespace MyApp.Services.Implementations;

/// <summary>
/// MQTT 서비스 구현체.
/// IOptions&lt;MqttSettings&gt;를 통해 설정값을 타입 안전하게 사용합니다.
/// </summary>
public class MqttService : IMqttService, IDisposable
{
    // 설정 클래스 (appsettings.json에서 자동 매핑됨)
    private readonly MqttSettings _settings;
    private readonly ILogger _logger;
    private IMqttClient? _client;

    /// <summary>
    /// 생성자 주입: IOptions&lt;MqttSettings&gt;가 자동으로 주입됨.
    /// .Value를 통해 실제 MqttSettings 객체에 접근합니다.
    /// </summary>
    public MqttService(IOptions<MqttSettings> mqttOptions, ILogger logger)
    {
        // IOptions<T>.Value로 실제 설정 객체를 가져옴
        _settings = mqttOptions.Value;
        _logger = logger;
    }

    public async Task ConnectAsync()
    {
        var factory = new MqttClientFactory();
        _client = factory.CreateMqttClient();

        // 설정 클래스의 속성을 직접 사용 (타입 안전!)
        var options = new MqttClientOptionsBuilder()
            .WithTcpServer(_settings.BrokerHost, _settings.BrokerPort)  // JSON에서 읽은 호스트/포트
            .WithClientId(_settings.ClientId)                           // JSON에서 읽은 클라이언트 ID
            .WithKeepAlivePeriod(TimeSpan.FromSeconds(_settings.KeepAliveSeconds))
            .Build();

        _logger.Information(
            "MQTT 브로커에 연결 중... Host={Host}, Port={Port}, ClientId={ClientId}",
            _settings.BrokerHost,
            _settings.BrokerPort,
            _settings.ClientId);

        await _client.ConnectAsync(options);

        // 설정에 정의된 토픽들을 구독
        foreach (var topic in _settings.Topics)
        {
            var subscribeOptions = new MqttClientSubscribeOptionsBuilder()
                .WithTopicFilter(topic)
                .Build();

            await _client.SubscribeAsync(subscribeOptions);
            _logger.Information("MQTT 토픽 구독: {Topic}", topic);
        }
    }

    public void Dispose()
    {
        _client?.Dispose();
    }
}
```

```csharp
// Services/Implementations/DatabaseService.cs
// 연결 문자열과 DatabaseSettings를 함께 사용하는 예시

using MyApp.Models.Settings;
using MyApp.Services.Interfaces;
using Npgsql;
using Serilog;
using System.Data;

namespace MyApp.Services.Implementations;

/// <summary>
/// 데이터베이스 연결을 관리하는 서비스.
/// 연결 문자열과 DatabaseSettings를 조합하여 사용합니다.
/// </summary>
public class DatabaseService : IDatabaseService
{
    private readonly string _connectionString;
    private readonly DatabaseSettings _settings;
    private readonly ILogger _logger;

    /// <summary>
    /// 연결 문자열은 직접 전달, 나머지 설정은 DatabaseSettings에서 가져옴
    /// </summary>
    public DatabaseService(
        string connectionString,
        DatabaseSettings settings,
        ILogger logger)
    {
        _connectionString = connectionString;
        _settings = settings;
        _logger = logger;
    }

    /// <summary>
    /// 새 DB 연결을 생성합니다.
    /// Dapper와 함께 사용됩니다.
    /// </summary>
    public IDbConnection CreateConnection()
    {
        _logger.Debug("새 DB 연결 생성 (Timeout={Timeout}초, MaxPool={MaxPool})",
            _settings.CommandTimeout, _settings.MaxPoolSize);

        // 연결 문자열에 설정값을 추가
        var builder = new NpgsqlConnectionStringBuilder(_connectionString)
        {
            CommandTimeout = _settings.CommandTimeout,
            MaxPoolSize = _settings.MaxPoolSize
        };

        return new NpgsqlConnection(builder.ConnectionString);
    }
}
```

### 4.5 ViewModel에서 설정 사용하기

ViewModel에서도 `IOptions<T>`를 통해 설정에 접근할 수 있습니다.

```csharp
// ViewModels/MainViewModel.cs
// IOptions<AppSettings>를 주입받아 앱 설정을 사용하는 예시

using CommunityToolkit.Mvvm.ComponentModel;
using Microsoft.Extensions.Options;
using MyApp.Models.Settings;
using Serilog;

namespace MyApp.ViewModels;

/// <summary>
/// 메인 윈도우의 ViewModel.
/// 앱 설정을 주입받아 UI에 표시합니다.
/// </summary>
public partial class MainViewModel : ObservableObject
{
    private readonly AppSettings _appSettings;
    private readonly ILogger _logger;

    public MainViewModel(IOptions<AppSettings> appOptions, ILogger logger)
    {
        _appSettings = appOptions.Value;
        _logger = logger;

        // 설정에서 앱 이름과 버전을 가져와 타이틀 설정
        Title = $"{_appSettings.AppName} v{_appSettings.Version}";

        _logger.Information("MainViewModel 초기화: {Title}", Title);
    }

    /// <summary>
    /// 윈도우 제목 (Title Bar에 바인딩)
    /// </summary>
    [ObservableProperty]
    private string _title = string.Empty;

    /// <summary>
    /// 앱 설정의 MinimizeToTray 값을 노출
    /// </summary>
    public bool MinimizeToTray => _appSettings.MinimizeToTray;
}
```

### 4.6 IOptions vs IOptionsSnapshot vs IOptionsMonitor

| 인터페이스 | 동작 | DI 수명 | 설정 변경 감지 | 사용 시기 |
|-----------|------|---------|--------------|-----------|
| `IOptions<T>` | 앱 시작 시 한 번 읽음 | Singleton | 안 됨 | 대부분의 경우 (변경 불필요) |
| `IOptionsSnapshot<T>` | Scope마다 새로 읽음 | Scoped | 됨 (Scope 단위) | ASP.NET (WPF에서는 잘 안 씀) |
| `IOptionsMonitor<T>` | 변경 시 실시간 알림 | Singleton | 됨 (실시간) | 설정 파일이 런타임에 바뀌는 경우 |

```csharp
// IOptionsMonitor 사용 예시 (설정 파일이 런타임에 바뀔 때)
public class SomeService
{
    private readonly IOptionsMonitor<AppSettings> _appOptionsMonitor;

    public SomeService(IOptionsMonitor<AppSettings> appOptionsMonitor)
    {
        _appOptionsMonitor = appOptionsMonitor;

        // 설정이 변경될 때 콜백 실행
        _appOptionsMonitor.OnChange(settings =>
        {
            Console.WriteLine($"설정 변경 감지! 새 앱 이름: {settings.AppName}");
        });
    }

    public void DoSomething()
    {
        // .CurrentValue로 항상 최신 설정에 접근
        var currentSettings = _appOptionsMonitor.CurrentValue;
        Console.WriteLine($"현재 페이지 크기: {currentSettings.PageSize}");
    }
}
```

> **권장**: WPF 데스크탑 앱에서는 보통 `IOptions<T>`로 충분합니다. 설정 파일이 런타임에 바뀌어야 하는 특수한 경우에만 `IOptionsMonitor<T>`를 사용하세요.

---

## 5. 환경별 설정 (appsettings.Development.json)

개발 환경과 운영 환경에서 다른 설정을 사용해야 하는 경우가 많습니다. 예를 들어, 개발 시에는 로컬 DB를 사용하고 운영 시에는 운영 DB를 사용해야 합니다.

### 5.1 환경별 설정 파일 구조

```
MyApp/
├── appsettings.json                 # 기본 설정 (모든 환경 공통)
├── appsettings.Development.json     # 개발 환경 전용 설정 (기본 설정을 덮어씀)
└── appsettings.Production.json      # 운영 환경 전용 설정 (필요 시)
```

### 5.2 appsettings.Development.json 예시

이 파일은 `appsettings.json`의 값을 **덮어쓰기(Override)** 합니다. 정의되지 않은 값은 기본 설정이 그대로 유지됩니다.

```json
{
  // 개발 환경에서만 사용하는 설정
  // appsettings.json의 값을 덮어씀

  "ConnectionStrings": {
    // 개발용 로컬 DB
    "DefaultConnection": "Host=localhost;Port=5432;Database=myapp_dev;Username=dev_user;Password=dev_pass"
  },

  "Mqtt": {
    // 개발용 MQTT 브로커 (로컬)
    "BrokerHost": "localhost",
    "BrokerPort": 1883,
    "ClientId": "MyApp-Dev-001"
  },

  "Serilog": {
    // 개발 시에는 더 상세한 로그
    "MinimumLevel": "Debug"
  },

  "AppSettings": {
    // 개발 중임을 표시
    "AppName": "MyApp [개발 모드]",
    // 개발 시에는 빠른 새로고침
    "AutoRefreshIntervalSeconds": 5
  }
}
```

### 5.3 환경 변수로 환경 지정하기

어떤 환경 설정 파일을 사용할지는 **환경 변수**로 지정합니다.

```bash
# Windows (PowerShell)
$env:DOTNET_ENVIRONMENT="Development"
dotnet run

# Windows (명령 프롬프트)
set DOTNET_ENVIRONMENT=Development
dotnet run

# Linux / Mac
export DOTNET_ENVIRONMENT=Development
dotnet run
```

### 5.4 App.xaml.cs에서 환경별 설정 파일 로드

```csharp
// App.xaml.cs의 BuildConfiguration() 메서드

private static IConfiguration BuildConfiguration()
{
    // 환경 변수에서 현재 환경을 읽음 (기본값: "Production")
    var environment = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT")
        ?? "Production";

    return new ConfigurationBuilder()
        .SetBasePath(Directory.GetCurrentDirectory())
        // 1단계: 기본 설정 파일 로드
        .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
        // 2단계: 환경별 설정 파일 로드 (기본 설정을 덮어씀)
        // 예: DOTNET_ENVIRONMENT=Development → appsettings.Development.json
        .AddJsonFile($"appsettings.{environment}.json", optional: true, reloadOnChange: true)
        .Build();
}
```

### 5.5 설정 병합(Override) 동작 원리

```
설정 로드 순서 (뒤에 로드된 것이 우선):

1. appsettings.json (기본)
   {
     "Serilog": { "MinimumLevel": "Information" },
     "AppSettings": { "AppName": "MyApp 관리 시스템", "PageSize": 50 }
   }

2. appsettings.Development.json (환경별)
   {
     "Serilog": { "MinimumLevel": "Debug" },       ← Information을 덮어씀!
     "AppSettings": { "AppName": "MyApp [개발 모드]" } ← AppName만 덮어씀!
   }

최종 결과:
   {
     "Serilog": { "MinimumLevel": "Debug" },           ← Development에서 덮어씀
     "AppSettings": { "AppName": "MyApp [개발 모드]",   ← Development에서 덮어씀
                      "PageSize": 50 }                  ← 기본값 유지 (Development에 없으므로)
   }
```

> **Python 개발자 참고**: 이것은 Python에서 `.env` 파일이 기본 설정을 덮어쓰는 것과 비슷합니다. `python-dotenv`에서 `override=True`로 설정하는 것과 같은 원리입니다.

---

## 6. 민감 정보 관리 (User Secrets)

### 6.1 왜 필요한가?

`appsettings.json`은 Git에 커밋됩니다. 따라서 **비밀번호, API 키, 액세스 키** 같은 민감 정보를 여기에 넣으면 보안 위험이 생깁니다.

```
⚠️ 절대 하면 안 되는 것:
   appsettings.json에 실제 비밀번호를 넣고 Git에 커밋!

   "ConnectionStrings": {
     "DefaultConnection": "...Password=실제운영비밀번호123..."  ← 위험!
   }
   "Aws": {
     "SecretAccessKey": "실제AWS시크릿키..."  ← 위험!
   }
```

### 6.2 User Secrets란?

.NET의 **User Secrets**는 개발 환경에서 민감 정보를 안전하게 관리하는 도구입니다. 비밀값을 프로젝트 외부(사용자 프로필 폴더)에 저장하므로, Git에 절대 커밋되지 않습니다.

```
저장 위치:
  Windows: %APPDATA%\Microsoft\UserSecrets\<user_secrets_id>\secrets.json
  Linux:   ~/.microsoft/usersecrets/<user_secrets_id>/secrets.json
  Mac:     ~/.microsoft/usersecrets/<user_secrets_id>/secrets.json
```

### 6.3 User Secrets 설정 방법

**1단계: 프로젝트에 User Secrets 초기화**

```bash
# 프로젝트 폴더에서 실행
dotnet user-secrets init
```

이 명령은 `.csproj`에 `UserSecretsId`를 추가합니다:

```xml
<PropertyGroup>
    <!-- dotnet user-secrets init이 자동으로 추가한 ID -->
    <UserSecretsId>a1b2c3d4-e5f6-7890-abcd-ef1234567890</UserSecretsId>
</PropertyGroup>
```

**2단계: 비밀값 설정**

```bash
# DB 비밀번호 설정
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Port=5432;Database=myapp_db;Username=admin;Password=실제비밀번호"

# AWS 키 설정
dotnet user-secrets set "Aws:AccessKeyId" "AKIAIOSFODNN7EXAMPLE"
dotnet user-secrets set "Aws:SecretAccessKey" "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# MQTT 비밀번호 설정
dotnet user-secrets set "Mqtt:Password" "mqtt_secret_password"

# 현재 설정된 비밀값 목록 확인
dotnet user-secrets list
```

**3단계: Configuration에 User Secrets 추가**

추가 NuGet 패키지가 필요합니다:

```bash
dotnet add package Microsoft.Extensions.Configuration.UserSecrets
```

```csharp
// App.xaml.cs의 BuildConfiguration() 수정
using System.Reflection;

private static IConfiguration BuildConfiguration()
{
    var environment = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT")
        ?? "Production";

    var builder = new ConfigurationBuilder()
        .SetBasePath(Directory.GetCurrentDirectory())
        .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
        .AddJsonFile($"appsettings.{environment}.json", optional: true, reloadOnChange: true);

    // 개발 환경에서만 User Secrets 사용
    if (environment == "Development")
    {
        builder.AddUserSecrets(Assembly.GetExecutingAssembly());
    }

    return builder.Build();
}
```

### 6.4 설정 로드 우선순위

```
설정 값 우선순위 (아래가 더 높음):

1. appsettings.json               (기본값)
       ↑ 덮어씀
2. appsettings.Development.json   (환경별)
       ↑ 덮어씀
3. User Secrets                   (개발 환경 비밀 정보)
       ↑ 덮어씀
4. 환경 변수                       (운영 환경에서 주로 사용)

예시:
   appsettings.json:       Password=dev_password_123
   User Secrets:           Password=실제비밀번호456
   → 최종값:               Password=실제비밀번호456  (User Secrets가 우선!)
```

### 6.5 운영 환경에서의 비밀 관리

운영 환경에서는 User Secrets 대신 다음 방법을 사용합니다:

| 방법 | 설명 | 사용 시기 |
|------|------|-----------|
| **환경 변수** | 시스템 환경 변수로 설정 | 간단한 배포 |
| **Azure Key Vault** | 클라우드 비밀 관리 서비스 | Azure 환경 |
| **AWS Secrets Manager** | AWS 비밀 관리 서비스 | AWS 환경 |
| **Docker Secrets** | Docker 컨테이너 비밀 | 컨테이너 배포 |

```bash
# 환경 변수로 설정하는 예시 (운영 서버에서)
# __(언더스코어 두 개)가 JSON의 :(콜론)을 대체
set ConnectionStrings__DefaultConnection=Host=prod-db;Port=5432;Database=myapp_prod;Username=prod_user;Password=운영비밀번호
set Aws__SecretAccessKey=운영AWS시크릿키
```

### 6.6 .gitignore 설정

민감 파일이 Git에 포함되지 않도록 `.gitignore`를 설정합니다.

```gitignore
# .gitignore에 추가

# 환경별 설정 파일 (민감 정보가 포함될 수 있음)
appsettings.Development.json
appsettings.Production.json

# User Secrets는 프로젝트 외부에 저장되므로 자동으로 제외됨
# (별도 설정 불필요)

# 로그 파일
logs/
```

---

## 7. 전체 코드 예제

지금까지의 내용을 종합한 완전한 코드입니다.

### 7.1 프로젝트 구조 (설정 관련 파일)

```
MyApp/
├── Models/
│   └── Settings/                # 설정 클래스 모음
│       ├── AppSettings.cs
│       ├── AwsSettings.cs
│       ├── DatabaseSettings.cs
│       ├── MqttSettings.cs
│       └── SerilogSettings.cs
├── Services/
│   ├── Interfaces/
│   │   └── IDatabaseService.cs
│   └── Implementations/
│       └── DatabaseService.cs
├── ViewModels/
│   └── MainViewModel.cs
├── Views/
│   └── MainWindow.xaml(.cs)
├── App.xaml                     # StartupUri 제거됨
├── App.xaml.cs                  # DI + Configuration 설정
├── appsettings.json             # 기본 설정
├── appsettings.Development.json # 개발 환경 설정
└── MyApp.csproj                 # 프로젝트 파일
```

### 7.2 완성된 App.xaml

```xml
<!-- App.xaml -->
<Application x:Class="MyApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <!-- StartupUri 없음: OnStartup에서 수동으로 윈도우를 생성 -->
    <Application.Resources />
</Application>
```

### 7.3 완성된 App.xaml.cs

```csharp
// App.xaml.cs
// 앱의 완전한 진입점 - Configuration + DI + Serilog 통합

using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;
using MyApp.Models.Settings;
using MyApp.Services.Implementations;
using MyApp.Services.Interfaces;
using MyApp.ViewModels;
using MyApp.Views;
using Serilog;
using Serilog.Events;
using System.IO;
using System.Reflection;
using System.Windows;

namespace MyApp;

public partial class App : Application
{
    private IServiceProvider _serviceProvider = null!;
    private IConfiguration _configuration = null!;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // 1단계: 설정 로드
        _configuration = BuildConfiguration();

        // 2단계: Serilog 초기화 (설정 파일의 값을 사용)
        ConfigureSerilog();

        Log.Information("===== 앱 시작 =====");
        Log.Information("환경: {Environment}",
            Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "Production");

        // 3단계: DI 컨테이너 구성
        _serviceProvider = ConfigureServices();

        // 4단계: 메인 윈도우 표시
        var mainWindow = _serviceProvider.GetRequiredService<MainWindow>();
        mainWindow.Show();

        Log.Information("메인 윈도우가 표시되었습니다.");
    }

    protected override void OnExit(ExitEventArgs e)
    {
        Log.Information("===== 앱 종료 =====");
        Log.CloseAndFlush();

        if (_serviceProvider is IDisposable disposable)
        {
            disposable.Dispose();
        }

        base.OnExit(e);
    }

    /// <summary>
    /// 설정 파일 로드.
    /// 로드 순서: appsettings.json → appsettings.{환경}.json → User Secrets
    /// </summary>
    private static IConfiguration BuildConfiguration()
    {
        var environment = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT")
            ?? "Production";

        var builder = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            // 기본 설정 (필수)
            .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
            // 환경별 설정 (선택, 기본값을 덮어씀)
            .AddJsonFile($"appsettings.{environment}.json", optional: true, reloadOnChange: true);

        // 개발 환경에서만 User Secrets 활성화
        if (environment == "Development")
        {
            builder.AddUserSecrets(Assembly.GetExecutingAssembly());
        }

        return builder.Build();
    }

    /// <summary>
    /// Serilog 설정. appsettings.json의 Serilog 섹션에서 설정값을 읽음.
    /// </summary>
    private void ConfigureSerilog()
    {
        // 설정 파일에서 Serilog 관련 값 읽기
        var serilogSection = _configuration.GetSection(SerilogSettings.SectionName);
        var minimumLevelStr = serilogSection["MinimumLevel"] ?? "Information";
        var logFilePath = serilogSection["LogFilePath"] ?? "logs/myapp-.log";
        var retainedFileCount = int.TryParse(serilogSection["RetainedFileCountLimit"], out var r) ? r : 30;
        var fileSizeLimit = long.TryParse(serilogSection["FileSizeLimitBytes"], out var s) ? s : 10_485_760L;

        // 문자열을 LogEventLevel로 변환
        var minimumLevel = Enum.TryParse<LogEventLevel>(minimumLevelStr, out var level)
            ? level
            : LogEventLevel.Information;

        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Is(minimumLevel)
            .WriteTo.Console(
                outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}")
            .WriteTo.File(
                path: logFilePath,
                rollingInterval: RollingInterval.Day,
                retainedFileCountLimit: retainedFileCount,
                fileSizeLimitBytes: fileSizeLimit,
                outputTemplate: "[{Timestamp:yyyy-MM-dd HH:mm:ss.fff} {Level:u3}] {Message:lj}{NewLine}{Exception}")
            .CreateLogger();
    }

    /// <summary>
    /// DI 서비스 등록.
    /// 모든 서비스, ViewModel, View를 한 곳에서 등록합니다.
    /// </summary>
    private IServiceProvider ConfigureServices()
    {
        var services = new ServiceCollection();

        // ─── 설정 (Configuration) ───
        services.AddSingleton(_configuration);

        // Options 패턴: JSON 섹션 → C# 클래스 매핑
        services.Configure<DatabaseSettings>(
            _configuration.GetSection(DatabaseSettings.SectionName));
        services.Configure<MqttSettings>(
            _configuration.GetSection(MqttSettings.SectionName));
        services.Configure<AwsSettings>(
            _configuration.GetSection(AwsSettings.SectionName));
        services.Configure<SerilogSettings>(
            _configuration.GetSection(SerilogSettings.SectionName));
        services.Configure<AppSettings>(
            _configuration.GetSection(AppSettings.SectionName));

        // 연결 문자열
        string connectionString = _configuration.GetConnectionString("DefaultConnection")
            ?? throw new InvalidOperationException(
                "appsettings.json에 ConnectionStrings:DefaultConnection이 없습니다.");

        // ─── 로깅 (Serilog) ───
        services.AddSingleton(Log.Logger);

        // ─── 서비스 ───
        // DatabaseService: 연결 문자열 + 설정을 팩토리 패턴으로 주입
        services.AddSingleton<IDatabaseService>(provider =>
        {
            var dbSettings = provider.GetRequiredService<IOptions<DatabaseSettings>>().Value;
            var logger = provider.GetRequiredService<Serilog.ILogger>();
            return new DatabaseService(connectionString, dbSettings, logger);
        });

        services.AddTransient<IEmployeeService, EmployeeService>();
        services.AddSingleton<IMqttService, MqttService>();
        services.AddSingleton<IAwsS3Service, AwsS3Service>();

        // ─── ViewModel ───
        services.AddTransient<MainViewModel>();
        services.AddTransient<EmployeeListViewModel>();
        services.AddTransient<EmployeeDetailViewModel>();

        // ─── View ───
        services.AddSingleton<MainWindow>();

        return services.BuildServiceProvider();
    }
}
```

### 7.4 완성된 appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=myapp_db;Username=myapp_user;Password=change_me"
  },

  "Database": {
    "CommandTimeout": 30,
    "MaxPoolSize": 20,
    "RetryCount": 3,
    "RetryDelaySeconds": 2
  },

  "Mqtt": {
    "BrokerHost": "localhost",
    "BrokerPort": 1883,
    "ClientId": "MyApp-Client-001",
    "Username": "",
    "Password": "",
    "Topics": [
      "devices/+/status",
      "devices/+/events",
      "notifications/#"
    ],
    "AutoReconnect": true,
    "ReconnectDelaySeconds": 5,
    "KeepAliveSeconds": 60
  },

  "Aws": {
    "Region": "ap-northeast-2",
    "S3BucketName": "myapp-photos",
    "AccessKeyId": "",
    "SecretAccessKey": "",
    "MaxUploadSizeMb": 10,
    "PresignedUrlExpirationMinutes": 60
  },

  "Serilog": {
    "MinimumLevel": "Information",
    "LogFilePath": "logs/myapp-.log",
    "RetainedFileCountLimit": 30,
    "FileSizeLimitBytes": 10485760
  },

  "AppSettings": {
    "AppName": "MyApp 관리 시스템",
    "Version": "1.0.0",
    "AutoRefreshIntervalSeconds": 30,
    "MinimizeToTray": true,
    "LoadDataOnStartup": true,
    "PageSize": 50,
    "DateFormat": "yyyy-MM-dd",
    "TimeFormat": "HH:mm:ss"
  }
}
```

### 7.5 완성된 appsettings.Development.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=myapp_dev;Username=dev_user;Password=dev_pass"
  },

  "Mqtt": {
    "BrokerHost": "localhost",
    "BrokerPort": 1883,
    "ClientId": "MyApp-Dev-001"
  },

  "Serilog": {
    "MinimumLevel": "Debug"
  },

  "AppSettings": {
    "AppName": "MyApp [개발 모드]",
    "AutoRefreshIntervalSeconds": 5
  }
}
```

### 7.6 설정이 올바르게 로드되었는지 확인하는 방법

```csharp
// ViewModels/MainViewModel.cs (디버깅용 코드)
// 앱 시작 시 현재 설정값을 로그에 출력하여 확인

public MainViewModel(
    IOptions<AppSettings> appOptions,
    IOptions<DatabaseSettings> dbOptions,
    IOptions<MqttSettings> mqttOptions,
    ILogger logger)
{
    var app = appOptions.Value;
    var db = dbOptions.Value;
    var mqtt = mqttOptions.Value;

    // 설정값 확인 로그
    logger.Information("=== 현재 설정 ===");
    logger.Information("앱 이름: {AppName}", app.AppName);
    logger.Information("앱 버전: {Version}", app.Version);
    logger.Information("DB Timeout: {Timeout}초", db.CommandTimeout);
    logger.Information("MQTT 브로커: {Host}:{Port}", mqtt.BrokerHost, mqtt.BrokerPort);
    logger.Information("MQTT 토픽 수: {Count}개", mqtt.Topics.Count);
    logger.Information("=================");

    Title = $"{app.AppName} v{app.Version}";
}
```

실행 결과 (콘솔 로그):

```
[14:30:15 INF] === 현재 설정 ===
[14:30:15 INF] 앱 이름: MyApp [개발 모드]
[14:30:15 INF] 앱 버전: 1.0.0
[14:30:15 INF] DB Timeout: 30초
[14:30:15 INF] MQTT 브로커: localhost:1883
[14:30:15 INF] MQTT 토픽 수: 3개
[14:30:15 INF] =================
```

### 7.7 전체 설정 흐름 요약

```
┌──────────────────────────────────────────────────────────┐
│                    설정 관리 전체 흐름                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  [파일 레벨]                                              │
│                                                          │
│  appsettings.json          (기본 설정, Git에 커밋)         │
│       │                                                  │
│       ▼ (덮어쓰기)                                        │
│  appsettings.Dev.json      (개발 환경, Git 제외 가능)      │
│       │                                                  │
│       ▼ (덮어쓰기)                                        │
│  User Secrets              (비밀번호/키, 프로젝트 외부)     │
│       │                                                  │
│       ▼                                                  │
│  IConfiguration            (최종 병합된 설정 객체)          │
│                                                          │
│  ─────────────────────────────────────────────────────── │
│                                                          │
│  [코드 레벨]                                              │
│                                                          │
│  IConfiguration                                          │
│       │                                                  │
│       ├─► Configure<DatabaseSettings>() ─► IOptions<T>   │
│       ├─► Configure<MqttSettings>()     ─► IOptions<T>   │
│       ├─► Configure<AwsSettings>()      ─► IOptions<T>   │
│       ├─► Configure<SerilogSettings>()  ─► IOptions<T>   │
│       └─► Configure<AppSettings>()      ─► IOptions<T>   │
│                                                          │
│  ─────────────────────────────────────────────────────── │
│                                                          │
│  [사용 레벨]                                              │
│                                                          │
│  ViewModel(IOptions<AppSettings> opts)                    │
│       │                                                  │
│       └─► opts.Value.AppName   ← "MyApp [개발 모드]"      │
│           opts.Value.PageSize  ← 50                      │
│                                                          │
│  Service(IOptions<MqttSettings> opts)                    │
│       │                                                  │
│       └─► opts.Value.BrokerHost ← "localhost"            │
│           opts.Value.BrokerPort ← 1883                   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 다음 단계

프로젝트 구성이 완료되었습니다! 프로젝트 생성, DI 설정, 설정 관리까지 마쳤으므로, 이제 기술 스택을 하나씩 통합하는 단계로 넘어갑니다.

> [다음 문서: PostgreSQL + Dapper 데이터베이스 연동 →](../04-tech-stack-integration/01-database-dapper-npgsql.md)
