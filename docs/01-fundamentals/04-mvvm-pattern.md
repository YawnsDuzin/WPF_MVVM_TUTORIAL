# MVVM 패턴 이해

> 이 문서는 WinForms의 이벤트 핸들러 방식에 익숙한 개발자를 대상으로,
> WPF의 핵심 디자인 패턴인 **MVVM(Model-View-ViewModel)** 을 처음부터 설명합니다.

---

## 목차

1. [디자인 패턴이란?](#1-디자인-패턴이란)
2. [MVC, MVP와 MVVM 비교](#2-mvc-mvp와-mvvm-비교)
3. [MVVM의 세 구성요소](#3-mvvm의-세-구성요소)
4. [각 구성요소의 역할과 책임](#4-각-구성요소의-역할과-책임)
5. [MVVM에서 데이터 흐름](#5-mvvm에서-데이터-흐름)
6. [ICommand 인터페이스](#6-icommand-인터페이스)
7. [왜 MVVM을 쓰는가?](#7-왜-mvvm을-쓰는가)
8. [MVVM의 단점](#8-mvvm의-단점)
9. [WinForms 개발자의 전형적인 실수](#9-winforms-개발자의-전형적인-실수)
10. [완전한 MVVM 예제](#10-완전한-mvvm-예제)

---

## 1. 디자인 패턴이란?

**디자인 패턴(Design Pattern)** 은 소프트웨어 개발에서 반복적으로 발생하는 문제를
해결하기 위한 **검증된 설계 방법**입니다.

"코드를 어떻게 **구조화(organize)** 할 것인가"에 대한 모범 답안이라고 생각하면 됩니다.

### WinForms에서 겪는 문제

```csharp
// WinForms: 모든 코드가 Form1.cs 하나에 집중 (수백~수천 줄)
public partial class Form1 : Form
{
    // UI 이벤트 핸들러
    private void btnSearch_Click(object sender, EventArgs e) { /* 검색 로직 */ }
    private void btnSave_Click(object sender, EventArgs e) { /* 저장 로직 */ }
    private void btnDelete_Click(object sender, EventArgs e) { /* 삭제 로직 */ }
    private void dgv_SelectionChanged(object sender, EventArgs e) { /* 선택 로직 */ }
    private void txtSearch_TextChanged(object sender, EventArgs e) { /* 필터 로직 */ }

    // 데이터 처리 로직
    private void LoadData() { /* DB 조회 */ }
    private void SaveData() { /* DB 저장 */ }
    private void ValidateInput() { /* 유효성 검사 */ }

    // UI 갱신 로직
    private void RefreshGrid() { /* 그리드 갱신 */ }
    private void UpdateStatusBar() { /* 상태바 갱신 */ }
    private void EnableButtons() { /* 버튼 활성화/비활성화 */ }

    // ... 이런 코드가 500줄, 1000줄, 2000줄...
    // 유지보수가 점점 어려워짐!
}
```

디자인 패턴은 이 문제를 해결합니다:
**"코드를 역할별로 나누자!"**

---

## 2. MVC, MVP와 MVVM 비교

### MVC (Model-View-Controller) — 웹에서 주로 사용

```
사용자 입력 → Controller → Model 업데이트 → View 갱신

┌───────┐     ┌────────────┐     ┌───────┐
│ View  │ ──→ │ Controller │ ──→ │ Model │
│ (UI)  │ ←── │ (입력 처리) │ ←── │(데이터)│
└───────┘     └────────────┘     └───────┘

Controller가 View와 Model을 모두 알고 있음
ASP.NET MVC, Django(Python), Spring(Java) 등에서 사용
```

### MVP (Model-View-Presenter) — WinForms에서 사용 가능

```
사용자 입력 → View → Presenter → Model 업데이트 → Presenter → View 갱신

┌───────┐     ┌───────────┐     ┌───────┐
│ View  │ ←─→ │ Presenter │ ←─→ │ Model │
│ (UI)  │     │ (중재자)   │     │(데이터)│
└───────┘     └───────────┘     └───────┘

Presenter가 View의 인터페이스를 통해 UI를 제어
View와 Presenter가 1:1 관계
```

### MVVM (Model-View-ViewModel) — WPF에서 사용

```
사용자 입력 → View ←─바인딩─→ ViewModel ←→ Model

┌───────┐   데이터   ┌───────────┐     ┌───────┐
│ View  │ ←─바인딩─→ │ ViewModel │ ←─→ │ Model │
│(XAML) │   커맨드   │ (중재자)   │     │(데이터)│
└───────┘            └───────────┘     └───────┘

View와 ViewModel은 바인딩으로만 연결 (서로 직접 참조 안 함)
ViewModel은 View의 존재를 모름! (View에 대한 의존성 없음!)
```

### WinForms 이벤트 핸들러 방식 vs MVVM

```
WinForms (이벤트 핸들러):

┌──────────────────────────────────────┐
│              Form1.cs                │
│                                      │
│  UI 코드 + 비즈니스 로직 + 데이터    │
│  (전부 섞여 있음!)                    │
│                                      │
│  btnSave_Click() {                   │
│    // 입력값 검증 (UI 로직)           │
│    // 데이터 변환 (비즈니스 로직)      │
│    // DB 저장 (데이터 액세스)          │
│    // UI 갱신 (UI 로직)               │
│    // 메시지 표시 (UI 로직)           │
│  }                                   │
└──────────────────────────────────────┘

MVVM:

┌──────────┐  ┌──────────────┐  ┌───────────┐
│  View    │  │  ViewModel   │  │  Model    │
│ (XAML)   │  │ (C# 클래스)  │  │ (C# 클래스)│
│          │  │              │  │           │
│ UI 정의  │  │ UI 로직      │  │ 데이터    │
│ 바인딩   │  │ 커맨드 처리  │  │ 비즈니스  │
│ 스타일   │  │ 데이터 가공  │  │ 로직      │
│          │  │ 상태 관리    │  │ DB 접근   │
└──────────┘  └──────────────┘  └───────────┘
  각각 독립!     View 모름!       UI 모름!
```

---

## 3. MVVM의 세 구성요소

### Model — 데이터와 비즈니스 로직

**WinForms에서의 데이터 클래스와 완전히 동일합니다!**
Model은 UI에 대해 전혀 모릅니다. 순수한 데이터와 비즈니스 로직만 담당합니다.

```csharp
// Model: WinForms에서 사용하던 데이터 클래스와 동일!
// UI에 대한 참조가 전혀 없음

public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public string Department { get; set; } = "";
    public decimal Salary { get; set; }
    public DateTime HireDate { get; set; }
    public bool IsActive { get; set; }

    // 비즈니스 로직 (UI와 무관한 순수 로직)
    public decimal CalculateBonus()
    {
        // 근속 연수에 따른 보너스 계산
        int yearsOfService = DateTime.Now.Year - HireDate.Year;
        return Salary * 0.1m * yearsOfService;
    }

    public bool IsEligibleForPromotion()
    {
        int yearsOfService = DateTime.Now.Year - HireDate.Year;
        return yearsOfService >= 3 && IsActive;
    }
}

// 데이터 접근 서비스 (DB와의 통신)
public class EmployeeService
{
    public List<Employee> GetAll() { /* DB에서 전체 조회 */ return new(); }
    public Employee? GetById(int id) { /* DB에서 단건 조회 */ return null; }
    public void Save(Employee employee) { /* DB에 저장 */ }
    public void Delete(int id) { /* DB에서 삭제 */ }
}
```

### View — XAML로 정의된 UI

**WinForms의 Form에 해당합니다.**
View는 UI의 시각적 구조만 정의하고, **로직은 포함하지 않습니다.**
(코드 비하인드에는 InitializeComponent()와 DataContext 설정 정도만 있어야 합니다.)

```xml
<!-- EmployeeView.xaml — View: UI 정의만! 로직 없음! -->
<Window x:Class="MyApp.Views.EmployeeView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="직원 관리" Height="500" Width="700">
    <Grid Margin="15">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
            <RowDefinition Height="Auto" />
        </Grid.RowDefinitions>

        <!-- 검색 영역 -->
        <StackPanel Grid.Row="0" Orientation="Horizontal" Margin="0,0,0,10">
            <TextBox Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
                     Width="200" Margin="0,0,10,0" />
            <Button Content="검색" Command="{Binding SearchCommand}" />
        </StackPanel>

        <!-- 직원 목록 -->
        <DataGrid Grid.Row="1"
                  ItemsSource="{Binding Employees}"
                  SelectedItem="{Binding SelectedEmployee}"
                  AutoGenerateColumns="False"
                  IsReadOnly="True">
            <DataGrid.Columns>
                <DataGridTextColumn Header="이름" Binding="{Binding Name}" Width="120" />
                <DataGridTextColumn Header="부서" Binding="{Binding Department}" Width="100" />
                <DataGridTextColumn Header="이메일" Binding="{Binding Email}" Width="200" />
            </DataGrid.Columns>
        </DataGrid>

        <!-- 하단 버튼 -->
        <StackPanel Grid.Row="2" Orientation="Horizontal"
                    HorizontalAlignment="Right" Margin="0,10,0,0">
            <Button Content="추가" Command="{Binding AddCommand}" Margin="0,0,5,0" />
            <Button Content="수정" Command="{Binding EditCommand}" Margin="0,0,5,0" />
            <Button Content="삭제" Command="{Binding DeleteCommand}" />
        </StackPanel>
    </Grid>
</Window>
```

```csharp
// EmployeeView.xaml.cs — 코드 비하인드는 최소한!
public partial class EmployeeView : Window
{
    public EmployeeView()
    {
        InitializeComponent();
        DataContext = new EmployeeViewModel(new EmployeeService());
        // 이게 코드 비하인드의 전부! 다른 로직 없음!
    }
}
```

### ViewModel — View와 Model 사이의 중재자

**WinForms에 없던 완전히 새로운 개념입니다!**

ViewModel은:
- Model의 데이터를 **View가 표시하기 좋은 형태로 가공**합니다.
- 사용자의 **액션(버튼 클릭 등)을 처리**합니다.
- View의 **상태(선택된 항목, 입력 값 등)를 관리**합니다.

**핵심: ViewModel은 View의 존재를 모릅니다!**
ViewModel에는 `using System.Windows;` 같은 UI 관련 참조가 없습니다.

```csharp
using System.Collections.ObjectModel;
using System.ComponentModel;
using System.Runtime.CompilerServices;
using System.Windows.Input;

// EmployeeViewModel.cs — View와 Model 사이의 중재자
public class EmployeeViewModel : ViewModelBase
{
    private readonly EmployeeService _employeeService;

    // ===== View에 노출할 데이터 (바인딩 속성) =====

    // 직원 목록 (ObservableCollection이므로 추가/제거 시 UI 자동 갱신)
    public ObservableCollection<Employee> Employees { get; } = new();

    // 선택된 직원
    private Employee? _selectedEmployee;
    public Employee? SelectedEmployee
    {
        get => _selectedEmployee;
        set
        {
            if (SetProperty(ref _selectedEmployee, value))
            {
                // 선택이 바뀌면 수정/삭제 버튼 활성화 상태도 바뀌어야 함
                (EditCommand as RelayCommand)?.RaiseCanExecuteChanged();
                (DeleteCommand as RelayCommand)?.RaiseCanExecuteChanged();
            }
        }
    }

    // 검색어
    private string _searchText = "";
    public string SearchText
    {
        get => _searchText;
        set => SetProperty(ref _searchText, value);
    }

    // ===== 커맨드 (사용자 액션 처리) =====

    public ICommand SearchCommand { get; }
    public ICommand AddCommand { get; }
    public ICommand EditCommand { get; }
    public ICommand DeleteCommand { get; }

    // ===== 생성자 =====

    public EmployeeViewModel(EmployeeService employeeService)
    {
        _employeeService = employeeService;

        // 커맨드 초기화
        SearchCommand = new RelayCommand(ExecuteSearch);
        AddCommand = new RelayCommand(ExecuteAdd);
        EditCommand = new RelayCommand(
            ExecuteEdit,
            () => SelectedEmployee != null  // 선택된 항목이 있을 때만 활성화
        );
        DeleteCommand = new RelayCommand(
            ExecuteDelete,
            () => SelectedEmployee != null  // 선택된 항목이 있을 때만 활성화
        );

        // 초기 데이터 로드
        LoadEmployees();
    }

    // ===== 비공개 메서드 (로직 구현) =====

    private void LoadEmployees()
    {
        Employees.Clear();
        foreach (var emp in _employeeService.GetAll())
        {
            Employees.Add(emp);
        }
    }

    private void ExecuteSearch()
    {
        Employees.Clear();
        var results = _employeeService.GetAll()
            .Where(e => e.Name.Contains(SearchText, StringComparison.OrdinalIgnoreCase))
            .ToList();

        foreach (var emp in results)
        {
            Employees.Add(emp);
        }
    }

    private void ExecuteAdd()
    {
        // 새 직원 추가 로직
        // (실제로는 다이얼로그를 열거나 입력 모드로 전환)
    }

    private void ExecuteEdit()
    {
        if (SelectedEmployee == null) return;
        // 선택된 직원 수정 로직
    }

    private void ExecuteDelete()
    {
        if (SelectedEmployee == null) return;
        _employeeService.Delete(SelectedEmployee.Id);
        Employees.Remove(SelectedEmployee);
    }
}
```

---

## 4. 각 구성요소의 역할과 책임

### 역할 분담 상세 표

| 역할 | Model | ViewModel | View |
|---|---|---|---|
| **데이터 정의** | O (속성, 관계) | | |
| **비즈니스 로직** | O (계산, 규칙) | | |
| **데이터 저장/조회** | O (DB, API) | | |
| **데이터 가공/변환** | | O (표시 형식 변환) | |
| **UI 상태 관리** | | O (선택, 입력값) | |
| **커맨드 처리** | | O (버튼 액션) | |
| **유효성 검사** | | O (입력값 검증) | |
| **UI 구조 정의** | | | O (XAML 레이아웃) |
| **바인딩 선언** | | | O ({Binding}) |
| **스타일/애니메이션** | | | O (Style, Trigger) |

### 각 구성요소의 "하면 안 되는 것"

```
Model:
  ✗ UI 요소 참조 금지 (using System.Windows 금지)
  ✗ ViewModel이나 View에 대한 참조 금지
  ✗ UI 관련 타입 사용 금지 (Visibility, Brush 등)

ViewModel:
  ✗ View 직접 참조 금지 (Window, TextBox 등 참조 금지)
  ✗ using System.Windows 금지 (System.Windows.Input의 ICommand는 예외)
  ✗ MessageBox.Show() 같은 UI 직접 호출 금지
  ✗ XAML 코드 조작 금지

View (XAML + 코드 비하인드):
  ✗ 비즈니스 로직 금지
  ✗ 데이터 접근 금지 (DB 호출 등)
  ✗ 복잡한 상태 관리 금지
  ✗ 코드 비하인드에서 데이터 처리 금지
```

### Python 개발자를 위한 비유

```python
# Python에서의 역할 분담 비유

# Model (models.py): 데이터 구조와 비즈니스 로직
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

    def calculate_bonus(self):
        return self.salary * 0.1

# ViewModel (viewmodel.py): UI 로직과 상태 관리
class EmployeeViewModel:
    def __init__(self):
        self.employees = []
        self.selected = None
        self.search_text = ""

    def search(self):
        self.employees = [e for e in all_employees
                         if self.search_text in e.name]

# View (XAML/UI): 화면 표시만 담당
# <TextBox Text="{Binding search_text}" />
# <ListBox ItemsSource="{Binding employees}" />
```

---

## 5. MVVM에서 데이터 흐름

### 전체 데이터 흐름도

```
┌──────────────────────────────────────────────────────────────────┐
│                    MVVM 데이터 흐름                               │
│                                                                  │
│                                                                  │
│  ┌─────────┐      데이터 바인딩      ┌──────────────┐           │
│  │         │  ←─────────────────── │              │           │
│  │  View   │   (자동 UI 갱신)       │  ViewModel   │           │
│  │ (XAML)  │                       │  (C# 클래스)  │           │
│  │         │  ───────────────────→ │              │           │
│  │         │   (사용자 입력/커맨드)  │              │           │
│  └─────────┘                       └──────────────┘           │
│       │                                   │                    │
│       │ 사용자가 보는 화면                  │ 데이터 요청/저장    │
│       │                                   │                    │
│       │                            ┌──────────────┐           │
│       │                            │    Model     │           │
│       │                            │   (데이터)    │           │
│       │                            │   Service    │           │
│       │                            └──────────────┘           │
│       │                                   │                    │
│       │                            ┌──────────────┐           │
│       │                            │  Database /  │           │
│       │                            │    API       │           │
│       │                            └──────────────┘           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

데이터 흐름 순서 (사용자가 "검색" 버튼 클릭):

  1. 사용자가 검색어 입력 → TextBox → (TwoWay 바인딩) → ViewModel.SearchText 갱신
  2. 사용자가 "검색" 클릭 → Button.Command → ViewModel.SearchCommand 실행
  3. ViewModel이 Model/Service에 검색 요청
  4. Service가 DB에서 결과 조회 → ViewModel로 반환
  5. ViewModel이 Employees 컬렉션 업데이트
  6. ObservableCollection 변경 알림 → DataGrid 자동 갱신
  7. 사용자가 갱신된 목록을 확인
```

### View → ViewModel 통신 방법

```
1. 데이터 바인딩 (속성 동기화)
   View의 TextBox.Text ←→ ViewModel의 SearchText 속성

2. Command (액션 전달)
   View의 Button.Command → ViewModel의 SearchCommand 실행

3. 이벤트 → Command 변환 (Behavior)
   View의 ListBox.SelectionChanged → ViewModel의 Command 실행
   (System.Windows.Interactivity 또는 Microsoft.Xaml.Behaviors 사용)
```

### ViewModel → View 통신 방법

```
1. INotifyPropertyChanged (속성 변경 알림)
   ViewModel: Name = "새 값" → PropertyChanged 이벤트
   → View: TextBox.Text 자동 갱신

2. ObservableCollection (컬렉션 변경 알림)
   ViewModel: Employees.Add(newEmployee) → CollectionChanged 이벤트
   → View: DataGrid에 새 행 자동 추가

3. ViewModel은 View를 직접 제어하지 않음!
   (MessageBox 표시 등은 서비스/인터페이스를 통해 간접적으로)
```

---

## 6. ICommand 인터페이스

### WinForms의 이벤트 핸들러 방식

```csharp
// WinForms: 버튼 클릭 = 이벤트 핸들러
// Form1.Designer.cs
this.btnSave.Click += new EventHandler(this.btnSave_Click);

// Form1.cs
private void btnSave_Click(object sender, EventArgs e)
{
    // 저장 로직을 여기에 직접 작성
    if (string.IsNullOrEmpty(txtName.Text))
    {
        MessageBox.Show("이름을 입력하세요.");
        return;
    }
    SaveToDatabase(txtName.Text, txtEmail.Text);
    MessageBox.Show("저장되었습니다.");
    RefreshGrid();
}
```

### WPF MVVM의 Command 바인딩 방식

```xml
<!-- View: 버튼에 Command를 바인딩 -->
<Button Content="저장" Command="{Binding SaveCommand}" />
<!--
    Click 이벤트 핸들러 대신 Command 바인딩!
    버튼이 클릭되면 ViewModel의 SaveCommand가 실행됨
-->
```

```csharp
// ViewModel: Command를 정의
public class MainViewModel : ViewModelBase
{
    public ICommand SaveCommand { get; }

    public MainViewModel()
    {
        SaveCommand = new RelayCommand(
            execute: () =>
            {
                // 저장 로직 (UI 참조 없음!)
                var employee = new Employee { Name = Name, Email = Email };
                _employeeService.Save(employee);
                LoadEmployees(); // 목록 갱신
            },
            canExecute: () =>
            {
                // 이 조건이 false이면 버튼이 자동으로 비활성화!
                return !string.IsNullOrEmpty(Name);
            }
        );
    }
}
```

### ICommand 인터페이스 설명

```csharp
// ICommand는 .NET에 내장된 인터페이스
// System.Windows.Input 네임스페이스에 있음
public interface ICommand
{
    // 커맨드를 실행할 수 있는지 판단
    // 반환값이 false이면 버튼이 자동으로 비활성화(회색)됨!
    bool CanExecute(object? parameter);

    // 커맨드 실행 (버튼 클릭 시 호출됨)
    void Execute(object? parameter);

    // CanExecute의 결과가 바뀌었을 때 UI에 알림
    event EventHandler? CanExecuteChanged;
}
```

```
ICommand 동작 흐름:

1. XAML에서 {Binding SaveCommand}
2. WPF가 SaveCommand의 CanExecute() 호출
3. CanExecute() → true면 버튼 활성화, false면 비활성화
4. 사용자가 버튼 클릭
5. WPF가 SaveCommand의 Execute() 호출
6. Execute()에서 로직 실행
7. 상태가 바뀌면 CanExecuteChanged 이벤트 발생
8. WPF가 다시 CanExecute() 호출 → 버튼 상태 갱신
```

### RelayCommand 직접 구현

WPF에는 ICommand의 기본 구현체가 없으므로, **RelayCommand를 직접 만들어야** 합니다.
(나중에 CommunityToolkit.Mvvm을 사용하면 직접 만들 필요 없습니다.)

```csharp
using System.Windows.Input;

/// <summary>
/// ICommand의 범용 구현체
/// 모든 ViewModel에서 재사용 가능
/// </summary>
public class RelayCommand : ICommand
{
    private readonly Action _execute;           // 실행할 동작
    private readonly Func<bool>? _canExecute;   // 실행 가능 여부 판단

    /// <summary>
    /// RelayCommand 생성자
    /// </summary>
    /// <param name="execute">커맨드 실행 시 호출할 메서드</param>
    /// <param name="canExecute">실행 가능 여부를 판단하는 메서드 (null이면 항상 실행 가능)</param>
    public RelayCommand(Action execute, Func<bool>? canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    // 실행 가능 여부 판단
    public bool CanExecute(object? parameter)
    {
        return _canExecute == null || _canExecute();
    }

    // 커맨드 실행
    public void Execute(object? parameter)
    {
        _execute();
    }

    // CanExecuteChanged 이벤트
    // CommandManager.RequerySuggested: WPF가 자동으로 CanExecute를 재평가하는 시점
    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }

    // 수동으로 CanExecute 재평가를 요청할 때 사용
    public void RaiseCanExecuteChanged()
    {
        CommandManager.InvalidateRequerySuggested();
    }
}
```

### 매개변수가 있는 RelayCommand

```csharp
/// <summary>
/// 매개변수를 받는 RelayCommand
/// 예: 리스트에서 항목을 클릭했을 때 해당 항목을 매개변수로 전달
/// </summary>
public class RelayCommand<T> : ICommand
{
    private readonly Action<T?> _execute;
    private readonly Predicate<T?>? _canExecute;

    public RelayCommand(Action<T?> execute, Predicate<T?>? canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    public bool CanExecute(object? parameter)
    {
        return _canExecute == null || _canExecute((T?)parameter);
    }

    public void Execute(object? parameter)
    {
        _execute((T?)parameter);
    }

    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
}
```

```xml
<!-- 매개변수가 있는 Command 사용 예시 -->
<Button Content="삭제"
        Command="{Binding DeleteCommand}"
        CommandParameter="{Binding SelectedEmployee}" />
<!--
    CommandParameter: Execute()에 전달될 매개변수
    SelectedEmployee가 매개변수로 전달됨
-->

<!-- ListBox에서 더블클릭 시 Command 실행 (InputBinding 사용) -->
<ListBox ItemsSource="{Binding Items}">
    <ListBox.InputBindings>
        <MouseBinding MouseAction="LeftDoubleClick"
                      Command="{Binding OpenDetailCommand}"
                      CommandParameter="{Binding SelectedItem,
                                         RelativeSource={RelativeSource AncestorType=ListBox}}" />
    </ListBox.InputBindings>
</ListBox>
```

### 비동기 Command (async/await)

```csharp
// 시간이 오래 걸리는 작업은 비동기로 처리해야 UI가 멈추지 않음
public class AsyncRelayCommand : ICommand
{
    private readonly Func<Task> _execute;
    private readonly Func<bool>? _canExecute;
    private bool _isExecuting;

    public AsyncRelayCommand(Func<Task> execute, Func<bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public bool CanExecute(object? parameter)
    {
        return !_isExecuting && (_canExecute == null || _canExecute());
    }

    public async void Execute(object? parameter)
    {
        if (CanExecute(parameter))
        {
            try
            {
                _isExecuting = true;
                CommandManager.InvalidateRequerySuggested();
                await _execute();
            }
            finally
            {
                _isExecuting = false;
                CommandManager.InvalidateRequerySuggested();
            }
        }
    }

    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
}

// ViewModel에서 사용
public class MainViewModel : ViewModelBase
{
    private bool _isLoading;
    public bool IsLoading
    {
        get => _isLoading;
        set => SetProperty(ref _isLoading, value);
    }

    public ICommand LoadDataCommand { get; }

    public MainViewModel()
    {
        LoadDataCommand = new AsyncRelayCommand(async () =>
        {
            IsLoading = true;  // 로딩 표시
            try
            {
                // 시간이 걸리는 작업 (DB 조회, API 호출 등)
                var data = await _service.GetAllAsync();
                Employees.Clear();
                foreach (var emp in data)
                {
                    Employees.Add(emp);
                }
            }
            finally
            {
                IsLoading = false;  // 로딩 종료
            }
        });
    }
}
```

---

## 7. 왜 MVVM을 쓰는가?

### 7.1 테스트 용이성

MVVM의 **가장 큰 장점**입니다. ViewModel에는 UI 참조가 없으므로,
**Window 객체 없이도 단위 테스트**를 작성할 수 있습니다.

```csharp
// WinForms: 테스트하기 매우 어려움
// Form1의 btnSave_Click을 테스트하려면 Form1을 생성해야 함
// → UI 스레드 필요, 시간 오래 걸림, 불안정

// MVVM: ViewModel은 일반 C# 클래스이므로 쉽게 테스트 가능!
[TestClass]
public class EmployeeViewModelTests
{
    [TestMethod]
    public void Search_WithMatchingName_ReturnsFilteredResults()
    {
        // Arrange (준비)
        var service = new MockEmployeeService();  // 가짜 서비스
        service.Employees.Add(new Employee { Name = "홍길동" });
        service.Employees.Add(new Employee { Name = "김철수" });
        service.Employees.Add(new Employee { Name = "홍길순" });

        var viewModel = new EmployeeViewModel(service);

        // Act (실행)
        viewModel.SearchText = "홍";
        viewModel.SearchCommand.Execute(null);

        // Assert (검증)
        Assert.AreEqual(2, viewModel.Employees.Count);
        // "홍길동"과 "홍길순"만 결과에 포함

        // Window나 UI 없이도 완벽하게 테스트 가능!
    }

    [TestMethod]
    public void DeleteCommand_CanExecute_IsFalseWhenNothingSelected()
    {
        var service = new MockEmployeeService();
        var viewModel = new EmployeeViewModel(service);

        // 아무것도 선택 안 한 상태
        viewModel.SelectedEmployee = null;

        // DeleteCommand가 실행 불가능한지 확인
        Assert.IsFalse(viewModel.DeleteCommand.CanExecute(null));
    }
}
```

### 7.2 관심사 분리 (Separation of Concerns)

```
WinForms (관심사가 섞임):
┌──────────────────────────────────────┐
│         Form1.cs (500줄)              │
│                                      │
│  UI 로직 + 비즈니스 로직 + 데이터     │
│  → 수정할 때 어디를 고쳐야 할지 모름   │
│  → 하나 고치면 다른 게 깨짐           │
└──────────────────────────────────────┘

MVVM (관심사가 분리됨):
┌──────────┐  ┌──────────────┐  ┌───────────┐
│ View     │  │ ViewModel    │  │ Model     │
│ 50줄     │  │ 150줄        │  │ 100줄     │
│          │  │              │  │           │
│ UI만!    │  │ UI 로직만!   │  │ 데이터만!  │
│          │  │              │  │           │
│ 디자이너가│  │ C# 개발자가  │  │ 백엔드     │
│ 수정 가능 │  │ 수정 가능    │  │ 개발자가   │
│          │  │              │  │ 수정 가능  │
└──────────┘  └──────────────┘  └───────────┘
```

### 7.3 재사용성

```csharp
// ViewModel은 UI에 독립적이므로 다양한 View에서 재사용 가능!

// 같은 EmployeeViewModel을 다른 View에서 사용:
// - EmployeeListView.xaml (목록 형태)
// - EmployeeCardView.xaml (카드 형태)
// - EmployeeDashboardView.xaml (대시보드 형태)
// 모두 같은 EmployeeViewModel을 DataContext로 사용!

// 심지어 WPF가 아닌 다른 플랫폼에서도 재사용 가능:
// - WPF 데스크톱 앱
// - Xamarin/MAUI 모바일 앱
// - Blazor 웹 앱 (일부 조정 필요)
```

### 7.4 디자이너-개발자 협업

```
MVVM에서는 View(XAML)와 ViewModel(C#)이 분리되어 있으므로:

디자이너: XAML만 수정 (레이아웃, 색상, 애니메이션)
개발자:   ViewModel만 수정 (로직, 데이터 처리)

→ 서로의 코드를 건드리지 않고 동시에 작업 가능!
→ Git 머지 충돌 감소!
```

---

## 8. MVVM의 단점

### 8.1 학습 곡선

```
WinForms:
  버튼 더블클릭 → 이벤트 핸들러 자동 생성 → 코드 작성 → 끝!
  (30초 만에 기능 구현)

MVVM:
  1. ViewModelBase 클래스 만들기
  2. INotifyPropertyChanged 구현하기
  3. RelayCommand 클래스 만들기
  4. ViewModel 클래스 만들기
  5. 속성마다 private 필드 + public 속성 + OnPropertyChanged 작성
  6. Command 속성 만들기
  7. DataContext 설정하기
  8. XAML에서 바인딩 작성하기
  (처음에는 30분 걸릴 수 있음!)
```

### 8.2 보일러플레이트 코드

MVVM의 가장 큰 불만: **반복적인 코드가 너무 많다!**

```csharp
// 속성 하나 추가하는데 이만큼의 코드가 필요:
private string _name = "";
public string Name
{
    get => _name;
    set
    {
        if (_name != value)
        {
            _name = value;
            OnPropertyChanged();
        }
    }
}
// 속성 10개면 이런 코드가 10번 반복!
```

### CommunityToolkit.Mvvm으로 해결!

**CommunityToolkit.Mvvm** (구 Microsoft.Toolkit.Mvvm)은
Microsoft가 제공하는 공식 MVVM 라이브러리로,
**소스 생성기(Source Generator)** 를 통해 보일러플레이트 코드를 자동으로 생성합니다.

```bash
# NuGet 패키지 설치
dotnet add package CommunityToolkit.Mvvm
```

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

// Before: CommunityToolkit 없이 (보일러플레이트 많음)
public class EmployeeViewModel : ViewModelBase
{
    private string _name = "";
    public string Name
    {
        get => _name;
        set => SetProperty(ref _name, value);
    }

    private string _email = "";
    public string Email
    {
        get => _email;
        set => SetProperty(ref _email, value);
    }

    private int _age;
    public int Age
    {
        get => _age;
        set => SetProperty(ref _age, value);
    }

    public ICommand SaveCommand { get; }

    public EmployeeViewModel()
    {
        SaveCommand = new RelayCommand(() =>
        {
            // 저장 로직
        });
    }
}
```

```csharp
// After: CommunityToolkit 사용 (깔끔!)
// partial 키워드 필수 (소스 생성기가 나머지 코드를 자동 생성)
public partial class EmployeeViewModel : ObservableObject
{
    // [ObservableProperty] 어트리뷰트 하나면 끝!
    // 자동으로 public 속성 Name, Email, Age가 생성됨
    // INotifyPropertyChanged 구현도 자동!
    [ObservableProperty]
    private string _name = "";

    [ObservableProperty]
    private string _email = "";

    [ObservableProperty]
    private int _age;

    // [RelayCommand] 어트리뷰트로 Command 자동 생성!
    // SaveCommand 속성이 자동으로 만들어짐
    [RelayCommand]
    private void Save()
    {
        // 저장 로직
    }

    // 비동기 Command도 지원!
    // LoadDataCommand 속성이 자동으로 만들어짐
    [RelayCommand]
    private async Task LoadData()
    {
        // 비동기 로직
    }

    // CanExecute 조건도 지정 가능!
    // DeleteCommand 속성이 자동으로 만들어지고,
    // SelectedEmployee가 null이면 버튼 자동 비활성화
    [RelayCommand(CanExecute = nameof(CanDelete))]
    private void Delete()
    {
        // 삭제 로직
    }

    private bool CanDelete() => SelectedEmployee != null;
}
```

```
코드 줄 수 비교:
  직접 구현:        약 50줄
  CommunityToolkit: 약 20줄

→ 약 60% 코드 감소!
```

### 8.3 단순한 앱에는 과도할 수 있음

```
앱 복잡도에 따른 권장 패턴:

간단한 도구 (계산기, 메모장 수준):
  → 코드 비하인드로도 충분
  → MVVM은 과도한 설계(over-engineering)일 수 있음

중간 규모 앱 (CRUD 앱, 설정 도구):
  → MVVM 도입 권장
  → CommunityToolkit.Mvvm 사용으로 보일러플레이트 최소화

대규모 앱 (ERP, 대시보드, 복합 기능):
  → MVVM 필수
  → 추가로 DI(의존성 주입), 네비게이션 프레임워크 등 도입
```

---

## 9. WinForms 개발자의 전형적인 실수

### 실수 1: View에서 직접 로직 처리

```csharp
// 나쁜 예: 코드 비하인드에 비즈니스 로직 작성 (WinForms 습관!)
public partial class MainWindow : Window
{
    private List<Employee> _employees = new();

    private void BtnSearch_Click(object sender, RoutedEventArgs e)
    {
        // 코드 비하인드에서 직접 DB 조회!
        string searchText = txtSearch.Text;  // UI 컨트롤 직접 참조!
        using var conn = new SqlConnection("...");
        conn.Open();
        var cmd = new SqlCommand("SELECT * FROM Employees WHERE Name LIKE @name", conn);
        cmd.Parameters.AddWithValue("@name", $"%{searchText}%");

        var reader = cmd.ExecuteReader();
        _employees.Clear();
        while (reader.Read())
        {
            _employees.Add(new Employee
            {
                Name = reader["Name"].ToString()!,
                Email = reader["Email"].ToString()!
            });
        }

        dgEmployees.ItemsSource = null;      // UI 직접 조작!
        dgEmployees.ItemsSource = _employees; // UI 직접 조작!
        lblCount.Content = $"검색 결과: {_employees.Count}건"; // UI 직접 조작!
    }
}
```

```csharp
// 좋은 예: MVVM 패턴 적용
// View: 바인딩만 선언
// <TextBox Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}" />
// <Button Command="{Binding SearchCommand}" />
// <DataGrid ItemsSource="{Binding Employees}" />
// <TextBlock Text="{Binding ResultCountText}" />

// ViewModel: 로직 처리 (UI 참조 없음!)
public partial class EmployeeViewModel : ObservableObject
{
    private readonly IEmployeeService _service;

    [ObservableProperty]
    private string _searchText = "";

    [ObservableProperty]
    private string _resultCountText = "";

    public ObservableCollection<Employee> Employees { get; } = new();

    [RelayCommand]
    private async Task Search()
    {
        var results = await _service.SearchAsync(SearchText);
        Employees.Clear();
        foreach (var emp in results)
        {
            Employees.Add(emp);
        }
        ResultCountText = $"검색 결과: {Employees.Count}건";
    }
}
```

### 실수 2: ViewModel에서 MessageBox 직접 호출

```csharp
// 나쁜 예: ViewModel에서 MessageBox 사용 (UI 의존성!)
public class OrderViewModel : ViewModelBase
{
    private void ExecuteDelete()
    {
        // ViewModel에서 MessageBox 직접 호출!
        // → 단위 테스트에서 MessageBox가 팝업됨!
        // → ViewModel이 WPF에 종속됨!
        var result = MessageBox.Show("정말 삭제하시겠습니까?",
                                     "확인",
                                     MessageBoxButton.YesNo);
        if (result == MessageBoxResult.Yes)
        {
            _service.Delete(SelectedOrder!.Id);
        }
    }
}
```

```csharp
// 좋은 예: 인터페이스를 통해 간접적으로 다이얼로그 표시
public interface IDialogService
{
    bool Confirm(string message, string title);
    void ShowMessage(string message, string title);
}

// 실제 구현 (WPF용)
public class WpfDialogService : IDialogService
{
    public bool Confirm(string message, string title)
    {
        return MessageBox.Show(message, title, MessageBoxButton.YesNo)
               == MessageBoxResult.Yes;
    }

    public void ShowMessage(string message, string title)
    {
        MessageBox.Show(message, title);
    }
}

// 테스트용 구현
public class MockDialogService : IDialogService
{
    public bool ConfirmResult { get; set; } = true; // 테스트에서 결과 제어
    public bool Confirm(string message, string title) => ConfirmResult;
    public void ShowMessage(string message, string title) { /* 아무것도 안 함 */ }
}

// ViewModel: 인터페이스를 통해 다이얼로그 사용
public class OrderViewModel : ViewModelBase
{
    private readonly IDialogService _dialog;

    public OrderViewModel(IDialogService dialog)
    {
        _dialog = dialog;
    }

    private void ExecuteDelete()
    {
        // 인터페이스를 통해 간접적으로 호출
        // → 테스트에서는 MockDialogService가 주입됨
        if (_dialog.Confirm("정말 삭제하시겠습니까?", "확인"))
        {
            _service.Delete(SelectedOrder!.Id);
        }
    }
}
```

### 실수 3: 바인딩 대신 x:Name으로 직접 접근

```xml
<!-- 나쁜 예: WinForms 습관대로 x:Name을 남용 -->
<TextBox x:Name="txtName" />
<TextBox x:Name="txtEmail" />
<TextBox x:Name="txtPhone" />
<Button x:Name="btnSave" Click="BtnSave_Click" />
```

```csharp
// 나쁜 예: 코드 비하인드에서 x:Name으로 직접 접근
private void BtnSave_Click(object sender, RoutedEventArgs e)
{
    string name = txtName.Text;    // 직접 참조
    string email = txtEmail.Text;  // 직접 참조
    string phone = txtPhone.Text;  // 직접 참조
    // ...
}
```

```xml
<!-- 좋은 예: 바인딩 사용 (x:Name 불필요!) -->
<TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />
<TextBox Text="{Binding Email, UpdateSourceTrigger=PropertyChanged}" />
<TextBox Text="{Binding Phone, UpdateSourceTrigger=PropertyChanged}" />
<Button Content="저장" Command="{Binding SaveCommand}" />
<!-- x:Name이 하나도 없음! -->
```

### 실수 4: ObservableCollection 대신 List 사용

```csharp
// 나쁜 예: List를 사용하면 아이템 추가/제거 시 UI가 갱신 안 됨!
public List<Employee> Employees { get; set; } = new();
// Employees.Add(newEmployee); → UI 변화 없음!

// 좋은 예: ObservableCollection 사용!
public ObservableCollection<Employee> Employees { get; } = new();
// Employees.Add(newEmployee); → UI에 새 항목 자동 표시!
```

### 실수 5: OnPropertyChanged 호출 빼먹기

```csharp
// 나쁜 예: OnPropertyChanged를 빼먹으면 UI가 갱신 안 됨!
private string _name = "";
public string Name
{
    get => _name;
    set
    {
        _name = value;
        // OnPropertyChanged() 빼먹음!
        // → Name을 바꿔도 UI에 반영 안 됨!
    }
}

// 좋은 예: 반드시 OnPropertyChanged 호출!
private string _name = "";
public string Name
{
    get => _name;
    set => SetProperty(ref _name, value); // 자동으로 OnPropertyChanged 호출!
}
```

---

## 10. 완전한 MVVM 예제

지금까지 배운 모든 개념을 종합한 **직원 관리 앱** 예제입니다.

### 프로젝트 구조

```
EmployeeManager/
├── Models/
│   └── Employee.cs              ← 데이터 모델
├── Services/
│   └── EmployeeService.cs       ← 데이터 액세스 서비스
├── ViewModels/
│   ├── ViewModelBase.cs         ← ViewModel 기본 클래스
│   ├── RelayCommand.cs          ← ICommand 구현체
│   └── EmployeeListViewModel.cs ← 메인 ViewModel
├── Views/
│   └── EmployeeListView.xaml    ← 메인 View
├── App.xaml
└── App.xaml.cs
```

### 1단계: Model

```csharp
// Models/Employee.cs
namespace EmployeeManager.Models
{
    /// <summary>
    /// 직원 데이터 모델
    /// UI에 대한 의존성이 전혀 없는 순수 데이터 클래스
    /// </summary>
    public class Employee
    {
        public int Id { get; set; }
        public string Name { get; set; } = "";
        public string Department { get; set; } = "";
        public string Email { get; set; } = "";
        public decimal Salary { get; set; }
        public bool IsActive { get; set; } = true;
    }
}
```

```csharp
// Services/EmployeeService.cs
using EmployeeManager.Models;

namespace EmployeeManager.Services
{
    /// <summary>
    /// 직원 데이터 서비스 (실제로는 DB를 사용하겠지만 여기서는 메모리 리스트)
    /// </summary>
    public class EmployeeService
    {
        // 메모리에 저장된 샘플 데이터 (실제로는 DB)
        private readonly List<Employee> _employees = new()
        {
            new() { Id = 1, Name = "홍길동", Department = "개발팀",
                    Email = "hong@example.com", Salary = 5000000 },
            new() { Id = 2, Name = "김철수", Department = "기획팀",
                    Email = "kim@example.com", Salary = 4500000 },
            new() { Id = 3, Name = "이영희", Department = "개발팀",
                    Email = "lee@example.com", Salary = 5500000 },
            new() { Id = 4, Name = "박지성", Department = "영업팀",
                    Email = "park@example.com", Salary = 4800000 },
            new() { Id = 5, Name = "최민수", Department = "개발팀",
                    Email = "choi@example.com", Salary = 6000000 },
        };

        private int _nextId = 6;

        /// <summary>
        /// 전체 직원 목록 조회
        /// </summary>
        public List<Employee> GetAll()
        {
            return _employees.Where(e => e.IsActive).ToList();
        }

        /// <summary>
        /// 이름으로 검색
        /// </summary>
        public List<Employee> Search(string keyword)
        {
            if (string.IsNullOrWhiteSpace(keyword))
                return GetAll();

            return _employees
                .Where(e => e.IsActive &&
                            e.Name.Contains(keyword, StringComparison.OrdinalIgnoreCase))
                .ToList();
        }

        /// <summary>
        /// 직원 추가
        /// </summary>
        public void Add(Employee employee)
        {
            employee.Id = _nextId++;
            _employees.Add(employee);
        }

        /// <summary>
        /// 직원 삭제 (논리적 삭제)
        /// </summary>
        public void Delete(int id)
        {
            var employee = _employees.FirstOrDefault(e => e.Id == id);
            if (employee != null)
            {
                employee.IsActive = false;
            }
        }
    }
}
```

### 2단계: ViewModel 인프라

```csharp
// ViewModels/ViewModelBase.cs
using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace EmployeeManager.ViewModels
{
    /// <summary>
    /// 모든 ViewModel의 기본 클래스
    /// INotifyPropertyChanged 구현을 제공
    /// </summary>
    public abstract class ViewModelBase : INotifyPropertyChanged
    {
        public event PropertyChangedEventHandler? PropertyChanged;

        /// <summary>
        /// 속성 변경을 UI에 알림
        /// </summary>
        protected void OnPropertyChanged([CallerMemberName] string? propertyName = null)
        {
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }

        /// <summary>
        /// 필드 값을 설정하고, 변경되었으면 알림을 보냄
        /// </summary>
        protected bool SetProperty<T>(ref T field, T value,
                                       [CallerMemberName] string? propertyName = null)
        {
            if (EqualityComparer<T>.Default.Equals(field, value))
                return false;

            field = value;
            OnPropertyChanged(propertyName);
            return true;
        }
    }
}
```

```csharp
// ViewModels/RelayCommand.cs
using System.Windows.Input;

namespace EmployeeManager.ViewModels
{
    /// <summary>
    /// ICommand 범용 구현체
    /// </summary>
    public class RelayCommand : ICommand
    {
        private readonly Action _execute;
        private readonly Func<bool>? _canExecute;

        public RelayCommand(Action execute, Func<bool>? canExecute = null)
        {
            _execute = execute ?? throw new ArgumentNullException(nameof(execute));
            _canExecute = canExecute;
        }

        public bool CanExecute(object? parameter)
        {
            return _canExecute == null || _canExecute();
        }

        public void Execute(object? parameter)
        {
            _execute();
        }

        public event EventHandler? CanExecuteChanged
        {
            add => CommandManager.RequerySuggested += value;
            remove => CommandManager.RequerySuggested -= value;
        }

        public void RaiseCanExecuteChanged()
        {
            CommandManager.InvalidateRequerySuggested();
        }
    }
}
```

### 3단계: ViewModel

```csharp
// ViewModels/EmployeeListViewModel.cs
using System.Collections.ObjectModel;
using System.Windows;
using System.Windows.Input;
using EmployeeManager.Models;
using EmployeeManager.Services;

namespace EmployeeManager.ViewModels
{
    /// <summary>
    /// 직원 목록 화면의 ViewModel
    /// View(XAML)의 모든 바인딩이 이 클래스의 속성/커맨드를 참조
    /// </summary>
    public class EmployeeListViewModel : ViewModelBase
    {
        // ===== 의존성 =====
        private readonly EmployeeService _service;

        // ===== 바인딩 속성: 검색 =====

        private string _searchText = "";
        /// <summary>검색어 (TextBox에 바인딩)</summary>
        public string SearchText
        {
            get => _searchText;
            set => SetProperty(ref _searchText, value);
        }

        // ===== 바인딩 속성: 직원 목록 =====

        /// <summary>직원 목록 (DataGrid에 바인딩)</summary>
        public ObservableCollection<Employee> Employees { get; } = new();

        private Employee? _selectedEmployee;
        /// <summary>선택된 직원 (DataGrid.SelectedItem에 바인딩)</summary>
        public Employee? SelectedEmployee
        {
            get => _selectedEmployee;
            set
            {
                if (SetProperty(ref _selectedEmployee, value))
                {
                    // 선택이 바뀌면 관련 속성과 커맨드 상태 갱신
                    OnPropertyChanged(nameof(IsEmployeeSelected));
                    OnPropertyChanged(nameof(SelectedEmployeeInfo));
                }
            }
        }

        // ===== 바인딩 속성: 상태 표시 =====

        /// <summary>직원이 선택되었는지 여부 (상세 패널 표시용)</summary>
        public bool IsEmployeeSelected => SelectedEmployee != null;

        /// <summary>선택된 직원 정보 요약 텍스트</summary>
        public string SelectedEmployeeInfo =>
            SelectedEmployee != null
                ? $"{SelectedEmployee.Name} ({SelectedEmployee.Department}) - " +
                  $"급여: {SelectedEmployee.Salary:N0}원"
                : "직원을 선택해주세요";

        /// <summary>상태바 텍스트</summary>
        private string _statusText = "준비";
        public string StatusText
        {
            get => _statusText;
            set => SetProperty(ref _statusText, value);
        }

        // ===== 바인딩 속성: 새 직원 입력 =====

        private string _newName = "";
        public string NewName
        {
            get => _newName;
            set => SetProperty(ref _newName, value);
        }

        private string _newDepartment = "";
        public string NewDepartment
        {
            get => _newDepartment;
            set => SetProperty(ref _newDepartment, value);
        }

        private string _newEmail = "";
        public string NewEmail
        {
            get => _newEmail;
            set => SetProperty(ref _newEmail, value);
        }

        // ===== 커맨드 =====

        /// <summary>검색 커맨드</summary>
        public ICommand SearchCommand { get; }

        /// <summary>추가 커맨드</summary>
        public ICommand AddCommand { get; }

        /// <summary>삭제 커맨드</summary>
        public ICommand DeleteCommand { get; }

        /// <summary>전체 보기 커맨드</summary>
        public ICommand ShowAllCommand { get; }

        // ===== 생성자 =====

        public EmployeeListViewModel(EmployeeService employeeService)
        {
            _service = employeeService;

            // 커맨드 초기화
            SearchCommand = new RelayCommand(
                execute: ExecuteSearch
            );

            AddCommand = new RelayCommand(
                execute: ExecuteAdd,
                canExecute: () => !string.IsNullOrWhiteSpace(NewName)
            );

            DeleteCommand = new RelayCommand(
                execute: ExecuteDelete,
                canExecute: () => SelectedEmployee != null
            );

            ShowAllCommand = new RelayCommand(
                execute: () =>
                {
                    SearchText = "";
                    LoadAllEmployees();
                }
            );

            // 초기 데이터 로드
            LoadAllEmployees();
        }

        // ===== 비공개 메서드 =====

        private void LoadAllEmployees()
        {
            Employees.Clear();
            foreach (var emp in _service.GetAll())
            {
                Employees.Add(emp);
            }
            StatusText = $"전체 직원: {Employees.Count}명";
        }

        private void ExecuteSearch()
        {
            Employees.Clear();
            var results = _service.Search(SearchText);
            foreach (var emp in results)
            {
                Employees.Add(emp);
            }
            StatusText = $"검색 결과: {Employees.Count}명 (검색어: \"{SearchText}\")";
        }

        private void ExecuteAdd()
        {
            var newEmployee = new Employee
            {
                Name = NewName,
                Department = NewDepartment,
                Email = NewEmail,
                Salary = 0,
                IsActive = true
            };

            _service.Add(newEmployee);
            Employees.Add(newEmployee);

            // 입력 필드 초기화
            NewName = "";
            NewDepartment = "";
            NewEmail = "";

            StatusText = $"'{newEmployee.Name}' 직원이 추가되었습니다.";
        }

        private void ExecuteDelete()
        {
            if (SelectedEmployee == null) return;

            string name = SelectedEmployee.Name;
            _service.Delete(SelectedEmployee.Id);
            Employees.Remove(SelectedEmployee);
            SelectedEmployee = null;

            StatusText = $"'{name}' 직원이 삭제되었습니다.";
        }
    }
}
```

### 4단계: View

```xml
<!-- Views/EmployeeListView.xaml -->
<Window x:Class="EmployeeManager.Views.EmployeeListView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="직원 관리 시스템 (MVVM)" Height="550" Width="750"
        WindowStartupLocation="CenterScreen">

    <Window.Resources>
        <!-- bool → Visibility 변환기 -->
        <BooleanToVisibilityConverter x:Key="BoolToVis" />

        <!-- 공통 버튼 스타일 -->
        <Style x:Key="ActionButton" TargetType="Button">
            <Setter Property="Padding" Value="15,5" />
            <Setter Property="Margin" Value="0,0,5,0" />
            <Setter Property="FontSize" Value="13" />
        </Style>
    </Window.Resources>

    <DockPanel>
        <!-- 상태바 (하단) -->
        <StatusBar DockPanel.Dock="Bottom">
            <StatusBarItem Content="{Binding StatusText}" />
        </StatusBar>

        <!-- 메인 콘텐츠 -->
        <Grid Margin="15">
            <Grid.RowDefinitions>
                <RowDefinition Height="Auto" />   <!-- 검색 영역 -->
                <RowDefinition Height="*" />      <!-- 목록 + 상세 -->
                <RowDefinition Height="Auto" />   <!-- 추가 영역 -->
            </Grid.RowDefinitions>

            <!-- === 검색 영역 === -->
            <StackPanel Grid.Row="0" Orientation="Horizontal" Margin="0,0,0,10">
                <TextBox Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
                         Width="250"
                         Padding="5,3"
                         FontSize="13"
                         Margin="0,0,5,0" />
                <Button Content="검색"
                        Command="{Binding SearchCommand}"
                        Style="{StaticResource ActionButton}" />
                <Button Content="전체 보기"
                        Command="{Binding ShowAllCommand}"
                        Style="{StaticResource ActionButton}" />
            </StackPanel>

            <!-- === 목록 + 상세 영역 === -->
            <Grid Grid.Row="1" Margin="0,0,0,10">
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="2*" />   <!-- 목록 (2/3) -->
                    <ColumnDefinition Width="*" />    <!-- 상세 (1/3) -->
                </Grid.ColumnDefinitions>

                <!-- 직원 목록 -->
                <DataGrid Grid.Column="0"
                          ItemsSource="{Binding Employees}"
                          SelectedItem="{Binding SelectedEmployee}"
                          AutoGenerateColumns="False"
                          IsReadOnly="True"
                          SelectionMode="Single"
                          Margin="0,0,10,0"
                          FontSize="13">
                    <DataGrid.Columns>
                        <DataGridTextColumn Header="이름"
                                            Binding="{Binding Name}"
                                            Width="100" />
                        <DataGridTextColumn Header="부서"
                                            Binding="{Binding Department}"
                                            Width="80" />
                        <DataGridTextColumn Header="이메일"
                                            Binding="{Binding Email}"
                                            Width="*" />
                        <DataGridTextColumn Header="급여"
                                            Binding="{Binding Salary, StringFormat={}{0:N0}원}"
                                            Width="120" />
                    </DataGrid.Columns>
                </DataGrid>

                <!-- 선택된 직원 상세 정보 -->
                <Border Grid.Column="1"
                        Background="#F5F5F5"
                        CornerRadius="5"
                        Padding="15">
                    <StackPanel>
                        <TextBlock Text="선택된 직원"
                                   FontSize="16"
                                   FontWeight="Bold"
                                   Margin="0,0,0,15" />

                        <!-- 직원이 선택되었을 때만 표시 -->
                        <StackPanel Visibility="{Binding IsEmployeeSelected,
                                                          Converter={StaticResource BoolToVis}}">
                            <TextBlock Text="이름" FontWeight="Bold" FontSize="11"
                                       Foreground="Gray" />
                            <TextBlock Text="{Binding SelectedEmployee.Name}"
                                       FontSize="14" Margin="0,0,0,10" />

                            <TextBlock Text="부서" FontWeight="Bold" FontSize="11"
                                       Foreground="Gray" />
                            <TextBlock Text="{Binding SelectedEmployee.Department}"
                                       FontSize="14" Margin="0,0,0,10" />

                            <TextBlock Text="이메일" FontWeight="Bold" FontSize="11"
                                       Foreground="Gray" />
                            <TextBlock Text="{Binding SelectedEmployee.Email}"
                                       FontSize="14" Margin="0,0,0,10" />

                            <TextBlock Text="급여" FontWeight="Bold" FontSize="11"
                                       Foreground="Gray" />
                            <TextBlock Text="{Binding SelectedEmployee.Salary,
                                              StringFormat={}{0:N0}원}"
                                       FontSize="14" Margin="0,0,0,15" />

                            <Button Content="이 직원 삭제"
                                    Command="{Binding DeleteCommand}"
                                    Background="#F44336"
                                    Foreground="White"
                                    Padding="10,5" />
                        </StackPanel>

                        <!-- 직원이 선택되지 않았을 때 표시 -->
                        <TextBlock Text="직원을 선택하면&#x0a;상세 정보가&#x0a;표시됩니다."
                                   Foreground="Gray"
                                   FontSize="13"
                                   TextWrapping="Wrap"
                                   Visibility="{Binding IsEmployeeSelected,
                                                         Converter={StaticResource BoolToVis},
                                                         ConverterParameter=Invert}" />
                        <!-- 참고: 위의 Invert 매개변수는 기본 BooleanToVisibilityConverter에서는
                             지원하지 않으므로, 실제로는 별도의 InverseBoolToVisibility 변환기 필요 -->
                    </StackPanel>
                </Border>
            </Grid>

            <!-- === 새 직원 추가 영역 === -->
            <Border Grid.Row="2" Background="#E3F2FD" CornerRadius="5" Padding="10">
                <StackPanel>
                    <TextBlock Text="새 직원 추가"
                               FontWeight="Bold"
                               FontSize="14"
                               Margin="0,0,0,8" />
                    <StackPanel Orientation="Horizontal">
                        <TextBox Text="{Binding NewName, UpdateSourceTrigger=PropertyChanged}"
                                 Width="120"
                                 Padding="5,3"
                                 Margin="0,0,5,0" />
                        <TextBox Text="{Binding NewDepartment, UpdateSourceTrigger=PropertyChanged}"
                                 Width="100"
                                 Padding="5,3"
                                 Margin="0,0,5,0" />
                        <TextBox Text="{Binding NewEmail, UpdateSourceTrigger=PropertyChanged}"
                                 Width="200"
                                 Padding="5,3"
                                 Margin="0,0,5,0" />
                        <Button Content="추가"
                                Command="{Binding AddCommand}"
                                Style="{StaticResource ActionButton}" />
                    </StackPanel>
                </StackPanel>
            </Border>
        </Grid>
    </DockPanel>
</Window>
```

```csharp
// Views/EmployeeListView.xaml.cs — 코드 비하인드는 DataContext 설정만!
using System.Windows;
using EmployeeManager.Services;
using EmployeeManager.ViewModels;

namespace EmployeeManager.Views
{
    public partial class EmployeeListView : Window
    {
        public EmployeeListView()
        {
            InitializeComponent();

            // DataContext 설정 = ViewModel 연결
            // 이것이 코드 비하인드의 유일한 역할!
            DataContext = new EmployeeListViewModel(new EmployeeService());
        }
    }
}
```

### 5단계: App.xaml 설정

```xml
<!-- App.xaml -->
<Application x:Class="EmployeeManager.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             StartupUri="Views/EmployeeListView.xaml">
    <Application.Resources>
        <!-- 앱 전역 스타일 -->
        <Style TargetType="TextBox">
            <Setter Property="FontSize" Value="13" />
        </Style>
    </Application.Resources>
</Application>
```

### 실행 결과

```
┌──────────────────────────────────────────────────────────────┐
│  직원 관리 시스템 (MVVM)                                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [홍길동          ] [검색] [전체 보기]                         │
│                                                              │
│  ┌──────────────────────────┐  ┌──────────────────┐        │
│  │ 이름    부서   이메일  급여 │  │ 선택된 직원       │        │
│  ├──────────────────────────┤  │                  │        │
│  │ 홍길동  개발팀  hong@  5M  │  │ 이름             │        │
│  │ 이영희  개발팀  lee@   5.5M│  │ 홍길동           │        │
│  │ 최민수  개발팀  choi@  6M  │  │                  │        │
│  │                          │  │ 부서             │        │
│  │                          │  │ 개발팀           │        │
│  │                          │  │                  │        │
│  └──────────────────────────┘  │ [이 직원 삭제]    │        │
│                                 └──────────────────┘        │
│  ┌ 새 직원 추가 ────────────────────────────────────┐       │
│  │ [이름     ] [부서   ] [이메일           ] [추가]  │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  검색 결과: 3명 (검색어: "홍길동")                            │
└──────────────────────────────────────────────────────────────┘
```

---

> **핵심 요약**
>
> 1. **MVVM**은 Model(데이터), View(UI), ViewModel(중재자)로 코드를 분리하는 패턴입니다.
> 2. **ViewModel은 View를 모릅니다.** 데이터 바인딩으로만 연결됩니다.
> 3. **ICommand**로 버튼 클릭 등 사용자 액션을 처리합니다 (이벤트 핸들러 대체).
> 4. **RelayCommand**는 ICommand의 범용 구현체로, 직접 만들거나 CommunityToolkit을 사용합니다.
> 5. MVVM의 최대 장점은 **테스트 용이성**과 **관심사 분리**입니다.
> 6. **CommunityToolkit.Mvvm**을 사용하면 보일러플레이트 코드를 대폭 줄일 수 있습니다.
> 7. WinForms 습관인 **코드 비하인드에서 로직 처리**, **x:Name 남용**을 경계하세요.
