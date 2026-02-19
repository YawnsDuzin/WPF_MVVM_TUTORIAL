# Polly를 이용한 HTTP 복원력 패턴

> **목표**: WPF MVVM 프로젝트에서 Polly 8.x를 사용하여 HTTP 호출의 복원력(Resilience)을 구현합니다. 재시도, 서킷 브레이커, 타임아웃 전략을 이해하고 조합하는 방법을 익힙니다.
> **대상 독자**: HttpClient로 API를 호출해 본 경험이 있는 개발자

---

## 목차

1. [네트워크 호출의 문제점](#1-네트워크-호출의-문제점)
2. [Polly란?](#2-polly란)
3. [Polly 8.x의 새로운 API](#3-polly-8x의-새로운-api)
4. [핵심 전략](#4-핵심-전략)
5. [HttpClient + Polly 통합](#5-httpclient--polly-통합)
6. [WPF ViewModel에서 사용하는 예제](#6-wpf-viewmodel에서-사용하는-예제)
7. [전체 코드 예제](#7-전체-코드-예제)

---

## 1. 네트워크 호출의 문제점

### 네트워크는 신뢰할 수 없다

WinForms나 WPF 앱에서 외부 API를 호출할 때, 다음과 같은 문제가 발생할 수 있습니다:

```csharp
// ❌ 아무런 보호 장치 없는 HTTP 호출
var client = new HttpClient();
var response = await client.GetAsync("https://api.example.com/employees");
var data = await response.Content.ReadAsStringAsync();
```

**이 코드에서 발생할 수 있는 문제들:**

| 문제 | 원인 | 결과 |
|------|------|------|
| **타임아웃** | 서버 과부하, 네트워크 지연 | `TaskCanceledException` 발생 |
| **일시적 오류** | 서버 재시작, 네트워크 순간 끊김 | `HttpRequestException` (500, 503) |
| **연쇄 장애** | 다운된 서버에 계속 요청 | 앱 전체가 느려짐, 리소스 고갈 |
| **연결 고갈** | `new HttpClient()` 반복 생성 | 소켓 고갈 (SNAT 포트 고갈) |

### WinForms에서 흔히 하던 방식

```csharp
// ❌ WinForms 방식 — 단순 try-catch만 사용
private async void btnGetData_Click(object sender, EventArgs e)
{
    try
    {
        using var client = new HttpClient();  // 매번 새로 생성 (소켓 고갈 위험!)
        client.Timeout = TimeSpan.FromSeconds(10);

        var response = await client.GetAsync("https://api.example.com/data");
        if (response.IsSuccessStatusCode)
        {
            var json = await response.Content.ReadAsStringAsync();
            // ... 처리
        }
        else
        {
            MessageBox.Show($"오류: {response.StatusCode}");
        }
    }
    catch (Exception ex)
    {
        MessageBox.Show($"네트워크 오류: {ex.Message}");
        // 재시도? 사용자가 버튼을 다시 클릭해야 함...
    }
}
```

**이 방식의 문제점:**
- `HttpClient`를 매번 생성하여 소켓 고갈 위험
- 재시도 로직이 없어 일시적 오류에 취약
- 서버가 다운되어도 계속 요청을 보냄
- 타임아웃 관리가 미흡

---

## 2. Polly란?

Polly는 .NET 생태계에서 가장 널리 사용되는 **복원력(Resilience) 및 일시적 오류 처리 라이브러리**입니다.

### Polly가 제공하는 전략

| 전략 | 설명 | 비유 |
|------|------|------|
| **Retry** | 실패 시 자동 재시도 | 전화가 안 되면 다시 걸기 |
| **Circuit Breaker** | 연속 실패 시 호출 차단 | 퓨즈가 내려가면 전기 차단 |
| **Timeout** | 지정 시간 초과 시 취소 | 30초 안에 응답 없으면 포기 |
| **Fallback** | 실패 시 대체 값 반환 | 서버 못 연결하면 캐시 데이터 사용 |
| **Rate Limiter** | 호출 빈도 제한 | 1초에 10번까지만 요청 |
| **Hedging** | 동시에 여러 요청 보내기 | 가장 빠른 응답 사용 |

### NuGet 패키지

```bash
# Polly 핵심 패키지
dotnet add package Polly --version 8.5.2

# HttpClient 통합 (Microsoft.Extensions.Http.Resilience)
dotnet add package Microsoft.Extensions.Http.Resilience --version 9.0.0
```

---

## 3. Polly 8.x의 새로운 API

Polly 8.x는 기존 7.x와 완전히 다른 API를 제공합니다. **ResiliencePipeline** 기반으로 재설계되었습니다.

### Polly 7.x vs 8.x 비교

```csharp
// ── Polly 7.x (이전 방식 — 참고용) ──
// Policy 기반, 동기/비동기 별도 API
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt =>
        TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

await retryPolicy.ExecuteAsync(async () =>
{
    var response = await client.GetAsync(url);
    response.EnsureSuccessStatusCode();
});
```

```csharp
// ── Polly 8.x (현재 방식) ──
// ResiliencePipeline 기반, 통합된 API
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential
    })
    .Build();

var response = await pipeline.ExecuteAsync(
    async ct => await client.GetAsync(url, ct),
    cancellationToken);
```

### 핵심 개념

| 개념 | 설명 |
|------|------|
| `ResiliencePipeline` | 여러 전략을 조합한 실행 파이프라인 |
| `ResiliencePipelineBuilder` | 파이프라인을 구성하는 빌더 |
| `ResilienceStrategy` | 개별 복원력 전략 (Retry, Timeout 등) |
| `ResilienceContext` | 실행 컨텍스트 (메타데이터, CancellationToken) |

---

## 4. 핵심 전략

### 4.1 Retry (재시도)

일시적 오류가 발생하면 지정된 횟수만큼 자동으로 재시도합니다.

```csharp
using Polly;
using Polly.Retry;

// ── 기본 재시도 설정 ──
var retryOptions = new RetryStrategyOptions<HttpResponseMessage>
{
    // 최대 재시도 횟수 (3번 재시도 = 총 4번 시도)
    MaxRetryAttempts = 3,

    // 재시도 간격의 기본값 (1초)
    Delay = TimeSpan.FromSeconds(1),

    // 백오프 유형: 지수적 증가
    // 1초 → 2초 → 4초 (지수 백오프)
    BackoffType = DelayBackoffType.Exponential,

    // 지터 추가: 여러 클라이언트가 동시에 재시도하는 것을 방지
    // (Thundering Herd 문제 완화)
    UseJitter = true,

    // 어떤 경우에 재시도할지 판단
    ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
        // HTTP 5xx (서버 오류) 또는 408 (타임아웃)일 때 재시도
        .HandleResult(response =>
            response.StatusCode == System.Net.HttpStatusCode.RequestTimeout ||
            response.StatusCode == System.Net.HttpStatusCode.TooManyRequests ||
            (int)response.StatusCode >= 500)
        // 네트워크 예외도 재시도
        .Handle<HttpRequestException>()
        .Handle<TaskCanceledException>(),

    // 재시도 시 실행할 콜백 (로깅 등)
    OnRetry = args =>
    {
        Console.WriteLine(
            $"재시도 {args.AttemptNumber + 1}/{3}: " +
            $"다음 시도까지 {args.RetryDelay.TotalSeconds:F1}초 대기");
        return ValueTask.CompletedTask;
    }
};
```

#### 지수 백오프(Exponential Backoff)란?

재시도 간격을 점차 늘려가는 전략입니다. 서버에 과부하를 주지 않기 위함입니다.

```
시도 1: 즉시 실행          →  실패
시도 2: 1초 후 재시도       →  실패
시도 3: 2초 후 재시도       →  실패
시도 4: 4초 후 재시도       →  성공 (또는 최종 실패)

지터(Jitter) 적용 시:
시도 2: 0.8~1.2초 후       →  (무작위 변동)
시도 3: 1.6~2.4초 후       →  (무작위 변동)
시도 4: 3.2~4.8초 후       →  (무작위 변동)
```

### 4.2 Circuit Breaker (서킷 브레이커)

연속으로 실패가 발생하면 일정 시간 동안 호출을 차단하여, 다운된 서버에 불필요한 요청을 보내지 않습니다.

#### 상태 다이어그램

```
                        실패율 임계값 초과
              ┌──────────────────────────────────┐
              │                                  ▼
        ┌───────────┐                    ┌──────────────┐
        │           │                    │              │
        │  Closed   │                    │    Open      │
        │ (정상)    │                    │  (차단 중)   │
        │           │                    │              │
        └───────────┘                    └──────┬───────┘
              ▲                                 │
              │                    Break Duration 경과
              │                                 │
              │                          ┌──────▼───────┐
              │                          │              │
              │        성공              │  Half-Open   │
              └──────────────────────────│ (시험 중)    │
                                         │              │
                                         └──────┬───────┘
                                                │
                                         실패 → Open으로 돌아감
```

**각 상태 설명:**

| 상태 | 설명 | 동작 |
|------|------|------|
| **Closed (닫힘)** | 정상 상태 | 모든 요청이 통과됨 |
| **Open (열림)** | 장애 감지 | 모든 요청이 즉시 차단됨 (`BrokenCircuitException`) |
| **Half-Open (반열림)** | 복구 확인 중 | 시험 요청 1개만 통과시킴 |

```csharp
using Polly.CircuitBreaker;

var circuitBreakerOptions = new CircuitBreakerStrategyOptions<HttpResponseMessage>
{
    // 실패율 계산에 사용할 샘플 기간
    SamplingDuration = TimeSpan.FromSeconds(30),

    // 샘플 기간 내 최소 처리 수 (이 수 이상이어야 서킷 브레이커가 동작)
    MinimumThroughput = 10,

    // 실패율 임계값 (50% 이상 실패하면 서킷 열림)
    FailureRatio = 0.5,

    // 서킷이 열린 후 대기 시간 (이 시간이 지나면 Half-Open으로 전환)
    BreakDuration = TimeSpan.FromSeconds(30),

    // 실패로 판단할 조건
    ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
        .HandleResult(r => (int)r.StatusCode >= 500)
        .Handle<HttpRequestException>()
        .Handle<TaskCanceledException>(),

    // 서킷 상태 변경 시 콜백 (로깅에 활용)
    OnOpened = args =>
    {
        Console.WriteLine(
            $"서킷 브레이커 열림! {args.BreakDuration.TotalSeconds}초간 요청 차단");
        return ValueTask.CompletedTask;
    },
    OnClosed = args =>
    {
        Console.WriteLine("서킷 브레이커 닫힘 — 정상 상태 복귀");
        return ValueTask.CompletedTask;
    },
    OnHalfOpened = args =>
    {
        Console.WriteLine("서킷 브레이커 반열림 — 시험 요청 허용");
        return ValueTask.CompletedTask;
    }
};
```

### 4.3 Timeout (타임아웃)

지정된 시간 내에 응답이 오지 않으면 요청을 취소합니다.

```csharp
using Polly.Timeout;

var timeoutOptions = new TimeoutStrategyOptions
{
    // 타임아웃 시간 (10초)
    Timeout = TimeSpan.FromSeconds(10),

    // 타임아웃 발생 시 콜백
    OnTimeout = args =>
    {
        Console.WriteLine(
            $"타임아웃 발생! 경과 시간: {args.Timeout.TotalSeconds}초");
        return ValueTask.CompletedTask;
    }
};
```

> **참고**: Polly의 Timeout은 `CancellationToken`을 통해 동작합니다. `HttpClient` 자체의 `Timeout` 속성과는 별개입니다. Polly Timeout은 더 세밀한 제어가 가능합니다.

### 4.4 전략 조합 (ResiliencePipelineBuilder)

여러 전략을 하나의 파이프라인으로 조합할 수 있습니다. **실행 순서가 중요**합니다.

```csharp
using Polly;
using Polly.CircuitBreaker;
using Polly.Retry;
using Polly.Timeout;

// 전략 조합 — 바깥쪽에서 안쪽 순서로 실행
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    // 1단계: 전체 타임아웃 (모든 재시도를 포함한 전체 시간 제한)
    .AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromSeconds(30),
        OnTimeout = args =>
        {
            Console.WriteLine("전체 타임아웃 초과!");
            return ValueTask.CompletedTask;
        }
    })

    // 2단계: 재시도 (서킷 브레이커가 허용할 때만 실행됨)
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .HandleResult(r => (int)r.StatusCode >= 500)
            .Handle<HttpRequestException>()
            .Handle<TaskCanceledException>()
            .Handle<BrokenCircuitException>(), // 서킷이 열려 있으면 재시도
        OnRetry = args =>
        {
            Console.WriteLine(
                $"재시도 {args.AttemptNumber + 1}: " +
                $"{args.RetryDelay.TotalSeconds:F1}초 후 재시도");
            return ValueTask.CompletedTask;
        }
    })

    // 3단계: 서킷 브레이커
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 10,
        FailureRatio = 0.5,
        BreakDuration = TimeSpan.FromSeconds(30),
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .HandleResult(r => (int)r.StatusCode >= 500)
            .Handle<HttpRequestException>()
    })

    // 4단계: 개별 요청 타임아웃 (각 시도당 시간 제한)
    .AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromSeconds(10)
    })

    .Build();
```

#### 실행 순서 다이어그램

```
요청 →  [전체 Timeout]  →  [Retry]  →  [Circuit Breaker]  →  [요청 Timeout]  →  HTTP 호출
         (30초 제한)      (3회 재시도)   (장애 감지)          (10초/각 시도)     (실제 전송)

실패 시 흐름:
HTTP 호출 실패 → 요청 Timeout 확인 → Circuit Breaker 기록 → Retry 판단 → 전체 Timeout 확인
                                                              │
                                                     재시도 가능 → 다시 시도
                                                     재시도 불가 → 예외 발생
```

---

## 5. HttpClient + Polly 통합

### IHttpClientFactory를 사용해야 하는 이유

```csharp
// ❌ 절대 이렇게 하지 마세요
using var client = new HttpClient();  // 매번 생성 → 소켓 고갈!

// ❌ 이것도 문제가 있음
private static readonly HttpClient _client = new HttpClient();  // DNS 변경 감지 못함
```

`IHttpClientFactory`는 `HttpClient` 인스턴스의 생명주기를 자동으로 관리하여 소켓 고갈과 DNS 캐시 문제를 해결합니다.

### DI에서 HttpClient + Polly 설정

#### 방법 1: Named HttpClient + Polly (기본 방식)

```csharp
// App.xaml.cs 또는 DI 설정 부분

using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Http.Resilience;
using Polly;

// ── HttpClient + Polly 등록 ──
services
    .AddHttpClient("EmployeeApi", client =>
    {
        // 기본 URL 설정
        client.BaseAddress = new Uri("https://api.example.com/");

        // 기본 헤더 설정
        client.DefaultRequestHeaders.Add("Accept", "application/json");

        // HttpClient 자체 타임아웃 (Polly 타임아웃과 별개)
        // Polly가 타임아웃을 관리하므로 여기서는 넉넉하게 설정
        client.Timeout = TimeSpan.FromSeconds(60);
    })
    // Polly 복원력 파이프라인 추가
    .AddResilienceHandler("EmployeeApiResilience", builder =>
    {
        // 전체 타임아웃
        builder.AddTimeout(new TimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(30)
        });

        // 재시도
        builder.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(1),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true
        });

        // 서킷 브레이커
        builder.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 10,
            FailureRatio = 0.5,
            BreakDuration = TimeSpan.FromSeconds(30)
        });

        // 개별 요청 타임아웃
        builder.AddTimeout(new TimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(10)
        });
    });
```

#### 방법 2: Typed HttpClient + Polly (권장)

Typed HttpClient를 사용하면 서비스 클래스에 `HttpClient`가 자동으로 주입됩니다.

```csharp
// Services/IEmployeeApiService.cs

namespace MyApp.Services;

/// <summary>
/// 직원 API 호출 서비스 인터페이스
/// </summary>
public interface IEmployeeApiService
{
    /// <summary>직원 목록 조회</summary>
    Task<IEnumerable<Employee>> GetEmployeesAsync(CancellationToken cancellationToken = default);

    /// <summary>직원 상세 조회</summary>
    Task<Employee?> GetEmployeeByIdAsync(int id, CancellationToken cancellationToken = default);

    /// <summary>직원 생성</summary>
    Task<Employee> CreateEmployeeAsync(Employee employee, CancellationToken cancellationToken = default);
}
```

```csharp
// Services/EmployeeApiService.cs

using System.Net.Http.Json;
using Microsoft.Extensions.Logging;

namespace MyApp.Services;

/// <summary>
/// 직원 API 호출 서비스 구현.
/// HttpClient는 DI에서 자동 주입됩니다 (Typed HttpClient).
/// Polly 복원력 파이프라인도 자동으로 적용됩니다.
/// </summary>
public sealed class EmployeeApiService : IEmployeeApiService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<EmployeeApiService> _logger;

    /// <summary>
    /// 생성자 — HttpClient는 IHttpClientFactory가 생성하고 Polly가 래핑한 것
    /// </summary>
    public EmployeeApiService(HttpClient httpClient, ILogger<EmployeeApiService> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<IEnumerable<Employee>> GetEmployeesAsync(
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("직원 목록 API 호출 시작");

        try
        {
            // GetFromJsonAsync: JSON 응답을 자동으로 역직렬화
            // Polly가 자동으로 재시도, 서킷 브레이커, 타임아웃을 적용
            var employees = await _httpClient.GetFromJsonAsync<IEnumerable<Employee>>(
                "api/employees",
                cancellationToken);

            _logger.LogInformation("직원 목록 API 호출 성공: {Count}건",
                employees?.Count() ?? 0);

            return employees ?? [];
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex,
                "직원 목록 API 호출 실패: StatusCode={StatusCode}",
                ex.StatusCode);
            throw;
        }
        catch (TaskCanceledException ex) when (!cancellationToken.IsCancellationRequested)
        {
            // CancellationToken이 취소되지 않았는데 TaskCanceledException이 발생하면
            // Polly 타임아웃이 동작한 것
            _logger.LogError(ex, "직원 목록 API 호출 타임아웃");
            throw;
        }
    }

    public async Task<Employee?> GetEmployeeByIdAsync(
        int id,
        CancellationToken cancellationToken = default)
    {
        _logger.LogDebug("직원 상세 API 호출: Id={EmployeeId}", id);

        try
        {
            var response = await _httpClient.GetAsync(
                $"api/employees/{id}",
                cancellationToken);

            if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
            {
                _logger.LogWarning("직원을 찾을 수 없음: Id={EmployeeId}", id);
                return null;
            }

            response.EnsureSuccessStatusCode();

            return await response.Content.ReadFromJsonAsync<Employee>(
                cancellationToken: cancellationToken);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "직원 상세 API 호출 실패: Id={EmployeeId}", id);
            throw;
        }
    }

    public async Task<Employee> CreateEmployeeAsync(
        Employee employee,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("직원 생성 API 호출: Name={EmployeeName}", employee.Name);

        try
        {
            // PostAsJsonAsync: 객체를 JSON으로 직렬화하여 POST
            var response = await _httpClient.PostAsJsonAsync(
                "api/employees",
                employee,
                cancellationToken);

            response.EnsureSuccessStatusCode();

            var created = await response.Content.ReadFromJsonAsync<Employee>(
                cancellationToken: cancellationToken);

            _logger.LogInformation(
                "직원 생성 API 호출 성공: Id={EmployeeId}, Name={EmployeeName}",
                created!.Id, created.Name);

            return created;
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex,
                "직원 생성 API 호출 실패: Name={EmployeeName}",
                employee.Name);
            throw;
        }
    }
}
```

```csharp
// App.xaml.cs — Typed HttpClient + Polly 등록

services
    .AddHttpClient<IEmployeeApiService, EmployeeApiService>(client =>
    {
        client.BaseAddress = new Uri("https://api.example.com/");
        client.DefaultRequestHeaders.Add("Accept", "application/json");
        client.Timeout = TimeSpan.FromSeconds(60);
    })
    .AddResilienceHandler("EmployeeApiResilience", builder =>
    {
        // 전체 타임아웃 (30초)
        builder.AddTimeout(new TimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(30)
        });

        // 재시도 (3회, 지수 백오프)
        builder.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(1),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true
        });

        // 서킷 브레이커
        builder.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 10,
            FailureRatio = 0.5,
            BreakDuration = TimeSpan.FromSeconds(30)
        });

        // 개별 요청 타임아웃 (10초)
        builder.AddTimeout(new TimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(10)
        });
    });
```

---

## 6. WPF ViewModel에서 사용하는 예제

### ViewModel에서 API 서비스 호출

```csharp
// ViewModels/EmployeeApiViewModel.cs

using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.Logging;
using Polly.CircuitBreaker;

namespace MyApp.ViewModels;

/// <summary>
/// REST API를 통해 직원 데이터를 관리하는 ViewModel.
/// Polly의 복원력 전략이 자동으로 적용됩니다.
/// </summary>
public partial class EmployeeApiViewModel : ObservableObject
{
    private readonly IEmployeeApiService _apiService;
    private readonly ILogger<EmployeeApiViewModel> _logger;

    public EmployeeApiViewModel(
        IEmployeeApiService apiService,
        ILogger<EmployeeApiViewModel> logger)
    {
        _apiService = apiService;
        _logger = logger;
    }

    // ── 속성 ──

    /// <summary>직원 목록</summary>
    public ObservableCollection<Employee> Employees { get; } = [];

    /// <summary>로딩 중 여부</summary>
    [ObservableProperty]
    private bool _isLoading;

    /// <summary>상태 메시지</summary>
    [ObservableProperty]
    private string _statusMessage = string.Empty;

    /// <summary>오류 발생 여부 (UI에서 오류 스타일 적용용)</summary>
    [ObservableProperty]
    private bool _hasError;

    /// <summary>입력: 직원 이름</summary>
    [ObservableProperty]
    private string _name = string.Empty;

    /// <summary>입력: 부서</summary>
    [ObservableProperty]
    private string _department = string.Empty;

    /// <summary>입력: 이메일</summary>
    [ObservableProperty]
    private string _email = string.Empty;

    // ── 명령 ──

    /// <summary>
    /// 직원 목록 로드.
    /// Polly가 자동으로 재시도, 서킷 브레이커, 타임아웃을 처리합니다.
    /// ViewModel은 실패 시 사용자에게 알리는 것만 책임집니다.
    /// </summary>
    [RelayCommand]
    private async Task LoadEmployeesAsync(CancellationToken cancellationToken)
    {
        try
        {
            IsLoading = true;
            HasError = false;
            StatusMessage = "직원 목록을 불러오는 중...";

            // API 호출 — Polly가 투명하게 복원력을 제공
            var employees = await _apiService.GetEmployeesAsync(cancellationToken);

            Employees.Clear();
            foreach (var emp in employees)
            {
                Employees.Add(emp);
            }

            StatusMessage = $"직원 {Employees.Count}명 로드 완료";
        }
        catch (BrokenCircuitException)
        {
            // 서킷 브레이커가 열린 상태 — 서버가 다운된 것으로 판단
            HasError = true;
            StatusMessage = "서버에 연결할 수 없습니다. 잠시 후 다시 시도해 주세요.";

            _logger.LogWarning("서킷 브레이커 열림 — API 호출이 차단됨");
        }
        catch (TaskCanceledException) when (!cancellationToken.IsCancellationRequested)
        {
            // Polly 타임아웃 (사용자가 취소한 것이 아님)
            HasError = true;
            StatusMessage = "서버 응답 시간이 초과되었습니다.";

            _logger.LogWarning("API 호출 타임아웃");
        }
        catch (TaskCanceledException)
        {
            // 사용자가 직접 취소한 경우
            StatusMessage = "작업이 취소되었습니다.";
            _logger.LogInformation("사용자가 API 호출을 취소함");
        }
        catch (HttpRequestException ex)
        {
            // 네트워크 오류 (재시도 후에도 실패)
            HasError = true;
            StatusMessage = $"네트워크 오류: {ex.Message}";

            _logger.LogError(ex, "API 호출 최종 실패 (재시도 모두 소진)");
        }
        catch (Exception ex)
        {
            // 기타 예상치 못한 오류
            HasError = true;
            StatusMessage = $"오류 발생: {ex.Message}";

            _logger.LogError(ex, "직원 목록 로드 중 예기치 않은 오류");
        }
        finally
        {
            IsLoading = false;
        }
    }

    /// <summary>
    /// 직원 생성.
    /// </summary>
    [RelayCommand]
    private async Task CreateEmployeeAsync(CancellationToken cancellationToken)
    {
        if (string.IsNullOrWhiteSpace(Name))
        {
            StatusMessage = "이름을 입력하세요.";
            return;
        }

        try
        {
            IsLoading = true;
            HasError = false;
            StatusMessage = "직원 생성 중...";

            var newEmployee = new Employee
            {
                Name = Name,
                Department = Department,
                Email = Email,
                HireDate = DateTime.Today,
                IsActive = true
            };

            var created = await _apiService.CreateEmployeeAsync(
                newEmployee, cancellationToken);

            StatusMessage = $"직원 '{created.Name}' 생성 완료 (ID: {created.Id})";

            // 입력 필드 초기화
            Name = string.Empty;
            Department = string.Empty;
            Email = string.Empty;

            // 목록 새로고침
            await LoadEmployeesAsync(cancellationToken);
        }
        catch (BrokenCircuitException)
        {
            HasError = true;
            StatusMessage = "서버에 연결할 수 없습니다. 잠시 후 다시 시도해 주세요.";
        }
        catch (HttpRequestException ex)
        {
            HasError = true;
            StatusMessage = $"직원 생성 실패: {ex.Message}";
            _logger.LogError(ex, "직원 생성 API 최종 실패: Name={EmployeeName}", Name);
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

### View (XAML) — 오류 상태 표시

```xml
<!-- Views/EmployeeApiWindow.xaml -->
<Window x:Class="MyApp.Views.EmployeeApiWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="직원 관리 (API)" Width="800" Height="600">

    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- ═══════ 입력 폼 ═══════ -->
        <StackPanel Grid.Row="0" Margin="0,0,0,8">
            <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}"
                     Margin="0,0,0,4"
                     Tag="이름"/>
            <TextBox Text="{Binding Department, UpdateSourceTrigger=PropertyChanged}"
                     Margin="0,0,0,4"
                     Tag="부서"/>
            <TextBox Text="{Binding Email, UpdateSourceTrigger=PropertyChanged}"
                     Margin="0,0,0,4"
                     Tag="이메일"/>
        </StackPanel>

        <!-- ═══════ 버튼 영역 ═══════ -->
        <StackPanel Grid.Row="1" Orientation="Horizontal" Margin="0,0,0,8">
            <Button Content="조회" Command="{Binding LoadEmployeesCommand}"
                    Width="80" Margin="0,0,8,0"/>
            <Button Content="생성" Command="{Binding CreateEmployeeCommand}"
                    Width="80" Margin="0,0,8,0"/>
        </StackPanel>

        <!-- ═══════ 직원 목록 ═══════ -->
        <DataGrid Grid.Row="2"
                  ItemsSource="{Binding Employees}"
                  AutoGenerateColumns="False"
                  IsReadOnly="True">
            <DataGrid.Columns>
                <DataGridTextColumn Header="ID" Binding="{Binding Id}" Width="50"/>
                <DataGridTextColumn Header="이름" Binding="{Binding Name}" Width="*"/>
                <DataGridTextColumn Header="부서" Binding="{Binding Department}" Width="100"/>
                <DataGridTextColumn Header="이메일" Binding="{Binding Email}" Width="*"/>
            </DataGrid.Columns>
        </DataGrid>

        <!-- ═══════ 상태 바 ═══════ -->
        <Border Grid.Row="3" Margin="0,8,0,0" Padding="8,4">
            <!--
                HasError 속성에 따라 배경색을 변경하여
                사용자가 오류 상태를 직관적으로 알 수 있게 합니다.
            -->
            <Border.Style>
                <Style TargetType="Border">
                    <Setter Property="Background" Value="Transparent"/>
                    <Style.Triggers>
                        <DataTrigger Binding="{Binding HasError}" Value="True">
                            <Setter Property="Background" Value="#FFF0F0"/>
                        </DataTrigger>
                    </Style.Triggers>
                </Style>
            </Border.Style>

            <StackPanel Orientation="Horizontal">
                <!-- 로딩 인디케이터 -->
                <ProgressBar IsIndeterminate="{Binding IsLoading}"
                             Width="80" Height="14"
                             Margin="0,0,8,0"
                             Visibility="{Binding IsLoading,
                                 Converter={StaticResource BooleanToVisibilityConverter}}"/>

                <!-- 상태 메시지 -->
                <TextBlock Text="{Binding StatusMessage}" VerticalAlignment="Center">
                    <TextBlock.Style>
                        <Style TargetType="TextBlock">
                            <Setter Property="Foreground" Value="Gray"/>
                            <Style.Triggers>
                                <DataTrigger Binding="{Binding HasError}" Value="True">
                                    <Setter Property="Foreground" Value="Red"/>
                                    <Setter Property="FontWeight" Value="Bold"/>
                                </DataTrigger>
                            </Style.Triggers>
                        </Style>
                    </TextBlock.Style>
                </TextBlock>
            </StackPanel>
        </Border>
    </Grid>
</Window>
```

---

## 7. 전체 코드 예제

### 7.1 프로젝트 구조

```
MyApp/
├── App.xaml
├── App.xaml.cs              ← DI 설정 + HttpClient + Polly 등록
├── appsettings.json         ← API URL, 타임아웃 설정
├── Models/
│   ├── Employee.cs
│   └── Settings/
│       └── ApiSettings.cs   ← API 설정 클래스
├── Services/
│   ├── IEmployeeApiService.cs  ← API 서비스 인터페이스
│   └── EmployeeApiService.cs   ← API 서비스 구현
├── ViewModels/
│   └── EmployeeApiViewModel.cs ← ViewModel
└── Views/
    └── EmployeeApiWindow.xaml  ← View
```

### 7.2 설정 파일

```jsonc
// appsettings.json
{
  "ApiSettings": {
    "BaseUrl": "https://api.example.com/",
    "TimeoutSeconds": 10,
    "RetryCount": 3,
    "CircuitBreakerFailureRatio": 0.5,
    "CircuitBreakerBreakDurationSeconds": 30
  }
}
```

```csharp
// Models/Settings/ApiSettings.cs

namespace MyApp.Models.Settings;

/// <summary>
/// API 호출 관련 설정.
/// appsettings.json의 "ApiSettings" 섹션과 매핑됩니다.
/// </summary>
public sealed class ApiSettings
{
    public const string SectionName = "ApiSettings";

    /// <summary>API 기본 URL</summary>
    public string BaseUrl { get; set; } = string.Empty;

    /// <summary>요청 당 타임아웃(초)</summary>
    public int TimeoutSeconds { get; set; } = 10;

    /// <summary>재시도 횟수</summary>
    public int RetryCount { get; set; } = 3;

    /// <summary>서킷 브레이커 실패율 임계값 (0.0 ~ 1.0)</summary>
    public double CircuitBreakerFailureRatio { get; set; } = 0.5;

    /// <summary>서킷 브레이커 차단 시간(초)</summary>
    public int CircuitBreakerBreakDurationSeconds { get; set; } = 30;
}
```

### 7.3 App.xaml.cs — 전체 DI 설정

```csharp
// App.xaml.cs — HttpClient + Polly 전체 설정

using System.Windows;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Http.Resilience;
using Polly;
using Polly.Timeout;
using Serilog;

namespace MyApp;

public partial class App : Application
{
    private readonly IHost _host;

    public App()
    {
        // Serilog 설정 (02-logging-serilog.md 참고)
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Information()
            .WriteTo.Console()
            .WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day)
            .CreateLogger();

        _host = Host.CreateDefaultBuilder()
            .ConfigureAppConfiguration((_, config) =>
            {
                config.SetBasePath(AppContext.BaseDirectory);
                config.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);
            })
            .UseSerilog()
            .ConfigureServices((context, services) =>
            {
                // ── 설정 등록 ──
                var apiSettings = new ApiSettings();
                context.Configuration.GetSection(ApiSettings.SectionName).Bind(apiSettings);
                services.Configure<ApiSettings>(
                    context.Configuration.GetSection(ApiSettings.SectionName));

                // ── HttpClient + Polly 등록 (Typed HttpClient) ──
                services
                    .AddHttpClient<IEmployeeApiService, EmployeeApiService>(client =>
                    {
                        // appsettings.json에서 읽은 기본 URL 설정
                        client.BaseAddress = new Uri(apiSettings.BaseUrl);
                        client.DefaultRequestHeaders.Add("Accept", "application/json");
                        // Polly가 타임아웃을 관리하므로 HttpClient 자체 타임아웃은 넉넉하게
                        client.Timeout = TimeSpan.FromSeconds(60);
                    })
                    .AddResilienceHandler("EmployeeApiPipeline", (builder, context) =>
                    {
                        // 설정 값을 DI에서 가져옴
                        var settings = context.ServiceProvider
                            .GetRequiredService<IOptions<ApiSettings>>().Value;

                        // 1. 전체 타임아웃 (모든 재시도 포함)
                        builder.AddTimeout(new TimeoutStrategyOptions
                        {
                            Timeout = TimeSpan.FromSeconds(settings.TimeoutSeconds * (settings.RetryCount + 1))
                        });

                        // 2. 재시도 (지수 백오프 + 지터)
                        builder.AddRetry(new HttpRetryStrategyOptions
                        {
                            MaxRetryAttempts = settings.RetryCount,
                            Delay = TimeSpan.FromSeconds(1),
                            BackoffType = DelayBackoffType.Exponential,
                            UseJitter = true
                        });

                        // 3. 서킷 브레이커
                        builder.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
                        {
                            SamplingDuration = TimeSpan.FromSeconds(30),
                            MinimumThroughput = 10,
                            FailureRatio = settings.CircuitBreakerFailureRatio,
                            BreakDuration = TimeSpan.FromSeconds(
                                settings.CircuitBreakerBreakDurationSeconds)
                        });

                        // 4. 개별 요청 타임아웃
                        builder.AddTimeout(new TimeoutStrategyOptions
                        {
                            Timeout = TimeSpan.FromSeconds(settings.TimeoutSeconds)
                        });
                    });

                // ── ViewModel 등록 ──
                services.AddTransient<EmployeeApiViewModel>();

                // ── View 등록 ──
                services.AddTransient<EmployeeApiWindow>();
            })
            .Build();
    }

    protected override async void OnStartup(StartupEventArgs e)
    {
        await _host.StartAsync();

        var mainWindow = _host.Services.GetRequiredService<EmployeeApiWindow>();
        mainWindow.Show();

        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        await _host.StopAsync();
        _host.Dispose();
        await Log.CloseAndFlushAsync();
        base.OnExit(e);
    }
}
```

### 7.4 Polly 동작 시나리오별 흐름

#### 시나리오 1: 정상 응답

```
요청 →  [전체 Timeout]  →  [Retry]  →  [Circuit Breaker]  →  [요청 Timeout]  →  HTTP 200 OK
                                                                                    │
응답 ←──────────────────────────────────────────────────────────────────────────────┘
                                                                     (1회 시도로 성공)
```

#### 시나리오 2: 일시적 오류 → 재시도 성공

```
시도 1:  HTTP 500 ──→ [Retry 판단: 재시도 가능] ──→ 1초 대기
시도 2:  HTTP 503 ──→ [Retry 판단: 재시도 가능] ──→ 2초 대기
시도 3:  HTTP 200 ──→ 성공!

ViewModel에서는 정상 응답만 받음 (재시도를 몰라도 됨)
```

#### 시나리오 3: 서킷 브레이커 작동

```
최근 30초 동안 10건 이상 호출, 50% 이상 실패
    │
    ▼
서킷 열림 (Open) ──→ 모든 요청 즉시 BrokenCircuitException
    │
    │  30초 경과
    ▼
서킷 반열림 (Half-Open) ──→ 시험 요청 1건 허용
    │
    ├─ 성공 → 서킷 닫힘 (Closed) → 정상 동작
    └─ 실패 → 서킷 열림 (Open) → 30초 더 대기
```

#### 시나리오 4: 타임아웃

```
요청 시작 ──→ [요청 Timeout: 10초] ──→ 10초 경과, 응답 없음
                                           │
                                    CancellationToken 취소
                                           │
                            TaskCanceledException 발생
                                           │
                              [Retry 판단: 재시도 가능]
                                           │
                                      재시도 진행...
```

---

## 요약

| 항목 | 핵심 |
|------|------|
| **Polly** | .NET 복원력 라이브러리 (재시도, 서킷 브레이커, 타임아웃) |
| **Polly 8.x** | ResiliencePipeline 기반의 새로운 API |
| **Retry** | 일시적 오류 시 자동 재시도 (지수 백오프 + 지터) |
| **Circuit Breaker** | 연속 실패 시 요청 차단 → 서버 보호 |
| **Timeout** | 전체 타임아웃 + 개별 요청 타임아웃 이중 설정 |
| **IHttpClientFactory** | HttpClient 생명주기 관리 (소켓 고갈 방지) |
| **Typed HttpClient** | 서비스 클래스에 HttpClient 자동 주입 |
| **ViewModel** | Polly 예외별 사용자 친화적 메시지 표시 |
| **설정 외부화** | appsettings.json에서 타임아웃, 재시도 횟수 등 관리 |

---

> **다음 문서**: [MQTTnet을 이용한 MQTT v5 통신](./04-mqtt-mqttnet.md) — IoT 장비와 MQTT 프로토콜로 실시간 통신을 구현합니다.
