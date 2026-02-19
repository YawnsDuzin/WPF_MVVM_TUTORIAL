# 일반적인 MVVM 구현 (수동 방식)

> **목표**: INotifyPropertyChanged, ICommand를 직접 구현하여 MVVM 패턴을 수동으로 구축하는 방법을 배웁니다.
> 이 문서를 먼저 읽은 뒤 [다음 문서](./02-communitytoolkit-mvvm.md)에서 CommunityToolkit.Mvvm으로 얼마나 간단해지는지 비교해 보세요.

---

## 목차

1. [ViewModelBase 클래스 — INotifyPropertyChanged 직접 구현](#1-viewmodelbase-클래스--inotifypropertychanged-직접-구현)
2. [RelayCommand 직접 구현 — ICommand 인터페이스](#2-relaycommand-직접-구현--icommand-인터페이스)
3. [AsyncRelayCommand 직접 구현](#3-asyncrelaycommand-직접-구현)
4. [완전한 예제: 사용자 정보 편집 화면](#4-완전한-예제-사용자-정보-편집-화면)
5. [단점 분석](#5-단점-분석)

---

## 1. ViewModelBase 클래스 — INotifyPropertyChanged 직접 구현

### 1.1 왜 INotifyPropertyChanged가 필요한가?

WinForms에서는 텍스트박스의 값을 직접 읽고 써야 했습니다:

```csharp
// WinForms 방식 — 컨트롤을 직접 조작
txtName.Text = user.Name;
user.Name = txtName.Text;
```

WPF에서는 **데이터 바인딩**으로 UI와 데이터를 연결합니다. 하지만 바인딩이 작동하려면 "데이터가 바뀌었다"는 것을 WPF에게 알려줘야 합니다. 이것이 바로 `INotifyPropertyChanged` 인터페이스의 역할입니다.

Python으로 비유하면, **프로퍼티 setter에서 이벤트를 발생시키는 것**과 비슷합니다:

```python
# Python 비유 — 값이 바뀔 때 알림을 보내는 개념
class Observable:
    def __init__(self):
        self._name = ""
        self._listeners = []

    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, value):
        if self._name != value:
            self._name = value
            self._notify("name")  # "name이 바뀌었어!" 라고 알림

    def _notify(self, property_name):
        for listener in self._listeners:
            listener(property_name)
```

### 1.2 INotifyPropertyChanged 인터페이스 살펴보기

```csharp
// .NET에 내장된 인터페이스 — 딱 하나의 이벤트만 정의되어 있음
namespace System.ComponentModel
{
    public interface INotifyPropertyChanged
    {
        event PropertyChangedEventHandler? PropertyChanged;
    }
}
```

이 인터페이스를 구현한 객체의 프로퍼티가 바뀌면, WPF 바인딩 엔진이 `PropertyChanged` 이벤트를 구독하고 있다가 UI를 자동으로 갱신합니다.

### 1.3 ViewModelBase 클래스 구현

모든 ViewModel에서 `INotifyPropertyChanged`를 반복 구현하지 않도록, **기본 클래스(Base Class)**를 만듭니다:

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace MyApp.ViewModels
{
    /// <summary>
    /// 모든 ViewModel의 기본 클래스.
    /// INotifyPropertyChanged를 구현하여 프로퍼티 변경 알림 기능을 제공합니다.
    /// </summary>
    public abstract class ViewModelBase : INotifyPropertyChanged
    {
        // WPF 바인딩 엔진이 이 이벤트를 구독합니다.
        // 프로퍼티 값이 바뀌면 이 이벤트를 발생시켜 UI를 갱신합니다.
        public event PropertyChangedEventHandler? PropertyChanged;

        /// <summary>
        /// PropertyChanged 이벤트를 발생시킵니다.
        /// </summary>
        /// <param name="propertyName">
        /// 변경된 프로퍼티 이름.
        /// [CallerMemberName]을 사용하면 호출한 프로퍼티 이름이 자동으로 채워집니다.
        /// </param>
        protected virtual void OnPropertyChanged([CallerMemberName] string? propertyName = null)
        {
            // null 조건 연산자(?.)로 구독자가 있을 때만 이벤트 발생
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }

        /// <summary>
        /// 프로퍼티 값을 설정하고, 값이 실제로 변경되었을 때만 알림을 보냅니다.
        /// </summary>
        /// <typeparam name="T">프로퍼티의 타입</typeparam>
        /// <param name="field">프로퍼티의 백킹 필드 (ref로 전달)</param>
        /// <param name="value">새로운 값</param>
        /// <param name="propertyName">
        /// 프로퍼티 이름 — [CallerMemberName]이 자동으로 채워줌
        /// </param>
        /// <returns>값이 실제로 변경되었으면 true, 같은 값이면 false</returns>
        protected bool SetProperty<T>(
            ref T field,
            T value,
            [CallerMemberName] string? propertyName = null)
        {
            // 현재 값과 새 값이 같으면 아무것도 하지 않음 (불필요한 UI 갱신 방지)
            if (EqualityComparer<T>.Default.Equals(field, value))
                return false;

            field = value;              // 백킹 필드에 새 값 저장
            OnPropertyChanged(propertyName);  // UI에 "값이 바뀌었어!" 알림
            return true;
        }
    }
}
```

### 1.4 [CallerMemberName]이 뭔가요?

`[CallerMemberName]`은 **컴파일 시점**에 호출한 멤버의 이름을 자동으로 채워주는 특성입니다:

```csharp
// [CallerMemberName] 덕분에 프로퍼티 이름을 문자열로 직접 쓰지 않아도 됩니다.

// ❌ CallerMemberName이 없다면 — 오타 위험!
public string Name
{
    get => _name;
    set
    {
        _name = value;
        OnPropertyChanged("Name");   // "Name"을 "Nme"로 오타 내면 바인딩이 안 됨!
        // OnPropertyChanged("Nme"); // 이렇게 오타 나도 컴파일 에러가 안 남!
    }
}

// ✅ CallerMemberName 사용 — 컴파일러가 "Name"을 자동으로 넣어줌
public string Name
{
    get => _name;
    set => SetProperty(ref _name, value);  // propertyName 인자를 생략해도 "Name"이 들어감
}
```

### 1.5 SetProperty 헬퍼 메서드의 동작 흐름

```
                 SetProperty(ref _name, "홍길동")
                              │
                   ┌──────────▼──────────┐
                   │ _name == "홍길동"?   │
                   │ (현재 값과 비교)      │
                   └──────────┬──────────┘
                      │              │
                    같음            다름
                      │              │
                      ▼              ▼
                 return false   _name = "홍길동"
                 (아무 일 없음)        │
                                     ▼
                            OnPropertyChanged("Name")
                                     │
                                     ▼
                            PropertyChanged 이벤트 발생
                                     │
                                     ▼
                            WPF 바인딩 엔진이 UI 갱신
```

---

## 2. RelayCommand 직접 구현 — ICommand 인터페이스

### 2.1 WinForms와의 차이: 이벤트 핸들러 vs 커맨드

WinForms에서는 버튼 클릭을 **이벤트 핸들러**로 처리했습니다:

```csharp
// WinForms 방식 — 코드비하인드에 이벤트 핸들러 직접 작성
private void btnSave_Click(object sender, EventArgs e)
{
    SaveUser();
}
```

WPF MVVM에서는 **커맨드(Command)** 패턴을 사용합니다. 버튼의 `Command` 프로퍼티에 ViewModel의 커맨드를 바인딩합니다:

```xml
<!-- WPF 방식 — XAML에서 커맨드를 바인딩 -->
<Button Content="저장" Command="{Binding SaveCommand}" />
```

### 2.2 ICommand 인터페이스

```csharp
// .NET에 내장된 인터페이스
namespace System.Windows.Input
{
    public interface ICommand
    {
        // 커맨드를 실행할 수 있는 상태인지 판단
        // false를 반환하면 버튼이 자동으로 비활성화됨 (IsEnabled = false)
        bool CanExecute(object? parameter);

        // 실제 실행할 동작
        void Execute(object? parameter);

        // CanExecute의 결과가 바뀌었을 때 발생시키는 이벤트
        // WPF가 이 이벤트를 듣고 버튼의 활성/비활성 상태를 갱신
        event EventHandler? CanExecuteChanged;
    }
}
```

### 2.3 RelayCommand 구현

```csharp
using System.Windows.Input;

namespace MyApp.Commands
{
    /// <summary>
    /// 재사용 가능한 ICommand 구현체.
    /// 실행 로직(Action)과 실행 가능 여부(Predicate)를 생성자에서 받습니다.
    /// "Relay(중계)"라는 이름은 실행 로직을 외부에서 주입받아 중계한다는 의미입니다.
    /// </summary>
    public class RelayCommand : ICommand
    {
        // 실행할 동작을 담는 델리게이트
        private readonly Action<object?> _execute;

        // 실행 가능 여부를 판단하는 델리게이트 (null이면 항상 실행 가능)
        private readonly Predicate<object?>? _canExecute;

        /// <summary>
        /// RelayCommand 생성자.
        /// </summary>
        /// <param name="execute">실행할 동작 (필수)</param>
        /// <param name="canExecute">실행 가능 여부 판단 (선택 — null이면 항상 true)</param>
        public RelayCommand(Action<object?> execute, Predicate<object?>? canExecute = null)
        {
            _execute = execute ?? throw new ArgumentNullException(nameof(execute));
            _canExecute = canExecute;
        }

        /// <summary>
        /// CanExecuteChanged 이벤트.
        /// CommandManager.RequerySuggested에 연결하면
        /// WPF가 적절한 시점에 자동으로 CanExecute를 다시 평가합니다.
        /// (예: 포커스 변경, 키 입력 등)
        /// </summary>
        public event EventHandler? CanExecuteChanged
        {
            add => CommandManager.RequerySuggested += value;
            remove => CommandManager.RequerySuggested -= value;
        }

        /// <summary>
        /// 커맨드 실행 가능 여부를 반환합니다.
        /// </summary>
        public bool CanExecute(object? parameter)
        {
            // _canExecute가 null이면 항상 true (=항상 실행 가능)
            return _canExecute is null || _canExecute(parameter);
        }

        /// <summary>
        /// 커맨드를 실행합니다.
        /// </summary>
        public void Execute(object? parameter)
        {
            _execute(parameter);
        }
    }
}
```

### 2.4 CommandManager.RequerySuggested란?

`CommandManager.RequerySuggested`는 WPF에 내장된 이벤트로, UI 상태가 변할 때(포커스 변경, 키 입력 등) 자동으로 발생합니다. 이 이벤트에 연결해 두면 WPF가 알아서 `CanExecute`를 다시 호출하여 버튼의 활성/비활성 상태를 갱신합니다.

```
사용자가 텍스트를 입력
        │
        ▼
CommandManager.RequerySuggested 이벤트 발생
        │
        ▼
WPF가 바인딩된 모든 커맨드의 CanExecute() 재호출
        │
        ▼
CanExecute()가 true/false 반환
        │
        ▼
버튼 활성/비활성 상태 자동 갱신
```

> **주의**: `CommandManager.RequerySuggested`는 편리하지만, 바인딩된 **모든** 커맨드의 `CanExecute`를 재호출하므로 커맨드 수가 많으면 성능에 영향을 줄 수 있습니다.

### 2.5 제네릭 버전 — RelayCommand\<T\>

파라미터 타입을 명시적으로 지정할 수 있는 제네릭 버전도 만들어 봅시다:

```csharp
using System.Windows.Input;

namespace MyApp.Commands
{
    /// <summary>
    /// 타입 안전한 제네릭 RelayCommand.
    /// CommandParameter로 특정 타입의 데이터를 받을 때 사용합니다.
    /// </summary>
    /// <typeparam name="T">커맨드 파라미터의 타입</typeparam>
    public class RelayCommand<T> : ICommand
    {
        private readonly Action<T?> _execute;
        private readonly Predicate<T?>? _canExecute;

        public RelayCommand(Action<T?> execute, Predicate<T?>? canExecute = null)
        {
            _execute = execute ?? throw new ArgumentNullException(nameof(execute));
            _canExecute = canExecute;
        }

        public event EventHandler? CanExecuteChanged
        {
            add => CommandManager.RequerySuggested += value;
            remove => CommandManager.RequerySuggested -= value;
        }

        public bool CanExecute(object? parameter)
        {
            // parameter를 T 타입으로 캐스팅
            if (parameter is null && default(T) is not null)
                return false;

            return _canExecute is null || _canExecute((T?)parameter);
        }

        public void Execute(object? parameter)
        {
            _execute((T?)parameter);
        }
    }
}
```

---

## 3. AsyncRelayCommand 직접 구현

### 3.1 왜 비동기 커맨드가 필요한가?

WPF는 **단일 UI 스레드**로 동작합니다. 파일 저장, DB 조회, API 호출 같은 시간이 걸리는 작업을 UI 스레드에서 동기적으로 실행하면 화면이 **멈춥니다(프리징)**:

```csharp
// ❌ 위험: UI 스레드에서 동기적으로 DB 조회 → 화면 프리징
private void SaveUser(object? parameter)
{
    Thread.Sleep(3000); // 3초 동안 화면이 멈춤!
    // DB에 사용자 저장...
}

// ✅ 안전: 비동기로 처리 → UI 스레드가 차단되지 않음
private async Task SaveUserAsync()
{
    await Task.Delay(3000); // 3초 동안 다른 UI 작업 가능
    // DB에 사용자 저장...
}
```

Python의 `asyncio`와 비슷한 개념입니다:

```python
# Python asyncio 비유
async def save_user():
    await asyncio.sleep(3)  # 비동기 대기 — 이벤트 루프가 다른 작업 처리 가능
    # DB에 사용자 저장...
```

### 3.2 AsyncRelayCommand 구현

```csharp
using System.Windows.Input;

namespace MyApp.Commands
{
    /// <summary>
    /// 비동기 작업을 지원하는 RelayCommand.
    /// async/await 패턴을 사용하는 메서드를 커맨드로 바인딩할 수 있게 해줍니다.
    /// </summary>
    public class AsyncRelayCommand : ICommand
    {
        // 비동기 실행 로직 (Task를 반환하는 Func)
        private readonly Func<object?, Task> _execute;
        private readonly Predicate<object?>? _canExecute;

        // 현재 실행 중인지 추적 — 중복 실행 방지
        private bool _isExecuting;

        public AsyncRelayCommand(
            Func<object?, Task> execute,
            Predicate<object?>? canExecute = null)
        {
            _execute = execute ?? throw new ArgumentNullException(nameof(execute));
            _canExecute = canExecute;
        }

        public event EventHandler? CanExecuteChanged
        {
            add => CommandManager.RequerySuggested += value;
            remove => CommandManager.RequerySuggested -= value;
        }

        public bool CanExecute(object? parameter)
        {
            // 이미 실행 중이면 실행 불가 (버튼 비활성화)
            // + 외부 조건(_canExecute)도 만족해야 함
            return !_isExecuting
                && (_canExecute is null || _canExecute(parameter));
        }

        public async void Execute(object? parameter)
        {
            // ICommand.Execute의 반환 타입이 void이므로 async void를 사용해야 합니다.
            // 일반적으로 async void는 피해야 하지만, 이벤트 핸들러 성격의 메서드는 예외입니다.

            if (!CanExecute(parameter))
                return;

            _isExecuting = true;
            RaiseCanExecuteChanged();  // 실행 중 상태를 UI에 반영 (버튼 비활성화)

            try
            {
                await _execute(parameter);
            }
            finally
            {
                _isExecuting = false;
                RaiseCanExecuteChanged();  // 실행 완료 후 UI에 반영 (버튼 활성화)
            }
        }

        /// <summary>
        /// CanExecute 상태가 변경되었음을 수동으로 알립니다.
        /// CommandManager에만 의존하지 않고, 필요한 시점에 직접 호출할 수 있습니다.
        /// </summary>
        public void RaiseCanExecuteChanged()
        {
            CommandManager.InvalidateRequerySuggested();
        }
    }
}
```

### 3.3 async void에 대한 참고

`ICommand.Execute`의 반환 타입은 `void`입니다. 따라서 비동기 커맨드에서는 불가피하게 `async void`를 사용합니다. 일반적으로 `async void`는 예외 처리가 어려워 권장되지 않지만, **이벤트 핸들러 성격의 메서드**에서는 허용됩니다. 단, `try-finally`로 반드시 감싸야 합니다.

---

## 4. 완전한 예제: 사용자 정보 편집 화면

지금까지 만든 `ViewModelBase`, `RelayCommand`, `AsyncRelayCommand`를 사용하여 **사용자 정보 편집 화면**을 구현해 봅시다.

### 4.1 프로젝트 구조

```
MyApp/
├── Models/
│   └── UserModel.cs              ← 데이터 모델
├── ViewModels/
│   ├── ViewModelBase.cs          ← INotifyPropertyChanged 기본 클래스
│   └── UserViewModel.cs          ← 사용자 편집 ViewModel
├── Commands/
│   ├── RelayCommand.cs           ← ICommand 구현
│   └── AsyncRelayCommand.cs      ← 비동기 ICommand 구현
└── Views/
    └── UserView.xaml             ← UI 화면
    └── UserView.xaml.cs          ← 코드비하인드 (거의 비어 있음)
```

### 4.2 Model — UserModel.cs

```csharp
namespace MyApp.Models
{
    /// <summary>
    /// 사용자 데이터를 담는 모델 클래스.
    /// 순수한 데이터 객체(POCO) — 비즈니스 로직이나 UI 로직 없음.
    /// Python의 dataclass와 비슷한 역할입니다.
    /// </summary>
    public class UserModel
    {
        // 사용자 고유 ID
        public int Id { get; set; }

        // 사용자 이름 (예: "홍길동")
        public string Name { get; set; } = string.Empty;

        // 이메일 주소
        public string Email { get; set; } = string.Empty;

        // 나이
        public int Age { get; set; }

        // 부서명
        public string Department { get; set; } = string.Empty;

        // 활성 상태 여부
        public bool IsActive { get; set; } = true;
    }
}
```

### 4.3 ViewModel — UserViewModel.cs (수동 구현)

**여기가 핵심입니다.** 모든 프로퍼티와 커맨드를 수동으로 구현하면 얼마나 많은 코드가 필요한지 확인하세요:

```csharp
using System.Windows;
using MyApp.Commands;
using MyApp.Models;

namespace MyApp.ViewModels
{
    /// <summary>
    /// 사용자 정보 편집을 위한 ViewModel.
    ///
    /// [수동 MVVM 구현]
    /// 모든 바인딩 프로퍼티를 하나하나 직접 작성해야 합니다.
    /// 프로퍼티 하나당 약 10줄의 반복적인 코드가 필요합니다.
    /// </summary>
    public class UserViewModel : ViewModelBase
    {
        // ============================================================
        // 백킹 필드 선언
        // 각 프로퍼티마다 하나씩 필요합니다.
        // ============================================================
        private string _name = string.Empty;
        private string _email = string.Empty;
        private int _age;
        private string _department = string.Empty;
        private bool _isActive = true;
        private string _statusMessage = string.Empty;
        private bool _isBusy;

        // ============================================================
        // 프로퍼티들
        // 프로퍼티 하나당 get/set + SetProperty 호출 + 추가 로직이 필요합니다.
        // ============================================================

        /// <summary>
        /// 사용자 이름.
        /// 이름이 바뀌면 SaveCommand의 CanExecute도 다시 평가해야 합니다.
        /// </summary>
        public string Name
        {
            get => _name;
            set
            {
                if (SetProperty(ref _name, value))
                {
                    // 이름이 바뀌면 저장 버튼 활성화 상태 재평가
                    // (이름이 비어있으면 저장 불가)
                    OnPropertyChanged(nameof(CanSave));
                }
            }
        }

        /// <summary>
        /// 이메일 주소.
        /// </summary>
        public string Email
        {
            get => _email;
            set
            {
                if (SetProperty(ref _email, value))
                {
                    // 이메일이 바뀌면 저장 버튼 활성화 상태 재평가
                    OnPropertyChanged(nameof(CanSave));
                }
            }
        }

        /// <summary>
        /// 나이.
        /// </summary>
        public int Age
        {
            get => _age;
            set
            {
                if (SetProperty(ref _age, value))
                {
                    // 나이가 바뀌면 저장 버튼과 표시 텍스트 재평가
                    OnPropertyChanged(nameof(CanSave));
                    OnPropertyChanged(nameof(DisplayText));
                }
            }
        }

        /// <summary>
        /// 부서명.
        /// </summary>
        public string Department
        {
            get => _department;
            set => SetProperty(ref _department, value);
        }

        /// <summary>
        /// 활성 상태 여부.
        /// </summary>
        public bool IsActive
        {
            get => _isActive;
            set => SetProperty(ref _isActive, value);
        }

        /// <summary>
        /// 상태 메시지 (예: "저장 완료", "저장 중...")
        /// </summary>
        public string StatusMessage
        {
            get => _statusMessage;
            set => SetProperty(ref _statusMessage, value);
        }

        /// <summary>
        /// 현재 작업 중인지 여부 (로딩 표시용).
        /// IsBusy가 true이면 저장/리셋 버튼을 비활성화합니다.
        /// </summary>
        public bool IsBusy
        {
            get => _isBusy;
            set
            {
                if (SetProperty(ref _isBusy, value))
                {
                    // IsBusy 상태가 바뀌면 모든 커맨드의 CanExecute 재평가
                    OnPropertyChanged(nameof(CanSave));
                }
            }
        }

        // ============================================================
        // 계산된 프로퍼티 (Computed Property)
        // 다른 프로퍼티에 의존하므로, 의존하는 프로퍼티가 바뀔 때
        // OnPropertyChanged를 수동으로 호출해야 합니다.
        // ============================================================

        /// <summary>
        /// 화면에 표시할 요약 텍스트.
        /// Name과 Age에 의존 → 두 프로퍼티의 setter에서 수동으로 알림 필요.
        /// </summary>
        public string DisplayText => $"{Name} ({Age}세)";

        /// <summary>
        /// 저장 가능 여부.
        /// Name, Email, Age, IsBusy에 의존 → 모두의 setter에서 수동으로 알림 필요.
        /// </summary>
        public bool CanSave =>
            !IsBusy
            && !string.IsNullOrWhiteSpace(Name)
            && !string.IsNullOrWhiteSpace(Email)
            && Age > 0 && Age < 150;

        // ============================================================
        // 커맨드들
        // 각 커맨드를 프로퍼티로 노출하고, 생성자에서 초기화합니다.
        // ============================================================

        /// <summary>
        /// 저장 커맨드 (비동기) — "저장" 버튼에 바인딩
        /// </summary>
        public AsyncRelayCommand SaveCommand { get; }

        /// <summary>
        /// 리셋 커맨드 (동기) — "초기화" 버튼에 바인딩
        /// </summary>
        public RelayCommand ResetCommand { get; }

        // ============================================================
        // 생성자
        // ============================================================
        public UserViewModel()
        {
            // 커맨드 생성 — 실행 로직과 CanExecute 조건을 전달
            SaveCommand = new AsyncRelayCommand(
                execute: async _ => await SaveUserAsync(),       // 실행할 비동기 메서드
                canExecute: _ => CanSave                         // 실행 가능 조건
            );

            ResetCommand = new RelayCommand(
                execute: _ => ResetForm(),                       // 실행할 동기 메서드
                canExecute: _ => !IsBusy                         // 실행 가능 조건
            );

            // 초기 데이터 로드 (예시)
            LoadSampleData();
        }

        // ============================================================
        // 커맨드 실행 메서드들
        // ============================================================

        /// <summary>
        /// 사용자 정보를 저장합니다 (비동기).
        /// 실제로는 DB나 API에 저장하겠지만, 예제에서는 지연으로 대체합니다.
        /// </summary>
        private async Task SaveUserAsync()
        {
            IsBusy = true;
            StatusMessage = "저장 중...";

            try
            {
                // 실제로는 DB 저장 로직이 들어갈 자리
                // 예: await _userRepository.SaveAsync(CreateUserModel());
                await Task.Delay(2000); // 네트워크 지연 시뮬레이션

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
        /// </summary>
        private void ResetForm()
        {
            Name = string.Empty;
            Email = string.Empty;
            Age = 0;
            Department = string.Empty;
            IsActive = true;
            StatusMessage = "폼이 초기화되었습니다.";
        }

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

### 4.4 View — UserView.xaml

```xml
<UserControl x:Class="MyApp.Views.UserView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             Padding="20">

    <!-- ViewModel을 DataContext로 설정 -->
    <!-- 실제 프로젝트에서는 DI를 사용하지만, 예제에서는 직접 생성 -->
    <UserControl.DataContext>
        <vm:UserViewModel />
    </UserControl.DataContext>

    <StackPanel MaxWidth="500">

        <!-- 제목 -->
        <TextBlock Text="사용자 정보 편집"
                   FontSize="24"
                   FontWeight="Bold"
                   Margin="0,0,0,20" />

        <!-- 표시 텍스트 (DisplayText 계산 프로퍼티에 바인딩) -->
        <TextBlock Text="{Binding DisplayText}"
                   FontSize="16"
                   Foreground="Gray"
                   Margin="0,0,0,15" />

        <!-- 이름 입력 -->
        <TextBlock Text="이름" FontWeight="SemiBold" />
        <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}"
                 Margin="0,4,0,12" />
        <!--
            UpdateSourceTrigger=PropertyChanged:
            기본값은 LostFocus (포커스를 잃을 때 반영)이지만,
            타이핑할 때마다 즉시 ViewModel에 반영되도록 PropertyChanged로 설정.

            WinForms의 TextChanged 이벤트와 비슷한 효과.
        -->

        <!-- 이메일 입력 -->
        <TextBlock Text="이메일" FontWeight="SemiBold" />
        <TextBox Text="{Binding Email, UpdateSourceTrigger=PropertyChanged}"
                 Margin="0,4,0,12" />

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
                Command 바인딩:
                SaveCommand.CanExecute()가 false를 반환하면
                버튼이 자동으로 비활성화(회색)됩니다.
            -->
            <Button Content="저장"
                    Command="{Binding SaveCommand}"
                    Width="100"
                    Height="35"
                    Margin="0,0,10,0" />

            <Button Content="초기화"
                    Command="{Binding ResetCommand}"
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

### 4.5 View 코드비하인드 — UserView.xaml.cs

```csharp
namespace MyApp.Views
{
    /// <summary>
    /// UserView의 코드비하인드.
    /// MVVM 패턴에서는 코드비하인드를 최소한으로 유지합니다.
    /// 모든 로직은 ViewModel에 있으므로 여기는 거의 비어 있습니다.
    ///
    /// WinForms 개발자 참고:
    /// WinForms에서는 Form1.cs에 수백 줄의 이벤트 핸들러가 있었지만,
    /// WPF MVVM에서는 이 파일이 거의 빈 상태로 유지됩니다.
    /// </summary>
    public partial class UserView : UserControl
    {
        public UserView()
        {
            InitializeComponent();
        }
    }
}
```

---

## 5. 단점 분석

### 5.1 보일러플레이트 코드가 너무 많음

위의 `UserViewModel`에서 프로퍼티 **하나**를 구현하는 데 필요한 코드를 세어 봅시다:

```csharp
// 프로퍼티 1개를 구현하려면 이 모든 코드가 필요합니다:

// 1) 백킹 필드 선언
private string _name = string.Empty;

// 2) 프로퍼티 선언 (getter + setter + SetProperty + 추가 알림)
public string Name
{
    get => _name;
    set
    {
        if (SetProperty(ref _name, value))
        {
            OnPropertyChanged(nameof(CanSave));
            OnPropertyChanged(nameof(DisplayText));
        }
    }
}
```

**프로퍼티 1개 = 최소 7~12줄**. 프로퍼티가 20개인 ViewModel이라면? **140~240줄이 반복 코드**입니다.

### 5.2 프로퍼티 추가 시 반복되는 작업

새 프로퍼티를 하나 추가할 때마다 해야 할 일:

| 단계 | 작업 | 누락 시 문제 |
|------|------|-------------|
| 1 | 백킹 필드 `private T _field` 선언 | 컴파일 에러 |
| 2 | 프로퍼티 `get` 구현 | 컴파일 에러 |
| 3 | 프로퍼티 `set` 구현 + `SetProperty` 호출 | UI가 갱신되지 않음 (디버깅 어려움) |
| 4 | 관련 계산 프로퍼티의 `OnPropertyChanged` 추가 | 의존 프로퍼티가 갱신되지 않음 |
| 5 | 관련 커맨드의 `CanExecute` 재평가 | 버튼 활성 상태가 안 바뀜 |

### 5.3 오타 위험과 유지보수 어려움

```csharp
// ❌ 실수 1: OnPropertyChanged 호출을 빼먹음
public string Name
{
    get => _name;
    set => _name = value;  // SetProperty 호출 안 함 → UI가 갱신 안 됨!
}

// ❌ 실수 2: 관련 프로퍼티 알림을 빼먹음
public string Name
{
    get => _name;
    set
    {
        if (SetProperty(ref _name, value))
        {
            // OnPropertyChanged(nameof(DisplayText)); ← 이걸 깜빡하면
            // DisplayText가 Name 변경에 반응하지 않음!
        }
    }
}

// ❌ 실수 3: 프로퍼티 이름 오타 (CallerMemberName을 안 쓸 경우)
OnPropertyChanged("Naem");  // "Name"이 아님 — 컴파일 에러 없이 바인딩만 실패
```

### 5.4 코드 줄 수 요약

위 예제에서 **수동 MVVM 구현**에 필요한 코드:

| 클래스 | 대략적인 줄 수 | 역할 |
|--------|--------------|------|
| `ViewModelBase.cs` | ~40줄 | 기본 클래스 (재사용) |
| `RelayCommand.cs` | ~45줄 | 커맨드 구현 (재사용) |
| `AsyncRelayCommand.cs` | ~55줄 | 비동기 커맨드 (재사용) |
| `UserViewModel.cs` | **~170줄** | ViewModel (프로퍼티 7개 + 커맨드 2개) |
| **합계** | **~310줄** | |

> **"프로퍼티 7개, 커맨드 2개"를 구현하는 데 310줄**이 필요합니다.
> 실제 업무 화면에는 프로퍼티가 20~30개, 커맨드가 10개 이상인 경우가 흔합니다.

### 5.5 전통 MVVM의 문제 요약

```
┌──────────────────────────────────────────────────────────┐
│                전통적 수동 MVVM의 문제점                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. 보일러플레이트 지옥                                    │
│     → 프로퍼티 1개에 7~12줄의 반복 코드                     │
│     → ViewModel이 수백 줄로 비대해짐                        │
│                                                          │
│  2. 실수하기 쉬움                                          │
│     → OnPropertyChanged 호출 누락                          │
│     → 관련 프로퍼티 알림 누락                                │
│     → 프로퍼티 이름 오타                                     │
│                                                          │
│  3. 유지보수 어려움                                         │
│     → 프로퍼티 추가/삭제 시 여러 곳을 수정해야 함               │
│     → 의존 관계 파악이 어려움                                 │
│                                                          │
│  4. 기본 클래스(ViewModelBase, RelayCommand)를               │
│     직접 만들고 관리해야 함                                   │
│     → 프로젝트마다 미묘하게 다른 구현                          │
│     → 검증되지 않은 자체 구현의 버그 위험                       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 다음 단계

이 문제들을 **CommunityToolkit.Mvvm**이 어떻게 해결하는지 다음 문서에서 확인하세요:

> **다음**: [CommunityToolkit.Mvvm 사용 방법](./02-communitytoolkit-mvvm.md)

---

## 핵심 정리

| 개념 | 설명 | Python/WinForms 비유 |
|------|------|---------------------|
| `INotifyPropertyChanged` | 프로퍼티 변경을 UI에 알리는 인터페이스 | Python의 `@property.setter`에서 옵저버 호출 |
| `ViewModelBase` | 모든 ViewModel의 공통 기본 클래스 | Python의 `Observable` 기본 클래스 |
| `SetProperty` | 값 비교 + 필드 설정 + 변경 알림을 한 번에 | setter 내부의 `if old != new` + notify 로직 |
| `[CallerMemberName]` | 호출자의 멤버 이름을 자동으로 전달 | Python의 `__name__` 같은 자동 이름 해결 |
| `ICommand` | 사용자 동작을 캡슐화하는 인터페이스 | WinForms의 버튼 클릭 이벤트 핸들러 |
| `RelayCommand` | 재사용 가능한 ICommand 구현체 | 콜백 함수를 감싸는 래퍼 |
| `AsyncRelayCommand` | 비동기 작업용 ICommand 구현체 | Python의 `async def` + 콜백 패턴 |
| `CommandManager` | CanExecute 자동 재평가 매니저 | WinForms에 없는 개념 (수동 Enable/Disable) |
