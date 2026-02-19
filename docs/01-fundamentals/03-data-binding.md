# 데이터 바인딩 이해

> 이 문서는 WinForms에서 `textBox1.Text = value`처럼 직접 값을 대입하던 개발자를 대상으로,
> WPF의 핵심 기능인 **데이터 바인딩**을 처음부터 설명합니다.

---

## 목차

1. [WinForms의 데이터 표시 방식 vs WPF 데이터 바인딩](#1-winforms의-데이터-표시-방식-vs-wpf-데이터-바인딩)
2. [바인딩이란?](#2-바인딩이란)
3. [바인딩 모드](#3-바인딩-모드)
4. [INotifyPropertyChanged 인터페이스](#4-inotifypropertychanged-인터페이스)
5. [DataContext란?](#5-datacontext란)
6. [바인딩 문법](#6-바인딩-문법)
7. [바인딩 디버깅 방법](#7-바인딩-디버깅-방법)
8. [컬렉션 바인딩 (ObservableCollection)](#8-컬렉션-바인딩-observablecollection)
9. [IValueConverter (값 변환기)](#9-ivalueconverter-값-변환기)
10. [전체 흐름 정리](#10-전체-흐름-정리)

---

## 1. WinForms의 데이터 표시 방식 vs WPF 데이터 바인딩

### WinForms 방식: 직접 값을 넣고 꺼내기

WinForms에서는 데이터를 화면에 표시하거나, 화면에서 데이터를 가져올 때
**컨트롤의 속성에 직접 접근**해서 값을 대입합니다.

```csharp
// WinForms: 데이터 → UI (수동으로 컨트롤에 값 대입)
private void LoadUser(User user)
{
    txtName.Text = user.Name;           // 데이터 → TextBox
    txtEmail.Text = user.Email;         // 데이터 → TextBox
    lblAge.Text = user.Age.ToString();  // 데이터 → Label
    chkActive.Checked = user.IsActive;  // 데이터 → CheckBox
}

// WinForms: UI → 데이터 (수동으로 컨트롤에서 값 가져오기)
private User SaveUser()
{
    return new User
    {
        Name = txtName.Text,                    // TextBox → 데이터
        Email = txtEmail.Text,                  // TextBox → 데이터
        Age = int.Parse(lblAge.Text),           // Label → 데이터
        IsActive = chkActive.Checked            // CheckBox → 데이터
    };
}
```

**문제점**:
- 컨트롤이 많아지면 **대입 코드가 비대해짐**
- 데이터가 바뀔 때마다 **수동으로 UI를 갱신**해야 함
- UI에서 입력이 바뀔 때마다 **수동으로 데이터를 읽어**야 함
- 컨트롤 이름이 바뀌면 **여러 곳의 코드를 수정**해야 함

### WPF 방식: 데이터 바인딩으로 자동 동기화

WPF에서는 "이 컨트롤은 이 속성을 보여줘"라고 **선언**만 하면,
데이터가 바뀔 때 UI가 **자동으로 갱신**됩니다.

```xml
<!-- WPF: 바인딩을 한 번 선언하면 자동 동기화 -->
<TextBox Text="{Binding Name}" />
<TextBox Text="{Binding Email}" />
<TextBlock Text="{Binding Age}" />
<CheckBox IsChecked="{Binding IsActive}" />
<!--
    Name 속성이 바뀌면 → TextBox가 자동 갱신
    TextBox에 입력하면 → Name 속성이 자동 갱신
    양방향 자동 동기화!
-->
```

```csharp
// WPF: 데이터 객체의 속성만 변경하면 UI가 자동으로 반응!
public void UpdateUserName(string newName)
{
    // 이것만 하면 화면의 TextBox도 자동으로 바뀜!
    user.Name = newName;
    // txtName.Text = newName; ← 이런 코드가 필요 없음!
}
```

---

## 2. 바인딩이란?

**바인딩(Binding)** 은 **소스(Source)** 의 데이터와 **타깃(Target)** 의 UI 속성을
**자동으로 연결**하는 메커니즘입니다.

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│   소스(Source)              타깃(Target)              │
│   ┌─────────────┐         ┌─────────────────┐       │
│   │  ViewModel  │ ──────→ │  XAML 컨트롤     │       │
│   │             │         │                 │       │
│   │  Name="홍길동"│ ──────→ │ TextBox.Text    │       │
│   │             │ ←────── │                 │       │
│   └─────────────┘  바인딩  └─────────────────┘       │
│                                                      │
│   데이터(C# 객체)           UI(XAML 요소)             │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 바인딩의 4가지 핵심 요소

```xml
<TextBox Text="{Binding Path=Name,
                        Mode=TwoWay,
                        UpdateSourceTrigger=PropertyChanged}" />
<!--
    1. Source (소스): 데이터를 제공하는 객체 → DataContext에서 결정
    2. Path (경로): 소스 객체의 어떤 속성을 참조할 것인가 → Name
    3. Mode (모드): 데이터 흐름 방향 → TwoWay (양방향)
    4. Target (타깃): 데이터를 표시할 UI 속성 → TextBox.Text
-->
```

---

## 3. 바인딩 모드

바인딩 모드는 **데이터가 흐르는 방향**을 결정합니다.

### OneWay (소스 → 타깃, 단방향)

소스 데이터가 바뀌면 UI가 갱신되지만, UI에서 변경해도 소스에는 영향 없음.

```xml
<!-- 읽기 전용 표시에 적합 -->
<TextBlock Text="{Binding Name, Mode=OneWay}" />
<TextBlock Text="{Binding TotalPrice, Mode=OneWay}" />
<!--
    Name이 바뀌면 → TextBlock에 자동 반영 ✓
    TextBlock은 입력이 안 되니까 소스로 역방향은 해당 없음
-->
```

```
소스(ViewModel)  ──────→  타깃(UI)
Name = "홍길동"   ──────→  TextBlock.Text = "홍길동"
```

### TwoWay (소스 ↔ 타깃, 양방향)

소스가 바뀌면 UI가 갱신되고, UI에서 변경하면 소스도 갱신됨.

```xml
<!-- 입력 컨트롤에 적합 -->
<TextBox Text="{Binding Name, Mode=TwoWay}" />
<!--
    Name이 바뀌면 → TextBox에 자동 반영 ✓
    TextBox에 입력하면 → Name이 자동 갱신 ✓
-->
```

```
소스(ViewModel)  ←────→  타깃(UI)
Name = "홍길동"   ←────→  TextBox.Text = "홍길동"

사용자가 "김철수" 입력 → Name이 "김철수"로 자동 변경!
코드에서 Name = "이영희" → TextBox에 "이영희" 자동 표시!
```

### OneTime (최초 한 번만)

바인딩 시점에 한 번만 값을 가져오고, 이후 변경은 반영하지 않음.

```xml
<!-- 변경되지 않는 데이터 표시에 적합 (성능 최적화) -->
<TextBlock Text="{Binding CreatedDate, Mode=OneTime}" />
<TextBlock Text="{Binding AppVersion, Mode=OneTime}" />
```

### OneWayToSource (타깃 → 소스)

UI에서 변경한 값만 소스로 전달 (소스→UI 갱신은 안 됨). 드물게 사용.

```xml
<PasswordBox Password="{Binding Password, Mode=OneWayToSource}" />
<!-- 사용자가 입력한 비밀번호를 ViewModel로 전달만 함 -->
```

### 바인딩 모드 요약

```
Mode             방향           사용 사례
──────────────────────────────────────────────────
OneWay           Source → Target    읽기 전용 표시 (TextBlock)
TwoWay           Source ↔ Target    입력 컨트롤 (TextBox, CheckBox)
OneTime          Source → Target    한 번만 (고정 데이터)
OneWayToSource   Source ← Target    입력만 전달 (PasswordBox)
```

### 기본 바인딩 모드 (컨트롤별)

모드를 생략하면 **컨트롤의 기본값**이 적용됩니다.

| 컨트롤 / 속성 | 기본 Mode |
|---|---|
| `TextBlock.Text` | OneWay |
| `TextBox.Text` | TwoWay |
| `CheckBox.IsChecked` | TwoWay |
| `Slider.Value` | TwoWay |
| `ListBox.SelectedItem` | TwoWay |

```xml
<!-- 따라서 이 두 줄은 동일한 동작 -->
<TextBox Text="{Binding Name}" />
<TextBox Text="{Binding Name, Mode=TwoWay}" />

<!-- 이 두 줄도 동일한 동작 -->
<TextBlock Text="{Binding Name}" />
<TextBlock Text="{Binding Name, Mode=OneWay}" />
```

---

## 4. INotifyPropertyChanged 인터페이스

### 왜 필요한가?

WPF의 바인딩 시스템은 소스 데이터가 바뀌었는지 **자동으로 감지하지 못합니다.**
소스 객체가 "내 속성이 바뀌었어!"라고 **명시적으로 알려줘야** 합니다.

```csharp
// 이 클래스는 바인딩이 동작하지 않음!
public class UserBad
{
    public string Name { get; set; } = "";
    // Name을 바꿔도 UI에 반영 안 됨!
    // WPF가 Name이 바뀐 사실을 모르기 때문!
}

// 이 클래스는 바인딩이 정상 동작!
public class UserGood : INotifyPropertyChanged
{
    private string _name = "";

    public string Name
    {
        get => _name;
        set
        {
            _name = value;
            PropertyChanged?.Invoke(this,
                new PropertyChangedEventArgs(nameof(Name)));
            // "Name 속성이 바뀌었어!"라고 WPF에게 알림
        }
    }

    public event PropertyChangedEventHandler? PropertyChanged;
}
```

```
"그냥 속성" (INotifyPropertyChanged 없이):

코드: user.Name = "김철수";
WPF:  (아무 일도 안 일어남... Name이 바뀐 걸 모름)
UI:   [홍길동]  ← 그대로

"알림 속성" (INotifyPropertyChanged 있을 때):

코드: user.Name = "김철수";
      ↓
      PropertyChanged 이벤트 발생! ("Name이 바뀌었어!")
      ↓
WPF:  아, Name이 바뀌었구나! 새 값을 가져와서 UI를 갱신해야지!
      ↓
UI:   [김철수]  ← 자동 갱신!
```

### PropertyChanged 이벤트가 하는 일

```csharp
// PropertyChanged 이벤트의 동작 원리

// 1. ViewModel에서 속성 값 변경
Name = "김철수";

// 2. setter에서 PropertyChanged 이벤트 발생
PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("Name"));
//                       ↑ this: 이벤트를 발생시킨 객체 (ViewModel)
//                                                        ↑ "Name": 바뀐 속성의 이름

// 3. WPF 바인딩 엔진이 이 이벤트를 수신
//    → "Name" 속성에 바인딩된 모든 UI 요소를 찾음
//    → 해당 UI 요소들의 값을 새로운 Name 값으로 갱신

// 이 과정이 자동으로 일어남! 개발자는 PropertyChanged만 발생시키면 됨!
```

### 직접 구현하는 코드 예제

#### 기본 구현 (모든 ViewModel에서 반복되는 코드)

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

// ViewModel 기본 클래스: 모든 ViewModel이 이 클래스를 상속
public class ViewModelBase : INotifyPropertyChanged
{
    // PropertyChanged 이벤트 선언
    public event PropertyChangedEventHandler? PropertyChanged;

    // 속성 변경 알림을 보내는 헬퍼 메서드
    protected void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        // CallerMemberName: 이 메서드를 호출한 속성의 이름을 자동으로 채워줌
        // 예: Name 속성의 setter에서 호출하면 propertyName = "Name"
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }

    // 값을 설정하고 변경되었으면 알림을 보내는 헬퍼 메서드
    protected bool SetProperty<T>(ref T field, T value,
                                   [CallerMemberName] string? propertyName = null)
    {
        // 값이 같으면 아무것도 하지 않음 (불필요한 UI 갱신 방지)
        if (EqualityComparer<T>.Default.Equals(field, value))
            return false;

        field = value;              // 실제 값 변경
        OnPropertyChanged(propertyName);  // 변경 알림
        return true;
    }
}
```

#### ViewModel에서 사용하기

```csharp
// 사용자 정보를 표시/편집하는 ViewModel
public class UserViewModel : ViewModelBase
{
    // === 방법 1: OnPropertyChanged 직접 호출 ===
    private string _name = "";
    public string Name
    {
        get => _name;
        set
        {
            if (_name != value)       // 같은 값이면 무시
            {
                _name = value;
                OnPropertyChanged();  // CallerMemberName이 자동으로 "Name"을 넘김
            }
        }
    }

    // === 방법 2: SetProperty 헬퍼 사용 (더 간결!) ===
    private string _email = "";
    public string Email
    {
        get => _email;
        set => SetProperty(ref _email, value);  // 한 줄로 끝!
    }

    private int _age;
    public int Age
    {
        get => _age;
        set => SetProperty(ref _age, value);
    }

    private bool _isActive;
    public bool IsActive
    {
        get => _isActive;
        set => SetProperty(ref _isActive, value);
    }

    // === 계산된 속성 (다른 속성에 의존하는 속성) ===
    // 읽기 전용이지만, 의존하는 속성이 바뀔 때 알림을 보내야 함
    public string DisplayText => $"{Name} ({Age}세)";

    // Name이나 Age가 바뀌면 DisplayText도 바뀌어야 함
    // 따라서 Name, Age의 setter를 수정:
    // (위의 Name을 아래처럼 수정)
    /*
    public string Name
    {
        get => _name;
        set
        {
            if (SetProperty(ref _name, value))
            {
                OnPropertyChanged(nameof(DisplayText)); // DisplayText도 갱신!
            }
        }
    }
    */
}
```

#### XAML에서 바인딩

```xml
<StackPanel Margin="20" DataContext="{Binding}">
    <!-- Name 속성에 양방향 바인딩 -->
    <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />

    <!-- Email 속성에 양방향 바인딩 -->
    <TextBox Text="{Binding Email, UpdateSourceTrigger=PropertyChanged}" />

    <!-- Age 속성에 양방향 바인딩 -->
    <TextBox Text="{Binding Age, UpdateSourceTrigger=PropertyChanged}" />

    <!-- IsActive 속성에 양방향 바인딩 -->
    <CheckBox Content="활성" IsChecked="{Binding IsActive}" />

    <!-- DisplayText (읽기 전용) -->
    <TextBlock Text="{Binding DisplayText}" FontSize="18" FontWeight="Bold" />
</StackPanel>
```

---

## 5. DataContext란?

### DataContext = 바인딩의 출발점

**DataContext**는 "이 UI 요소의 바인딩이 **어떤 데이터 객체를 참조**하는가"를 결정합니다.
`{Binding Name}`이라고 쓰면, WPF는 해당 요소의 **DataContext에서 Name 속성을 찾습니다.**

```csharp
// MainWindow.xaml.cs
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();

        // DataContext 설정 = "이 창의 모든 바인딩은 이 객체를 참조해!"
        DataContext = new UserViewModel
        {
            Name = "홍길동",
            Email = "hong@example.com",
            Age = 25
        };
    }
}
```

```xml
<!-- MainWindow.xaml -->
<!-- DataContext가 UserViewModel이므로, {Binding Name}은 UserViewModel.Name을 찾음 -->
<TextBox Text="{Binding Name}" />
```

```
DataContext 설정:
Window.DataContext = new UserViewModel { Name = "홍길동" }

{Binding Name}의 해석:
1. "이 컨트롤의 DataContext에서 Name 속성을 찾아!"
2. DataContext = UserViewModel 인스턴스
3. UserViewModel.Name = "홍길동"
4. → TextBox.Text = "홍길동"
```

### DataContext의 상속 구조 (부모 → 자식)

DataContext는 **자식 요소에게 자동으로 상속**됩니다.
Window에 DataContext를 설정하면, Window 안의 모든 자식 요소가 같은 DataContext를 공유합니다.

```xml
<!-- Window의 DataContext를 설정하면... -->
<Window DataContext="{...}">  <!-- DataContext = UserViewModel -->

    <Grid>  <!-- DataContext = UserViewModel (Window에서 상속) -->

        <StackPanel>  <!-- DataContext = UserViewModel (Grid에서 상속) -->

            <!-- 모든 자식이 같은 DataContext를 사용! -->
            <TextBox Text="{Binding Name}" />   <!-- UserViewModel.Name -->
            <TextBox Text="{Binding Email}" />  <!-- UserViewModel.Email -->

        </StackPanel>
    </Grid>
</Window>
```

```
DataContext 상속 흐름:

Window (DataContext = UserViewModel)
  │
  └─ Grid (DataContext = UserViewModel ← 부모에서 상속)
       │
       └─ StackPanel (DataContext = UserViewModel ← 부모에서 상속)
            │
            ├─ TextBox (DataContext = UserViewModel ← 부모에서 상속)
            │         {Binding Name} → UserViewModel.Name ✓
            │
            └─ TextBox (DataContext = UserViewModel ← 부모에서 상속)
                      {Binding Email} → UserViewModel.Email ✓
```

### DataContext 재설정 (특정 자식에서 변경)

자식 요소에서 DataContext를 다시 설정하면, **그 하위 요소**는 새로운 DataContext를 사용합니다.

```xml
<Window DataContext="{Binding MainViewModel}">

    <!-- 이 영역은 MainViewModel의 속성에 바인딩 -->
    <TextBlock Text="{Binding Title}" />

    <!-- 이 Border 이하에서는 DataContext가 바뀜! -->
    <Border DataContext="{Binding SelectedUser}">
        <!-- 이제 SelectedUser 객체의 속성에 바인딩 -->
        <StackPanel>
            <TextBox Text="{Binding Name}" />   <!-- SelectedUser.Name -->
            <TextBox Text="{Binding Email}" />  <!-- SelectedUser.Email -->
        </StackPanel>
    </Border>

</Window>
```

---

## 6. 바인딩 문법

### 기본 문법: {Binding PropertyName}

```xml
<!-- 가장 기본적인 바인딩 -->
<TextBlock Text="{Binding Name}" />
<!-- DataContext.Name의 값을 표시 -->

<!-- Path= 을 명시적으로 쓸 수도 있음 (동일한 의미) -->
<TextBlock Text="{Binding Path=Name}" />
```

### 중첩 속성 접근

```xml
<!-- 점(.)으로 중첩 속성에 접근 -->
<TextBlock Text="{Binding Address.City}" />
<!-- DataContext.Address 객체의 City 속성 -->

<TextBlock Text="{Binding SelectedUser.Name}" />
<!-- DataContext.SelectedUser 객체의 Name 속성 -->
```

### StringFormat — 문자열 형식 지정

```xml
<!-- 숫자 형식 -->
<TextBlock Text="{Binding Price, StringFormat={}{0:N0}원}" />
<!-- 결과: "1,234,567원" -->

<TextBlock Text="{Binding Rate, StringFormat={}{0:P1}}" />
<!-- 결과: "85.5%" -->

<!-- 날짜 형식 -->
<TextBlock Text="{Binding BirthDate, StringFormat={}{0:yyyy년 MM월 dd일}}" />
<!-- 결과: "1990년 03월 15일" -->

<!-- 복합 형식 -->
<TextBlock Text="{Binding Name, StringFormat=이름: {0}}" />
<!-- 결과: "이름: 홍길동" -->

<!--
    주의: StringFormat에서 { } 를 사용할 때,
    맨 앞에 {}를 붙여야 XAML 파서가 마크업 확장으로 오해하지 않음
    StringFormat={}{0:N0}  ← 맨 앞의 {}는 이스케이프 용도
-->
```

### FallbackValue, TargetNullValue — 기본값 처리

```xml
<!-- 바인딩이 실패했을 때 표시할 기본값 -->
<TextBlock Text="{Binding Name, FallbackValue=이름 없음}" />
<!-- Name 속성을 찾지 못하면 "이름 없음" 표시 -->

<!-- 바인딩 값이 null일 때 표시할 값 -->
<TextBlock Text="{Binding Name, TargetNullValue=미입력}" />
<!-- Name이 null이면 "미입력" 표시 -->
```

### UpdateSourceTrigger — 소스 갱신 시점

```xml
<!-- 기본값: LostFocus (포커스를 잃을 때 소스 갱신) -->
<TextBox Text="{Binding Name}" />
<!-- 텍스트를 입력해도 포커스를 옮기기 전까지는 ViewModel.Name이 갱신 안 됨 -->

<!-- PropertyChanged: 키를 입력할 때마다 즉시 소스 갱신 -->
<TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />
<!-- 키를 누를 때마다 ViewModel.Name이 즉시 갱신됨 (실시간 검색 등에 필요) -->

<!-- Explicit: 명시적으로 갱신을 호출할 때만 소스 갱신 -->
<TextBox x:Name="txtName" Text="{Binding Name, UpdateSourceTrigger=Explicit}" />
<!-- 코드에서 수동으로: txtName.GetBindingExpression(TextBox.TextProperty).UpdateSource(); -->
```

```
UpdateSourceTrigger 비교:

LostFocus (기본값):
  사용자 입력: [홍 → 홍길 → 홍길동] → Tab 키 → ViewModel.Name = "홍길동"
  포커스를 옮겨야만 갱신됨

PropertyChanged:
  사용자 입력: [홍] → ViewModel.Name = "홍"
              [홍길] → ViewModel.Name = "홍길"
              [홍길동] → ViewModel.Name = "홍길동"
  글자 하나하나 입력할 때마다 즉시 갱신

Explicit:
  사용자 입력: [홍길동] → Tab → (갱신 안 됨)
  코드에서 UpdateSource() 호출해야만 갱신됨
```

### ElementName — 다른 컨트롤의 속성에 바인딩

```xml
<!-- Slider의 값을 TextBlock에 표시 (DataContext 없이!) -->
<Slider x:Name="slider" Minimum="0" Maximum="100" />
<TextBlock Text="{Binding ElementName=slider, Path=Value,
                          StringFormat=값: {0:F0}}" />
<!--
    DataContext가 아닌, 같은 XAML의 다른 컨트롤을 소스로 사용
    slider 컨트롤의 Value 속성에 바인딩
-->

<!-- CheckBox로 패널 표시/숨김 -->
<CheckBox x:Name="chkShowDetails" Content="상세 정보 표시" />
<StackPanel Visibility="{Binding ElementName=chkShowDetails,
                                  Path=IsChecked,
                                  Converter={StaticResource BoolToVisConverter}}">
    <TextBlock Text="상세 정보가 여기에 표시됩니다" />
</StackPanel>
```

---

## 7. 바인딩 디버깅 방법

바인딩이 동작하지 않을 때 **원인을 찾는 방법**입니다.

### 방법 1: Visual Studio Output 창 확인

WPF는 바인딩 실패 시 **Output(출력) 창**에 경고 메시지를 출력합니다.

```
// Output 창에 표시되는 바인딩 오류 예시:

System.Windows.Data Error: 40 :
    BindingExpression path error:
    'Nmae' property not found on 'object' ''UserViewModel' (HashCode=12345678).
    BindingExpression:Path=Nmae;
    DataItem='UserViewModel' (HashCode=12345678);
    target element is 'TextBox' (Name='');
    target property is 'Text' (type 'String')
```

```
위 메시지 해석:
- 'Nmae' property not found → 속성 이름 오타! (Nmae → Name)
- on 'UserViewModel' → UserViewModel에서 찾으려 했음
- target is 'TextBox' → TextBox.Text에 바인딩하려 했음
```

### 방법 2: 바인딩에 디버그 추적 레벨 설정

```xml
<!-- 특정 바인딩에 대해 상세한 디버그 정보를 출력 -->
<Window xmlns:diag="clr-namespace:System.Diagnostics;assembly=WindowsBase">
    <TextBox Text="{Binding Name,
                    diag:PresentationTraceSources.TraceLevel=High}" />
</Window>
<!--
    High 레벨로 설정하면 Output 창에 바인딩의 모든 단계가 출력됨:
    - DataContext를 찾는 과정
    - 속성을 찾는 과정
    - 값 변환 과정
    - 최종 값 적용 과정
-->
```

### 방법 3: 코드에서 PresentationTraceSources 설정

```csharp
// App.xaml.cs에서 전역적으로 바인딩 추적 활성화
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // 바인딩 오류를 Error 레벨로 추적
        PresentationTraceSources.DataBindingSource.Switch.Level =
            SourceLevels.Error;

        // 더 상세한 추적이 필요하면:
        // PresentationTraceSources.DataBindingSource.Switch.Level =
        //     SourceLevels.Verbose;
    }
}
```

### 자주 발생하는 바인딩 실패 원인

| 증상 | 원인 | 해결 |
|---|---|---|
| 값이 표시 안 됨 | DataContext가 설정 안 됨 | `DataContext = new ViewModel()` 추가 |
| 값이 표시 안 됨 | 속성 이름 오타 | Output 창 확인 후 오타 수정 |
| 초기값만 보이고 갱신 안 됨 | INotifyPropertyChanged 미구현 | `OnPropertyChanged()` 호출 추가 |
| 입력해도 소스가 안 바뀜 | Mode가 OneWay | `Mode=TwoWay` 명시 |
| 포커스 이동해야 갱신 | UpdateSourceTrigger 기본값 | `UpdateSourceTrigger=PropertyChanged` |

---

## 8. 컬렉션 바인딩 (ObservableCollection)

### List vs ObservableCollection

WinForms에서 ListBox에 아이템을 추가할 때:

```csharp
// WinForms: 직접 아이템 추가/제거
listBox1.Items.Add("새 항목");
listBox1.Items.RemoveAt(0);
```

WPF에서는 **컬렉션에 아이템을 추가/제거하면 UI가 자동 갱신**됩니다.
하지만 일반 `List<T>`를 사용하면 **UI가 갱신되지 않습니다.**
`ObservableCollection<T>`를 사용해야 합니다.

```csharp
// List<T>: 아이템 추가/제거해도 UI에 반영 안 됨!
public List<string> Items { get; set; } = new List<string>();

// ObservableCollection<T>: 아이템 추가/제거하면 UI에 자동 반영!
public ObservableCollection<string> Items { get; set; } = new();
```

```
List<T>와 ObservableCollection<T>의 차이:

List<T>:
  Items.Add("새 항목")  →  리스트에 추가됨  →  UI: (아무 변화 없음)

ObservableCollection<T>:
  Items.Add("새 항목")  →  리스트에 추가됨
                        →  CollectionChanged 이벤트 발생!
                        →  WPF가 이벤트 수신
                        →  UI: [새 항목] 표시됨!
```

### 완전한 예제: 할 일 목록

```csharp
// TodoItem.cs — Model (데이터 클래스)
public class TodoItem : ViewModelBase
{
    private string _title = "";
    public string Title
    {
        get => _title;
        set => SetProperty(ref _title, value);
    }

    private bool _isDone;
    public bool IsDone
    {
        get => _isDone;
        set => SetProperty(ref _isDone, value);
    }
}
```

```csharp
// TodoViewModel.cs — ViewModel
using System.Collections.ObjectModel;

public class TodoViewModel : ViewModelBase
{
    // ObservableCollection: 아이템 추가/제거 시 UI 자동 갱신!
    public ObservableCollection<TodoItem> Todos { get; } = new();

    // 새 할 일 입력 텍스트
    private string _newTodoTitle = "";
    public string NewTodoTitle
    {
        get => _newTodoTitle;
        set => SetProperty(ref _newTodoTitle, value);
    }

    // 선택된 항목
    private TodoItem? _selectedTodo;
    public TodoItem? SelectedTodo
    {
        get => _selectedTodo;
        set => SetProperty(ref _selectedTodo, value);
    }

    // 할 일 개수 표시
    public int TodoCount => Todos.Count;

    // 추가 커맨드
    public ICommand AddCommand { get; }

    // 삭제 커맨드
    public ICommand DeleteCommand { get; }

    public TodoViewModel()
    {
        // 샘플 데이터
        Todos.Add(new TodoItem { Title = "WPF 기초 학습", IsDone = true });
        Todos.Add(new TodoItem { Title = "XAML 문법 공부", IsDone = false });
        Todos.Add(new TodoItem { Title = "MVVM 패턴 이해", IsDone = false });

        // 컬렉션 변경 시 TodoCount도 갱신
        Todos.CollectionChanged += (s, e) => OnPropertyChanged(nameof(TodoCount));

        // 추가 커맨드
        AddCommand = new RelayCommand(() =>
        {
            if (!string.IsNullOrWhiteSpace(NewTodoTitle))
            {
                Todos.Add(new TodoItem { Title = NewTodoTitle });
                NewTodoTitle = "";  // 입력 필드 초기화
            }
        });

        // 삭제 커맨드
        DeleteCommand = new RelayCommand(() =>
        {
            if (SelectedTodo != null)
            {
                Todos.Remove(SelectedTodo);
            }
        });
    }
}
```

```xml
<!-- TodoView.xaml -->
<Window x:Class="TodoApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="할 일 목록" Height="400" Width="350">
    <Grid Margin="15">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />   <!-- 제목 -->
            <RowDefinition Height="Auto" />   <!-- 입력 -->
            <RowDefinition Height="*" />      <!-- 목록 -->
            <RowDefinition Height="Auto" />   <!-- 하단 버튼 -->
        </Grid.RowDefinitions>

        <!-- 제목 -->
        <TextBlock Grid.Row="0"
                   FontSize="20" FontWeight="Bold"
                   Margin="0,0,0,10">
            <Run Text="할 일 목록 (" />
            <Run Text="{Binding TodoCount, Mode=OneWay}" />
            <Run Text="개)" />
        </TextBlock>

        <!-- 입력 영역 -->
        <Grid Grid.Row="1" Margin="0,0,0,10">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="*" />
                <ColumnDefinition Width="Auto" />
            </Grid.ColumnDefinitions>

            <TextBox Grid.Column="0"
                     Text="{Binding NewTodoTitle, UpdateSourceTrigger=PropertyChanged}"
                     Padding="5,3"
                     Margin="0,0,5,0" />
            <Button Grid.Column="1"
                    Content="추가"
                    Command="{Binding AddCommand}"
                    Width="60" />
        </Grid>

        <!-- 할 일 목록 -->
        <ListBox Grid.Row="2"
                 ItemsSource="{Binding Todos}"
                 SelectedItem="{Binding SelectedTodo}">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <StackPanel Orientation="Horizontal" Margin="5,3">
                        <CheckBox IsChecked="{Binding IsDone}"
                                  Margin="0,0,8,0"
                                  VerticalAlignment="Center" />
                        <TextBlock Text="{Binding Title}"
                                   FontSize="14"
                                   VerticalAlignment="Center" />
                    </StackPanel>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>

        <!-- 삭제 버튼 -->
        <Button Grid.Row="3"
                Content="선택 항목 삭제"
                Command="{Binding DeleteCommand}"
                Margin="0,10,0,0"
                Height="30" />
    </Grid>
</Window>
```

### ObservableCollection 주의사항

```csharp
// 주의: ObservableCollection은 아이템 추가/제거만 감지!
// 아이템 내부 속성 변경은 아이템 자체가 INotifyPropertyChanged를 구현해야 함!

// 이것은 UI에 반영됨 ✓
Todos.Add(new TodoItem { Title = "새 항목" });     // 추가
Todos.Remove(selectedItem);                         // 제거
Todos.Clear();                                      // 전체 삭제
Todos.Move(0, 2);                                   // 이동

// 이것은 아이템이 INotifyPropertyChanged를 구현해야 UI에 반영됨!
Todos[0].Title = "수정된 제목";    // TodoItem이 INotifyPropertyChanged 구현 필요
Todos[0].IsDone = true;            // TodoItem이 INotifyPropertyChanged 구현 필요

// 이것은 UI에 반영 안 됨! (전체 목록 교체)
Todos = new ObservableCollection<TodoItem>(newList);
// → 대신 Todos.Clear() 후 foreach로 Add 해야 함
// → 또는 Todos 속성 자체에 OnPropertyChanged 구현
```

---

## 9. IValueConverter (값 변환기)

### 왜 필요한가?

ViewModel의 데이터 타입과 UI에서 필요한 타입이 **다른 경우**가 많습니다.

```
문제 상황:
- ViewModel: bool IsActive (true/false)
- UI: Visibility 속성 (Visible/Hidden/Collapsed)

bool을 직접 Visibility에 바인딩할 수 없음!
→ Converter(변환기)로 bool → Visibility 변환이 필요!
```

### IValueConverter 인터페이스

```csharp
using System.Globalization;
using System.Windows;
using System.Windows.Data;

// bool → Visibility 변환기
public class BoolToVisibilityConverter : IValueConverter
{
    // 소스 → 타깃 변환 (ViewModel → UI)
    public object Convert(object value, Type targetType,
                          object parameter, CultureInfo culture)
    {
        if (value is bool boolValue)
        {
            return boolValue ? Visibility.Visible : Visibility.Collapsed;
        }
        return Visibility.Collapsed;
    }

    // 타깃 → 소스 변환 (UI → ViewModel, TwoWay 바인딩 시 필요)
    public object ConvertBack(object value, Type targetType,
                              object parameter, CultureInfo culture)
    {
        if (value is Visibility visibility)
        {
            return visibility == Visibility.Visible;
        }
        return false;
    }
}
```

### Converter 사용 방법

```xml
<Window.Resources>
    <!-- 변환기를 리소스로 등록 -->
    <local:BoolToVisibilityConverter x:Key="BoolToVisConverter" />
</Window.Resources>

<!-- Converter를 통해 bool → Visibility 변환 -->
<StackPanel Visibility="{Binding IsDetailVisible,
                                  Converter={StaticResource BoolToVisConverter}}">
    <TextBlock Text="상세 정보가 여기에 표시됩니다" />
</StackPanel>

<!-- Converter에 매개변수 전달 -->
<TextBlock Visibility="{Binding IsAdmin,
                                 Converter={StaticResource BoolToVisConverter},
                                 ConverterParameter=Invert}" />
```

### 자주 사용하는 Converter 예제들

```csharp
// 1. 반전 bool 변환기 (true → false, false → true)
public class InverseBoolConverter : IValueConverter
{
    public object Convert(object value, Type targetType,
                          object parameter, CultureInfo culture)
    {
        if (value is bool b) return !b;
        return value;
    }

    public object ConvertBack(object value, Type targetType,
                              object parameter, CultureInfo culture)
    {
        if (value is bool b) return !b;
        return value;
    }
}

// 사용: 로딩 중이면 버튼 비활성화
// <Button IsEnabled="{Binding IsLoading, Converter={StaticResource InverseBoolConverter}}" />
```

```csharp
// 2. null → bool 변환기 (null이면 false, 아니면 true)
public class NullToBoolConverter : IValueConverter
{
    public object Convert(object value, Type targetType,
                          object parameter, CultureInfo culture)
    {
        return value != null;
    }

    public object ConvertBack(object value, Type targetType,
                              object parameter, CultureInfo culture)
    {
        throw new NotImplementedException();
    }
}

// 사용: 선택된 항목이 있을 때만 삭제 버튼 활성화
// <Button IsEnabled="{Binding SelectedItem, Converter={StaticResource NullToBoolConverter}}" />
```

```csharp
// 3. 숫자 → 색상 변환기 (점수에 따라 색상 변경)
public class ScoreToColorConverter : IValueConverter
{
    public object Convert(object value, Type targetType,
                          object parameter, CultureInfo culture)
    {
        if (value is int score)
        {
            if (score >= 90) return new SolidColorBrush(Colors.Green);
            if (score >= 70) return new SolidColorBrush(Colors.Orange);
            return new SolidColorBrush(Colors.Red);
        }
        return new SolidColorBrush(Colors.Black);
    }

    public object ConvertBack(object value, Type targetType,
                              object parameter, CultureInfo culture)
    {
        throw new NotImplementedException();
    }
}

// 사용:
// <TextBlock Text="{Binding Score}"
//            Foreground="{Binding Score, Converter={StaticResource ScoreToColorConverter}}" />
```

### WPF 기본 제공 BooleanToVisibilityConverter

```xml
<!-- WPF에 기본 내장된 변환기 (직접 만들 필요 없음!) -->
<Window.Resources>
    <BooleanToVisibilityConverter x:Key="BoolToVis" />
</Window.Resources>

<StackPanel Visibility="{Binding IsVisible,
                                  Converter={StaticResource BoolToVis}}" />
<!-- true → Visible, false → Collapsed -->
```

---

## 10. 전체 흐름 정리

### 바인딩 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    WPF 데이터 바인딩 전체 흐름                │
│                                                             │
│  ┌───────────┐    바인딩     ┌──────────────┐              │
│  │           │ ←──────────→ │              │              │
│  │   View    │              │  DataContext  │              │
│  │  (XAML)   │              │ (ViewModel)   │              │
│  │           │              │              │              │
│  └───────────┘              └──────────────┘              │
│       │                           │                       │
│       │ UI에 표시                  │ 데이터 소유            │
│       │                           │                       │
│  ┌─────────────────────────────────────────────┐          │
│  │              바인딩 엔진                      │          │
│  │                                             │          │
│  │  1. View의 {Binding Name} 발견              │          │
│  │  2. DataContext에서 Name 속성 찾기           │          │
│  │  3. Name 값을 가져와서 UI에 표시             │          │
│  │  4. INotifyPropertyChanged 이벤트 구독      │          │
│  │  5. Name이 바뀌면 → UI 자동 갱신            │          │
│  │  6. (TwoWay인 경우) UI 입력 → Name 자동 갱신│          │
│  │  7. Converter가 있으면 값 변환 적용          │          │
│  └─────────────────────────────────────────────┘          │
│                                                             │
│  ┌───────────┐                                             │
│  │   Model   │ ← ViewModel이 Model의 데이터를 가공해서     │
│  │  (데이터)  │    바인딩에 적합한 형태로 노출              │
│  └───────────┘                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 단계별 동작 과정

```
[1단계: 초기화]
  Window 생성 → DataContext = new ViewModel() 설정

[2단계: 바인딩 연결]
  XAML 파서가 {Binding Name}을 발견
  → DataContext에서 Name 속성을 찾음
  → Name의 현재 값 "홍길동"을 TextBox.Text에 설정
  → INotifyPropertyChanged.PropertyChanged 이벤트에 구독

[3단계: 소스 → 타깃 (ViewModel → UI)]
  코드에서 Name = "김철수" 실행
  → setter에서 OnPropertyChanged("Name") 호출
  → PropertyChanged 이벤트 발생
  → WPF 바인딩 엔진이 이벤트 수신
  → Name의 새 값 "김철수"를 가져옴
  → Converter가 있으면 변환 적용
  → TextBox.Text = "김철수" 설정 (UI 갱신!)

[4단계: 타깃 → 소스 (UI → ViewModel, TwoWay인 경우)]
  사용자가 TextBox에 "이영희" 입력
  → UpdateSourceTrigger에 따라 갱신 시점 결정
  → Converter가 있으면 역변환 적용
  → ViewModel.Name = "이영희" 설정
  → setter에서 OnPropertyChanged("Name") 호출
  → 같은 속성에 바인딩된 다른 UI 요소도 갱신
```

### WinForms 방식과 WPF 바인딩 방식 최종 비교

```csharp
// ========== WinForms 방식 ==========
// Form1.cs에 모든 로직이 집중

public partial class Form1 : Form
{
    private User _user = new User();

    private void Form1_Load(object sender, EventArgs e)
    {
        // 데이터 → UI (수동)
        txtName.Text = _user.Name;
        txtEmail.Text = _user.Email;
        chkActive.Checked = _user.IsActive;

        // 이벤트 핸들러 등록 (수동 동기화)
        txtName.TextChanged += (s, e) => _user.Name = txtName.Text;
        txtEmail.TextChanged += (s, e) => _user.Email = txtEmail.Text;
        chkActive.CheckedChanged += (s, e) => _user.IsActive = chkActive.Checked;
    }

    private void btnSave_Click(object sender, EventArgs e)
    {
        // UI → 데이터 (수동)
        _user.Name = txtName.Text;
        _user.Email = txtEmail.Text;
        _user.IsActive = chkActive.Checked;
        SaveToDatabase(_user);
    }

    private void RefreshUI()
    {
        // 데이터가 바뀔 때마다 이 메서드를 호출해야 함 (수동)
        txtName.Text = _user.Name;
        txtEmail.Text = _user.Email;
        chkActive.Checked = _user.IsActive;
    }
}
```

```xml
<!-- ========== WPF 바인딩 방식 ========== -->
<!-- MainWindow.xaml: UI 선언만 -->
<StackPanel>
    <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />
    <TextBox Text="{Binding Email, UpdateSourceTrigger=PropertyChanged}" />
    <CheckBox IsChecked="{Binding IsActive}" Content="활성" />
    <Button Content="저장" Command="{Binding SaveCommand}" />
</StackPanel>
<!-- 끝! UI에서 할 일은 바인딩 선언뿐! -->
```

```csharp
// UserViewModel.cs: 로직만 담당 (UI 참조 없음!)
public class UserViewModel : ViewModelBase
{
    private string _name = "";
    public string Name
    {
        get => _name;
        set => SetProperty(ref _name, value);
        // 이것만으로 UI 자동 갱신!
    }

    private string _email = "";
    public string Email
    {
        get => _email;
        set => SetProperty(ref _email, value);
    }

    private bool _isActive;
    public bool IsActive
    {
        get => _isActive;
        set => SetProperty(ref _isActive, value);
    }

    public ICommand SaveCommand { get; }

    public UserViewModel()
    {
        SaveCommand = new RelayCommand(() =>
        {
            // Name, Email, IsActive는 이미 바인딩으로 최신 상태!
            var user = new User { Name = Name, Email = Email, IsActive = IsActive };
            SaveToDatabase(user);
        });
    }
}
```

---

> **핵심 요약**
>
> 1. **데이터 바인딩**은 소스(ViewModel)와 타깃(UI)을 자동으로 동기화합니다.
> 2. **INotifyPropertyChanged**를 구현해야 속성 변경이 UI에 반영됩니다.
> 3. **DataContext**는 바인딩의 출발점이며, 부모에서 자식으로 상속됩니다.
> 4. **ObservableCollection**을 사용해야 컬렉션 변경이 UI에 반영됩니다.
> 5. **IValueConverter**로 데이터 타입을 UI에 맞게 변환할 수 있습니다.
> 6. 바인딩 오류는 **Output 창**에서 확인할 수 있습니다.
> 7. WinForms의 수동 대입 코드가 WPF에서는 `{Binding}` 한 줄로 대체됩니다.
