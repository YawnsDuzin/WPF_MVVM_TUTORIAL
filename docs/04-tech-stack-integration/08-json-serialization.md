# 8. System.Text.Json을 이용한 JSON 직렬화/역직렬화

> **목표**: System.Text.Json을 사용하여 WPF + MVVM 프로젝트에서 JSON 직렬화/역직렬화를 수행하고, DI를 통한 옵션 공유, 커스텀 컨버터 작성, Source Generator를 활용한 성능 최적화 방법을 익힙니다.

---

## 목차

1. [JSON이란?](#1-json이란)
2. [Newtonsoft.Json vs System.Text.Json 비교](#2-newtonsoftjson-vs-systemtextjson-비교)
3. [System.Text.Json 기본 사용법](#3-systemtextjson-기본-사용법)
4. [자주 쓰는 특성(Attribute)](#4-자주-쓰는-특성attribute)
5. [커스텀 컨버터 작성](#5-커스텀-컨버터-작성)
6. [DI에서 JsonSerializerOptions 공유](#6-di에서-jsonserializeroptions-공유)
7. [MVVM에서 활용](#7-mvvm에서-활용)
8. [Source Generator 모드 (JsonSerializerContext)](#8-source-generator-모드-jsonserializercontext)
9. [전체 코드 예제](#9-전체-코드-예제)

---

## 1. JSON이란?

**JSON (JavaScript Object Notation)**은 텍스트 기반의 데이터 교환 형식입니다.

```json
{
  "name": "홍길동",
  "age": 30,
  "isActive": true,
  "skills": ["C#", "WPF", "MVVM"],
  "address": {
    "city": "서울",
    "zipCode": "12345"
  }
}
```

### Python과의 비교

Python에서 JSON을 다뤄본 경험이 있다면 아래 비교가 도움이 됩니다.

| Python | C# (System.Text.Json) | 설명 |
|--------|----------------------|------|
| `json.dumps(obj)` | `JsonSerializer.Serialize(obj)` | 객체 → JSON 문자열 |
| `json.loads(s)` | `JsonSerializer.Deserialize<T>(s)` | JSON 문자열 → 객체 |
| `json.dump(obj, f)` | `JsonSerializer.Serialize(stream, obj)` | 객체 → 파일에 쓰기 |
| `json.load(f)` | `JsonSerializer.Deserialize<T>(stream)` | 파일에서 읽기 → 객체 |
| `indent=2` | `WriteIndented = true` | 들여쓰기 포맷팅 |
| `dict` | `Dictionary<string, object>` 또는 `class` | 데이터 컨테이너 |

```python
# Python 예제
import json

data = {"name": "홍길동", "age": 30}
json_str = json.dumps(data, ensure_ascii=False, indent=2)
parsed = json.loads(json_str)
```

```csharp
// C# 대응 예제
using System.Text.Json;

var data = new { Name = "홍길동", Age = 30 };
string jsonStr = JsonSerializer.Serialize(data,
    new JsonSerializerOptions { WriteIndented = true });
var parsed = JsonSerializer.Deserialize<Person>(jsonStr);
```

---

## 2. Newtonsoft.Json vs System.Text.Json 비교

.NET 생태계에는 두 가지 주요 JSON 라이브러리가 있습니다.

| 항목 | Newtonsoft.Json | System.Text.Json |
|------|----------------|------------------|
| **설치** | NuGet 패키지 설치 필요 | .NET 내장 (추가 설치 불필요) |
| **성능** | 상대적으로 느림 | 2~5배 빠름 (벤치마크 기준) |
| **메모리** | 더 많이 사용 | 적은 메모리 사용 |
| **기능** | 매우 풍부 (모든 시나리오 지원) | 대부분의 시나리오 지원 |
| **유연성** | 동적 타입, 복잡한 변환 쉬움 | 정적 타입 중심, 제한적 동적 타입 |
| **Source Generator** | 미지원 | 지원 (AOT 호환, 최고 성능) |
| **유지보수** | 개인 프로젝트 (James Newton-King) | Microsoft 공식 지원 |
| **AOT 호환** | 미지원 | 완벽 지원 |

> **권장**: 새 프로젝트에서는 **System.Text.Json**을 사용하세요. .NET 10.0에 내장되어 있어 별도 설치가 필요 없고, 성능이 우수하며, Source Generator로 AOT(Ahead-of-Time) 컴파일도 지원합니다. Newtonsoft.Json은 레거시 프로젝트 유지보수 시에만 사용하는 것이 좋습니다.

---

## 3. System.Text.Json 기본 사용법

### 3-1. Serialize / Deserialize

```csharp
// 기본 사용법 예제
using System.Text.Json;

namespace MyApp.Examples;

/// <summary>
/// JSON 직렬화/역직렬화 기본 예제입니다.
/// </summary>
public static class JsonBasicExample
{
    /// <summary>
    /// 직렬화: C# 객체 → JSON 문자열
    /// </summary>
    public static void SerializeExample()
    {
        // 직렬화할 객체 생성
        var employee = new Employee
        {
            Id = 1,
            Name = "홍길동",
            Department = "개발팀",
            HireDate = new DateTime(2023, 3, 15),
            IsActive = true,
            Skills = ["C#", "WPF", "MVVM"]
        };

        // 객체 → JSON 문자열
        string json = JsonSerializer.Serialize(employee);
        // 결과: {"Id":1,"Name":"홍길동","Department":"개발팀",...}

        // 보기 좋게 들여쓰기
        string prettyJson = JsonSerializer.Serialize(employee,
            new JsonSerializerOptions { WriteIndented = true });
        // 결과:
        // {
        //   "Id": 1,
        //   "Name": "홍길동",
        //   ...
        // }
    }

    /// <summary>
    /// 역직렬화: JSON 문자열 → C# 객체
    /// </summary>
    public static void DeserializeExample()
    {
        string json = """
            {
                "Id": 1,
                "Name": "홍길동",
                "Department": "개발팀",
                "HireDate": "2023-03-15T00:00:00",
                "IsActive": true,
                "Skills": ["C#", "WPF", "MVVM"]
            }
            """;

        // JSON 문자열 → C# 객체
        // 제네릭 타입 파라미터로 대상 타입을 지정합니다.
        Employee? employee = JsonSerializer.Deserialize<Employee>(json);

        if (employee is not null)
        {
            Console.WriteLine($"이름: {employee.Name}");
            Console.WriteLine($"부서: {employee.Department}");
        }
    }
}

/// <summary>직원 데이터 모델</summary>
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Department { get; set; } = string.Empty;
    public DateTime HireDate { get; set; }
    public bool IsActive { get; set; }
    public List<string> Skills { get; set; } = [];
}
```

### 3-2. JsonSerializerOptions 설정

```csharp
// JsonSerializerOptions는 직렬화/역직렬화의 동작을 세밀하게 제어합니다.
using System.Text.Encodings.Web;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace MyApp.Configuration;

/// <summary>
/// JsonSerializerOptions의 주요 설정을 보여주는 예제입니다.
/// </summary>
public static class JsonOptionsExample
{
    /// <summary>
    /// 앱 전체에서 사용할 표준 JSON 옵션을 생성합니다.
    /// </summary>
    public static JsonSerializerOptions CreateStandardOptions()
    {
        return new JsonSerializerOptions
        {
            // ──────────────────────────────────
            // 속성 이름 정책
            // ──────────────────────────────────

            // CamelCase: C#의 PascalCase를 camelCase로 변환
            // 예: "FirstName" → "firstName"
            // REST API와 통신할 때 가장 많이 사용됩니다.
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,

            // ──────────────────────────────────
            // 역직렬화 시 속성 이름 대소문자 무시
            // ──────────────────────────────────

            // true로 설정하면 "name", "Name", "NAME" 모두 매칭됩니다.
            // API 서버가 일관되지 않은 케이싱을 사용할 때 유용합니다.
            PropertyNameCaseInsensitive = true,

            // ──────────────────────────────────
            // 들여쓰기 (가독성)
            // ──────────────────────────────────

            // true: 들여쓰기 포함 (디버깅/파일 저장용)
            // false: 한 줄로 출력 (API 전송용, 크기 절약)
            WriteIndented = true,

            // ──────────────────────────────────
            // null 값 처리
            // ──────────────────────────────────

            // WhenWritingNull: null인 속성은 JSON에 포함하지 않음
            // WhenWritingDefault: 기본값인 속성도 포함하지 않음
            // Never: 항상 포함 (기본값)
            DefaultIgnoreCondition =
                JsonIgnoreCondition.WhenWritingNull,

            // ──────────────────────────────────
            // Enum을 문자열로 직렬화
            // ──────────────────────────────────

            Converters =
            {
                // 숫자 대신 이름으로: 0 → "Active"
                new JsonStringEnumConverter(JsonNamingPolicy.CamelCase)
            },

            // ──────────────────────────────────
            // 인코딩 설정
            // ──────────────────────────────────

            // 한국어 등 유니코드 문자를 이스케이프하지 않고 그대로 출력
            // 기본값은 ASCII가 아닌 문자를 \uXXXX로 이스케이프합니다.
            Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping,

            // ──────────────────────────────────
            // 숫자를 문자열에서 읽기 허용
            // ──────────────────────────────────

            // "123" (문자열)을 int 123으로 역직렬화 허용
            NumberHandling =
                JsonNumberHandling.AllowReadingFromString,

            // ──────────────────────────────────
            // 주석 및 후행 쉼표 허용
            // ──────────────────────────────────

            // JSON에 주석이 포함된 경우 (예: jsonc 파일)
            ReadCommentHandling = JsonCommentHandling.Skip,

            // 마지막 요소 뒤의 쉼표 허용 (예: ["a", "b",])
            AllowTrailingCommas = true,

            // ──────────────────────────────────
            // 순환 참조 처리
            // ──────────────────────────────────

            // 객체 간 순환 참조가 있을 때 처리 방법
            ReferenceHandler = ReferenceHandler.IgnoreCycles
        };
    }
}
```

### PropertyNamingPolicy 종류

| 정책 | C# 속성 | JSON 키 | 용도 |
|------|---------|---------|------|
| `null` (기본값) | `FirstName` | `"FirstName"` | PascalCase 유지 |
| `CamelCase` | `FirstName` | `"firstName"` | REST API 표준 |
| `SnakeCaseLower` | `FirstName` | `"first_name"` | Python/Ruby API |
| `SnakeCaseUpper` | `FirstName` | `"FIRST_NAME"` | 특수 케이스 |
| `KebabCaseLower` | `FirstName` | `"first-name"` | HTTP 헤더 스타일 |

```csharp
// SnakeCase 예제 (Python API와 통신할 때)
var snakeOptions = new JsonSerializerOptions
{
    // .NET 8+에서 지원
    PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower
};

var employee = new Employee { FirstName = "길동", LastName = "홍" };
string json = JsonSerializer.Serialize(employee, snakeOptions);
// 결과: {"first_name":"길동","last_name":"홍"}
```

---

## 4. 자주 쓰는 특성(Attribute)

특성(Attribute)을 사용하면 클래스 단위로 직렬화 동작을 제어할 수 있습니다.

### 4-1. [JsonPropertyName] - 속성 이름 지정

```csharp
using System.Text.Json.Serialization;

/// <summary>
/// API 응답 모델입니다.
/// 서버의 JSON 키 이름과 C# 속성 이름이 다를 때
/// [JsonPropertyName]으로 매핑합니다.
/// </summary>
public class ApiResponse
{
    // JSON: "result_code" ↔ C#: ResultCode
    [JsonPropertyName("result_code")]
    public int ResultCode { get; set; }

    // JSON: "result_message" ↔ C#: ResultMessage
    [JsonPropertyName("result_message")]
    public string ResultMessage { get; set; } = string.Empty;

    // JSON: "data" ↔ C#: Data
    [JsonPropertyName("data")]
    public object? Data { get; set; }

    // JSON: "total_count" ↔ C#: TotalCount
    [JsonPropertyName("total_count")]
    public int TotalCount { get; set; }
}
```

> **참고**: `PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower`를 설정하면 `[JsonPropertyName]`을 하나하나 붙일 필요가 없습니다. 전역 정책과 개별 속성 지정을 적절히 조합하세요.

### 4-2. [JsonIgnore] - 속성 제외

```csharp
/// <summary>
/// 사용자 모델입니다.
/// 민감한 정보나 불필요한 속성을 직렬화에서 제외합니다.
/// </summary>
public class User
{
    public int Id { get; set; }

    public string Username { get; set; } = string.Empty;

    // 비밀번호는 절대 JSON에 포함하지 않음
    [JsonIgnore]
    public string PasswordHash { get; set; } = string.Empty;

    // 조건부 제외: null일 때만 제외
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public string? Email { get; set; }

    // 조건부 제외: 기본값(0, false, null 등)일 때 제외
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingDefault)]
    public int LoginCount { get; set; }

    // 내부 관리용 속성은 직렬화에서 제외
    [JsonIgnore]
    public DateTime LastAccessed { get; set; }
}
```

### 4-3. [JsonConverter] - 커스텀 변환기 지정

```csharp
/// <summary>
/// 특정 속성에만 커스텀 컨버터를 적용합니다.
/// </summary>
public class EventLog
{
    public int Id { get; set; }

    public string Message { get; set; } = string.Empty;

    // 이 속성만 커스텀 DateTime 포맷을 사용
    [JsonConverter(typeof(DateTimeFormatConverter))]
    public DateTime CreatedAt { get; set; }

    // 일반 DateTime은 기본 ISO 8601 포맷 사용
    public DateTime? UpdatedAt { get; set; }
}
```

### 4-4. [JsonConstructor] - 역직렬화에 사용할 생성자 지정

```csharp
/// <summary>
/// 불변(immutable) 객체를 역직렬화할 때
/// [JsonConstructor]로 사용할 생성자를 지정합니다.
/// </summary>
public class DeviceInfo
{
    // 읽기 전용 속성들 (init 또는 생성자에서만 설정 가능)
    public string DeviceId { get; }
    public string DeviceName { get; }
    public string IpAddress { get; }

    // 이 생성자를 역직렬화에 사용
    // 파라미터 이름이 JSON 속성 이름과 매칭됩니다 (대소문자 무시)
    [JsonConstructor]
    public DeviceInfo(string deviceId, string deviceName, string ipAddress)
    {
        DeviceId = deviceId;
        DeviceName = deviceName;
        IpAddress = ipAddress;
    }
}
```

### 특성 요약

| 특성 | 용도 | 예시 |
|------|------|------|
| `[JsonPropertyName("name")]` | JSON 키 이름 변경 | API 응답 필드 매핑 |
| `[JsonIgnore]` | 직렬화에서 제외 | 비밀번호, 내부 관리용 속성 |
| `[JsonConverter(typeof(T))]` | 커스텀 변환기 적용 | 특수 날짜 포맷 |
| `[JsonConstructor]` | 역직렬화 생성자 지정 | 불변 객체 |
| `[JsonInclude]` | private 또는 internal 속성을 직렬화에 포함 | 특수한 경우 |
| `[JsonRequired]` | 필수 속성 (없으면 예외) | 필수 필드 검증 |
| `[JsonNumberHandling(...)]` | 숫자↔문자열 변환 허용 | 유연한 API 호환 |

---

## 5. 커스텀 컨버터 작성

기본 제공되는 변환 동작이 맞지 않을 때 커스텀 컨버터를 작성합니다.

### 5-1. DateTime 포맷 변환 컨버터

```csharp
// Converters/DateTimeFormatConverter.cs
using System.Globalization;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace MyApp.Converters;

/// <summary>
/// DateTime을 "yyyy-MM-dd HH:mm:ss" 포맷으로 직렬화/역직렬화하는
/// 커스텀 컨버터입니다.
///
/// 기본 포맷(ISO 8601: "2024-01-15T14:30:00")이 아닌
/// 한국식 포맷("2024-01-15 14:30:00")을 사용합니다.
/// </summary>
public sealed class DateTimeFormatConverter : JsonConverter<DateTime>
{
    // 사용할 날짜/시간 포맷
    private const string Format = "yyyy-MM-dd HH:mm:ss";

    /// <summary>
    /// JSON 문자열 → DateTime (역직렬화)
    /// </summary>
    public override DateTime Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options)
    {
        // JSON에서 문자열 값 읽기
        var dateString = reader.GetString();

        if (string.IsNullOrEmpty(dateString))
        {
            return default; // DateTime.MinValue
        }

        // 지정된 포맷으로 파싱
        if (DateTime.TryParseExact(
                dateString,
                Format,
                CultureInfo.InvariantCulture,
                DateTimeStyles.None,
                out var result))
        {
            return result;
        }

        // 지정 포맷이 아니면 일반 파싱 시도
        return DateTime.Parse(dateString, CultureInfo.InvariantCulture);
    }

    /// <summary>
    /// DateTime → JSON 문자열 (직렬화)
    /// </summary>
    public override void Write(
        Utf8JsonWriter writer,
        DateTime value,
        JsonSerializerOptions options)
    {
        // 지정된 포맷으로 문자열 변환 후 JSON에 쓰기
        writer.WriteStringValue(value.ToString(Format));
    }
}
```

### 5-2. Nullable DateTime 컨버터

```csharp
// Converters/NullableDateTimeFormatConverter.cs
using System.Globalization;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace MyApp.Converters;

/// <summary>
/// DateTime?를 처리하는 커스텀 컨버터입니다.
/// null이면 JSON null로, 값이 있으면 지정된 포맷으로 변환합니다.
/// </summary>
public sealed class NullableDateTimeFormatConverter
    : JsonConverter<DateTime?>
{
    private const string Format = "yyyy-MM-dd HH:mm:ss";

    public override DateTime? Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options)
    {
        // JSON null 처리
        if (reader.TokenType == JsonTokenType.Null)
            return null;

        var dateString = reader.GetString();
        if (string.IsNullOrEmpty(dateString))
            return null;

        if (DateTime.TryParseExact(
                dateString, Format,
                CultureInfo.InvariantCulture,
                DateTimeStyles.None,
                out var result))
        {
            return result;
        }

        return DateTime.Parse(dateString, CultureInfo.InvariantCulture);
    }

    public override void Write(
        Utf8JsonWriter writer,
        DateTime? value,
        JsonSerializerOptions options)
    {
        if (value.HasValue)
        {
            writer.WriteStringValue(value.Value.ToString(Format));
        }
        else
        {
            writer.WriteNullValue();
        }
    }
}
```

### 5-3. 범용 Enum+Description 컨버터 (고급)

```csharp
// Converters/EnumDescriptionConverter.cs
using System.ComponentModel;
using System.Reflection;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace MyApp.Converters;

/// <summary>
/// Enum을 [Description] 특성의 값으로 직렬화하는 컨버터입니다.
///
/// 예: EventStatus.InProgress → "진행 중"
///     (JsonStringEnumConverter는 "InProgress"로 변환)
/// </summary>
public sealed class EnumDescriptionConverter<TEnum>
    : JsonConverter<TEnum> where TEnum : struct, Enum
{
    public override TEnum Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options)
    {
        var description = reader.GetString();
        if (string.IsNullOrEmpty(description))
            return default;

        // Description → Enum 값 매핑
        foreach (var field in typeof(TEnum).GetFields(
            BindingFlags.Public | BindingFlags.Static))
        {
            var attr = field.GetCustomAttribute<DescriptionAttribute>();
            if (attr?.Description == description)
            {
                return (TEnum)field.GetValue(null)!;
            }

            // Description이 없으면 필드 이름으로 매칭
            if (field.Name.Equals(
                description, StringComparison.OrdinalIgnoreCase))
            {
                return (TEnum)field.GetValue(null)!;
            }
        }

        return default;
    }

    public override void Write(
        Utf8JsonWriter writer,
        TEnum value,
        JsonSerializerOptions options)
    {
        // Enum 값의 [Description] 특성 읽기
        var field = typeof(TEnum).GetField(value.ToString());
        var attr = field?.GetCustomAttribute<DescriptionAttribute>();

        // Description이 있으면 그 값을, 없으면 Enum 이름 사용
        writer.WriteStringValue(attr?.Description ?? value.ToString());
    }
}

/// <summary>
/// 사용 예제 Enum
/// </summary>
public enum EventStatus
{
    [Description("대기")]
    Pending,

    [Description("진행 중")]
    InProgress,

    [Description("완료")]
    Completed,

    [Description("실패")]
    Failed
}
```

---

## 6. DI에서 JsonSerializerOptions 공유

`JsonSerializerOptions`를 매번 새로 생성하면 성능이 저하됩니다. **싱글톤으로 등록하여 앱 전체에서 공유**하는 것이 좋습니다.

### 6-1. 옵션 구성 및 DI 등록

```csharp
// App.xaml.cs (DI 설정)
using System.Text.Encodings.Web;
using System.Text.Json;
using System.Text.Json.Serialization;
using Microsoft.Extensions.DependencyInjection;
using MyApp.Converters;

namespace MyApp;

public partial class App : Application
{
    private ServiceProvider? _serviceProvider;

    public App()
    {
        var services = new ServiceCollection();

        // ──────────────────────────────────────────────
        // JsonSerializerOptions를 싱글톤으로 등록
        // ──────────────────────────────────────────────
        services.AddSingleton(_ =>
        {
            var options = new JsonSerializerOptions
            {
                // 속성 이름: camelCase (API 통신 표준)
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase,

                // 역직렬화 시 대소문자 무시
                PropertyNameCaseInsensitive = true,

                // null 속성은 JSON에 포함하지 않음
                DefaultIgnoreCondition =
                    JsonIgnoreCondition.WhenWritingNull,

                // 한국어를 유니코드 이스케이프하지 않음
                Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping,

                // 숫자를 문자열에서도 읽기 허용
                NumberHandling =
                    JsonNumberHandling.AllowReadingFromString,

                // 순환 참조 무시
                ReferenceHandler = ReferenceHandler.IgnoreCycles,

                // 커스텀 컨버터 등록
                Converters =
                {
                    // Enum을 문자열로 변환
                    new JsonStringEnumConverter(JsonNamingPolicy.CamelCase),
                    // DateTime 커스텀 포맷
                    new DateTimeFormatConverter(),
                    new NullableDateTimeFormatConverter()
                }
            };

            return options;
        });

        // 서비스 등록
        services.AddSingleton<IJsonFileService, JsonFileService>();
        services.AddTransient<SettingsViewModel>();
        services.AddLogging();

        _serviceProvider = services.BuildServiceProvider();
    }
}
```

### 6-2. 왜 싱글톤인가?

```csharp
// ❌ 잘못된 패턴: 매번 새 옵션 생성
public string Serialize<T>(T obj)
{
    // 매번 new JsonSerializerOptions()를 생성하면
    // 내부 캐시가 공유되지 않아 성능이 크게 저하됩니다!
    var options = new JsonSerializerOptions
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    };
    return JsonSerializer.Serialize(obj, options);
}

// ✅ 올바른 패턴: 싱글톤 옵션 재사용
public class MyService
{
    private readonly JsonSerializerOptions _jsonOptions;

    // DI를 통해 싱글톤 옵션을 주입받음
    public MyService(JsonSerializerOptions jsonOptions)
    {
        _jsonOptions = jsonOptions;
    }

    public string Serialize<T>(T obj)
    {
        // 캐시된 옵션을 재사용하여 최적의 성능
        return JsonSerializer.Serialize(obj, _jsonOptions);
    }
}
```

> **성능 참고**: `JsonSerializerOptions`는 내부적으로 타입별 메타데이터를 캐싱합니다. 같은 인스턴스를 재사용하면 두 번째 직렬화부터 캐시 히트로 훨씬 빠릅니다.

---

## 7. MVVM에서 활용

### 7-1. API 응답 역직렬화

```csharp
// Services/ApiService.cs
using System.Net.Http;
using System.Text.Json;
using Microsoft.Extensions.Logging;

namespace MyApp.Services;

/// <summary>
/// REST API 호출 서비스입니다.
/// HttpClient와 System.Text.Json을 조합하여 API 응답을 역직렬화합니다.
/// </summary>
public sealed class ApiService
{
    private readonly HttpClient _httpClient;
    private readonly JsonSerializerOptions _jsonOptions;
    private readonly ILogger<ApiService> _logger;

    public ApiService(
        HttpClient httpClient,
        JsonSerializerOptions jsonOptions,
        ILogger<ApiService> logger)
    {
        _httpClient = httpClient;
        _jsonOptions = jsonOptions;
        _logger = logger;
    }

    /// <summary>
    /// GET 요청을 보내고 응답을 역직렬화합니다.
    /// </summary>
    public async Task<T?> GetAsync<T>(
        string url, CancellationToken ct = default)
    {
        _logger.LogDebug("API GET: {Url}", url);

        // HttpClient로 GET 요청
        using var response = await _httpClient.GetAsync(url, ct);
        response.EnsureSuccessStatusCode();

        // 응답 본문을 스트림으로 읽어서 역직렬화 (메모리 효율적)
        await using var stream = await response.Content
            .ReadAsStreamAsync(ct);

        // 스트림에서 직접 역직렬화 (문자열 변환 단계 생략)
        var result = await JsonSerializer
            .DeserializeAsync<T>(stream, _jsonOptions, ct);

        return result;
    }

    /// <summary>
    /// POST 요청을 보내고 응답을 역직렬화합니다.
    /// </summary>
    public async Task<TResponse?> PostAsync<TRequest, TResponse>(
        string url, TRequest data, CancellationToken ct = default)
    {
        _logger.LogDebug("API POST: {Url}", url);

        // 요청 본문 직렬화
        var jsonContent = JsonSerializer.Serialize(data, _jsonOptions);
        using var content = new StringContent(
            jsonContent, System.Text.Encoding.UTF8, "application/json");

        // POST 요청
        using var response = await _httpClient.PostAsync(url, content, ct);
        response.EnsureSuccessStatusCode();

        // 응답 역직렬화
        await using var stream = await response.Content
            .ReadAsStreamAsync(ct);
        return await JsonSerializer
            .DeserializeAsync<TResponse>(stream, _jsonOptions, ct);
    }
}
```

### 7-2. 로컬 파일 저장/로드 (사용자 설정 등)

```csharp
// Services/IJsonFileService.cs
namespace MyApp.Services;

/// <summary>
/// JSON 파일 저장/로드 서비스 인터페이스입니다.
/// 사용자 설정, 캐시 데이터 등을 로컬 JSON 파일로 관리합니다.
/// </summary>
public interface IJsonFileService
{
    /// <summary>객체를 JSON 파일로 저장합니다.</summary>
    Task SaveAsync<T>(string filePath, T data, CancellationToken ct = default);

    /// <summary>JSON 파일을 읽어 객체로 역직렬화합니다.</summary>
    Task<T?> LoadAsync<T>(string filePath, CancellationToken ct = default);
}
```

```csharp
// Services/JsonFileService.cs
using System.IO;
using System.Text.Json;
using Microsoft.Extensions.Logging;

namespace MyApp.Services;

/// <summary>
/// JSON 파일 저장/로드 서비스 구현입니다.
/// </summary>
public sealed class JsonFileService : IJsonFileService
{
    private readonly JsonSerializerOptions _jsonOptions;
    private readonly ILogger<JsonFileService> _logger;

    // 파일 저장용 옵션 (들여쓰기 포함)
    private readonly JsonSerializerOptions _writeOptions;

    public JsonFileService(
        JsonSerializerOptions jsonOptions,
        ILogger<JsonFileService> logger)
    {
        _jsonOptions = jsonOptions;
        _logger = logger;

        // 파일 저장 시에는 들여쓰기를 포함합니다.
        // 기존 옵션을 복사한 후 WriteIndented만 변경합니다.
        _writeOptions = new JsonSerializerOptions(_jsonOptions)
        {
            WriteIndented = true
        };
    }

    /// <inheritdoc />
    public async Task SaveAsync<T>(
        string filePath, T data, CancellationToken ct = default)
    {
        // 디렉터리가 없으면 생성
        var directory = Path.GetDirectoryName(filePath);
        if (!string.IsNullOrEmpty(directory))
        {
            Directory.CreateDirectory(directory);
        }

        // 파일 스트림으로 직접 직렬화 (메모리 효율적)
        await using var stream = File.Create(filePath);
        await JsonSerializer.SerializeAsync(
            stream, data, _writeOptions, ct);

        _logger.LogDebug("JSON 파일 저장 완료: {Path}", filePath);
    }

    /// <inheritdoc />
    public async Task<T?> LoadAsync<T>(
        string filePath, CancellationToken ct = default)
    {
        if (!File.Exists(filePath))
        {
            _logger.LogDebug("JSON 파일 없음: {Path}", filePath);
            return default;
        }

        await using var stream = File.OpenRead(filePath);
        var result = await JsonSerializer
            .DeserializeAsync<T>(stream, _jsonOptions, ct);

        _logger.LogDebug("JSON 파일 로드 완료: {Path}", filePath);
        return result;
    }
}
```

### 7-3. SettingsViewModel 예제

```csharp
// ViewModels/SettingsViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.Logging;
using MyApp.Services;

namespace MyApp.ViewModels;

/// <summary>
/// 사용자 설정을 JSON 파일로 저장/로드하는 ViewModel입니다.
/// </summary>
public partial class SettingsViewModel : ObservableObject
{
    private readonly IJsonFileService _jsonFileService;
    private readonly ILogger<SettingsViewModel> _logger;

    // 설정 파일 경로
    private static readonly string SettingsPath = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
        "MyApp",
        "settings.json");

    public SettingsViewModel(
        IJsonFileService jsonFileService,
        ILogger<SettingsViewModel> logger)
    {
        _jsonFileService = jsonFileService;
        _logger = logger;
    }

    // ──────────────────────────────────────────────
    // 설정 속성들
    // ──────────────────────────────────────────────

    [ObservableProperty]
    private string _serverUrl = "https://api.example.com";

    [ObservableProperty]
    private int _refreshInterval = 30;

    [ObservableProperty]
    private bool _isDarkMode;

    [ObservableProperty]
    private string _language = "ko-KR";

    [ObservableProperty]
    private string _statusMessage = string.Empty;

    // ──────────────────────────────────────────────
    // 커맨드
    // ──────────────────────────────────────────────

    /// <summary>
    /// 설정을 JSON 파일로 저장합니다.
    /// </summary>
    [RelayCommand]
    private async Task SaveSettingsAsync(CancellationToken ct)
    {
        try
        {
            // 현재 설정을 DTO 객체로 변환
            var settings = new AppSettings
            {
                ServerUrl = ServerUrl,
                RefreshInterval = RefreshInterval,
                IsDarkMode = IsDarkMode,
                Language = Language,
                LastSaved = DateTime.Now
            };

            // JSON 파일로 저장
            await _jsonFileService.SaveAsync(SettingsPath, settings, ct);

            StatusMessage = $"설정이 저장되었습니다. ({DateTime.Now:HH:mm:ss})";
            _logger.LogInformation("설정 저장 완료: {Path}", SettingsPath);
        }
        catch (Exception ex)
        {
            StatusMessage = $"저장 실패: {ex.Message}";
            _logger.LogError(ex, "설정 저장 실패");
        }
    }

    /// <summary>
    /// JSON 파일에서 설정을 로드합니다.
    /// </summary>
    [RelayCommand]
    private async Task LoadSettingsAsync(CancellationToken ct)
    {
        try
        {
            var settings = await _jsonFileService
                .LoadAsync<AppSettings>(SettingsPath, ct);

            if (settings is null)
            {
                StatusMessage = "저장된 설정이 없습니다. 기본값을 사용합니다.";
                return;
            }

            // DTO의 값을 ViewModel 속성에 반영
            ServerUrl = settings.ServerUrl;
            RefreshInterval = settings.RefreshInterval;
            IsDarkMode = settings.IsDarkMode;
            Language = settings.Language;

            StatusMessage = $"설정 로드 완료. " +
                            $"(마지막 저장: {settings.LastSaved:yyyy-MM-dd HH:mm:ss})";
        }
        catch (Exception ex)
        {
            StatusMessage = $"로드 실패: {ex.Message}";
            _logger.LogError(ex, "설정 로드 실패");
        }
    }
}

/// <summary>
/// 앱 설정 DTO (Data Transfer Object)입니다.
/// JSON 파일에 저장되는 데이터 구조입니다.
/// </summary>
public sealed class AppSettings
{
    public string ServerUrl { get; set; } = "https://api.example.com";
    public int RefreshInterval { get; set; } = 30;
    public bool IsDarkMode { get; set; }
    public string Language { get; set; } = "ko-KR";
    public DateTime LastSaved { get; set; }
}
```

### 7-4. MQTT 메시지 직렬화/역직렬화

```csharp
// MQTT 메시지 처리 예제
// (04-mqtt-mqttnet.md의 MqttService와 함께 사용)
using System.Text;
using System.Text.Json;

namespace MyApp.Services;

/// <summary>
/// MQTT 메시지를 JSON으로 직렬화/역직렬화하는 헬퍼입니다.
/// </summary>
public sealed class MqttMessageHelper
{
    private readonly JsonSerializerOptions _jsonOptions;

    public MqttMessageHelper(JsonSerializerOptions jsonOptions)
    {
        _jsonOptions = jsonOptions;
    }

    /// <summary>
    /// 객체를 MQTT 메시지 페이로드(바이트 배열)로 변환합니다.
    /// </summary>
    public byte[] SerializeToPayload<T>(T message)
    {
        // 객체 → JSON 문자열 → UTF-8 바이트 배열
        var json = JsonSerializer.Serialize(message, _jsonOptions);
        return Encoding.UTF8.GetBytes(json);

        // 또는 더 효율적인 방법:
        // return JsonSerializer.SerializeToUtf8Bytes(message, _jsonOptions);
    }

    /// <summary>
    /// MQTT 메시지 페이로드(바이트 배열)를 객체로 변환합니다.
    /// </summary>
    public T? DeserializeFromPayload<T>(byte[] payload)
    {
        // UTF-8 바이트 배열에서 직접 역직렬화 (문자열 변환 단계 생략)
        return JsonSerializer.Deserialize<T>(payload, _jsonOptions);
    }
}

// 사용 예제:
// var helper = new MqttMessageHelper(jsonOptions);
//
// // 발행: 객체 → 바이트 배열
// var sensorData = new SensorData { Temperature = 25.5, Humidity = 60.0 };
// byte[] payload = helper.SerializeToPayload(sensorData);
// await mqttClient.PublishAsync("sensor/data", payload);
//
// // 수신: 바이트 배열 → 객체
// var received = helper.DeserializeFromPayload<SensorData>(e.ApplicationMessage.Payload);
```

---

## 8. Source Generator 모드 (JsonSerializerContext)

### Source Generator란?

**Source Generator**는 컴파일 시점에 직렬화/역직렬화 코드를 자동 생성하는 기능입니다.

| 항목 | 리플렉션 모드 (기본) | Source Generator 모드 |
|------|---------------------|----------------------|
| **코드 생성 시점** | 런타임 (실행 중) | 컴파일 타임 |
| **성능** | 보통 | 매우 빠름 (리플렉션 제거) |
| **메모리** | 더 많이 사용 | 적게 사용 |
| **AOT 호환** | 미지원 (Trimming 문제) | 완벽 지원 |
| **설정 복잡도** | 간단 | 약간의 추가 설정 필요 |

### 8-1. JsonSerializerContext 정의

```csharp
// Serialization/AppJsonContext.cs
using System.Text.Json.Serialization;
using MyApp.Models;

namespace MyApp.Serialization;

/// <summary>
/// 앱 전체에서 사용하는 JSON Source Generator 컨텍스트입니다.
///
/// [JsonSerializable] 특성으로 직렬화 대상 타입을 등록하면,
/// 컴파일러가 해당 타입의 직렬화/역직렬화 코드를 미리 생성합니다.
///
/// 이점:
/// - 런타임 리플렉션 제거 → 성능 향상
/// - AOT(Ahead-of-Time) 컴파일 호환
/// - 앱 시작 시간 단축
/// </summary>
[JsonSourceGenerationOptions(
    // 옵션을 여기에서도 설정할 수 있습니다.
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    WriteIndented = false,
    GenerationMode = JsonSourceGenerationMode.Default)]

// 직렬화할 타입들을 모두 등록합니다.
[JsonSerializable(typeof(Employee))]
[JsonSerializable(typeof(List<Employee>))]
[JsonSerializable(typeof(AppSettings))]
[JsonSerializable(typeof(ApiResponse))]
[JsonSerializable(typeof(DeviceInfo))]
[JsonSerializable(typeof(FaceRecognitionEvent))]
[JsonSerializable(typeof(SensorData))]
[JsonSerializable(typeof(Dictionary<string, string>))]
public partial class AppJsonContext : JsonSerializerContext
{
    // 컴파일러가 이 클래스의 나머지를 자동 생성합니다.
    // 'partial' 키워드가 반드시 필요합니다.
}

/// <summary>센서 데이터 (MQTT 예제용)</summary>
public class SensorData
{
    public double Temperature { get; set; }
    public double Humidity { get; set; }
    public DateTime Timestamp { get; set; }
}

/// <summary>얼굴인식 이벤트 (Hikvision 연동용)</summary>
public class FaceRecognitionEvent
{
    public string PersonName { get; set; } = string.Empty;
    public string EmployeeId { get; set; } = string.Empty;
    public double Similarity { get; set; }
    public DateTime Timestamp { get; set; }
    public string DeviceIp { get; set; } = string.Empty;
}
```

### 8-2. Source Generator 사용법

```csharp
// Source Generator 모드로 직렬화/역직렬화하는 예제
using System.Text.Json;
using MyApp.Serialization;

namespace MyApp.Examples;

public static class SourceGeneratorExample
{
    public static void Demo()
    {
        var employee = new Employee
        {
            Id = 1,
            Name = "홍길동",
            Department = "개발팀"
        };

        // ──────────────────────────────────────────
        // 방법 1: 컨텍스트의 타입 정보를 직접 사용
        // ──────────────────────────────────────────

        // 직렬화
        string json1 = JsonSerializer.Serialize(
            employee,
            AppJsonContext.Default.Employee); // 타입별 메타데이터

        // 역직렬화
        Employee? parsed1 = JsonSerializer.Deserialize(
            json1,
            AppJsonContext.Default.Employee);

        // ──────────────────────────────────────────
        // 방법 2: 컨텍스트를 옵션으로 전달
        // ──────────────────────────────────────────

        // 직렬화
        string json2 = JsonSerializer.Serialize(
            employee,
            typeof(Employee),
            AppJsonContext.Default); // 컨텍스트 전체 전달

        // 역직렬화
        Employee? parsed2 = (Employee?)JsonSerializer.Deserialize(
            json2,
            typeof(Employee),
            AppJsonContext.Default);

        // ──────────────────────────────────────────
        // 방법 3: 제네릭 + 옵션에 컨텍스트 추가
        // ──────────────────────────────────────────

        var options = new JsonSerializerOptions
        {
            TypeInfoResolver = AppJsonContext.Default
        };

        // 기존 코드와 동일한 API 사용 가능
        string json3 = JsonSerializer.Serialize(employee, options);
        Employee? parsed3 = JsonSerializer.Deserialize<Employee>(
            json3, options);

        // ──────────────────────────────────────────
        // 컬렉션 직렬화
        // ──────────────────────────────────────────

        var employees = new List<Employee>
        {
            new() { Id = 1, Name = "홍길동" },
            new() { Id = 2, Name = "이순신" }
        };

        // List<Employee>도 [JsonSerializable]에 등록해야 합니다!
        string listJson = JsonSerializer.Serialize(
            employees,
            AppJsonContext.Default.ListEmployee);
    }
}
```

### 8-3. DI에서 Source Generator 사용

```csharp
// App.xaml.cs에서 Source Generator 기반 옵션 등록
services.AddSingleton(_ =>
{
    var options = new JsonSerializerOptions
    {
        // Source Generator 컨텍스트를 TypeInfoResolver로 설정
        TypeInfoResolver = AppJsonContext.Default,

        // 추가 옵션은 여전히 설정 가능
        WriteIndented = true,
        Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
    };

    return options;
});
```

### Source Generator 주의사항

| 주의사항 | 설명 |
|---------|------|
| **모든 타입 등록 필수** | `[JsonSerializable]`에 등록하지 않은 타입은 Source Generator가 처리하지 않음 |
| **제네릭 컬렉션도 등록** | `List<Employee>`를 사용하려면 별도로 `[JsonSerializable(typeof(List<Employee>))]` 등록 |
| **partial 클래스** | `AppJsonContext`는 반드시 `partial` 키워드가 필요 |
| **동적 타입 제한** | `object`, `dynamic` 등 동적 타입은 Source Generator로 처리하기 어려움 |
| **옵션 중복 주의** | `[JsonSourceGenerationOptions]`와 `JsonSerializerOptions` 양쪽에 설정하면 충돌 가능 |

---

## 9. 전체 코드 예제

### 프로젝트 구조

```
MyApp/
├── Converters/
│   ├── DateTimeFormatConverter.cs        ← DateTime 커스텀 컨버터
│   ├── NullableDateTimeFormatConverter.cs ← DateTime? 컨버터
│   └── EnumDescriptionConverter.cs        ← Enum+Description 컨버터
├── Models/
│   ├── Employee.cs                        ← 직원 모델
│   ├── AppSettings.cs                     ← 앱 설정 DTO
│   └── ApiResponse.cs                     ← API 응답 모델
├── Serialization/
│   └── AppJsonContext.cs                  ← Source Generator 컨텍스트
├── Services/
│   ├── IJsonFileService.cs               ← JSON 파일 서비스 인터페이스
│   ├── JsonFileService.cs                ← JSON 파일 서비스 구현
│   ├── ApiService.cs                     ← API 호출 서비스
│   └── MqttMessageHelper.cs              ← MQTT 메시지 헬퍼
├── ViewModels/
│   └── SettingsViewModel.cs              ← 설정 ViewModel
├── App.xaml.cs                            ← DI 설정
└── appsettings.json
```

### 모드별 사용 가이드

```
어떤 모드를 사용할까?

Q: AOT 컴파일이나 최고 성능이 필요한가?
│
├── YES → Source Generator 모드 (JsonSerializerContext)
│         - AppJsonContext에 모든 타입 등록
│         - 컴파일 타임 코드 생성
│         - 최고 성능, AOT 호환
│
└── NO  → 리플렉션 모드 (기본)
          - JsonSerializerOptions만 설정
          - 타입 등록 불필요
          - 간편한 사용
```

### 핵심 포인트 정리

| 항목 | 설명 |
|------|------|
| **라이브러리 선택** | 새 프로젝트 → System.Text.Json, 레거시 → Newtonsoft.Json |
| **NamingPolicy** | CamelCase(API), SnakeCaseLower(Python API) |
| **옵션 싱글톤** | DI에 `JsonSerializerOptions`를 싱글톤으로 등록하여 성능 확보 |
| **커스텀 컨버터** | `JsonConverter<T>` 상속 → `Read()`/`Write()` 구현 |
| **파일 저장/로드** | `IJsonFileService`로 추상화, 스트림 직렬화로 메모리 효율 |
| **API 통신** | `HttpClient` + `ReadAsStreamAsync()` + `DeserializeAsync()` |
| **MQTT 메시지** | `SerializeToUtf8Bytes()` / `Deserialize(byte[])` |
| **Source Generator** | `[JsonSerializable]` + `partial class` → 컴파일 타임 코드 생성 |
| **특성 활용** | `[JsonPropertyName]`, `[JsonIgnore]`, `[JsonConstructor]` |

> **이전 단계**: [Hikvision HCNetSDK P/Invoke 연동](./07-hikvision-sdk.md)에서는 비관리 DLL을 서비스로 래핑하는 방법을 다뤘습니다.
>
> **다음 단계**: [실전 예제](../05-practical-examples/)에서는 지금까지 배운 모든 기술 스택을 조합하여 실제 앱을 구현합니다.
