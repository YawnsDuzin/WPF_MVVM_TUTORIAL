# MQTTnet을 이용한 MQTT v5 통신

> **목표**: WPF MVVM 프로젝트에서 MQTTnet 5.x를 사용하여 MQTT v5 브로커에 연결하고, 토픽 구독/발행, 자동 재연결, UI 갱신까지의 전체 흐름을 구현합니다.
> **대상 독자**: MQTT를 처음 접하거나 TCP 소켓 통신 경험이 있는 WinForms/Python 개발자

---

## 목차

1. [MQTT란?](#1-mqtt란)
2. [MQTTnet 5.x 소개](#2-mqttnet-5x-소개)
3. [MQTT 기본 개념](#3-mqtt-기본-개념)
4. [MQTT 서비스 구현](#4-mqtt-서비스-구현)
5. [DI에 등록 (Singleton)](#5-di에-등록-singleton)
6. [ViewModel에서 MQTT 사용](#6-viewmodel에서-mqtt-사용)
7. [WPF 앱 종료 시 연결 정리](#7-wpf-앱-종료-시-연결-정리)
8. [appsettings.json에서 MQTT 설정 관리](#8-appsettingsjson에서-mqtt-설정-관리)
9. [전체 코드 예제](#9-전체-코드-예제)

---

## 1. MQTT란?

### 간단한 설명

MQTT(Message Queuing Telemetry Transport)는 **경량 메시지 프로토콜**입니다. IoT(사물인터넷) 장비, 센서, 산업 설비 등 **리소스가 제한된 환경**에서 실시간으로 데이터를 주고받기 위해 설계되었습니다.

### HTTP vs MQTT 비교

| 항목 | HTTP | MQTT |
|------|------|------|
| 통신 방식 | 요청-응답 (Request-Response) | 발행-구독 (Publish-Subscribe) |
| 연결 | 매번 연결/해제 | 지속 연결 (Long-lived) |
| 프로토콜 크기 | 헤더가 큼 (수백 바이트) | 헤더가 작음 (최소 2바이트) |
| 실시간성 | 폴링 필요 | 즉시 수신 |
| 용도 | 웹 API, REST | IoT, 센서, 장비 통신 |
| 방향 | 클라이언트 → 서버 (단방향) | 양방향 |

### 실제 사용 예시

```
[얼굴인식 단말기] ──MQTT──→ [브로커] ──MQTT──→ [WPF 관리 앱]
  "출입 이벤트 발행"          "메시지 라우팅"     "실시간 수신 및 표시"

[온도 센서] ──MQTT──→ [브로커] ──MQTT──→ [모니터링 앱]
  "온도 데이터 발행"            ↘
                            [데이터 저장 서비스]
                              "DB에 기록"
```

---

## 2. MQTTnet 5.x 소개

MQTTnet은 .NET에서 가장 널리 사용되는 **고성능 MQTT 클라이언트/서버 라이브러리**입니다.

### 특징

| 특징 | 설명 |
|------|------|
| **MQTT v5 지원** | 최신 MQTT v5.0 프로토콜 완전 지원 |
| **고성능** | 비동기 기반, 효율적인 메모리 사용 |
| **클라이언트/서버** | MQTT 클라이언트와 브로커 모두 구현 가능 |
| **크로스 플랫폼** | Windows, Linux, macOS 지원 |
| **.NET 10 지원** | 최신 .NET 버전 호환 |

### NuGet 패키지

```bash
# MQTTnet 클라이언트
dotnet add package MQTTnet --version 5.0.1
```

---

## 3. MQTT 기본 개념

### 3.1 Broker (브로커)

브로커는 **메시지 중개 서버**입니다. 모든 MQTT 통신은 브로커를 거칩니다. 클라이언트 간에 직접 통신하지 않습니다.

```
발행자 (Publisher)           브로커 (Broker)          구독자 (Subscriber)
─────────────────          ──────────────          ──────────────────
  장비, 센서            →    Mosquitto,       →     WPF 앱
                             EMQX, HiveMQ           모니터링 앱
```

대표적인 MQTT 브로커:
- **Mosquitto**: 오픈소스, 경량 (개발/테스트용)
- **EMQX**: 고성능, 클러스터 지원 (운영용)
- **HiveMQ**: 기업용, 클라우드 지원

### 3.2 Topic (토픽)

토픽은 메시지를 분류하는 **계층적 문자열**입니다. 파일 경로와 비슷하게 `/`로 구분합니다.

```
building/1f/temperature     ← 1층 온도
building/2f/temperature     ← 2층 온도
building/1f/humidity        ← 1층 습도
device/door-01/event        ← 출입문 1번 이벤트
device/door-02/event        ← 출입문 2번 이벤트
```

#### 와일드카드

| 와일드카드 | 설명 | 예시 |
|-----------|------|------|
| `+` | 한 레벨 와일드카드 | `building/+/temperature` → 모든 층의 온도 |
| `#` | 다중 레벨 와일드카드 | `building/#` → building 하위 모든 토픽 |

```csharp
// 구독 예시
"device/+/event"     // 모든 장비의 이벤트 구독
"building/1f/#"      // 1층의 모든 데이터 구독
"system/#"           // 시스템 관련 모든 토픽 구독
```

### 3.3 Publish / Subscribe (발행 / 구독)

```
[발행자]                        [브로커]                      [구독자]
   │                               │                            │
   │  PUBLISH                      │                            │
   │  topic: "device/door-01/event"│                            │
   │  payload: {"type":"entry"...} │                            │
   │──────────────────────────────→│                            │
   │                               │  이 토픽을 구독한 클라이언트│
   │                               │  에게 메시지를 전달         │
   │                               │───────────────────────────→│
   │                               │                            │
   │                               │            SUBSCRIBE       │
   │                               │←───────────────────────────│
   │                               │  topic: "device/+/event"   │
   │                               │  (와일드카드 구독)         │
```

### 3.4 QoS (Quality of Service)

QoS는 메시지 전달의 **신뢰성 수준**을 결정합니다.

| QoS | 이름 | 설명 | 사용 예 |
|-----|------|------|--------|
| **0** | At most once | 최대 1번 전달 (손실 가능) | 센서 데이터 (빈번하게 전송) |
| **1** | At least once | 최소 1번 전달 (중복 가능) | 이벤트 알림 (일반적 선택) |
| **2** | Exactly once | 정확히 1번 전달 | 결제, 중요 명령 |

```csharp
// QoS 선택 가이드
MqttQualityOfServiceLevel.AtMostOnce   // QoS 0 — 빠르지만 손실 가능
MqttQualityOfServiceLevel.AtLeastOnce  // QoS 1 — 대부분의 경우 권장
MqttQualityOfServiceLevel.ExactlyOnce  // QoS 2 — 느리지만 정확
```

---

## 4. MQTT 서비스 구현

### 4.1 IMqttService 인터페이스

```csharp
// Services/IMqttService.cs

namespace MyApp.Services;

/// <summary>
/// MQTT 클라이언트 서비스 인터페이스.
/// 연결, 구독, 발행, 메시지 수신 기능을 제공합니다.
/// </summary>
public interface IMqttService : IAsyncDisposable
{
    /// <summary>
    /// 현재 연결 상태.
    /// UI에서 바인딩하여 연결 상태를 표시할 수 있습니다.
    /// </summary>
    bool IsConnected { get; }

    /// <summary>
    /// 연결 상태가 변경될 때 발생하는 이벤트.
    /// </summary>
    event EventHandler<bool>? ConnectionStateChanged;

    /// <summary>
    /// MQTT 메시지를 수신했을 때 발생하는 이벤트.
    /// </summary>
    event EventHandler<MqttMessageReceivedEventArgs>? MessageReceived;

    /// <summary>
    /// MQTT 브로커에 연결합니다.
    /// 자동 재연결이 포함되어 있습니다.
    /// </summary>
    Task ConnectAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// MQTT 브로커 연결을 해제합니다.
    /// </summary>
    Task DisconnectAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// 토픽을 구독합니다.
    /// </summary>
    /// <param name="topic">구독할 토픽 (와일드카드 사용 가능)</param>
    /// <param name="qos">QoS 레벨</param>
    Task SubscribeAsync(string topic, MqttQualityOfServiceLevel qos = MqttQualityOfServiceLevel.AtLeastOnce, CancellationToken cancellationToken = default);

    /// <summary>
    /// 메시지를 발행합니다.
    /// </summary>
    /// <param name="topic">발행할 토픽</param>
    /// <param name="payload">메시지 내용</param>
    /// <param name="qos">QoS 레벨</param>
    Task PublishAsync(string topic, string payload, MqttQualityOfServiceLevel qos = MqttQualityOfServiceLevel.AtLeastOnce, CancellationToken cancellationToken = default);
}
```

### 4.2 MqttMessageReceivedEventArgs

```csharp
// Services/MqttMessageReceivedEventArgs.cs

namespace MyApp.Services;

/// <summary>
/// MQTT 메시지 수신 이벤트 인자.
/// </summary>
public sealed class MqttMessageReceivedEventArgs : EventArgs
{
    /// <summary>메시지가 발행된 토픽</summary>
    public required string Topic { get; init; }

    /// <summary>메시지 내용 (문자열)</summary>
    public required string Payload { get; init; }

    /// <summary>수신 시각</summary>
    public DateTime ReceivedAt { get; init; } = DateTime.Now;

    /// <summary>QoS 레벨</summary>
    public MqttQualityOfServiceLevel QualityOfServiceLevel { get; init; }
}
```

### 4.3 MqttService 구현

```csharp
// Services/MqttService.cs

using System.Text;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using MQTTnet;
using MQTTnet.Protocol;

namespace MyApp.Services;

/// <summary>
/// MQTTnet 5.x 기반 MQTT 서비스 구현.
/// 자동 재연결, 토픽 구독/발행, 연결 상태 관리를 처리합니다.
/// DI에서 Singleton으로 등록해야 합니다.
/// </summary>
public sealed class MqttService : IMqttService
{
    // ── 의존성 ──
    private readonly MqttSettings _settings;
    private readonly ILogger<MqttService> _logger;

    // ── MQTTnet 클라이언트 ──
    private IMqttClient? _mqttClient;

    // ── 자동 재연결을 위한 타이머 ──
    private Timer? _reconnectTimer;

    // ── 스레드 안전을 위한 락 ──
    private readonly SemaphoreSlim _connectLock = new(1, 1);

    // ── 구독한 토픽 목록 (재연결 시 다시 구독하기 위해 저장) ──
    private readonly List<(string Topic, MqttQualityOfServiceLevel Qos)> _subscribedTopics = [];

    // ── 연결 상태 ──
    private bool _isConnected;
    public bool IsConnected
    {
        get => _isConnected;
        private set
        {
            if (_isConnected != value)
            {
                _isConnected = value;
                // 상태 변경 이벤트 발생
                ConnectionStateChanged?.Invoke(this, value);
            }
        }
    }

    // ── 이벤트 ──
    public event EventHandler<bool>? ConnectionStateChanged;
    public event EventHandler<MqttMessageReceivedEventArgs>? MessageReceived;

    /// <summary>
    /// 생성자 — DI에서 설정과 로거를 주입받습니다.
    /// </summary>
    public MqttService(
        IOptions<MqttSettings> options,
        ILogger<MqttService> logger)
    {
        _settings = options.Value;
        _logger = logger;
    }

    // ─────────────────────────────────────────────
    // 연결
    // ─────────────────────────────────────────────

    /// <summary>
    /// MQTT 브로커에 연결합니다.
    /// 이미 연결되어 있으면 무시합니다.
    /// 연결 실패 시 자동 재연결 타이머를 시작합니다.
    /// </summary>
    public async Task ConnectAsync(CancellationToken cancellationToken = default)
    {
        // 동시에 여러 번 연결 시도를 방지
        await _connectLock.WaitAsync(cancellationToken);
        try
        {
            if (_mqttClient?.IsConnected == true)
            {
                _logger.LogDebug("이미 MQTT 브로커에 연결되어 있음");
                return;
            }

            _logger.LogInformation(
                "MQTT 브로커 연결 시도: {Host}:{Port}",
                _settings.Host, _settings.Port);

            // MQTTnet 5.x 클라이언트 생성
            var factory = new MqttClientFactory();
            _mqttClient = factory.CreateMqttClient();

            // 메시지 수신 이벤트 핸들러 등록
            _mqttClient.ApplicationMessageReceivedAsync += OnMessageReceivedAsync;

            // 연결 끊김 이벤트 핸들러 등록
            _mqttClient.DisconnectedAsync += OnDisconnectedAsync;

            // 연결 옵션 구성
            var options = new MqttClientOptionsBuilder()
                // 브로커 주소 및 포트
                .WithTcpServer(_settings.Host, _settings.Port)
                // 클라이언트 ID (고유해야 함)
                .WithClientId(_settings.ClientId)
                // MQTT 프로토콜 버전 (v5)
                .WithProtocolVersion(MQTTnet.Formatter.MqttProtocolVersion.V500)
                // 인증 (설정에 있는 경우)
                .WithCredentials(_settings.Username, _settings.Password)
                // Keep Alive 간격 (브로커가 연결 살아있음을 확인하는 주기)
                .WithKeepAlivePeriod(TimeSpan.FromSeconds(_settings.KeepAliveSeconds))
                // Clean Start — true면 이전 세션 정보 삭제
                .WithCleanStart(true)
                .Build();

            // 연결 실행
            var result = await _mqttClient.ConnectAsync(options, cancellationToken);

            if (result.ResultCode == MqttClientConnectResultCode.Success)
            {
                IsConnected = true;
                _logger.LogInformation("MQTT 브로커 연결 성공: {Host}:{Port}",
                    _settings.Host, _settings.Port);

                // 자동 재연결 타이머 중지 (연결 성공했으므로)
                StopReconnectTimer();

                // 이전에 구독했던 토픽 다시 구독
                await ResubscribeAsync(cancellationToken);
            }
            else
            {
                _logger.LogWarning(
                    "MQTT 브로커 연결 실패: ResultCode={ResultCode}",
                    result.ResultCode);
                StartReconnectTimer();
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "MQTT 브로커 연결 중 오류 발생");
            StartReconnectTimer();
        }
        finally
        {
            _connectLock.Release();
        }
    }

    // ─────────────────────────────────────────────
    // 연결 해제
    // ─────────────────────────────────────────────

    /// <summary>
    /// MQTT 브로커 연결을 해제합니다.
    /// </summary>
    public async Task DisconnectAsync(CancellationToken cancellationToken = default)
    {
        StopReconnectTimer();

        if (_mqttClient?.IsConnected == true)
        {
            _logger.LogInformation("MQTT 브로커 연결 해제 시작");

            try
            {
                var disconnectOptions = new MqttClientDisconnectOptionsBuilder()
                    .WithReason(MqttClientDisconnectOptionsReason.NormalDisconnection)
                    .Build();

                await _mqttClient.DisconnectAsync(disconnectOptions, cancellationToken);

                _logger.LogInformation("MQTT 브로커 연결 해제 완료");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "MQTT 연결 해제 중 오류");
            }
        }

        IsConnected = false;
    }

    // ─────────────────────────────────────────────
    // 자동 재연결
    // ─────────────────────────────────────────────

    /// <summary>
    /// 자동 재연결 타이머를 시작합니다.
    /// 설정된 간격(기본 5초)마다 연결을 시도합니다.
    /// </summary>
    private void StartReconnectTimer()
    {
        if (_reconnectTimer is not null) return;

        var interval = TimeSpan.FromSeconds(_settings.ReconnectIntervalSeconds);

        _logger.LogInformation(
            "자동 재연결 타이머 시작: {Interval}초 간격",
            _settings.ReconnectIntervalSeconds);

        _reconnectTimer = new Timer(
            callback: async _ =>
            {
                _logger.LogDebug("자동 재연결 시도 중...");
                try
                {
                    await ConnectAsync();
                }
                catch (Exception ex)
                {
                    _logger.LogWarning(ex, "자동 재연결 실패, 다음 시도까지 대기");
                }
            },
            state: null,
            dueTime: interval,   // 첫 실행까지 대기 시간
            period: interval);   // 반복 간격
    }

    /// <summary>
    /// 자동 재연결 타이머를 중지합니다.
    /// </summary>
    private void StopReconnectTimer()
    {
        if (_reconnectTimer is not null)
        {
            _reconnectTimer.Dispose();
            _reconnectTimer = null;
            _logger.LogDebug("자동 재연결 타이머 중지");
        }
    }

    /// <summary>
    /// 연결이 끊겼을 때 호출되는 이벤트 핸들러.
    /// </summary>
    private Task OnDisconnectedAsync(MqttClientDisconnectedEventArgs args)
    {
        IsConnected = false;

        if (args.ClientWasConnected)
        {
            // 이전에 연결되어 있었는데 끊긴 경우 — 자동 재연결 시작
            _logger.LogWarning(
                "MQTT 연결 끊김. 이유: {Reason}. 자동 재연결 시도 예정",
                args.Reason);
            StartReconnectTimer();
        }
        else
        {
            _logger.LogDebug("MQTT 연결 해제됨 (의도적)");
        }

        return Task.CompletedTask;
    }

    // ─────────────────────────────────────────────
    // 토픽 구독
    // ─────────────────────────────────────────────

    /// <summary>
    /// 토픽을 구독합니다.
    /// 구독한 토픽 목록을 저장하여 재연결 시 자동으로 다시 구독합니다.
    /// </summary>
    public async Task SubscribeAsync(
        string topic,
        MqttQualityOfServiceLevel qos = MqttQualityOfServiceLevel.AtLeastOnce,
        CancellationToken cancellationToken = default)
    {
        if (_mqttClient?.IsConnected != true)
        {
            _logger.LogWarning("MQTT 미연결 상태에서 구독 시도: {Topic}", topic);
            // 구독 목록에는 저장 (연결 후 자동 구독됨)
            AddToSubscribedTopics(topic, qos);
            return;
        }

        _logger.LogInformation("토픽 구독: {Topic}, QoS={QoS}", topic, qos);

        try
        {
            var subscribeOptions = new MqttClientSubscribeOptionsBuilder()
                .WithTopicFilter(f => f
                    .WithTopic(topic)
                    .WithQualityOfServiceLevel(qos))
                .Build();

            var result = await _mqttClient.SubscribeAsync(
                subscribeOptions, cancellationToken);

            // 구독 결과 확인
            foreach (var item in result.Items)
            {
                _logger.LogInformation(
                    "토픽 구독 결과: {Topic}, ResultCode={ResultCode}",
                    item.TopicFilter.Topic, item.ResultCode);
            }

            // 구독 목록에 추가 (재연결 시 사용)
            AddToSubscribedTopics(topic, qos);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "토픽 구독 실패: {Topic}", topic);
            throw;
        }
    }

    /// <summary>
    /// 구독 목록에 토픽을 추가합니다 (중복 방지).
    /// </summary>
    private void AddToSubscribedTopics(string topic, MqttQualityOfServiceLevel qos)
    {
        if (!_subscribedTopics.Exists(t => t.Topic == topic))
        {
            _subscribedTopics.Add((topic, qos));
        }
    }

    /// <summary>
    /// 재연결 후 이전에 구독했던 토픽을 다시 구독합니다.
    /// </summary>
    private async Task ResubscribeAsync(CancellationToken cancellationToken)
    {
        if (_subscribedTopics.Count == 0) return;

        _logger.LogInformation("재연결 후 토픽 재구독: {Count}개", _subscribedTopics.Count);

        foreach (var (topic, qos) in _subscribedTopics)
        {
            try
            {
                var subscribeOptions = new MqttClientSubscribeOptionsBuilder()
                    .WithTopicFilter(f => f
                        .WithTopic(topic)
                        .WithQualityOfServiceLevel(qos))
                    .Build();

                await _mqttClient!.SubscribeAsync(subscribeOptions, cancellationToken);
                _logger.LogDebug("토픽 재구독 완료: {Topic}", topic);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "토픽 재구독 실패: {Topic}", topic);
            }
        }
    }

    // ─────────────────────────────────────────────
    // 메시지 발행
    // ─────────────────────────────────────────────

    /// <summary>
    /// 지정된 토픽에 메시지를 발행합니다.
    /// </summary>
    public async Task PublishAsync(
        string topic,
        string payload,
        MqttQualityOfServiceLevel qos = MqttQualityOfServiceLevel.AtLeastOnce,
        CancellationToken cancellationToken = default)
    {
        if (_mqttClient?.IsConnected != true)
        {
            _logger.LogWarning("MQTT 미연결 상태에서 발행 시도: {Topic}", topic);
            throw new InvalidOperationException("MQTT 브로커에 연결되어 있지 않습니다.");
        }

        _logger.LogDebug("메시지 발행: {Topic}, Payload 길이={PayloadLength}",
            topic, payload.Length);

        try
        {
            var message = new MqttApplicationMessageBuilder()
                .WithTopic(topic)
                .WithPayload(Encoding.UTF8.GetBytes(payload))
                .WithQualityOfServiceLevel(qos)
                // Retain: true로 설정하면 브로커가 마지막 메시지를 보관
                // 새 구독자가 연결되면 이 메시지를 즉시 받음
                .WithRetainFlag(false)
                .Build();

            var result = await _mqttClient.PublishAsync(message, cancellationToken);

            if (result.IsSuccess)
            {
                _logger.LogInformation(
                    "메시지 발행 성공: {Topic}, QoS={QoS}",
                    topic, qos);
            }
            else
            {
                _logger.LogWarning(
                    "메시지 발행 실패: {Topic}, ReasonCode={ReasonCode}",
                    topic, result.ReasonCode);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "메시지 발행 오류: {Topic}", topic);
            throw;
        }
    }

    // ─────────────────────────────────────────────
    // 메시지 수신
    // ─────────────────────────────────────────────

    /// <summary>
    /// 메시지를 수신했을 때 호출되는 내부 핸들러.
    /// 수신한 메시지를 이벤트로 전파합니다.
    /// </summary>
    private Task OnMessageReceivedAsync(MqttApplicationMessageReceivedEventArgs args)
    {
        var topic = args.ApplicationMessage.Topic;
        var payload = Encoding.UTF8.GetString(args.ApplicationMessage.PayloadSegment);

        _logger.LogDebug(
            "메시지 수신: Topic={Topic}, Payload 길이={PayloadLength}",
            topic, payload.Length);

        // 이벤트를 발생시켜 ViewModel 등에서 처리하게 함
        // 주의: 이 콜백은 MQTT 스레드에서 실행됨 (UI 스레드가 아님!)
        MessageReceived?.Invoke(this, new MqttMessageReceivedEventArgs
        {
            Topic = topic,
            Payload = payload,
            ReceivedAt = DateTime.Now,
            QualityOfServiceLevel = args.ApplicationMessage.QualityOfServiceLevel
        });

        return Task.CompletedTask;
    }

    // ─────────────────────────────────────────────
    // 리소스 정리
    // ─────────────────────────────────────────────

    /// <summary>
    /// 리소스를 정리합니다 (IAsyncDisposable 구현).
    /// 앱 종료 시 호출됩니다.
    /// </summary>
    public async ValueTask DisposeAsync()
    {
        _logger.LogInformation("MqttService 리소스 정리 시작");

        StopReconnectTimer();

        if (_mqttClient is not null)
        {
            if (_mqttClient.IsConnected)
            {
                try
                {
                    await DisconnectAsync();
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "MQTT 연결 해제 중 오류 (Dispose)");
                }
            }

            _mqttClient.Dispose();
            _mqttClient = null;
        }

        _connectLock.Dispose();

        _logger.LogInformation("MqttService 리소스 정리 완료");
    }
}
```

---

## 5. DI에 등록 (Singleton)

MQTT 서비스는 앱 전체에서 **하나의 연결만 유지**해야 하므로 **Singleton**으로 등록합니다.

```csharp
// App.xaml.cs — DI 설정 부분

services.Configure<MqttSettings>(
    context.Configuration.GetSection(MqttSettings.SectionName));

// Singleton으로 등록 — 앱 전체에서 하나의 MQTT 연결을 공유
// IMqttService와 IAsyncDisposable 두 인터페이스 모두 같은 인스턴스를 반환
services.AddSingleton<IMqttService, MqttService>();
```

### 왜 Singleton인가?

| Lifetime | MQTT 서비스에 적합? | 이유 |
|----------|---------------------|------|
| Transient | 부적합 | 매번 새 연결을 만들면 브로커 부하 증가 |
| Scoped | 부적합 | WPF에서는 Scope 개념이 약함 |
| **Singleton** | **적합** | 하나의 연결로 모든 토픽을 관리 |

---

## 6. ViewModel에서 MQTT 사용

### 6.1 ViewModel 구현

```csharp
// ViewModels/MqttMonitorViewModel.cs

using System.Collections.ObjectModel;
using System.Windows.Threading;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.Logging;
using MQTTnet.Protocol;

namespace MyApp.ViewModels;

/// <summary>
/// MQTT 모니터링 ViewModel.
/// 연결 상태 표시, 메시지 수신/발행을 관리합니다.
/// </summary>
public partial class MqttMonitorViewModel : ObservableObject, IDisposable
{
    private readonly IMqttService _mqttService;
    private readonly ILogger<MqttMonitorViewModel> _logger;

    // WPF의 Dispatcher — UI 스레드에서 작업을 실행하기 위해 필요
    private readonly Dispatcher _dispatcher;

    public MqttMonitorViewModel(
        IMqttService mqttService,
        ILogger<MqttMonitorViewModel> logger)
    {
        _mqttService = mqttService;
        _logger = logger;

        // 현재 UI 스레드의 Dispatcher를 저장
        _dispatcher = Dispatcher.CurrentDispatcher;

        // MQTT 이벤트 구독
        _mqttService.ConnectionStateChanged += OnConnectionStateChanged;
        _mqttService.MessageReceived += OnMessageReceived;

        // 초기 연결 상태 반영
        IsConnected = _mqttService.IsConnected;

        _logger.LogDebug("MqttMonitorViewModel 인스턴스 생성");
    }

    // ── 속성 ──

    /// <summary>MQTT 연결 상태</summary>
    [ObservableProperty]
    private bool _isConnected;

    /// <summary>연결 상태 텍스트 (UI 표시용)</summary>
    [ObservableProperty]
    private string _connectionStatus = "미연결";

    /// <summary>수신된 메시지 목록 (UI에 표시)</summary>
    public ObservableCollection<MqttMessageItem> ReceivedMessages { get; } = [];

    /// <summary>발행할 토픽</summary>
    [ObservableProperty]
    private string _publishTopic = "test/message";

    /// <summary>발행할 메시지 내용</summary>
    [ObservableProperty]
    private string _publishPayload = string.Empty;

    /// <summary>구독할 토픽</summary>
    [ObservableProperty]
    private string _subscribeTopic = "device/+/event";

    /// <summary>상태 메시지</summary>
    [ObservableProperty]
    private string _statusMessage = string.Empty;

    // ── 연결 상태 변경 처리 ──

    /// <summary>
    /// 연결 상태가 변경될 때 호출됩니다.
    /// MQTT 스레드에서 호출되므로 Dispatcher를 통해 UI를 갱신해야 합니다.
    /// </summary>
    private void OnConnectionStateChanged(object? sender, bool isConnected)
    {
        // ★ 핵심: Dispatcher.Invoke로 UI 스레드에서 실행
        // MQTT 이벤트는 백그라운드 스레드에서 발생하므로,
        // ObservableProperty를 직접 변경하면 예외가 발생합니다.
        _dispatcher.Invoke(() =>
        {
            IsConnected = isConnected;
            ConnectionStatus = isConnected ? "연결됨" : "연결 끊김";
            StatusMessage = isConnected
                ? "MQTT 브로커에 연결되었습니다."
                : "MQTT 브로커 연결이 끊겼습니다. 재연결 시도 중...";
        });
    }

    // ── 메시지 수신 처리 ──

    /// <summary>
    /// MQTT 메시지를 수신했을 때 호출됩니다.
    /// 역시 MQTT 스레드에서 호출되므로 Dispatcher 필요!
    /// </summary>
    private void OnMessageReceived(object? sender, MqttMessageReceivedEventArgs args)
    {
        _logger.LogDebug(
            "ViewModel에서 메시지 수신: Topic={Topic}",
            args.Topic);

        // ★ 핵심: Dispatcher.Invoke로 UI 스레드에서 ObservableCollection에 추가
        _dispatcher.Invoke(() =>
        {
            ReceivedMessages.Insert(0, new MqttMessageItem
            {
                Topic = args.Topic,
                Payload = args.Payload,
                ReceivedAt = args.ReceivedAt
            });

            // 최대 100개만 유지 (메모리 관리)
            while (ReceivedMessages.Count > 100)
            {
                ReceivedMessages.RemoveAt(ReceivedMessages.Count - 1);
            }

            StatusMessage = $"메시지 수신: {args.Topic}";
        });
    }

    // ── 명령 ──

    /// <summary>
    /// MQTT 브로커에 연결합니다.
    /// </summary>
    [RelayCommand]
    private async Task ConnectAsync()
    {
        try
        {
            StatusMessage = "연결 중...";
            await _mqttService.ConnectAsync();
        }
        catch (Exception ex)
        {
            StatusMessage = $"연결 실패: {ex.Message}";
            _logger.LogError(ex, "MQTT 연결 실패");
        }
    }

    /// <summary>
    /// MQTT 브로커 연결을 해제합니다.
    /// </summary>
    [RelayCommand]
    private async Task DisconnectAsync()
    {
        try
        {
            await _mqttService.DisconnectAsync();
            StatusMessage = "연결 해제됨";
        }
        catch (Exception ex)
        {
            StatusMessage = $"연결 해제 실패: {ex.Message}";
            _logger.LogError(ex, "MQTT 연결 해제 실패");
        }
    }

    /// <summary>
    /// 토픽을 구독합니다.
    /// </summary>
    [RelayCommand]
    private async Task SubscribeAsync()
    {
        if (string.IsNullOrWhiteSpace(SubscribeTopic))
        {
            StatusMessage = "구독할 토픽을 입력하세요.";
            return;
        }

        try
        {
            await _mqttService.SubscribeAsync(
                SubscribeTopic,
                MqttQualityOfServiceLevel.AtLeastOnce);

            StatusMessage = $"토픽 구독 완료: {SubscribeTopic}";
        }
        catch (Exception ex)
        {
            StatusMessage = $"구독 실패: {ex.Message}";
            _logger.LogError(ex, "토픽 구독 실패: {Topic}", SubscribeTopic);
        }
    }

    /// <summary>
    /// 메시지를 발행합니다.
    /// </summary>
    [RelayCommand]
    private async Task PublishAsync()
    {
        if (string.IsNullOrWhiteSpace(PublishTopic) ||
            string.IsNullOrWhiteSpace(PublishPayload))
        {
            StatusMessage = "토픽과 메시지를 입력하세요.";
            return;
        }

        try
        {
            await _mqttService.PublishAsync(
                PublishTopic,
                PublishPayload,
                MqttQualityOfServiceLevel.AtLeastOnce);

            StatusMessage = $"메시지 발행 완료: {PublishTopic}";
            PublishPayload = string.Empty; // 입력 필드 초기화
        }
        catch (Exception ex)
        {
            StatusMessage = $"발행 실패: {ex.Message}";
            _logger.LogError(ex, "메시지 발행 실패: {Topic}", PublishTopic);
        }
    }

    // ── 리소스 정리 ──

    /// <summary>
    /// ViewModel이 소멸될 때 이벤트 구독을 해제합니다.
    /// 이벤트를 해제하지 않으면 메모리 누수가 발생합니다.
    /// </summary>
    public void Dispose()
    {
        _mqttService.ConnectionStateChanged -= OnConnectionStateChanged;
        _mqttService.MessageReceived -= OnMessageReceived;
        _logger.LogDebug("MqttMonitorViewModel 리소스 정리 완료");
    }
}
```

### 6.2 메시지 표시용 모델

```csharp
// Models/MqttMessageItem.cs

namespace MyApp.Models;

/// <summary>
/// UI에 표시할 MQTT 메시지 항목.
/// </summary>
public sealed class MqttMessageItem
{
    /// <summary>메시지 토픽</summary>
    public string Topic { get; init; } = string.Empty;

    /// <summary>메시지 내용</summary>
    public string Payload { get; init; } = string.Empty;

    /// <summary>수신 시각</summary>
    public DateTime ReceivedAt { get; init; }

    /// <summary>수신 시각 포맷 (UI 표시용)</summary>
    public string ReceivedAtFormatted => ReceivedAt.ToString("HH:mm:ss.fff");
}
```

### 6.3 Dispatcher 사용 주의사항

MQTT 메시지 수신은 **백그라운드 스레드(MQTT I/O 스레드)**에서 발생합니다. WPF에서는 **UI 요소를 UI 스레드에서만** 변경할 수 있으므로, 반드시 `Dispatcher`를 사용해야 합니다.

```csharp
// ❌ MQTT 스레드에서 직접 UI 속성 변경 — 예외 발생!
private void OnMessageReceived(object? sender, MqttMessageReceivedEventArgs args)
{
    // System.InvalidOperationException:
    // "The calling thread cannot access this object because a different thread owns it."
    ReceivedMessages.Add(new MqttMessageItem { ... });
}

// ✅ Dispatcher를 통해 UI 스레드에서 실행
private void OnMessageReceived(object? sender, MqttMessageReceivedEventArgs args)
{
    _dispatcher.Invoke(() =>
    {
        // 이 블록은 UI 스레드에서 실행됨 — 안전
        ReceivedMessages.Add(new MqttMessageItem { ... });
    });
}

// ✅ 대안: Dispatcher.InvokeAsync (비동기, UI를 블로킹하지 않음)
private void OnMessageReceived(object? sender, MqttMessageReceivedEventArgs args)
{
    _dispatcher.InvokeAsync(() =>
    {
        ReceivedMessages.Add(new MqttMessageItem { ... });
    });
}
```

**Invoke vs InvokeAsync:**

| 메서드 | 동작 | 사용 시점 |
|--------|------|----------|
| `Invoke` | UI 스레드에서 실행될 때까지 대기 | 결과가 필요한 경우 |
| `InvokeAsync` | 즉시 반환, UI 스레드에서 나중에 실행 | 대부분의 경우 (권장) |

---

## 7. WPF 앱 종료 시 연결 정리

MQTT 연결을 제대로 정리하지 않으면 브로커 측에서 비정상 종료로 감지합니다. 앱 종료 시 반드시 연결을 해제해야 합니다.

### App.xaml.cs에서 정리

```csharp
// App.xaml.cs

protected override async void OnExit(ExitEventArgs e)
{
    _logger.LogInformation("앱 종료 시작 — 리소스 정리 중");

    // ── MQTT 연결 정리 ──
    // IAsyncDisposable을 구현했으므로 DisposeAsync() 호출
    var mqttService = _host.Services.GetService<IMqttService>();
    if (mqttService is IAsyncDisposable disposable)
    {
        await disposable.DisposeAsync();
    }

    // ── Host 정리 ──
    await _host.StopAsync();
    _host.Dispose();

    // ── 로그 정리 ──
    await Log.CloseAndFlushAsync();

    base.OnExit(e);
}
```

### IDisposable vs IAsyncDisposable

```csharp
// MqttService는 IAsyncDisposable을 구현
// → 비동기 정리가 필요 (네트워크 연결 해제)
public sealed class MqttService : IMqttService, IAsyncDisposable
{
    public async ValueTask DisposeAsync()
    {
        // 비동기로 연결 해제
        await DisconnectAsync();
        _mqttClient?.Dispose();
    }
}

// ViewModel은 IDisposable을 구현
// → 동기 정리만 필요 (이벤트 구독 해제)
public partial class MqttMonitorViewModel : ObservableObject, IDisposable
{
    public void Dispose()
    {
        // 이벤트 구독 해제 (동기)
        _mqttService.ConnectionStateChanged -= OnConnectionStateChanged;
        _mqttService.MessageReceived -= OnMessageReceived;
    }
}
```

---

## 8. appsettings.json에서 MQTT 설정 관리

### 설정 파일

```jsonc
// appsettings.json
{
  "MqttSettings": {
    "Host": "localhost",
    "Port": 1883,
    "ClientId": "wpf-app-001",
    "Username": "",
    "Password": "",
    "KeepAliveSeconds": 60,
    "ReconnectIntervalSeconds": 5,
    "DefaultTopics": [
      "device/+/event",
      "system/status"
    ]
  }
}
```

### 설정 클래스

```csharp
// Models/Settings/MqttSettings.cs

namespace MyApp.Models.Settings;

/// <summary>
/// MQTT 연결 설정.
/// appsettings.json의 "MqttSettings" 섹션과 매핑됩니다.
/// </summary>
public sealed class MqttSettings
{
    public const string SectionName = "MqttSettings";

    /// <summary>MQTT 브로커 호스트 주소</summary>
    public string Host { get; set; } = "localhost";

    /// <summary>MQTT 브로커 포트 (기본 1883, TLS는 8883)</summary>
    public int Port { get; set; } = 1883;

    /// <summary>
    /// MQTT 클라이언트 ID.
    /// 브로커에서 클라이언트를 식별하는 고유 값입니다.
    /// 같은 ClientId로 두 클라이언트가 연결하면 먼저 연결된 쪽이 끊어집니다.
    /// </summary>
    public string ClientId { get; set; } = $"wpf-{Environment.MachineName}";

    /// <summary>인증 사용자명 (없으면 빈 문자열)</summary>
    public string Username { get; set; } = string.Empty;

    /// <summary>인증 비밀번호 (없으면 빈 문자열)</summary>
    public string Password { get; set; } = string.Empty;

    /// <summary>
    /// Keep Alive 간격(초).
    /// 이 시간 동안 아무 통신이 없으면 PING 패킷을 보내 연결을 유지합니다.
    /// </summary>
    public int KeepAliveSeconds { get; set; } = 60;

    /// <summary>
    /// 자동 재연결 간격(초).
    /// 연결이 끊기면 이 간격으로 재연결을 시도합니다.
    /// </summary>
    public int ReconnectIntervalSeconds { get; set; } = 5;

    /// <summary>
    /// 앱 시작 시 자동으로 구독할 토픽 목록.
    /// </summary>
    public List<string> DefaultTopics { get; set; } = [];
}
```

### DI에 설정 등록

```csharp
// App.xaml.cs

services.Configure<MqttSettings>(
    context.Configuration.GetSection(MqttSettings.SectionName));
```

---

## 9. 전체 코드 예제

### 9.1 프로젝트 구조

```
MyApp/
├── App.xaml
├── App.xaml.cs              ← DI 설정 + 종료 시 정리
├── appsettings.json         ← MQTT 설정
├── Models/
│   ├── MqttMessageItem.cs   ← UI 표시용 메시지 모델
│   └── Settings/
│       └── MqttSettings.cs  ← MQTT 설정 클래스
├── Services/
│   ├── IMqttService.cs      ← MQTT 서비스 인터페이스
│   ├── MqttService.cs       ← MQTT 서비스 구현
│   └── MqttMessageReceivedEventArgs.cs  ← 이벤트 인자
├── ViewModels/
│   └── MqttMonitorViewModel.cs  ← ViewModel
└── Views/
    └── MqttMonitorWindow.xaml   ← View
```

### 9.2 App.xaml.cs — 전체 설정

```csharp
// App.xaml.cs — MQTT 포함 전체 설정

using System.Windows;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;

namespace MyApp;

public partial class App : Application
{
    private readonly IHost _host;

    public App()
    {
        // Serilog 설정
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Debug()
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
                // ── 설정 ──
                services.Configure<MqttSettings>(
                    context.Configuration.GetSection(MqttSettings.SectionName));

                // ── MQTT 서비스 (Singleton) ──
                services.AddSingleton<IMqttService, MqttService>();

                // ── ViewModel ──
                services.AddTransient<MqttMonitorViewModel>();

                // ── View ──
                services.AddTransient<MqttMonitorWindow>();
            })
            .Build();
    }

    protected override async void OnStartup(StartupEventArgs e)
    {
        await _host.StartAsync();

        // MQTT 연결 시작 (앱 시작과 동시에)
        var mqttService = _host.Services.GetRequiredService<IMqttService>();
        _ = Task.Run(async () =>
        {
            try
            {
                await mqttService.ConnectAsync();
            }
            catch (Exception ex)
            {
                Log.Error(ex, "MQTT 초기 연결 실패 — 자동 재연결이 시도됩니다");
            }
        });

        var mainWindow = _host.Services.GetRequiredService<MqttMonitorWindow>();
        mainWindow.Show();

        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        Log.Information("앱 종료 — 리소스 정리 중");

        // MQTT 연결 해제
        var mqttService = _host.Services.GetRequiredService<IMqttService>();
        if (mqttService is IAsyncDisposable disposable)
        {
            await disposable.DisposeAsync();
        }

        await _host.StopAsync();
        _host.Dispose();
        await Log.CloseAndFlushAsync();

        base.OnExit(e);
    }
}
```

### 9.3 View (XAML) — MQTT 모니터 화면

```xml
<!-- Views/MqttMonitorWindow.xaml -->
<Window x:Class="MyApp.Views.MqttMonitorWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="MQTT 모니터" Width="900" Height="700">

    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>  <!-- 연결 상태 + 버튼 -->
            <RowDefinition Height="Auto"/>  <!-- 구독 -->
            <RowDefinition Height="Auto"/>  <!-- 발행 -->
            <RowDefinition Height="*"/>     <!-- 수신 메시지 목록 -->
            <RowDefinition Height="Auto"/>  <!-- 상태 바 -->
        </Grid.RowDefinitions>

        <!-- ═══════ 연결 상태 + 버튼 ═══════ -->
        <GroupBox Grid.Row="0" Header="연결 관리" Margin="0,0,0,8">
            <StackPanel Orientation="Horizontal" Margin="8">
                <!-- 연결 상태 표시 (원형 인디케이터) -->
                <Ellipse Width="12" Height="12" Margin="0,0,8,0">
                    <Ellipse.Style>
                        <Style TargetType="Ellipse">
                            <Setter Property="Fill" Value="Red"/>
                            <Style.Triggers>
                                <!-- IsConnected가 true면 초록색 -->
                                <DataTrigger Binding="{Binding IsConnected}" Value="True">
                                    <Setter Property="Fill" Value="Green"/>
                                </DataTrigger>
                            </Style.Triggers>
                        </Style>
                    </Ellipse.Style>
                </Ellipse>

                <TextBlock Text="{Binding ConnectionStatus}"
                           VerticalAlignment="Center" Margin="0,0,16,0"
                           FontWeight="Bold"/>

                <Button Content="연결" Command="{Binding ConnectCommand}"
                        Width="80" Margin="0,0,8,0"/>
                <Button Content="연결 해제" Command="{Binding DisconnectCommand}"
                        Width="80"/>
            </StackPanel>
        </GroupBox>

        <!-- ═══════ 토픽 구독 ═══════ -->
        <GroupBox Grid.Row="1" Header="토픽 구독" Margin="0,0,0,8">
            <StackPanel Orientation="Horizontal" Margin="8">
                <TextBlock Text="토픽:" VerticalAlignment="Center" Margin="0,0,8,0"/>
                <TextBox Text="{Binding SubscribeTopic, UpdateSourceTrigger=PropertyChanged}"
                         Width="300" Margin="0,0,8,0"/>
                <Button Content="구독" Command="{Binding SubscribeCommand}" Width="80"/>
            </StackPanel>
        </GroupBox>

        <!-- ═══════ 메시지 발행 ═══════ -->
        <GroupBox Grid.Row="2" Header="메시지 발행" Margin="0,0,0,8">
            <Grid Margin="8">
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="200"/>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="Auto"/>
                </Grid.ColumnDefinitions>

                <TextBlock Grid.Column="0" Text="토픽:" VerticalAlignment="Center"
                           Margin="0,0,8,0"/>
                <TextBox Grid.Column="1"
                         Text="{Binding PublishTopic, UpdateSourceTrigger=PropertyChanged}"
                         Margin="0,0,8,0"/>

                <TextBlock Grid.Column="2" Text="메시지:" VerticalAlignment="Center"
                           Margin="0,0,8,0"/>
                <TextBox Grid.Column="3"
                         Text="{Binding PublishPayload, UpdateSourceTrigger=PropertyChanged}"
                         Margin="0,0,8,0"/>

                <Button Grid.Column="4" Content="발행"
                        Command="{Binding PublishCommand}" Width="80"/>
            </Grid>
        </GroupBox>

        <!-- ═══════ 수신 메시지 목록 ═══════ -->
        <GroupBox Grid.Row="3" Header="수신 메시지">
            <ListView ItemsSource="{Binding ReceivedMessages}"
                      Margin="4">
                <ListView.View>
                    <GridView>
                        <GridViewColumn Header="시간" Width="100"
                                        DisplayMemberBinding="{Binding ReceivedAtFormatted}"/>
                        <GridViewColumn Header="토픽" Width="250"
                                        DisplayMemberBinding="{Binding Topic}"/>
                        <GridViewColumn Header="내용" Width="400"
                                        DisplayMemberBinding="{Binding Payload}"/>
                    </GridView>
                </ListView.View>
            </ListView>
        </GroupBox>

        <!-- ═══════ 상태 바 ═══════ -->
        <StatusBar Grid.Row="4" Margin="0,8,0,0">
            <TextBlock Text="{Binding StatusMessage}"/>
        </StatusBar>
    </Grid>
</Window>
```

### 9.4 View 코드비하인드

```csharp
// Views/MqttMonitorWindow.xaml.cs

namespace MyApp.Views;

public partial class MqttMonitorWindow : Window
{
    private readonly MqttMonitorViewModel _viewModel;

    public MqttMonitorWindow(MqttMonitorViewModel viewModel)
    {
        InitializeComponent();

        _viewModel = viewModel;
        DataContext = viewModel;

        // 윈도우 닫힐 때 ViewModel 정리
        Closed += (_, _) =>
        {
            _viewModel.Dispose();
        };
    }
}
```

### 9.5 동작 시나리오

#### 시나리오 1: 정상 연결 및 메시지 수신

```
1. 앱 시작
   → App.OnStartup()에서 MqttService.ConnectAsync() 호출
   → MQTT 브로커에 연결 성공
   → ConnectionStateChanged 이벤트 발생
   → ViewModel에서 IsConnected = true, 초록색 표시

2. 토픽 구독
   → 사용자가 "device/+/event" 입력 후 "구독" 클릭
   → MqttService.SubscribeAsync() 호출
   → 브로커가 구독 확인

3. 메시지 수신
   → 장비가 "device/door-01/event" 토픽에 메시지 발행
   → 브로커가 WPF 앱에 메시지 전달
   → MqttService.OnMessageReceivedAsync() → MessageReceived 이벤트
   → ViewModel.OnMessageReceived() → Dispatcher.Invoke()
   → ReceivedMessages에 추가 → UI 자동 갱신
```

#### 시나리오 2: 연결 끊김 + 자동 재연결

```
1. 네트워크 끊김 또는 브로커 재시작
   → MqttService.OnDisconnectedAsync() 호출
   → IsConnected = false → UI에 빨간색 표시
   → StartReconnectTimer() → 5초마다 재연결 시도

2. 네트워크 복구
   → 재연결 타이머에서 ConnectAsync() 성공
   → StopReconnectTimer()
   → ResubscribeAsync() — 이전에 구독했던 토픽 자동 재구독
   → IsConnected = true → UI에 초록색 표시
```

#### 시나리오 3: 앱 종료

```
1. 사용자가 창을 닫음
   → Window.Closed → ViewModel.Dispose() — 이벤트 구독 해제

2. App.OnExit()
   → MqttService.DisposeAsync()
     → StopReconnectTimer()
     → DisconnectAsync() — 정상적인 DISCONNECT 패킷 전송
     → _mqttClient.Dispose()
   → _host.StopAsync()
   → Log.CloseAndFlushAsync()
```

---

## 요약

| 항목 | 핵심 |
|------|------|
| **MQTT** | 경량 발행-구독 메시지 프로토콜 (IoT, 장비 통신) |
| **MQTTnet 5.x** | .NET 고성능 MQTT 클라이언트 라이브러리 |
| **토픽** | 계층적 문자열 (`device/door-01/event`), 와일드카드(`+`, `#`) 지원 |
| **QoS** | 0(최대 1번), 1(최소 1번, 권장), 2(정확히 1번) |
| **서비스 등록** | Singleton으로 등록 (앱 전체에서 하나의 연결 유지) |
| **자동 재연결** | Timer 기반, 연결 끊김 시 자동으로 재연결 시도 |
| **토픽 재구독** | 재연결 시 이전 구독 토픽을 자동으로 다시 구독 |
| **Dispatcher** | MQTT 스레드에서 UI 갱신 시 반드시 `Dispatcher.Invoke` 사용 |
| **리소스 정리** | `IAsyncDisposable`로 앱 종료 시 연결 정리 |
| **설정 외부화** | `appsettings.json`에서 호스트, 포트, 토픽 등 관리 |

---

> **다음 문서**: [AWS S3 저장소](./05-aws-s3.md) — S3에 사진을 업로드하고 다운로드하는 기능을 구현합니다.
