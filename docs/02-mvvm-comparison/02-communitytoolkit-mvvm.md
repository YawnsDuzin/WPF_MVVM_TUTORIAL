# CommunityToolkit.Mvvm 사용 방법

> **목표**: Microsoft 공식 MVVM 툴킷인 CommunityToolkit.Mvvm의 핵심 기능을 배우고,
> [이전 문서](./01-traditional-mvvm.md)에서 수동으로 구현했던 코드가 얼마나 간결해지는지 확인합니다.

---

## 목차

1. [CommunityToolkit.Mvvm이란?](#1-communitytoolkitmvvm이란)
2. [설치 방법](#2-설치-방법)
3. [핵심 기능 1 — ObservableObject 기본 클래스](#3-핵심-기능-1--observableobject-기본-클래스)
4. [핵심 기능 2 — \[ObservableProperty\] 특성](#4-핵심-기능-2--observableproperty-특성)
5. [핵심 기능 3 — \[RelayCommand\] 특성](#5-핵심-기능-3--relaycommand-특성)
6. [핵심 기능 4 — \[NotifyPropertyChangedFor\] 특성](#6-핵심-기능-4--notifypropertychangedfor-특성)
7. [핵심 기능 5 — \[NotifyCanExecuteChangedFor\] 특성](#7-핵심-기능-5--notifycanexecutechangedfor-특성)
8. [핵심 기능 6 — ObservableValidator로 유효성 검사](#8-핵심-기능-6--observablevalidator로-유효성-검사)
9. [핵심 기능 7 — Messenger로 ViewModel 간 통신](#9-핵심-기능-7--messenger로-viewmodel-간-통신)
10. [완전한 예제: 사용자 정보 편집 화면 (CommunityToolkit 방식)](#10-완전한-예제-사용자-정보-편집-화면-communitytoolkit-방식)
11. [partial class 주의사항](#11-partial-class-주의사항)

---

## 1. CommunityToolkit.Mvvm이란?

### 1.1 개요

**CommunityToolkit.Mvvm**(이하 "Toolkit")은 Microsoft가 공식으로 개발하고 유지보수하는 **오픈소스 MVVM 라이브러리**입니다.

| 항목 | 내용 |
|------|------|
| 공식 명칭 | CommunityToolkit.Mvvm (구: Microsoft.Toolkit.Mvvm) |
| 개발/유지보수 | Microsoft (.NET Foundation) |
| 라이선스 | MIT (무료, 상용 사용 가능) |
| GitHub | [CommunityToolkit/dotnet](https://github.com/CommunityToolkit/dotnet) |
| NuGet 패키지 | `CommunityToolkit.Mvvm` |
| 권장 버전 | 8.4.0 (이 튜토리얼 기준) |
| .NET 지원 | .NET Standard 2.0, .NET 6+, .NET 10 |

### 1.2 소스 제네레이터란?

Toolkit의 가장 강력한 특성은 **소스 제네레이터(Source Generator)** 기반이라는 점입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    소스 제네레이터 동작 원리                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [컴파일 시점]                                                   │
│                                                                 │
│  여러분이 작성한 코드:                                            │
│  ┌─────────────────────────────┐                                │
│  │ [ObservableProperty]        │                                │
│  │ private string _name;       │                                │
│  └──────────────┬──────────────┘                                │
│                 │                                                │
│                 ▼                                                │
│  소스 제네레이터가 자동 생성:                                      │
│  ┌─────────────────────────────────────────────┐                │
│  │ public string Name                          │                │
│  │ {                                           │                │
│  │     get => _name;                           │                │
│  │     set                                     │                │
│  │     {                                       │                │
│  │         if (SetProperty(ref _name, value))  │                │
│  │         {                                   │                │
│  │             OnNameChanged(value);            │                │
│  │         }                                   │                │
│  │     }                                       │                │
│  │ }                                           │                │
│  └─────────────────────────────────────────────┘                │
│                                                                 │
│  → 컴파일 시점에 코드가 생성되므로                                  │
│  → 런타임 리플렉션이 전혀 없음                                     │
│  → 성능 오버헤드 ZERO                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Python 개발자에게 비유하면, **메타클래스(metaclass)**나 **데코레이터**가 클래스를 자동으로 확장하는 것과 비슷합니다. 다만 **런타임이 아닌 컴파일 시점**에 일어난다는 것이 핵심 차이입니다:

```python
# Python 비유: 데코레이터가 클래스를 자동 확장
# (CommunityToolkit은 이것을 "컴파일 시점"에 수행)
@dataclass  # ← __init__, __repr__ 등을 자동 생성
class User:
    name: str
    age: int
```

### 1.3 왜 CommunityToolkit.Mvvm인가?

| 기존 수동 방식 | CommunityToolkit.Mvvm |
|-------------|----------------------|
| ViewModelBase 직접 작성 | `ObservableObject` 기본 클래스 제공 |
| RelayCommand 직접 작성 | `[RelayCommand]` 특성 하나로 끝 |
| 프로퍼티마다 7~12줄 반복 | `[ObservableProperty]` 한 줄로 끝 |
| OnPropertyChanged 수동 호출 | `[NotifyPropertyChangedFor]`로 선언적 |
| CanExecute 재평가 수동 관리 | `[NotifyCanExecuteChangedFor]`로 자동 |
| 유효성 검사 직접 구현 | `ObservableValidator` + 데이터 어노테이션 |
| ViewModel 간 통신 직접 구현 | `WeakReferenceMessenger` 내장 |
| 자체 구현의 버그 위험 | Microsoft가 테스트하고 유지보수 |

---

## 2. 설치 방법

### 2.1 NuGet 패키지 설치

```bash
# dotnet CLI로 설치
dotnet add package CommunityToolkit.Mvvm --version 8.4.0
```

또는 Visual Studio에서:
1. 프로젝트 우클릭 → **NuGet 패키지 관리**
2. **CommunityToolkit.Mvvm** 검색
3. 버전 **8.4.0** 선택 후 **설치**

### 2.2 .csproj 확인

설치 후 프로젝트 파일에 다음이 추가됩니다:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0-windows</TargetFramework>
    <UseWPF>true</UseWPF>
  </PropertyGroup>

  <ItemGroup>
    <!-- CommunityToolkit.Mvvm — MVVM 소스 제네레이터 -->
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.4.0" />
  </ItemGroup>
</Project>
```

---

## 3. 핵심 기능 1 — ObservableObject 기본 클래스

### 3.1 수동 방식 vs Toolkit 비교

**수동 방식**: ViewModelBase를 직접 만들어야 했습니다 (~40줄).

**Toolkit 방식**: `ObservableObject`를 상속하면 끝입니다.

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace MyApp.ViewModels
{
    // ObservableObject를 상속하면 INotifyPropertyChanged가 자동으로 구현됩니다.
    // ViewModelBase를 직접 만들 필요가 없습니다!
    public partial class UserViewModel : ObservableObject
    {
        // SetProperty, OnPropertyChanged 등이 이미 내장되어 있음
    }
}
```

> **중요**: `partial` 키워드를 반드시 써야 합니다! 소스 제네레이터가 이 클래스의 나머지 부분을 자동으로 생성하기 때문입니다.

### 3.2 ObservableObject가 제공하는 것들

`ObservableObject`는 [이전 문서](./01-traditional-mvvm.md)에서 직접 만든 `ViewModelBase`의 모든 기능을 포함합니다:

```csharp
// ObservableObject에 이미 구현되어 있는 주요 멤버들:

// INotifyPropertyChanged 이벤트
public event PropertyChangedEventHandler? PropertyChanged;

// INotifyPropertyChanging 이벤트 (변경 "전" 알림도 지원!)
public event PropertyChangingEventHandler? PropertyChanging;

// SetProperty 헬퍼 — 값 비교 + 필드 설정 + 알림
protected bool SetProperty<T>(ref T field, T newValue, ...);

// OnPropertyChanged — 변경 알림 발생
protected virtual void OnPropertyChanged(PropertyChangedEventArgs e);

// 그 외 다양한 오버로드 제공
```

---

## 4. 핵심 기능 2 — [ObservableProperty] 특성

### 4.1 기본 사용법

이것이 CommunityToolkit.Mvvm의 **가장 핵심적인 기능**입니다:

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

public partial class UserViewModel : ObservableObject
{
    // ✅ Toolkit 방식 — 이 한 줄이면 됩니다!
    [ObservableProperty]
    private string _name = string.Empty;

    // 위의 한 줄이 소스 제네레이터에 의해 아래 코드로 자동 변환됩니다:
    // ┌──────────────────────────────────────────────────┐
    // │ public string Name                               │
    // │ {                                                │
    // │     get => _name;                                │
    // │     set                                          │
    // │     {                                            │
    // │         if (!EqualityComparer<string>.Default     │
    // │              .Equals(_name, value))               │
    // │         {                                        │
    // │             OnNameChanging(value);                │
    // │             OnNameChanging(default, value);       │
    // │             OnPropertyChanging("Name");           │
    // │             _name = value;                        │
    // │             OnNameChanged(value);                 │
    // │             OnNameChanged(default, value);        │
    // │             OnPropertyChanged("Name");            │
    // │         }                                        │
    // │     }                                            │
    // │ }                                                │
    // └──────────────────────────────────────────────────┘
}
```

**수동 방식에서 7~12줄 걸리던 것이 단 2줄(특성 + 필드)로 줄어듭니다!**

### 4.2 필드 네이밍 규칙

소스 제네레이터가 필드 이름에서 프로퍼티 이름을 자동으로 만듭니다. 두 가지 네이밍 규칙을 지원합니다:

```csharp
// 규칙 1: _접두사 (언더스코어)
[ObservableProperty]
private string _name;           // → 프로퍼티: Name

[ObservableProperty]
private string _firstName;      // → 프로퍼티: FirstName

// 규칙 2: 소문자 시작 (접두사 없음)
[ObservableProperty]
private string name;            // → 프로퍼티: Name

[ObservableProperty]
private string firstName;       // → 프로퍼티: FirstName

// ❌ 이미 대문자로 시작하는 이름은 사용 불가
// [ObservableProperty]
// private string Name;         // 에러! 생성될 프로퍼티 이름과 충돌

// 💡 권장: _접두사 방식 (C# 컨벤션에 부합)
[ObservableProperty]
private string _email = string.Empty;   // → 프로퍼티: Email
```

### 4.3 생성되는 코드 직접 확인하기

Visual Studio에서 소스 제네레이터가 생성한 코드를 직접 볼 수 있습니다:

1. **솔루션 탐색기** → 프로젝트 → **종속성** → **분석기** 확장
2. **CommunityToolkit.Mvvm.SourceGenerators** 확장
3. 생성된 `.g.cs` 파일 확인

또는 프로퍼티 이름 위에서 **F12 (정의로 이동)**를 누르면 생성된 코드를 볼 수 있습니다.

### 4.4 partial 메서드 — 변경 시 추가 로직 실행

소스 제네레이터는 프로퍼티 변경 전/후에 호출되는 **partial 메서드**도 함께 생성합니다. 필요할 때만 구현하면 됩니다:

```csharp
public partial class UserViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name = string.Empty;

    // Name이 변경되기 "전"에 호출 (값을 검증하거나 이전 값 저장 등)
    partial void OnNameChanging(string value)
    {
        // 예: 이전 이름을 로그에 기록
        Console.WriteLine($"이름이 '{_name}'에서 '{value}'(으)로 변경됩니다.");
    }

    // Name이 변경된 "후"에 호출 (관련 작업 수행)
    partial void OnNameChanged(string value)
    {
        // 예: 이름이 바뀌면 전체 이름 다시 계산
        Console.WriteLine($"이름이 '{value}'(으)로 변경되었습니다.");
    }

    // 이전 값과 새 값을 모두 받는 오버로드도 있습니다
    partial void OnNameChanged(string? oldValue, string newValue)
    {
        Console.WriteLine($"이름: '{oldValue}' → '{newValue}'");
    }
}
```

Python으로 비유하면:

```python
# Python 비유: property setter에서 변경 전/후 로직
class User:
    @name.setter
    def name(self, value):
        self.on_name_changing(value)   # 변경 전
        self._name = value
        self.on_name_changed(value)    # 변경 후
```

---

## 5. 핵심 기능 3 — [RelayCommand] 특성

### 5.1 동기 커맨드

수동으로 `RelayCommand` 클래스를 만들고, 생성자에서 초기화하던 것이 **특성 하나**로 끝납니다:

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class UserViewModel : ObservableObject
{
    // ✅ Toolkit 방식 — 메서드에 [RelayCommand]만 붙이면 됨!
    [RelayCommand]
    private void ResetForm()
    {
        Name = string.Empty;
        Email = string.Empty;
        Age = 0;
        StatusMessage = "폼이 초기화되었습니다.";
    }

    // 위의 코드가 소스 제네레이터에 의해 자동으로 이 프로퍼티를 생성합니다:
    // ┌──────────────────────────────────────────────────────┐
    // │ public IRelayCommand ResetFormCommand { get; }       │
    // │                                                      │
    // │ // 메서드 이름 "ResetForm" + "Command" = 프로퍼티 이름 │
    // │ // XAML에서: Command="{Binding ResetFormCommand}"     │
    // └──────────────────────────────────────────────────────┘
}
```

**네이밍 규칙**: 메서드 이름에 `Command`가 자동으로 붙어 프로퍼티 이름이 됩니다.

| 메서드 이름 | 생성되는 커맨드 프로퍼티 |
|------------|---------------------|
| `Save()` | `SaveCommand` |
| `ResetForm()` | `ResetFormCommand` |
| `DeleteUser()` | `DeleteUserCommand` |
| `LoadData()` | `LoadDataCommand` |

### 5.2 비동기 커맨드 (Task 반환)

`Task`를 반환하는 메서드에 `[RelayCommand]`를 붙이면 **자동으로 비동기 커맨드**가 생성됩니다:

```csharp
public partial class UserViewModel : ObservableObject
{
    // 반환 타입이 Task이면 자동으로 AsyncRelayCommand가 생성됨
    [RelayCommand]
    private async Task SaveUserAsync()
    {
        IsBusy = true;
        StatusMessage = "저장 중...";

        try
        {
            await Task.Delay(2000); // DB 저장 시뮬레이션
            StatusMessage = $"'{Name}' 정보가 저장되었습니다.";
        }
        catch (Exception ex)
        {
            StatusMessage = $"저장 실패: {ex.Message}";
        }
        finally
        {
            IsBusy = false;
        }
    }

    // 생성되는 코드:
    // ┌─────────────────────────────────────────────────────────┐
    // │ public IAsyncRelayCommand SaveUserCommand { get; }      │
    // │                                                         │
    // │ // "SaveUserAsync"에서 "Async" 접미사를 자동으로 제거하고  │
    // │ // "Command"를 붙여서 "SaveUserCommand"가 됨               │
    // └─────────────────────────────────────────────────────────┘

    // IAsyncRelayCommand는 추가로 다음을 제공합니다:
    // - IsRunning 프로퍼티 (실행 중 여부)
    // - ExecutionTask 프로퍼티 (현재 실행 중인 Task)
    // - Cancel() 메서드 (CancellationToken과 함께 사용 시)
}
```

### 5.3 CanExecute 연결

커맨드의 실행 가능 조건을 `CanExecute` 매개변수로 지정합니다:

```csharp
public partial class UserViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name = string.Empty;

    [ObservableProperty]
    private bool _isBusy;

    // CanExecute 조건을 별도 프로퍼티나 메서드로 정의
    private bool CanSaveUser()
    {
        return !IsBusy && !string.IsNullOrWhiteSpace(Name);
    }

    // CanExecute 매개변수로 연결
    [RelayCommand(CanExecute = nameof(CanSaveUser))]
    private async Task SaveUserAsync()
    {
        // 저장 로직...
        await Task.Delay(1000);
    }

    // 생성되는 코드:
    // ┌──────────────────────────────────────────────────────────────┐
    // │ SaveUserCommand = new AsyncRelayCommand(                    │
    // │     SaveUserAsync,                                          │
    // │     CanSaveUser    // ← CanExecute 조건이 자동으로 연결됨     │
    // │ );                                                          │
    // └──────────────────────────────────────────────────────────────┘
}
```

### 5.4 커맨드 파라미터 지원

메서드에 파라미터가 있으면 자동으로 `RelayCommand<T>`가 생성됩니다:

```csharp
public partial class UserViewModel : ObservableObject
{
    // 파라미터가 있는 커맨드
    [RelayCommand]
    private void DeleteUser(int userId)
    {
        // userId를 사용하여 삭제 로직 수행
        Console.WriteLine($"사용자 {userId} 삭제");
    }

    // XAML에서 사용:
    // <Button Content="삭제"
    //         Command="{Binding DeleteUserCommand}"
    //         CommandParameter="{Binding SelectedUser.Id}" />
}
```

---

## 6. 핵심 기능 4 — [NotifyPropertyChangedFor] 특성

### 6.1 문제: 계산 프로퍼티의 의존성 관리

수동 방식에서는 관련 프로퍼티의 변경 알림을 setter마다 직접 추가해야 했습니다:

```csharp
// ❌ 수동 방식 — 각 setter에서 의존 프로퍼티를 직접 알림
public string Name
{
    get => _name;
    set
    {
        if (SetProperty(ref _name, value))
        {
            OnPropertyChanged(nameof(DisplayText));  // ← 매번 수동!
        }
    }
}

public int Age
{
    get => _age;
    set
    {
        if (SetProperty(ref _age, value))
        {
            OnPropertyChanged(nameof(DisplayText));  // ← 또 수동!
        }
    }
}

// DisplayText는 Name과 Age에 의존
public string DisplayText => $"{Name} ({Age}세)";
```

### 6.2 Toolkit 방식 — 선언적으로 해결

```csharp
public partial class UserViewModel : ObservableObject
{
    // Name이 바뀌면 DisplayText도 변경 알림을 보냄
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(DisplayText))]
    private string _name = string.Empty;

    // Age가 바뀌면 DisplayText도 변경 알림을 보냄
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(DisplayText))]
    private int _age;

    // 계산 프로퍼티 — Name과 Age에 의존
    public string DisplayText => $"{Name} ({Age}세)";
}

// [NotifyPropertyChangedFor]가 생성하는 코드:
// ┌────────────────────────────────────────────────────┐
// │ public string Name                                 │
// │ {                                                  │
// │     set                                            │
// │     {                                              │
// │         if (SetProperty(ref _name, value))         │
// │         {                                          │
// │             OnPropertyChanged("DisplayText");  ← 자동! │
// │         }                                          │
// │     }                                              │
// │ }                                                  │
// └────────────────────────────────────────────────────┘
```

### 6.3 여러 프로퍼티에 동시에 알림

하나의 필드 변경이 여러 계산 프로퍼티에 영향을 줄 수도 있습니다:

```csharp
// Name이 바뀌면 DisplayText, FullInfo, CanSave 모두 알림
[ObservableProperty]
[NotifyPropertyChangedFor(nameof(DisplayText))]
[NotifyPropertyChangedFor(nameof(FullInfo))]
[NotifyPropertyChangedFor(nameof(CanSave))]
private string _name = string.Empty;

// 또는 한 번에 여러 개를 지정할 수도 있습니다:
[ObservableProperty]
[NotifyPropertyChangedFor(nameof(DisplayText), nameof(FullInfo), nameof(CanSave))]
private string _name = string.Empty;
```

---

## 7. 핵심 기능 5 — [NotifyCanExecuteChangedFor] 특성

### 7.1 문제: 커맨드의 CanExecute 재평가

수동 방식에서는 프로퍼티가 바뀔 때 관련 커맨드의 `CanExecute`를 수동으로 재평가해야 했습니다:

```csharp
// ❌ 수동 방식
public string Name
{
    get => _name;
    set
    {
        if (SetProperty(ref _name, value))
        {
            // Name이 바뀌면 SaveCommand의 CanExecute를 재평가해야 함
            SaveCommand.RaiseCanExecuteChanged();  // ← 이걸 빼먹기 쉬움!
        }
    }
}
```

### 7.2 Toolkit 방식 — 선언적으로 해결

```csharp
public partial class UserViewModel : ObservableObject
{
    // Name이 바뀌면 SaveUserCommand의 CanExecute를 자동으로 재평가
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(SaveUserCommand))]
    private string _name = string.Empty;

    // Email이 바뀌면 SaveUserCommand의 CanExecute를 자동으로 재평가
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(SaveUserCommand))]
    private string _email = string.Empty;

    // IsBusy가 바뀌면 SaveUserCommand와 ResetFormCommand 둘 다 재평가
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(SaveUserCommand))]
    [NotifyCanExecuteChangedFor(nameof(ResetFormCommand))]
    private bool _isBusy;

    private bool CanSaveUser() =>
        !IsBusy && !string.IsNullOrWhiteSpace(Name) && !string.IsNullOrWhiteSpace(Email);

    [RelayCommand(CanExecute = nameof(CanSaveUser))]
    private async Task SaveUserAsync()
    {
        // 저장 로직...
    }

    [RelayCommand(CanExecute = nameof(CanResetForm))]
    private void ResetForm()
    {
        // 리셋 로직...
    }

    private bool CanResetForm() => !IsBusy;
}
```

### 7.3 전체 의존 관계를 선언적으로 표현

모든 의존 관계가 **필드 선언부에 명시적으로** 나타나므로, 코드를 읽기만 해도 관계를 파악할 수 있습니다:

```csharp
// 이 필드의 선언부만 보면 모든 의존 관계를 알 수 있습니다:
[ObservableProperty]
[NotifyPropertyChangedFor(nameof(DisplayText))]           // Name → DisplayText 연동
[NotifyPropertyChangedFor(nameof(CanSave))]               // Name → CanSave 연동
[NotifyCanExecuteChangedFor(nameof(SaveUserCommand))]     // Name → SaveUserCommand 재평가
private string _name = string.Empty;

// 수동 방식에서는 setter 본문을 읽어야만 관계를 파악할 수 있었습니다.
// Toolkit 방식에서는 특성만 보면 됩니다 → 가독성 대폭 향상!
```

---

## 8. 핵심 기능 6 — ObservableValidator로 유효성 검사

### 8.1 ObservableValidator란?

`ObservableValidator`는 `ObservableObject`를 확장하여 **데이터 어노테이션 기반의 유효성 검사**를 지원합니다. C#의 `System.ComponentModel.DataAnnotations`와 통합됩니다.

```csharp
using System.ComponentModel.DataAnnotations;
using CommunityToolkit.Mvvm.ComponentModel;

// ObservableObject 대신 ObservableValidator를 상속
public partial class UserViewModel : ObservableValidator
{
    // 유효성 검사 규칙을 데이터 어노테이션으로 지정
    [ObservableProperty]
    [NotifyDataErrorInfo]                    // ← 유효성 검사 자동 실행
    [Required(ErrorMessage = "이름은 필수입니다.")]
    [MinLength(2, ErrorMessage = "이름은 최소 2자 이상이어야 합니다.")]
    [MaxLength(50, ErrorMessage = "이름은 50자를 초과할 수 없습니다.")]
    private string _name = string.Empty;

    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Required(ErrorMessage = "이메일은 필수입니다.")]
    [EmailAddress(ErrorMessage = "올바른 이메일 형식이 아닙니다.")]
    private string _email = string.Empty;

    [ObservableProperty]
    [NotifyDataErrorInfo]
    [Range(1, 149, ErrorMessage = "나이는 1~149 사이여야 합니다.")]
    private int _age;
}
```

### 8.2 XAML에서 유효성 검사 결과 표시

```xml
<!-- ValidatesOnDataErrors=True로 유효성 검사 오류를 자동 표시 -->
<TextBox Text="{Binding Name,
                UpdateSourceTrigger=PropertyChanged,
                ValidatesOnDataErrors=True}" />
<!--
    유효성 검사 실패 시:
    - TextBox의 테두리가 빨간색으로 변합니다 (WPF 기본 동작).
    - Validation.Errors에 오류 메시지가 담깁니다.
    - 커스텀 스타일로 오류 메시지를 표시할 수 있습니다.
-->

<!-- 오류 메시지 표시 예 -->
<TextBlock Text="{Binding (Validation.Errors)[0].ErrorContent,
                  ElementName=nameTextBox}"
           Foreground="Red"
           FontSize="11" />
```

### 8.3 수동 유효성 검사 실행

```csharp
public partial class UserViewModel : ObservableValidator
{
    // 모든 프로퍼티를 한 번에 검사
    [RelayCommand]
    private void ValidateAll()
    {
        ValidateAllProperties();

        if (HasErrors)
        {
            StatusMessage = "입력 값에 오류가 있습니다.";
        }
        else
        {
            StatusMessage = "모든 값이 유효합니다.";
        }
    }

    // 특정 프로퍼티만 검사
    private void ValidateName()
    {
        ValidateProperty(Name, nameof(Name));
    }

    // GetErrors로 특정 프로퍼티의 오류 목록 조회
    private IEnumerable<ValidationResult> GetNameErrors()
    {
        return GetErrors(nameof(Name)).Cast<ValidationResult>();
    }
}
```

---

## 9. 핵심 기능 7 — Messenger로 ViewModel 간 통신

### 9.1 왜 Messenger가 필요한가?

MVVM에서는 ViewModel끼리 직접 참조하지 않습니다. 하지만 "사용자 목록 화면에서 사용자를 선택하면, 상세 화면이 업데이트되어야 하는" 경우처럼 ViewModel 간에 데이터를 전달해야 할 때가 있습니다.

```
┌─────────────────┐                     ┌──────────────────┐
│  UserListVM      │ ── 직접 참조 ❌ ──→ │  UserDetailVM     │
│  (사용자 목록)    │                     │  (사용자 상세)     │
└─────────────────┘                     └──────────────────┘

        │                                       ▲
        │        ┌─────────────────┐            │
        └──────→ │   Messenger     │ ──────────┘
         메시지   │  (중개자 역할)   │  메시지 수신
         발송     └─────────────────┘
```

Python으로 비유하면 **이벤트 버스(Event Bus)** 또는 **pub/sub 패턴**과 같습니다:

```python
# Python 비유: 이벤트 버스 패턴
event_bus = EventBus()

# 발행자 (Publisher)
event_bus.publish("user_selected", user_id=42)

# 구독자 (Subscriber)
event_bus.subscribe("user_selected", lambda data: show_detail(data["user_id"]))
```

### 9.2 메시지 정의

먼저 전달할 메시지 클래스를 정의합니다:

```csharp
using CommunityToolkit.Mvvm.Messaging.Messages;

namespace MyApp.Messages
{
    // 사용자가 선택되었을 때 보내는 메시지
    // ValueChangedMessage<T>를 상속하면 Value 프로퍼티가 자동으로 제공됨
    public class UserSelectedMessage : ValueChangedMessage<int>
    {
        // userId를 담아서 전달
        public UserSelectedMessage(int userId) : base(userId)
        {
        }
    }

    // 사용자 정보가 저장되었을 때 보내는 메시지
    public class UserSavedMessage : ValueChangedMessage<UserModel>
    {
        public UserSavedMessage(UserModel user) : base(user)
        {
        }
    }
}
```

### 9.3 메시지 보내기 (Send)

```csharp
using CommunityToolkit.Mvvm.Messaging;

public partial class UserListViewModel : ObservableObject
{
    [ObservableProperty]
    private int _selectedUserId;

    // 사용자가 선택되면 메시지를 보냄
    partial void OnSelectedUserIdChanged(int value)
    {
        // WeakReferenceMessenger: 약한 참조 기반 → 메모리 누수 방지
        WeakReferenceMessenger.Default.Send(new UserSelectedMessage(value));
    }
}
```

### 9.4 메시지 받기 (Receive) — 방법 1: IRecipient 인터페이스

```csharp
using CommunityToolkit.Mvvm.Messaging;

// IRecipient<T> 인터페이스를 구현하여 메시지를 수신
public partial class UserDetailViewModel : ObservableObject,
    IRecipient<UserSelectedMessage>
{
    public UserDetailViewModel()
    {
        // 이 ViewModel을 메시지 수신자로 등록
        WeakReferenceMessenger.Default.Register(this);
    }

    // IRecipient<UserSelectedMessage>.Receive — 메시지를 받으면 호출됨
    public void Receive(UserSelectedMessage message)
    {
        int selectedUserId = message.Value;
        // 선택된 사용자의 상세 정보를 로드
        LoadUserDetail(selectedUserId);
    }

    private void LoadUserDetail(int userId)
    {
        // DB에서 사용자 정보를 조회하여 화면에 표시
        Console.WriteLine($"사용자 {userId}의 상세 정보를 로드합니다.");
    }
}
```

### 9.5 메시지 받기 (Receive) — 방법 2: 람다 등록

```csharp
public partial class UserDetailViewModel : ObservableObject
{
    public UserDetailViewModel()
    {
        // 람다로 간단하게 등록
        WeakReferenceMessenger.Default.Register<UserSelectedMessage>(
            this,
            (recipient, message) =>
            {
                // recipient는 이 ViewModel 자신
                var vm = (UserDetailViewModel)recipient;
                vm.LoadUserDetail(message.Value);
            });
    }

    private void LoadUserDetail(int userId)
    {
        Console.WriteLine($"사용자 {userId}의 상세 정보를 로드합니다.");
    }
}
```

### 9.6 메시지 수신 해제

ViewModel이 더 이상 사용되지 않을 때 메시지 수신을 해제해야 합니다:

```csharp
public partial class UserDetailViewModel : ObservableObject
{
    // 특정 메시지 타입만 해제
    public void Cleanup()
    {
        WeakReferenceMessenger.Default.Unregister<UserSelectedMessage>(this);
    }

    // 또는 모든 메시지 수신 해제
    public void CleanupAll()
    {
        WeakReferenceMessenger.Default.UnregisterAll(this);
    }
}
```

> **참고**: `WeakReferenceMessenger`는 **약한 참조(Weak Reference)** 기반이므로, ViewModel이 GC에 의해 수거되면 메시지 수신도 자동으로 해제됩니다. 명시적 해제는 필수가 아니지만, 깔끔한 정리를 위해 권장됩니다.

### 9.7 요청-응답 패턴 (RequestMessage)

단방향 알림뿐 아니라, 요청-응답 패턴도 지원합니다:

```csharp
using CommunityToolkit.Mvvm.Messaging.Messages;

// 현재 로그인한 사용자를 요청하는 메시지
public class CurrentUserRequestMessage : RequestMessage<UserModel>
{
}

// 응답하는 쪽 (예: MainViewModel)
public partial class MainViewModel : ObservableObject,
    IRecipient<CurrentUserRequestMessage>
{
    public MainViewModel()
    {
        WeakReferenceMessenger.Default.Register(this);
    }

    public void Receive(CurrentUserRequestMessage message)
    {
        // 현재 로그인한 사용자 정보를 응답
        message.Reply(new UserModel { Name = "홍길동", Id = 1 });
    }
}

// 요청하는 쪽 (예: SettingsViewModel)
public partial class SettingsViewModel : ObservableObject
{
    [RelayCommand]
    private void LoadCurrentUser()
    {
        // 메시지를 보내고 응답을 받음
        var response = WeakReferenceMessenger.Default.Send<CurrentUserRequestMessage>();
        UserModel currentUser = response.Response;
        Console.WriteLine($"현재 사용자: {currentUser.Name}");
    }
}
```

---

## 10. 완전한 예제: 사용자 정보 편집 화면 (CommunityToolkit 방식)

[이전 문서](./01-traditional-mvvm.md)에서 수동으로 구현했던 **동일한 사용자 정보 편집 화면**을 CommunityToolkit.Mvvm으로 다시 구현합니다.

### 10.1 프로젝트 구조

```
MyApp/
├── Models/
│   └── UserModel.cs              ← 데이터 모델 (변경 없음)
├── ViewModels/
│   └── UserViewModel.cs          ← ViewModel (Toolkit 방식 — 대폭 간소화!)
├── Views/
│   └── UserView.xaml             ← UI 화면 (거의 변경 없음)
│   └── UserView.xaml.cs          ← 코드비하인드 (변경 없음)
└── Messages/
    └── UserMessages.cs           ← Messenger용 메시지 (선택사항)
```

> **주목**: `Commands/` 폴더가 사라졌습니다! RelayCommand, AsyncRelayCommand를 직접 만들 필요가 없으므로 해당 파일들이 필요 없습니다.

### 10.2 Model — UserModel.cs (변경 없음)

```csharp
namespace MyApp.Models
{
    /// <summary>
    /// 사용자 데이터를 담는 모델 클래스.
    /// Model은 순수 데이터 객체이므로 MVVM 방식에 관계없이 동일합니다.
    /// </summary>
    public class UserModel
    {
        public int Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public string Email { get; set; } = string.Empty;
        public int Age { get; set; }
        public string Department { get; set; } = string.Empty;
        public bool IsActive { get; set; } = true;
    }
}
```

### 10.3 ViewModel — UserViewModel.cs (CommunityToolkit 방식)

**이것이 핵심입니다.** 수동 구현에서 ~170줄이던 ViewModel이 얼마나 줄어드는지 확인하세요:

```csharp
using System.ComponentModel.DataAnnotations;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using MyApp.Models;

namespace MyApp.ViewModels
{
    /// <summary>
    /// 사용자 정보 편집을 위한 ViewModel.
    ///
    /// [CommunityToolkit.Mvvm 방식]
    /// 소스 제네레이터가 프로퍼티와 커맨드를 자동으로 생성합니다.
    /// 수동 방식 대비 코드가 약 60~70% 줄어듭니다.
    /// </summary>
    public partial class UserViewModel : ObservableValidator
    {
        // ============================================================
        // 프로퍼티 선언
        //
        // [ObservableProperty]만 붙이면 끝!
        // → public 프로퍼티, getter, setter, 변경 알림이 모두 자동 생성됨
        // ============================================================

        /// <summary>
        /// 사용자 이름.
        /// - Name이 바뀌면 DisplayText도 다시 계산
        /// - Name이 바뀌면 SaveUserCommand의 CanExecute 재평가
        /// - 유효성 검사: 필수, 최소 2자
        /// </summary>
        [ObservableProperty]
        [NotifyPropertyChangedFor(nameof(DisplayText))]
        [NotifyPropertyChangedFor(nameof(CanSave))]
        [NotifyCanExecuteChangedFor(nameof(SaveUserCommand))]
        [NotifyDataErrorInfo]
        [Required(ErrorMessage = "이름은 필수입니다.")]
        [MinLength(2, ErrorMessage = "이름은 최소 2자 이상이어야 합니다.")]
        private string _name = string.Empty;

        /// <summary>
        /// 이메일 주소.
        /// </summary>
        [ObservableProperty]
        [NotifyPropertyChangedFor(nameof(CanSave))]
        [NotifyCanExecuteChangedFor(nameof(SaveUserCommand))]
        [NotifyDataErrorInfo]
        [Required(ErrorMessage = "이메일은 필수입니다.")]
        [EmailAddress(ErrorMessage = "올바른 이메일 형식이 아닙니다.")]
        private string _email = string.Empty;

        /// <summary>
        /// 나이.
        /// </summary>
        [ObservableProperty]
        [NotifyPropertyChangedFor(nameof(DisplayText))]
        [NotifyPropertyChangedFor(nameof(CanSave))]
        [NotifyCanExecuteChangedFor(nameof(SaveUserCommand))]
        [NotifyDataErrorInfo]
        [Range(1, 149, ErrorMessage = "나이는 1~149 사이여야 합니다.")]
        private int _age;

        /// <summary>
        /// 부서명.
        /// </summary>
        [ObservableProperty]
        private string _department = string.Empty;

        /// <summary>
        /// 활성 상태 여부.
        /// </summary>
        [ObservableProperty]
        private bool _isActive = true;

        /// <summary>
        /// 상태 메시지.
        /// </summary>
        [ObservableProperty]
        private string _statusMessage = string.Empty;

        /// <summary>
        /// 현재 작업 중인지 여부.
        /// </summary>
        [ObservableProperty]
        [NotifyPropertyChangedFor(nameof(CanSave))]
        [NotifyCanExecuteChangedFor(nameof(SaveUserCommand))]
        [NotifyCanExecuteChangedFor(nameof(ResetFormCommand))]
        private bool _isBusy;

        // ============================================================
        // 계산 프로퍼티 (Computed Property)
        // ============================================================

        /// <summary>
        /// 화면에 표시할 요약 텍스트.
        /// [NotifyPropertyChangedFor]에 의해 Name, Age 변경 시 자동 알림.
        /// </summary>
        public string DisplayText => $"{Name} ({Age}세)";

        /// <summary>
        /// 저장 가능 여부.
        /// </summary>
        public bool CanSave =>
            !IsBusy
            && !string.IsNullOrWhiteSpace(Name)
            && !string.IsNullOrWhiteSpace(Email)
            && Age > 0 && Age < 150;

        // ============================================================
        // 생성자
        // ============================================================
        public UserViewModel()
        {
            // 초기 데이터 로드
            LoadSampleData();
        }

        // ============================================================
        // 커맨드
        //
        // [RelayCommand]만 붙이면 끝!
        // → ICommand 프로퍼티가 자동 생성됨
        // → 비동기 메서드면 자동으로 AsyncRelayCommand 사용
        // ============================================================

        /// <summary>
        /// 사용자 정보를 저장합니다 (비동기).
        /// → SaveUserCommand 프로퍼티가 자동 생성됨
        /// </summary>
        [RelayCommand(CanExecute = nameof(CanSave))]
        private async Task SaveUserAsync()
        {
            IsBusy = true;
            StatusMessage = "저장 중...";

            try
            {
                // 실제로는 DB 저장 로직
                await Task.Delay(2000);
                StatusMessage = $"'{Name}' 정보가 저장되었습니다.";
            }
            catch (Exception ex)
            {
                StatusMessage = $"저장 실패: {ex.Message}";
            }
            finally
            {
                IsBusy = false;
            }
        }

        /// <summary>
        /// 폼을 초기 상태로 리셋합니다.
        /// → ResetFormCommand 프로퍼티가 자동 생성됨
        /// </summary>
        [RelayCommand(CanExecute = nameof(CanResetForm))]
        private void ResetForm()
        {
            Name = string.Empty;
            Email = string.Empty;
            Age = 0;
            Department = string.Empty;
            IsActive = true;
            StatusMessage = "폼이 초기화되었습니다.";

            // 유효성 검사 오류도 초기화
            ClearErrors();
        }

        /// <summary>
        /// 리셋 가능 여부.
        /// </summary>
        private bool CanResetForm() => !IsBusy;

        // ============================================================
        // 헬퍼 메서드
        // ============================================================

        /// <summary>
        /// 샘플 데이터를 로드합니다.
        /// </summary>
        private void LoadSampleData()
        {
            Name = "홍길동";
            Email = "hong@example.com";
            Age = 30;
            Department = "개발팀";
            IsActive = true;
        }

        /// <summary>
        /// 현재 ViewModel의 데이터를 Model 객체로 변환합니다.
        /// </summary>
        private UserModel CreateUserModel()
        {
            return new UserModel
            {
                Name = this.Name,
                Email = this.Email,
                Age = this.Age,
                Department = this.Department,
                IsActive = this.IsActive
            };
        }
    }
}
```

### 10.4 코드 양 비교

같은 기능을 구현한 ViewModel의 코드 줄 수를 비교합니다:

| 항목 | 수동 방식 | Toolkit 방식 | 감소율 |
|------|----------|-------------|--------|
| 기본 클래스 (ViewModelBase) | ~40줄 (직접 구현) | 0줄 (ObservableObject 상속) | **100%** |
| RelayCommand | ~45줄 (직접 구현) | 0줄 ([RelayCommand] 특성) | **100%** |
| AsyncRelayCommand | ~55줄 (직접 구현) | 0줄 ([RelayCommand] 특성) | **100%** |
| UserViewModel | ~170줄 | **~100줄** | **~40%** |
| **합계** | **~310줄** | **~100줄** | **약 68% 감소** |

> 기본 인프라 코드(ViewModelBase, RelayCommand 등)가 **완전히 사라지고**, ViewModel 자체의 보일러플레이트도 크게 줄었습니다.

### 10.5 View — UserView.xaml (거의 변경 없음)

```xml
<UserControl x:Class="MyApp.Views.UserView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             Padding="20">

    <UserControl.DataContext>
        <vm:UserViewModel />
    </UserControl.DataContext>

    <StackPanel MaxWidth="500">

        <!-- 제목 -->
        <TextBlock Text="사용자 정보 편집"
                   FontSize="24"
                   FontWeight="Bold"
                   Margin="0,0,0,20" />

        <!-- 표시 텍스트 -->
        <TextBlock Text="{Binding DisplayText}"
                   FontSize="16"
                   Foreground="Gray"
                   Margin="0,0,0,15" />

        <!-- 이름 입력 + 유효성 검사 -->
        <TextBlock Text="이름" FontWeight="SemiBold" />
        <TextBox x:Name="nameTextBox"
                 Text="{Binding Name,
                        UpdateSourceTrigger=PropertyChanged,
                        ValidatesOnDataErrors=True}"
                 Margin="0,4,0,2" />
        <!-- 유효성 검사 오류 메시지 표시 -->
        <TextBlock Text="{Binding (Validation.Errors)[0].ErrorContent,
                          ElementName=nameTextBox}"
                   Foreground="Red"
                   FontSize="11"
                   Margin="0,0,0,8" />

        <!-- 이메일 입력 + 유효성 검사 -->
        <TextBlock Text="이메일" FontWeight="SemiBold" />
        <TextBox x:Name="emailTextBox"
                 Text="{Binding Email,
                        UpdateSourceTrigger=PropertyChanged,
                        ValidatesOnDataErrors=True}"
                 Margin="0,4,0,2" />
        <TextBlock Text="{Binding (Validation.Errors)[0].ErrorContent,
                          ElementName=emailTextBox}"
                   Foreground="Red"
                   FontSize="11"
                   Margin="0,0,0,8" />

        <!-- 나이 입력 -->
        <TextBlock Text="나이" FontWeight="SemiBold" />
        <TextBox Text="{Binding Age, UpdateSourceTrigger=PropertyChanged}"
                 Margin="0,4,0,12" />

        <!-- 부서 입력 -->
        <TextBlock Text="부서" FontWeight="SemiBold" />
        <TextBox Text="{Binding Department, UpdateSourceTrigger=PropertyChanged}"
                 Margin="0,4,0,12" />

        <!-- 활성 상태 체크박스 -->
        <CheckBox Content="활성 상태"
                  IsChecked="{Binding IsActive}"
                  Margin="0,0,0,15" />

        <!-- 버튼들 -->
        <StackPanel Orientation="Horizontal" Margin="0,0,0,15">
            <!--
                Command 바인딩은 수동 방식과 동일합니다!
                차이점은 ViewModel 쪽에서 커맨드를 만드는 방식뿐.
                View(XAML) 입장에서는 전혀 차이가 없습니다.
            -->
            <Button Content="저장"
                    Command="{Binding SaveUserCommand}"
                    Width="100"
                    Height="35"
                    Margin="0,0,10,0" />

            <Button Content="초기화"
                    Command="{Binding ResetFormCommand}"
                    Width="100"
                    Height="35" />
        </StackPanel>

        <!-- 로딩 표시 -->
        <ProgressBar IsIndeterminate="{Binding IsBusy}"
                     Height="4"
                     Margin="0,0,0,10"
                     Visibility="{Binding IsBusy,
                         Converter={StaticResource BooleanToVisibilityConverter}}" />

        <!-- 상태 메시지 -->
        <TextBlock Text="{Binding StatusMessage}"
                   Foreground="DarkGreen"
                   FontSize="13" />

    </StackPanel>
</UserControl>
```

> **View의 변경 사항**: 커맨드 이름이 `SaveCommand` → `SaveUserCommand`, `ResetCommand` → `ResetFormCommand`로 바뀐 것뿐입니다. 이는 Toolkit의 네이밍 규칙(메서드 이름 + `Command`)에 따른 것입니다. 유효성 검사 관련 부분(`ValidatesOnDataErrors`, 오류 메시지 표시)이 추가되었습니다.

---

## 11. partial class 주의사항

CommunityToolkit.Mvvm의 소스 제네레이터를 사용하려면 반드시 `partial` 키워드를 사용해야 합니다. 여기에 몇 가지 주의사항이 있습니다.

### 11.1 반드시 partial이어야 하는 이유

```csharp
// ❌ 컴파일 에러! partial이 없으면 소스 제네레이터가 코드를 추가할 수 없음
public class UserViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name;  // 에러 발생!
}

// ✅ 정상 작동 — partial 필수!
public partial class UserViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name;  // 소스 제네레이터가 Name 프로퍼티를 생성
}
```

`partial` 키워드를 사용하면 **하나의 클래스를 여러 파일에 나눠서 작성**할 수 있습니다. 소스 제네레이터는 별도의 파일(`.g.cs`)에 프로퍼티와 커맨드 코드를 생성하고, 컴파일 시 원본 클래스와 합쳐집니다.

```
[컴파일 시점에 일어나는 일]

여러분이 작성:                     소스 제네레이터가 생성:
┌──────────────────────────┐     ┌──────────────────────────┐
│ UserViewModel.cs          │     │ UserViewModel.g.cs       │
│                           │     │                          │
│ public partial class      │  +  │ public partial class     │
│   UserViewModel           │     │   UserViewModel          │
│ {                         │     │ {                        │
│   [ObservableProperty]    │     │   public string Name     │
│   private string _name;   │     │   {                      │
│ }                         │     │     get => _name;         │
│                           │     │     set => ...            │
│                           │     │   }                      │
│                           │     │ }                        │
└──────────────────────────┘     └──────────────────────────┘
               │                              │
               └──────────┬───────────────────┘
                          ▼
              컴파일 시 하나의 클래스로 합쳐짐
```

### 11.2 중첩 클래스(Nested Class)에서의 사용

포함하는 클래스(바깥 클래스)도 `partial`이어야 합니다:

```csharp
// ❌ 바깥 클래스에 partial이 없으면 에러
public class MainWindow
{
    public partial class InnerViewModel : ObservableObject
    {
        [ObservableProperty]
        private string _title;  // 에러!
    }
}

// ✅ 바깥 클래스도 partial이어야 함
public partial class MainWindow
{
    public partial class InnerViewModel : ObservableObject
    {
        [ObservableProperty]
        private string _title;  // 정상 작동
    }
}
```

### 11.3 생성된 프로퍼티와의 이름 충돌 주의

소스 제네레이터가 생성할 프로퍼티 이름과 이미 존재하는 멤버 이름이 충돌하면 안 됩니다:

```csharp
public partial class UserViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name;         // → Name 프로퍼티가 생성됨

    // ❌ Name이라는 이름의 다른 멤버가 있으면 충돌!
    // public string Name => "충돌!";  // 컴파일 에러

    // ❌ 같은 이름의 메서드도 충돌 가능
    // public void Name() { }

    // ✅ 다른 이름을 사용하면 OK
    public string DisplayName => $"사용자: {Name}";
}
```

### 11.4 필드 접근에 대한 주의

`[ObservableProperty]`를 사용할 때, ViewModel 내부에서는 **필드(_name)**와 **프로퍼티(Name)** 모두 접근 가능합니다. 하지만 **반드시 프로퍼티를 통해 값을 변경**해야 합니다:

```csharp
public partial class UserViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name = string.Empty;

    private void SomeMethod()
    {
        // ❌ 필드에 직접 대입 → PropertyChanged 이벤트가 발생하지 않음!
        _name = "홍길동";  // UI가 갱신되지 않음!

        // ✅ 프로퍼티를 통해 대입 → PropertyChanged 이벤트 발생
        Name = "홍길동";   // UI가 정상적으로 갱신됨
    }
}
```

> **예외**: `OnNameChanged` partial 메서드 안에서 `_name` 값을 읽는 것은 괜찮습니다 (이미 값이 설정된 후이므로). 하지만 값을 **변경**할 때는 항상 프로퍼티(`Name`)를 사용하세요.

---

## 핵심 정리

| 기능 | 수동 방식 | CommunityToolkit.Mvvm |
|------|----------|----------------------|
| 기본 클래스 | ViewModelBase 직접 구현 (~40줄) | `ObservableObject` 상속 (0줄) |
| 프로퍼티 | 백킹 필드 + getter + setter + SetProperty (7~12줄/개) | `[ObservableProperty]` (2줄/개) |
| 변경 연동 | setter에서 OnPropertyChanged 수동 호출 | `[NotifyPropertyChangedFor]` 선언 |
| 커맨드 | RelayCommand 클래스 직접 구현 (~45줄) + 생성자에서 초기화 | `[RelayCommand]` 특성 (0줄 추가) |
| 비동기 커맨드 | AsyncRelayCommand 직접 구현 (~55줄) | `[RelayCommand]` + Task 반환 메서드 |
| CanExecute 연동 | setter에서 수동 재평가 | `[NotifyCanExecuteChangedFor]` |
| 유효성 검사 | 직접 구현 | `ObservableValidator` + 데이터 어노테이션 |
| VM 간 통신 | 직접 구현 (이벤트 또는 서비스) | `WeakReferenceMessenger` |

---

## 다음 단계

수동 방식과 CommunityToolkit.Mvvm을 모두 살펴봤습니다. 다음 문서에서 두 방식을 나란히 놓고 비교 요약합니다:

> **다음**: [일반 MVVM vs CommunityToolkit.Mvvm 비교 요약](./03-comparison-summary.md)
