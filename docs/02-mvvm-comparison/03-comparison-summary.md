# 일반 MVVM vs CommunityToolkit.Mvvm 비교 요약

> **목표**: [수동 MVVM 구현](./01-traditional-mvvm.md)과 [CommunityToolkit.Mvvm](./02-communitytoolkit-mvvm.md)을
> 항목별로 나란히 비교하여, 왜 CommunityToolkit.Mvvm을 권장하는지 명확하게 정리합니다.

---

## 목차

1. [코드 양 비교](#1-코드-양-비교)
2. [Side-by-Side 코드 비교](#2-side-by-side-코드-비교)
3. [성능 비교](#3-성능-비교)
4. [학습 곡선 비교](#4-학습-곡선-비교)
5. [언제 전통 방식을 쓰는가?](#5-언제-전통-방식을-쓰는가)
6. [최종 권장: CommunityToolkit.Mvvm](#6-최종-권장-communitytoolkitmvvm)
7. [마이그레이션 가이드 (전통 → Toolkit)](#7-마이그레이션-가이드-전통--toolkit)

---

## 1. 코드 양 비교

### 1.1 같은 기능을 구현할 때 필요한 줄 수

사용자 정보 편집 화면 (프로퍼티 7개, 커맨드 2개, 유효성 검사 포함)을 기준으로 비교합니다:

| 항목 | 전통 방식 (줄 수) | Toolkit 방식 (줄 수) | 감소율 |
|------|:-:|:-:|:-:|
| 기본 클래스 (ViewModelBase / ObservableObject) | ~40줄 (직접 구현) | 0줄 (라이브러리 제공) | **-100%** |
| RelayCommand 클래스 | ~45줄 (직접 구현) | 0줄 (라이브러리 제공) | **-100%** |
| RelayCommand\<T\> 클래스 | ~50줄 (직접 구현) | 0줄 (라이브러리 제공) | **-100%** |
| AsyncRelayCommand 클래스 | ~55줄 (직접 구현) | 0줄 (라이브러리 제공) | **-100%** |
| UserViewModel (비즈니스 코드) | ~170줄 | ~100줄 | **-41%** |
| 유효성 검사 | ~50줄 추가 (직접 구현) | 데이터 어노테이션으로 통합 (0줄 추가) | **-100%** |
| **전체 합계** | **~410줄** | **~100줄** | **약 76% 감소** |

### 1.2 시각적 비교

```
전통 방식 (~410줄)
████████████████████████████████████████████████████████████████████████████████████

CommunityToolkit.Mvvm (~100줄)
████████████████████

→ 약 76%의 코드가 사라집니다!
```

### 1.3 프로퍼티 수에 따른 ViewModel 줄 수 증가

프로퍼티가 늘어날수록 차이는 더 벌어집니다:

| 프로퍼티 수 | 전통 방식 (ViewModel만) | Toolkit 방식 (ViewModel만) | 차이 |
|:-:|:-:|:-:|:-:|
| 5개 | ~120줄 | ~50줄 | 70줄 |
| 10개 | ~200줄 | ~70줄 | 130줄 |
| 20개 | ~360줄 | ~110줄 | 250줄 |
| 30개 | ~520줄 | ~150줄 | 370줄 |
| 50개 | ~840줄 | ~230줄 | **610줄** |

> 실무에서는 프로퍼티가 20~30개인 ViewModel이 흔합니다.
> 전통 방식은 500줄이 넘지만, Toolkit은 150줄 내외로 유지됩니다.

---

## 2. Side-by-Side 코드 비교

### 2.1 프로퍼티 선언

<table>
<tr>
<th>전통 방식 (수동 구현)</th>
<th>CommunityToolkit.Mvvm</th>
</tr>
<tr>
<td>

```csharp
// 백킹 필드
private string _name = string.Empty;

// 프로퍼티 (매번 이 패턴을 반복)
public string Name
{
    get => _name;
    set
    {
        if (SetProperty(ref _name, value))
        {
            OnPropertyChanged(nameof(DisplayText));
            OnPropertyChanged(nameof(CanSave));
        }
    }
}
```
**12줄** — 프로퍼티마다 반복

</td>
<td>

```csharp
// 특성 + 필드 선언이 전부
[ObservableProperty]
[NotifyPropertyChangedFor(nameof(DisplayText))]
[NotifyPropertyChangedFor(nameof(CanSave))]
private string _name = string.Empty;
```
**4줄** — 특성만 추가하면 끝

</td>
</tr>
</table>

### 2.2 커맨드 선언 (동기)

<table>
<tr>
<th>전통 방식 (수동 구현)</th>
<th>CommunityToolkit.Mvvm</th>
</tr>
<tr>
<td>

```csharp
// 1. RelayCommand 클래스를 직접 구현 (~45줄)
public class RelayCommand : ICommand
{
    private readonly Action<object?> _execute;
    private readonly Predicate<object?>? _canExecute;

    public RelayCommand(
        Action<object?> execute,
        Predicate<object?>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager
               .RequerySuggested += value;
        remove => CommandManager
               .RequerySuggested -= value;
    }

    public bool CanExecute(object? parameter)
        => _canExecute?.Invoke(parameter) ?? true;

    public void Execute(object? parameter)
        => _execute(parameter);
}

// 2. ViewModel에서 프로퍼티 선언
public RelayCommand ResetCommand { get; }

// 3. 생성자에서 초기화
public UserViewModel()
{
    ResetCommand = new RelayCommand(
        execute: _ => ResetForm(),
        canExecute: _ => !IsBusy
    );
}

// 4. 실행 메서드 정의
private void ResetForm()
{
    Name = string.Empty;
    Email = string.Empty;
}
```
**RelayCommand 클래스 ~45줄 + ViewModel ~15줄**

</td>
<td>

```csharp
// 메서드에 [RelayCommand]만 붙이면 됨!
// RelayCommand 클래스 구현 불필요!
// 프로퍼티 선언 불필요!
// 생성자 초기화 불필요!

[RelayCommand(CanExecute = nameof(CanResetForm))]
private void ResetForm()
{
    Name = string.Empty;
    Email = string.Empty;
}

private bool CanResetForm() => !IsBusy;

// → ResetFormCommand 프로퍼티 자동 생성
```
**10줄** — 비즈니스 로직만 작성

</td>
</tr>
</table>

### 2.3 커맨드 선언 (비동기)

<table>
<tr>
<th>전통 방식 (수동 구현)</th>
<th>CommunityToolkit.Mvvm</th>
</tr>
<tr>
<td>

```csharp
// 1. AsyncRelayCommand 직접 구현 (~55줄)
public class AsyncRelayCommand : ICommand
{
    private readonly Func<object?, Task> _execute;
    private readonly Predicate<object?>? _canExecute;
    private bool _isExecuting;

    // ... (생성자, CanExecute, Execute 등)

    public async void Execute(object? parameter)
    {
        _isExecuting = true;
        RaiseCanExecuteChanged();
        try
        {
            await _execute(parameter);
        }
        finally
        {
            _isExecuting = false;
            RaiseCanExecuteChanged();
        }
    }
    // ...
}

// 2. ViewModel에서 프로퍼티 선언
public AsyncRelayCommand SaveCommand { get; }

// 3. 생성자에서 초기화
SaveCommand = new AsyncRelayCommand(
    execute: async _ => await SaveUserAsync(),
    canExecute: _ => CanSave
);

// 4. 비동기 메서드 정의
private async Task SaveUserAsync()
{
    await Task.Delay(2000);
    // 저장 로직...
}
```
**AsyncRelayCommand ~55줄 + ViewModel ~20줄**

</td>
<td>

```csharp
// Task를 반환하면 자동으로
// AsyncRelayCommand가 생성됨!

[RelayCommand(CanExecute = nameof(CanSave))]
private async Task SaveUserAsync()
{
    await Task.Delay(2000);
    // 저장 로직...
}

// → SaveUserCommand 프로퍼티 자동 생성
// → IAsyncRelayCommand 타입
// → IsRunning, Cancel() 등 추가 기능 포함
```
**8줄** — 비즈니스 로직만 작성

</td>
</tr>
</table>

### 2.4 CanExecute 연동

<table>
<tr>
<th>전통 방식 (수동 구현)</th>
<th>CommunityToolkit.Mvvm</th>
</tr>
<tr>
<td>

```csharp
// 각 setter에서 수동으로 재평가 요청
public string Name
{
    get => _name;
    set
    {
        if (SetProperty(ref _name, value))
        {
            // SaveCommand의 CanExecute를
            // 수동으로 재평가해야 함
            // (빼먹으면 버튼 상태가 안 바뀜!)
            CommandManager
                .InvalidateRequerySuggested();
        }
    }
}

public bool IsBusy
{
    get => _isBusy;
    set
    {
        if (SetProperty(ref _isBusy, value))
        {
            // 여기도 수동으로...
            CommandManager
                .InvalidateRequerySuggested();
        }
    }
}
```
**각 setter에 수동 추가 (빼먹기 쉬움)**

</td>
<td>

```csharp
// 필드 선언부에 특성만 추가하면 끝!
[ObservableProperty]
[NotifyCanExecuteChangedFor(
    nameof(SaveUserCommand))]
private string _name = string.Empty;

[ObservableProperty]
[NotifyCanExecuteChangedFor(
    nameof(SaveUserCommand))]
[NotifyCanExecuteChangedFor(
    nameof(ResetFormCommand))]
private bool _isBusy;

// → Name이나 IsBusy가 바뀌면
//   SaveUserCommand.NotifyCanExecuteChanged()
//   가 자동으로 호출됨
```
**선언적이고 빼먹을 수 없음**

</td>
</tr>
</table>

### 2.5 유효성 검사

<table>
<tr>
<th>전통 방식 (수동 구현)</th>
<th>CommunityToolkit.Mvvm</th>
</tr>
<tr>
<td>

```csharp
// IDataErrorInfo 또는
// INotifyDataErrorInfo를 직접 구현해야 함

public class UserViewModel : ViewModelBase,
    INotifyDataErrorInfo
{
    private readonly Dictionary<string,
        List<string>> _errors = new();

    public bool HasErrors =>
        _errors.Any(e => e.Value.Count > 0);

    public event EventHandler<
        DataErrorsChangedEventArgs>?
        ErrorsChanged;

    public IEnumerable GetErrors(
        string? propertyName)
    {
        if (propertyName is null)
            return Enumerable.Empty<string>();
        return _errors.GetValueOrDefault(
            propertyName,
            new List<string>());
    }

    // 각 프로퍼티 setter에서 수동 검증
    public string Name
    {
        get => _name;
        set
        {
            SetProperty(ref _name, value);
            ValidateName();
        }
    }

    private void ValidateName()
    {
        _errors.Remove(nameof(Name));
        var errors = new List<string>();

        if (string.IsNullOrWhiteSpace(Name))
            errors.Add("이름은 필수입니다.");
        if (Name.Length < 2)
            errors.Add("최소 2자 이상");
        if (Name.Length > 50)
            errors.Add("50자 이하");

        if (errors.Any())
            _errors[nameof(Name)] = errors;

        ErrorsChanged?.Invoke(this,
            new DataErrorsChangedEventArgs(
                nameof(Name)));
    }
}
```
**유효성 검사 인프라 ~50줄 + 프로퍼티당 ~15줄**

</td>
<td>

```csharp
// ObservableValidator 상속 +
// 데이터 어노테이션만 추가하면 끝!

public partial class UserViewModel
    : ObservableValidator
{
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Required(
        ErrorMessage = "이름은 필수입니다.")]
    [MinLength(2,
        ErrorMessage = "최소 2자 이상")]
    [MaxLength(50,
        ErrorMessage = "50자 이하")]
    private string _name = string.Empty;
}

// INotifyDataErrorInfo가 자동 구현됨!
// HasErrors, GetErrors 등 모두 자동!
// ErrorsChanged 이벤트도 자동 발생!
```
**프로퍼티당 특성 4~5줄 추가 (인프라 코드 0줄)**

</td>
</tr>
</table>

### 2.6 ViewModel 간 메시징

<table>
<tr>
<th>전통 방식 (수동 구현)</th>
<th>CommunityToolkit.Mvvm</th>
</tr>
<tr>
<td>

```csharp
// 1. 이벤트 기반 직접 구현
// (또는 EventAggregator 패턴 직접 구현)

// 메시지 중개자 직접 구현 (~80줄 이상)
public class EventAggregator
{
    private static EventAggregator? _instance;
    public static EventAggregator Instance =>
        _instance ??= new();

    private readonly Dictionary<Type,
        List<Delegate>> _subscribers = new();

    public void Subscribe<T>(Action<T> handler)
    {
        var type = typeof(T);
        if (!_subscribers.ContainsKey(type))
            _subscribers[type] = new();
        _subscribers[type].Add(handler);
    }

    public void Publish<T>(T message)
    {
        var type = typeof(T);
        if (_subscribers.TryGetValue(
            type, out var handlers))
        {
            foreach (var handler in handlers)
                ((Action<T>)handler)(message);
        }
    }

    // Unsubscribe, 메모리 누수 관리 등
    // 직접 구현해야 함...
}

// 사용
EventAggregator.Instance
    .Publish(new UserSelected(userId));
EventAggregator.Instance
    .Subscribe<UserSelected>(
        msg => LoadDetail(msg.UserId));
```
**중개자 클래스 ~80줄 + 메모리 누수 위험**

</td>
<td>

```csharp
// WeakReferenceMessenger 내장!
// 약한 참조 기반 → 메모리 누수 방지

// 메시지 정의
public class UserSelectedMessage
    : ValueChangedMessage<int>
{
    public UserSelectedMessage(int userId)
        : base(userId) { }
}

// 보내기
WeakReferenceMessenger.Default.Send(
    new UserSelectedMessage(userId));

// 받기 (방법 1: 인터페이스)
public partial class DetailVM
    : ObservableObject,
      IRecipient<UserSelectedMessage>
{
    public DetailVM()
    {
        WeakReferenceMessenger.Default
            .Register(this);
    }

    public void Receive(
        UserSelectedMessage message)
    {
        LoadDetail(message.Value);
    }
}

// 받기 (방법 2: 람다)
WeakReferenceMessenger.Default
    .Register<UserSelectedMessage>(
        this, (r, m) =>
        LoadDetail(m.Value));
```
**메시지 클래스 ~5줄 + 사용 코드만 작성**

</td>
</tr>
</table>

---

## 3. 성능 비교

### 3.1 소스 제네레이터 → 런타임 오버헤드 없음

CommunityToolkit.Mvvm의 핵심 장점 중 하나는 **소스 제네레이터** 기반이라는 것입니다.

```
┌───────────────────────────────────────────────────────────────┐
│                        성능 비교                               │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  [전통 방식]                                                   │
│  ✅ 런타임 리플렉션 없음 (수동으로 코드를 작성했으므로)            │
│  ✅ 직접 작성한 코드이므로 최적화 가능                            │
│  ❌ 하지만 수동 코드의 품질은 개발자에 따라 다름                   │
│                                                               │
│  [CommunityToolkit.Mvvm — 소스 제네레이터]                     │
│  ✅ 런타임 리플렉션 없음 (컴파일 시점에 코드 생성)                │
│  ✅ Microsoft가 최적화한 코드 생성                               │
│  ✅ 수동 방식과 동일한 성능                                      │
│  ✅ 일관되게 최적화된 코드 보장                                   │
│                                                               │
│  [다른 MVVM 프레임워크 — 리플렉션 기반] (참고용)                 │
│  ❌ 런타임 리플렉션 사용 → 오버헤드 있음                          │
│  ❌ IL Weaving 방식도 빌드 시간 증가                             │
│                                                               │
│  결론: 전통 방식 ≈ CommunityToolkit.Mvvm (성능 동일!)           │
│  → Toolkit을 쓴다고 성능이 나빠지지 않음                         │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 3.2 소스 제네레이터 vs 리플렉션 vs IL Weaving

| 방식 | 코드 생성 시점 | 런타임 비용 | 대표 프레임워크 |
|------|:---:|:---:|------|
| 수동 구현 | 개발자가 직접 | 없음 | 전통 MVVM |
| **소스 제네레이터** | **컴파일 시점** | **없음** | **CommunityToolkit.Mvvm** |
| 리플렉션 | 런타임 | 있음 (느림) | 일부 구형 프레임워크 |
| IL Weaving | 빌드 후 처리 | 없음 (빌드 느려짐) | Fody/PropertyChanged |

> **핵심**: CommunityToolkit.Mvvm은 "편리함"과 "성능" 모두를 만족합니다.
> 편리하다고 해서 성능을 희생하는 것이 아닙니다.

### 3.3 빌드 시간 영향

소스 제네레이터는 빌드 시간에 약간의 영향을 줍니다:

| 항목 | 전통 방식 | Toolkit 방식 |
|------|:-:|:-:|
| 빌드 시간 | 기준 | +0.1~0.3초 정도 (무시할 수 있는 수준) |
| IntelliSense | 즉시 | 소스 제네레이터 분석 후 제공 (거의 즉시) |
| 런타임 성능 | 기준 | 동일 (차이 없음) |
| 메모리 사용 | 기준 | 동일 (차이 없음) |

---

## 4. 학습 곡선 비교

### 4.1 전통 방식의 학습 곡선

```
이해도
  ▲
  │                                    ┌───── 실무 수준
  │                               ╱────┘
  │                          ╱────┘
  │                     ╱────┘        ← CanExecute 연동, 유효성 검사,
  │                ╱────┘               ViewModel 간 통신 등
  │           ╱────┘                    모두 직접 구현해야 함
  │      ╱────┘                         (시간이 오래 걸림)
  │ ╱────┘
  │╱  ← INotifyPropertyChanged,
  │     ICommand 개념 이해
  └─────────────────────────────────────────→ 학습 시간
  0         1주         2주         3주

  총 학습 시간: 약 2~3주
  (인프라 구현에 많은 시간 소요)
```

### 4.2 CommunityToolkit.Mvvm의 학습 곡선

```
이해도
  ▲
  │               ┌──────────────────── 실무 수준
  │          ╱────┘
  │     ╱────┘    ← Messenger, ObservableValidator 등
  │╱────┘           활용법까지 익히기
  │╱  ← [ObservableProperty], [RelayCommand]
  │     특성 사용법만 익히면 바로 시작 가능!
  └─────────────────────────────────────────→ 학습 시간
  0         1주         2주         3주

  총 학습 시간: 약 1주
  (개념만 이해하면 특성으로 바로 적용)
```

### 4.3 학습 권장 순서

| 순서 | 전통 방식 | CommunityToolkit.Mvvm |
|:---:|----------|----------------------|
| 1 | INotifyPropertyChanged 개념 이해 | INotifyPropertyChanged 개념 이해 (동일) |
| 2 | ViewModelBase 직접 구현 | `ObservableObject` 사용법 (상속만 하면 됨) |
| 3 | ICommand 개념 이해 | ICommand 개념 이해 (동일) |
| 4 | RelayCommand 직접 구현 | `[RelayCommand]` 특성 사용법 |
| 5 | AsyncRelayCommand 직접 구현 | `[ObservableProperty]` 특성 사용법 |
| 6 | 유효성 검사 직접 구현 | `[NotifyPropertyChangedFor]` 등 연동 특성 |
| 7 | ViewModel 간 통신 직접 구현 | `ObservableValidator` 유효성 검사 |
| 8 | - | `WeakReferenceMessenger` 통신 |

> **권장**: 이전 문서에서 전통 방식의 **개념**은 이해하되, **실제 코드는 CommunityToolkit.Mvvm으로 작성**하세요.
> 원리를 이해하고 있으면 Toolkit의 동작 방식도 더 잘 이해할 수 있습니다.

---

## 5. 언제 전통 방식을 쓰는가?

### 5.1 전통 방식을 사용하는 (드문) 경우

| 상황 | 설명 | 빈도 |
|------|------|:---:|
| 레거시 프로젝트 유지보수 | .NET Framework 4.x 기반의 기존 프로젝트에서 CommunityToolkit을 도입하기 어려운 경우 | 드묾 |
| NuGet 패키지 사용 금지 | 보안 정책상 외부 패키지를 사용할 수 없는 환경 | 매우 드묾 |
| 극도로 특수한 커스터마이징 | Toolkit이 제공하지 않는 매우 특수한 동작이 필요한 경우 (거의 없음) | 극히 드묾 |
| 교육 목적 | MVVM 패턴의 내부 동작을 깊이 이해하기 위해 직접 구현해 보는 학습 | 학습 시에만 |

### 5.2 전통 방식을 쓰지 않아야 하는 이유

```
"직접 구현하면 더 잘 이해할 수 있지 않나요?"
→ 원리를 이해하는 것은 좋지만, 실무에서는 Toolkit을 사용하세요.

비유:
- 자동차 엔진의 원리를 이해하는 것은 좋지만,
  매일 출퇴근할 때 엔진을 직접 조립하지는 않습니다.
- Python에서 sort()를 쓰지,
  매번 퀵소트를 직접 구현하지는 않습니다.
```

---

## 6. 최종 권장: CommunityToolkit.Mvvm

### 6.1 권장 이유 총정리

| 항목 | 이유 |
|------|------|
| **코드 양** | 동일 기능 대비 약 70~80% 코드 감소 |
| **안정성** | Microsoft가 개발하고 테스트한 검증된 라이브러리 |
| **성능** | 소스 제네레이터 기반 → 런타임 오버헤드 제로 |
| **유지보수** | 선언적 코드 → 의존 관계가 명시적 → 실수 방지 |
| **생태계** | .NET 공식 도구, Microsoft Learn 문서 충실 |
| **무료** | MIT 라이선스, 상용 프로젝트에서 자유롭게 사용 |
| **업데이트** | 지속적인 업데이트와 버그 수정 |
| **커뮤니티** | 활발한 GitHub 이슈/PR, Stack Overflow 답변 |

### 6.2 이 튜토리얼에서의 방향

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  이 튜토리얼의 모든 예제는 CommunityToolkit.Mvvm을 사용합니다.  │
│                                                               │
│  ✅ 모든 ViewModel → ObservableObject 또는                     │
│                      ObservableValidator 상속                  │
│  ✅ 모든 프로퍼티 → [ObservableProperty] 특성                   │
│  ✅ 모든 커맨드 → [RelayCommand] 특성                           │
│  ✅ ViewModel 간 통신 → WeakReferenceMessenger                 │
│                                                               │
│  전통 방식은 "원리 이해용"으로만 참고하세요.                      │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## 7. 마이그레이션 가이드 (전통 → Toolkit)

기존 프로젝트를 전통 MVVM에서 CommunityToolkit.Mvvm으로 변환하는 단계별 가이드입니다.

### 7.1 마이그레이션 순서 개요

```
단계 1: NuGet 패키지 설치
         │
         ▼
단계 2: ViewModelBase → ObservableObject로 교체
         │
         ▼
단계 3: RelayCommand/AsyncRelayCommand 클래스 제거
         │
         ▼
단계 4: 프로퍼티를 [ObservableProperty]로 변환
         │
         ▼
단계 5: 커맨드를 [RelayCommand]로 변환
         │
         ▼
단계 6: OnPropertyChanged 수동 호출 → [NotifyPropertyChangedFor]로 변환
         │
         ▼
단계 7: CanExecute 연동 → [NotifyCanExecuteChangedFor]로 변환
         │
         ▼
단계 8: (선택) 유효성 검사를 ObservableValidator로 전환
         │
         ▼
단계 9: (선택) ViewModel 간 통신을 Messenger로 전환
         │
         ▼
단계 10: 사용하지 않는 인프라 코드 제거
```

### 7.2 단계 1: NuGet 패키지 설치

```bash
dotnet add package CommunityToolkit.Mvvm --version 8.4.0
```

### 7.3 단계 2: ViewModelBase → ObservableObject

```csharp
// ❌ 변환 전: 자체 ViewModelBase 사용
public class UserViewModel : ViewModelBase
{
    // ...
}

// ✅ 변환 후: ObservableObject 사용 (partial 추가!)
using CommunityToolkit.Mvvm.ComponentModel;

public partial class UserViewModel : ObservableObject
{
    // ...
}
```

> **주의**: `partial` 키워드를 반드시 추가하세요!

### 7.4 단계 3: 프로퍼티 변환

프로퍼티를 하나씩 `[ObservableProperty]`로 변환합니다:

```csharp
// ❌ 변환 전: 수동 프로퍼티
private string _name = string.Empty;

public string Name
{
    get => _name;
    set
    {
        if (SetProperty(ref _name, value))
        {
            OnPropertyChanged(nameof(DisplayText));
        }
    }
}

// ✅ 변환 후: [ObservableProperty] + [NotifyPropertyChangedFor]
[ObservableProperty]
[NotifyPropertyChangedFor(nameof(DisplayText))]
private string _name = string.Empty;
```

**변환 체크리스트**:

- [ ] `private` 백킹 필드에 `[ObservableProperty]` 추가
- [ ] `public` 프로퍼티(getter/setter) 삭제
- [ ] setter의 `OnPropertyChanged(nameof(...))` 호출 → `[NotifyPropertyChangedFor(...)]`로 변환
- [ ] setter의 커맨드 재평가 로직 → `[NotifyCanExecuteChangedFor(...)]`로 변환
- [ ] setter에 추가 로직이 있다면 → `partial void OnXxxChanged(...)` 메서드로 이동

### 7.5 단계 4: 커맨드 변환

```csharp
// ❌ 변환 전: 수동 커맨드
public RelayCommand ResetCommand { get; }

public UserViewModel()
{
    ResetCommand = new RelayCommand(
        execute: _ => ResetForm(),
        canExecute: _ => !IsBusy
    );
}

private void ResetForm()
{
    Name = string.Empty;
    Email = string.Empty;
}

// ✅ 변환 후: [RelayCommand]
[RelayCommand(CanExecute = nameof(CanResetForm))]
private void ResetForm()
{
    Name = string.Empty;
    Email = string.Empty;
}

private bool CanResetForm() => !IsBusy;
```

**변환 체크리스트**:

- [ ] `public ... Command { get; }` 프로퍼티 삭제
- [ ] 생성자에서 커맨드 초기화 코드 삭제
- [ ] 실행 메서드에 `[RelayCommand]` 추가
- [ ] `canExecute` 람다 → `CanExecute = nameof(...)` 매개변수로 이동
- [ ] 비동기 메서드면 그대로 `async Task` 반환 (Toolkit이 자동 처리)
- [ ] XAML에서 커맨드 이름 변경 확인 (예: `SaveCommand` → `SaveUserCommand`)

### 7.6 단계 5: XAML 커맨드 이름 업데이트

Toolkit의 네이밍 규칙에 맞춰 XAML의 커맨드 바인딩을 업데이트합니다:

```xml
<!-- ❌ 변환 전: 커맨드 이름이 다를 수 있음 -->
<Button Command="{Binding SaveCommand}" />
<Button Command="{Binding ResetCommand}" />

<!-- ✅ 변환 후: 메서드 이름 + "Command" -->
<Button Command="{Binding SaveUserCommand}" />
<Button Command="{Binding ResetFormCommand}" />

<!--
    네이밍 규칙:
    메서드 이름: SaveUserAsync → 커맨드 이름: SaveUserCommand ("Async" 자동 제거)
    메서드 이름: ResetForm     → 커맨드 이름: ResetFormCommand
-->
```

### 7.7 단계 6: 인프라 코드 삭제

마이그레이션이 완료되면 더 이상 필요 없는 파일들을 삭제합니다:

```
삭제할 파일들:
├── ViewModels/
│   └── ViewModelBase.cs          ← 삭제 (ObservableObject로 대체)
├── Commands/
│   ├── RelayCommand.cs           ← 삭제 ([RelayCommand] 특성으로 대체)
│   ├── RelayCommand{T}.cs        ← 삭제
│   └── AsyncRelayCommand.cs      ← 삭제
```

### 7.8 마이그레이션 시 주의사항

| 주의사항 | 설명 |
|---------|------|
| **점진적 마이그레이션 가능** | 모든 ViewModel을 한 번에 바꿀 필요 없습니다. 하나씩 변환해도 됩니다. `ObservableObject`가 `INotifyPropertyChanged`를 구현하므로 기존 바인딩과 호환됩니다. |
| **커맨드 이름 변경** | Toolkit의 네이밍 규칙이 기존과 다를 수 있습니다. XAML의 `Command` 바인딩을 반드시 확인하세요. |
| **partial 키워드** | 변환하는 모든 ViewModel 클래스에 `partial`을 추가해야 합니다. 중첩 클래스라면 바깥 클래스도 `partial`이어야 합니다. |
| **setter 추가 로직** | 기존 setter에 비즈니스 로직이 있다면 `partial void OnXxxChanged(...)` 메서드로 이동해야 합니다. |
| **테스트** | 변환 후 기존 단위 테스트가 통과하는지 확인하세요. 프로퍼티 이름과 커맨드 이름이 바뀌었을 수 있습니다. |

### 7.9 마이그레이션 전후 비교 요약

```
변환 전:                              변환 후:
┌──────────────────────────┐        ┌──────────────────────────┐
│ MyApp/                    │        │ MyApp/                    │
│ ├── Commands/             │        │ ├── Commands/ (삭제됨!)    │
│ │   ├── RelayCommand.cs   │   →    │ ├── ViewModels/           │
│ │   └── AsyncRelay...cs   │        │ │   └── UserViewModel.cs  │
│ ├── ViewModels/           │        │ │       (~100줄)           │
│ │   ├── ViewModelBase.cs  │        │ ├── Models/               │
│ │   └── UserViewModel.cs  │        │ │   └── UserModel.cs      │
│ │       (~170줄)           │        │ └── Views/                │
│ ├── Models/               │        │     └── UserView.xaml     │
│ │   └── UserModel.cs      │        │                           │
│ └── Views/                │        │ 파일 수: 3개 (5 → 3)       │
│     └── UserView.xaml     │        │ 총 줄 수: ~100줄           │
│                           │        │                           │
│ 파일 수: 5개               │        └──────────────────────────┘
│ 총 줄 수: ~410줄           │
└──────────────────────────┘

→ 파일 2개 삭제, 코드 약 310줄 감소!
```

---

## 최종 요약 테이블

| 비교 항목 | 전통 방식 (수동) | CommunityToolkit.Mvvm | 판정 |
|----------|:-:|:-:|:-:|
| 코드 양 | 많음 (~410줄) | 적음 (~100줄) | **Toolkit** |
| 보일러플레이트 | 매우 많음 | 거의 없음 | **Toolkit** |
| 실수 가능성 | 높음 (누락, 오타) | 낮음 (선언적) | **Toolkit** |
| 유지보수성 | 어려움 | 쉬움 | **Toolkit** |
| 가독성 | 보통 (코드가 길어 핵심이 묻힘) | 좋음 (비즈니스 로직에 집중) | **Toolkit** |
| 성능 | 좋음 | 좋음 (동일) | **동일** |
| 학습 곡선 | 길음 (~2~3주) | 짧음 (~1주) | **Toolkit** |
| 유효성 검사 | 직접 구현 | 내장 (데이터 어노테이션) | **Toolkit** |
| VM 간 통신 | 직접 구현 | 내장 (Messenger) | **Toolkit** |
| 테스트 용이성 | 좋음 | 좋음 (동일) | **동일** |
| 외부 의존성 | 없음 | NuGet 1개 | 전통 방식 (미미한 차이) |
| 공식 지원 | 없음 (자체 구현) | Microsoft 공식 | **Toolkit** |

> **결론**: 외부 의존성 1개를 제외하면 **모든 면에서 CommunityToolkit.Mvvm이 우위**입니다.
> 새 프로젝트를 시작한다면 **CommunityToolkit.Mvvm을 사용하세요.**

---

## 다음 단계

MVVM 구현 방식을 비교했으니, 이제 실제 프로젝트를 만들어 봅시다:

> **다음**: [프로젝트 생성](../03-project-setup/01-project-creation.md)
