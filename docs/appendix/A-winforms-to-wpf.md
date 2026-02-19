# WinForms → WPF 전환 가이드 (사고방식 전환)

> **목표**: WinForms 개발자가 WPF + MVVM으로 넘어올 때 겪는 **사고방식의 전환**을 돕습니다.
> 기술적 차이보다 **"왜 이렇게 하는지"**에 초점을 맞춥니다.

---

## 목차

1. [핵심 마인드셋 변화](#1-핵심-마인드셋-변화)
2. [WinForms → WPF 대응표](#2-winforms--wpf-대응표)
3. [WinForms 개발자가 가장 혼란스러워하는 5가지](#3-winforms-개발자가-가장-혼란스러워하는-5가지)
4. [점진적 전환 전략](#4-점진적-전환-전략)

---

## 1. 핵심 마인드셋 변화

### 한 문장 요약

> **"컨트롤을 직접 조작한다"** → **"데이터를 바인딩한다"**

### WinForms의 사고방식

```csharp
// WinForms: "내가 직접 컨트롤을 조작한다"
// 개발자가 모든 컨트롤의 상태를 하나하나 직접 제어

private void btnSave_Click(object sender, EventArgs e)
{
    // 입력값을 컨트롤에서 직접 읽기
    string name = textBoxName.Text;
    string email = textBoxEmail.Text;

    // 유효성 검사
    if (string.IsNullOrEmpty(name))
    {
        labelError.Text = "이름을 입력하세요";
        labelError.ForeColor = Color.Red;
        labelError.Visible = true;
        return;
    }

    // 버튼 비활성화 (중복 클릭 방지)
    btnSave.Enabled = false;
    progressBar1.Visible = true;

    // DB 저장 후 그리드 갱신
    SaveToDatabase(name, email);

    // 그리드에 직접 행 추가
    dataGridView1.Rows.Add(name, email);

    // 컨트롤 상태 복원
    textBoxName.Text = "";
    textBoxEmail.Text = "";
    btnSave.Enabled = true;
    progressBar1.Visible = false;
    labelError.Visible = false;
}
```

### WPF MVVM의 사고방식

```csharp
// WPF MVVM: "데이터만 변경하면 UI가 알아서 따라온다"
// 개발자는 데이터(ViewModel)만 관리, UI 업데이트는 바인딩이 자동 처리

// ViewModel (데이터 + 로직)
public partial class EmployeeViewModel : ObservableValidator
{
    [ObservableProperty]
    [Required(ErrorMessage = "이름을 입력하세요")]
    private string _name = string.Empty;

    [ObservableProperty]
    private string _email = string.Empty;

    [ObservableProperty]
    private bool _isLoading;

    // 직원 목록 (ObservableCollection이 변경을 자동 통지)
    public ObservableCollection<Employee> Employees { get; } = [];

    [RelayCommand]
    private async Task SaveAsync()
    {
        ValidateAllProperties();
        if (HasErrors) return;  // 에러 있으면 UI가 자동으로 에러 표시

        IsLoading = true;       // UI가 자동으로 프로그레스 바 표시 + 버튼 비활성화

        var employee = new Employee { Name = Name, Email = Email };
        await _repository.AddAsync(employee);

        Employees.Add(employee); // UI의 DataGrid가 자동으로 행 추가

        Name = "";               // UI의 TextBox가 자동으로 비워짐
        Email = "";
        IsLoading = false;       // UI가 자동으로 프로그레스 바 숨김 + 버튼 활성화
    }
}
```

### 시각적 비교

```
WinForms:
┌─ 코드 ─────────────────────────────────────────┐
│                                                   │
│  textBox1.Text = "김철수";                        │
│       │                                           │
│       └───────► textBox1 컨트롤에 직접 값 설정     │
│                                                   │
│  button1.Enabled = false;                         │
│       │                                           │
│       └───────► button1 컨트롤을 직접 비활성화      │
│                                                   │
│  label1.ForeColor = Color.Red;                    │
│       │                                           │
│       └───────► label1 컨트롤의 색상을 직접 변경    │
│                                                   │
│  → 개발자가 모든 UI 상태를 직접 관리!              │
└───────────────────────────────────────────────────┘

WPF MVVM:
┌─ ViewModel ──────────┐     ┌─ View (XAML) ──────────────┐
│                        │     │                              │
│  Name = "김철수";      │ ──► │ Text="{Binding Name}"       │
│                        │     │ (자동으로 TextBox에 반영)     │
│                        │     │                              │
│  IsLoading = true;     │ ──► │ IsEnabled="{Binding          │
│                        │     │   IsLoading, Converter=…}"   │
│                        │     │ (자동으로 버튼 비활성화)       │
│                        │     │                              │
│  HasErrors = true;     │ ──► │ Validation.ErrorTemplate     │
│                        │     │ (자동으로 에러 표시)           │
│                        │     │                              │
│  → 데이터만 변경!      │     │  → UI가 자동 반응!           │
└────────────────────────┘     └──────────────────────────────┘
```

---

## 2. WinForms → WPF 대응표

### 기본 패턴 대응표

| WinForms (익숙한 방식) | WPF MVVM (새로운 방식) | 설명 |
|------------------------|----------------------|------|
| `textBox1.Text = "Hello"` | `ViewModel.Name = "Hello"` | 데이터 바인딩이 자동으로 TextBox에 반영 |
| `button1_Click` 이벤트 핸들러 | `ICommand` 바인딩 (`[RelayCommand]`) | ViewModel의 커맨드를 버튼에 바인딩 |
| `listView1.Items.Add()` | `ObservableCollection.Add()` | 컬렉션 변경을 UI가 자동 감지 |
| `Form.Show()` / `ShowDialog()` | `INavigationService.NavigateTo<T>()` | 서비스를 통한 간접 네비게이션 |
| `MessageBox.Show()` | `IDialogService.ShowConfirmAsync()` | 서비스를 통한 간접 다이얼로그 |
| `BackgroundWorker` | `async/await` + `Task` | 현대적 비동기 패턴 |
| `System.Windows.Forms.Timer` | `DispatcherTimer` | UI 스레드에서 실행되는 타이머 |
| `this.Invoke()` / `BeginInvoke()` | `Dispatcher.Invoke()` | 백그라운드 → UI 스레드 마샬링 |
| `Settings.Default.Save()` | `IOptions<T>` + `appsettings.json` | DI 기반 설정 관리 |
| `Global` 변수 / `static` 클래스 | DI로 주입받는 서비스 | 의존성 주입 패턴 |

### UI 컨트롤 대응표

| WinForms 컨트롤 | WPF 컨트롤 | 주요 차이 |
|----------------|-----------|----------|
| `TextBox` | `TextBox` | `Text="{Binding Name}"` 바인딩 |
| `Label` | `TextBlock` | WPF의 `Label`은 `Content` 속성 사용 |
| `Button` | `Button` | `Command="{Binding SaveCommand}"` |
| `CheckBox` | `CheckBox` | `IsChecked="{Binding IsActive}"` |
| `ComboBox` | `ComboBox` | `ItemsSource` + `SelectedItem` 바인딩 |
| `DataGridView` | `DataGrid` | `ItemsSource="{Binding Items}"` |
| `ListView` | `ListView` / `ListBox` | `ItemTemplate`으로 커스텀 표시 |
| `Panel` | `StackPanel` / `Grid` | 레이아웃 컨테이너 |
| `GroupBox` | `GroupBox` / `Border` | `Header` 속성 |
| `TabControl` | `TabControl` | `ItemsSource` 바인딩 가능 |
| `PictureBox` | `Image` | `Source="{Binding ImagePath}"` |
| `ProgressBar` | `ProgressBar` | `Value="{Binding Progress}"` |
| `MenuStrip` | `Menu` | `ItemsSource` 바인딩 가능 |
| `StatusStrip` | `StatusBar` | 유사함 |

### 코드 패턴 대응 예제

#### 텍스트 표시

```csharp
// WinForms
labelWelcome.Text = $"환영합니다, {userName}님";
```

```csharp
// WPF MVVM — ViewModel
[ObservableProperty]
private string _welcomeMessage = string.Empty;

// 어딘가에서:
WelcomeMessage = $"환영합니다, {userName}님";
```

```xml
<!-- WPF MVVM — XAML -->
<TextBlock Text="{Binding WelcomeMessage}"/>
```

#### 리스트 항목 추가

```csharp
// WinForms
listBox1.Items.Add("새 항목");
listBox1.Items.RemoveAt(0);
```

```csharp
// WPF MVVM — ViewModel
public ObservableCollection<string> Items { get; } = [];

// 어딘가에서:
Items.Add("새 항목");
Items.RemoveAt(0);
// → UI의 ListBox가 자동으로 갱신됨
```

```xml
<!-- WPF MVVM — XAML -->
<ListBox ItemsSource="{Binding Items}"/>
```

#### 버튼 클릭 처리

```csharp
// WinForms
private void btnDelete_Click(object sender, EventArgs e)
{
    if (dataGridView1.SelectedRows.Count == 0)
    {
        MessageBox.Show("항목을 선택하세요");
        return;
    }

    var result = MessageBox.Show("삭제할까요?", "확인",
        MessageBoxButtons.YesNo);
    if (result == DialogResult.Yes)
    {
        var id = (int)dataGridView1.SelectedRows[0].Cells["Id"].Value;
        DeleteFromDB(id);
        dataGridView1.Rows.Remove(dataGridView1.SelectedRows[0]);
    }
}
```

```csharp
// WPF MVVM — ViewModel
[ObservableProperty]
[NotifyCanExecuteChangedFor(nameof(DeleteCommand))]
private Employee? _selectedEmployee;

[RelayCommand(CanExecute = nameof(CanDelete))]
private async Task DeleteAsync()
{
    var confirmed = await _dialogService.ShowConfirmAsync("확인", "삭제할까요?");
    if (!confirmed) return;

    await _repository.DeleteAsync(SelectedEmployee!.Id);
    Employees.Remove(SelectedEmployee!);
    SelectedEmployee = null;
}

private bool CanDelete() => SelectedEmployee is not null;
// → 선택 안 되면 자동으로 버튼 비활성화 (CanExecute)
```

```xml
<!-- WPF MVVM — XAML -->
<DataGrid SelectedItem="{Binding SelectedEmployee}" .../>
<Button Content="삭제" Command="{Binding DeleteCommand}"/>
<!-- 선택된 항목이 없으면 버튼이 자동으로 비활성화됨 -->
```

#### 조건부 UI 표시

```csharp
// WinForms
if (isLoggedIn)
{
    panelMain.Visible = true;
    panelLogin.Visible = false;
    labelStatus.Text = $"{userName}님 로그인됨";
    btnLogin.Text = "로그아웃";
}
else
{
    panelMain.Visible = false;
    panelLogin.Visible = true;
    labelStatus.Text = "로그인하세요";
    btnLogin.Text = "로그인";
}
```

```csharp
// WPF MVVM — ViewModel
[ObservableProperty]
private bool _isLoggedIn;

[ObservableProperty]
private string _userName = string.Empty;

// IsLoggedIn만 변경하면 UI가 알아서 반응!
```

```xml
<!-- WPF MVVM — XAML -->
<!-- IsLoggedIn에 따라 자동으로 표시/숨김 -->
<StackPanel Visibility="{Binding IsLoggedIn,
            Converter={StaticResource BooleanToVisibilityConverter}}">
    <!-- 메인 화면 -->
</StackPanel>

<StackPanel Visibility="{Binding IsLoggedIn,
            Converter={StaticResource InverseBoolToVisibilityConverter}}">
    <!-- 로그인 화면 -->
</StackPanel>

<!-- 로그인 상태에 따라 버튼 텍스트 변경 -->
<Button>
    <Button.Style>
        <Style TargetType="Button">
            <Setter Property="Content" Value="로그인"/>
            <Style.Triggers>
                <DataTrigger Binding="{Binding IsLoggedIn}" Value="True">
                    <Setter Property="Content" Value="로그아웃"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </Button.Style>
</Button>
```

---

## 3. WinForms 개발자가 가장 혼란스러워하는 5가지

### 질문 1: "DataContext가 뭔가요?"

**짧은 답변**: DataContext는 View(화면)와 ViewModel(데이터)을 연결하는 **접착제**입니다.

```csharp
// WinForms에는 DataContext 개념이 없습니다.
// 데이터를 표시하려면 컨트롤에 직접 값을 설정했습니다.
textBox1.Text = employee.Name;  // 직접 설정

// WPF에서 DataContext를 설정하면,
// 그 아래의 모든 컨트롤이 ViewModel의 프로퍼티에 바인딩할 수 있습니다.
```

```csharp
// DataContext 설정 (보통 DI를 통해 자동으로 설정됨)
// 파일: Views/MainWindow.xaml.cs
public partial class MainWindow : Window
{
    // DI로 ViewModel을 주입받아 DataContext에 설정
    public MainWindow(MainViewModel viewModel)
    {
        InitializeComponent();
        DataContext = viewModel;  // ← 이 한 줄이 View와 ViewModel을 연결
    }
}
```

```xml
<!-- DataContext가 설정되면, 바인딩이 동작합니다 -->
<Window>
    <!-- DataContext = MainViewModel 인스턴스 -->

    <!-- {Binding Name}은 MainViewModel.Name 프로퍼티를 참조 -->
    <TextBox Text="{Binding Name}"/>

    <!-- {Binding SaveCommand}는 MainViewModel.SaveCommand를 참조 -->
    <Button Command="{Binding SaveCommand}"/>

    <!-- 자식 컨트롤도 부모의 DataContext를 상속받음 -->
    <StackPanel>
        <!-- 이 TextBlock도 MainViewModel의 Email을 참조 가능 -->
        <TextBlock Text="{Binding Email}"/>
    </StackPanel>
</Window>
```

```
DataContext 상속 구조:

Window (DataContext = MainViewModel)
  ├── Grid (DataContext 상속 = MainViewModel)
  │   ├── TextBox (DataContext 상속 = MainViewModel)
  │   │   └── Text="{Binding Name}" → MainViewModel.Name
  │   ├── Button (DataContext 상속 = MainViewModel)
  │   │   └── Command="{Binding SaveCommand}" → MainViewModel.SaveCommand
  │   └── DataGrid (DataContext 상속 = MainViewModel)
  │       └── ItemsSource="{Binding Employees}" → MainViewModel.Employees
  │           └── 각 행의 DataContext = Employee 객체
  │               └── 열: {Binding Name} → Employee.Name
  └── StatusBar (DataContext 상속 = MainViewModel)
      └── Text="{Binding StatusMessage}" → MainViewModel.StatusMessage
```

> **비유**: DataContext는 "이 화면은 이 데이터를 보여줘"라는 지시서입니다.
> WinForms에서 `dataGridView1.DataSource = dataTable`로 데이터를 연결하는 것과
> 비슷하지만, WPF에서는 **모든 컨트롤**에 대해 동작합니다.

---

### 질문 2: "코드 비하인드에 코드 쓰면 안 되나요?"

**짧은 답변**: 쓸 수 있습니다. 하지만 **비즈니스 로직**은 ViewModel에 작성하세요.

```csharp
// ★ 코드 비하인드에 써도 되는 것들 (순수 UI 관련)
public partial class EmployeeListView : UserControl
{
    public EmployeeListView()
    {
        InitializeComponent();
    }

    // OK: 포커스 이동 같은 순수 UI 동작
    private void TextBox_GotFocus(object sender, RoutedEventArgs e)
    {
        if (sender is TextBox textBox)
        {
            textBox.SelectAll();  // 포커스 받으면 전체 선택
        }
    }

    // OK: 스크롤 동작 같은 순수 UI 로직
    private void DataGrid_ScrollChanged(object sender, ScrollChangedEventArgs e)
    {
        // 스크롤이 맨 아래에 도달하면 자동 스크롤 유지
        if (e.ExtentHeightChange > 0)
        {
            var grid = sender as DataGrid;
            grid?.ScrollIntoView(grid.Items[^1]);
        }
    }

    // OK: 드래그 앤 드롭 같은 복잡한 UI 인터랙션
    private void Grid_Drop(object sender, DragEventArgs e)
    {
        if (e.Data.GetDataPresent(DataFormats.FileDrop))
        {
            var files = (string[])e.Data.GetData(DataFormats.FileDrop);
            // ViewModel의 커맨드 호출
            if (DataContext is EmployeeListViewModel vm)
            {
                vm.HandleDroppedFiles(files);
            }
        }
    }
}
```

```csharp
// ✗ 코드 비하인드에 쓰면 안 되는 것들 (비즈니스 로직)
public partial class EmployeeListView : UserControl
{
    // ✗ DB 조회 → ViewModel에 작성해야 함
    private void LoadData()
    {
        using var conn = new NpgsqlConnection(connStr);
        var employees = conn.Query<Employee>("SELECT * FROM employees");
        dataGrid.ItemsSource = employees;
    }

    // ✗ 유효성 검사 → ViewModel에 작성해야 함
    private bool Validate()
    {
        if (string.IsNullOrEmpty(txtName.Text))
        {
            MessageBox.Show("이름을 입력하세요");
            return false;
        }
        return true;
    }

    // ✗ 비즈니스 로직 → ViewModel에 작성해야 함
    private void CalculateSalary()
    {
        var baseSalary = int.Parse(txtSalary.Text);
        var bonus = baseSalary * 0.1;
        lblTotal.Content = baseSalary + bonus;
    }
}
```

**판단 기준**:

| 코드 비하인드에 써도 OK | ViewModel에 작성해야 함 |
|----------------------|----------------------|
| 포커스 이동, 스크롤 | DB 조회/저장 |
| 드래그 앤 드롭 | 유효성 검사 |
| 애니메이션 트리거 | 비즈니스 계산 |
| 컨트롤 측정/레이아웃 | 상태 관리 |
| 키보드 단축키 | 에러 처리 |

---

### 질문 3: "컨트롤 이름(x:Name)을 거의 안 쓰는 이유"

**짧은 답변**: 바인딩으로 데이터를 주고받기 때문에 컨트롤을 **이름으로 참조할 필요가 없습니다.**

```csharp
// WinForms: 컨트롤 이름이 필수
// 코드에서 textBox1, button1 등의 이름으로 직접 접근
textBox1.Text = "값 설정";           // textBox1이라는 이름으로 접근
button1.Enabled = false;             // button1이라는 이름으로 접근
label1.ForeColor = Color.Red;        // label1이라는 이름으로 접근
dataGridView1.Rows.Clear();          // dataGridView1이라는 이름으로 접근
```

```xml
<!-- WPF MVVM: x:Name 없이 바인딩으로 모든 것을 처리 -->
<!-- 컨트롤 이름을 붙일 필요가 없음! -->

<!-- 이름 없음! → ViewModel.Name에 바인딩 -->
<TextBox Text="{Binding Name}"/>

<!-- 이름 없음! → ViewModel.SaveCommand에 바인딩 -->
<Button Content="저장" Command="{Binding SaveCommand}"/>

<!-- 이름 없음! → ViewModel.HasErrors로 자동 표시/숨김 -->
<TextBlock Text="{Binding ErrorMessage}"
           Foreground="Red"
           Visibility="{Binding HasErrors,
                        Converter={StaticResource BooleanToVisibilityConverter}}"/>

<!-- 이름 없음! → ViewModel.Employees에 바인딩 -->
<DataGrid ItemsSource="{Binding Employees}"
          SelectedItem="{Binding SelectedEmployee}"/>
```

**언제 x:Name을 쓰는가?**

```xml
<!-- 예외 1: 코드 비하인드에서 순수 UI 조작이 필요할 때 -->
<TextBox x:Name="SearchTextBox"/>
<!-- 코드 비하인드: SearchTextBox.Focus(); -->

<!-- 예외 2: ElementName 바인딩이 필요할 때 -->
<Slider x:Name="OpacitySlider" Minimum="0" Maximum="1" Value="0.8"/>
<Border Opacity="{Binding Value, ElementName=OpacitySlider}"/>

<!-- 예외 3: Storyboard 애니메이션 타겟 지정 -->
<Border x:Name="AnimatedBorder"/>
```

---

### 질문 4: "이벤트 핸들러 대신 Command를 쓰는 이유"

**짧은 답변**: Command를 쓰면 **로직이 ViewModel에 모이고**, **단위 테스트가 가능**해지며, **CanExecute로 활성화 조건**을 선언적으로 관리할 수 있습니다.

```csharp
// WinForms: 이벤트 핸들러 방식
// 장점: 직관적
// 단점: 1) 코드가 UI에 종속 2) 테스트 불가 3) 활성화 조건 수동 관리

private void btnSave_Click(object sender, EventArgs e)
{
    // ← 이 코드는 Form 없이는 테스트할 수 없음!
    // ← 버튼 활성화/비활성화도 수동으로 관리해야 함
    btnSave.Enabled = false;  // 수동 비활성화
    SaveEmployee();
    btnSave.Enabled = true;   // 수동 활성화
}

// 버튼 활성화 조건 관리 (수동, 실수하기 쉬움)
private void textBox1_TextChanged(object sender, EventArgs e)
{
    btnSave.Enabled = !string.IsNullOrEmpty(textBox1.Text)
                   && !string.IsNullOrEmpty(textBox2.Text);
}
```

```csharp
// WPF MVVM: Command 방식
// 장점: 1) ViewModel에서 테스트 가능 2) CanExecute 자동 관리 3) 관심사 분리

[RelayCommand(CanExecute = nameof(CanSave))]
private async Task SaveAsync()
{
    // ← ViewModel만으로 테스트 가능! (UI 없이)
    await _repository.AddAsync(new Employee { Name = Name, Email = Email });
}

// 활성화 조건을 선언적으로 정의
private bool CanSave()
{
    return !string.IsNullOrEmpty(Name)
        && !string.IsNullOrEmpty(Email)
        && !HasErrors;
}

// Name이 바뀌면 자동으로 CanSave() 재평가 → 버튼 활성화/비활성화 자동!
[ObservableProperty]
[NotifyCanExecuteChangedFor(nameof(SaveCommand))]
private string _name = string.Empty;
```

```csharp
// Command의 최대 장점: 단위 테스트 가능!
[Fact]
public async Task SaveCommand_ValidData_ShouldCallRepository()
{
    // Arrange: ViewModel을 직접 생성 (UI 필요 없음!)
    var mockRepo = new Mock<IEmployeeRepository>();
    var vm = new EmployeeEditViewModel(mockRepo.Object, /* ... */);

    vm.Name = "김철수";
    vm.Email = "chulsoo@test.com";

    // Act: Command를 직접 실행
    await vm.SaveCommand.ExecuteAsync(null);

    // Assert: Repository가 호출되었는지 확인
    mockRepo.Verify(r => r.AddAsync(It.IsAny<Employee>()), Times.Once);
}
```

---

### 질문 5: "왜 ViewModel이 View를 모르는 게 좋은 건가요?"

**짧은 답변**: ViewModel이 View를 모르면 **테스트가 쉽고**, **재사용이 가능하고**, **유지보수가 편합니다.**

```csharp
// ✗ 나쁜 예: ViewModel이 View를 알고 있음
public class EmployeeEditViewModel
{
    // ✗ View의 타입을 직접 참조
    private readonly EmployeeEditView _view;

    public void Save()
    {
        // ✗ View의 컨트롤에 직접 접근
        var name = _view.TextBoxName.Text;

        // ✗ MessageBox 직접 호출 (UI에 종속)
        MessageBox.Show("저장 완료");

        // ✗ 다른 View를 직접 생성
        var listView = new EmployeeListView();
        listView.Show();
    }
    // → 이 ViewModel은 EmployeeEditView 없이는 동작할 수 없음
    // → 단위 테스트 불가능
    // → View를 변경하면 ViewModel도 수정해야 함
}
```

```csharp
// ○ 좋은 예: ViewModel이 View를 전혀 모름
public class EmployeeEditViewModel : ObservableValidator
{
    // ○ 인터페이스에만 의존 (구현체를 모름)
    private readonly IEmployeeRepository _repository;
    private readonly IDialogService _dialogService;
    private readonly INavigationService _navigationService;

    [ObservableProperty]
    private string _name = string.Empty;

    [RelayCommand]
    private async Task SaveAsync()
    {
        // ○ 바인딩된 프로퍼티에서 값 읽기 (View 참조 없음)
        var employee = new Employee { Name = Name };
        await _repository.AddAsync(employee);

        // ○ 인터페이스를 통한 다이얼로그 (테스트 시 Mock 가능)
        await _dialogService.ShowInfoAsync("완료", "저장되었습니다.");

        // ○ 인터페이스를 통한 네비게이션 (View를 모름)
        _navigationService.GoBack();
    }
    // → View 없이도 동작 가능 (단위 테스트 가능!)
    // → View를 변경해도 ViewModel은 수정 불필요
    // → 같은 ViewModel을 다른 View에서 재사용 가능
}
```

**왜 중요한가? — 실제 이점**:

```
1. 단위 테스트
   ViewModel이 View를 모름
   → Mock 객체로 모든 의존성을 대체 가능
   → UI를 띄우지 않고도 비즈니스 로직 테스트 가능

2. UI 디자인 변경
   View(XAML)만 수정하면 됨
   → ViewModel 코드는 한 줄도 안 바꿔도 됨
   → 디자이너와 개발자가 독립적으로 작업 가능

3. 재사용
   같은 EmployeeEditViewModel을
   → 데스크탑 앱의 Window에서도 사용
   → 탭 안의 UserControl에서도 사용
   → 다이얼로그로도 사용

4. 유지보수
   버그 발생 시:
   → "UI 문제 → View(XAML) 확인"
   → "로직 문제 → ViewModel 확인"
   → "데이터 문제 → Repository 확인"
   명확한 책임 분리!
```

---

## 4. 점진적 전환 전략

WinForms에서 WPF MVVM으로 한 번에 전환하는 것은 어렵습니다. 단계적으로 접근하세요.

### 1단계: XAML과 바인딩에 익숙해지기 (1~2주)

```csharp
// 목표: "코드로 컨트롤을 조작하는 습관"을 "바인딩 습관"으로 바꾸기
// 작은 화면 하나를 만들어 보세요.

// ViewModel (간단한 것부터)
public partial class HelloViewModel : ObservableObject
{
    [ObservableProperty]
    private string _userName = string.Empty;

    // 계산된 프로퍼티: UserName이 바뀌면 자동 갱신
    public string Greeting => $"안녕하세요, {UserName}님!";

    partial void OnUserNameChanged(string value)
    {
        OnPropertyChanged(nameof(Greeting));
    }
}
```

```xml
<!-- View: 바인딩으로 연결 -->
<StackPanel Margin="20">
    <TextBox Text="{Binding UserName, UpdateSourceTrigger=PropertyChanged}"/>
    <TextBlock Text="{Binding Greeting}" FontSize="20" Margin="0,10,0,0"/>
</StackPanel>
```

### 2단계: Command 패턴에 익숙해지기 (1~2주)

```csharp
// 목표: 이벤트 핸들러 대신 Command 사용

// "버튼을 누르면 ○○한다" → [RelayCommand]로 변환
[RelayCommand]
private async Task LoadDataAsync()
{
    // WinForms에서 btnLoad_Click에 있던 코드를 여기로 이동
    IsLoading = true;
    var data = await _repository.GetAllAsync();
    // ...
    IsLoading = false;
}
```

### 3단계: DI와 서비스 패턴 적용 (1~2주)

```csharp
// 목표: new 대신 DI, MessageBox 대신 IDialogService

// 전:
var repo = new EmployeeRepository("연결문자열");  // ✗ 직접 생성
MessageBox.Show("완료");                            // ✗ 직접 호출

// 후:
public EmployeeViewModel(                           // ○ DI 주입
    IEmployeeRepository repo,
    IDialogService dialog)
{
    _repo = repo;
    _dialog = dialog;
}
await _dialog.ShowInfoAsync("완료", "저장되었습니다"); // ○ 서비스 통해 호출
```

### 4단계: 네비게이션 적용 (1주)

```csharp
// 목표: Form.Show() 대신 NavigationService

// 전:
var editForm = new EditForm(selectedId);
editForm.ShowDialog();

// 후:
_navigationService.NavigateTo<EmployeeEditViewModel>(parameter: selectedId);
```

### 5단계: 테스트 작성 (지속)

```csharp
// 목표: ViewModel 단위 테스트

[Fact]
public async Task LoadEmployees_ShouldPopulateCollection()
{
    // Arrange
    var mockRepo = new Mock<IEmployeeRepository>();
    mockRepo.Setup(r => r.GetAllAsync())
            .ReturnsAsync(new[] { new Employee { Name = "테스트" } });

    var vm = new EmployeeListViewModel(
        mockRepo.Object,
        Mock.Of<INavigationService>(),
        Mock.Of<IDialogService>(),
        Mock.Of<ILogger>());

    // Act
    await vm.LoadEmployeesCommand.ExecuteAsync(null);

    // Assert
    Assert.Single(vm.Employees);
    Assert.Equal("테스트", vm.Employees[0].Name);
}
```

### 전환 단계 요약

```
단계 1: XAML + 바인딩
         ↓
단계 2: Command 패턴
         ↓
단계 3: DI + 서비스
         ↓
단계 4: 네비게이션
         ↓
단계 5: 테스트
```

> **조언**: 각 단계를 완전히 이해하기 전에 다음 단계로 넘어가지 마세요.
> 특히 **1단계(바인딩)**가 가장 중요합니다. 바인딩이 자연스러워지면 나머지는 따라옵니다.
> "컨트롤에 값을 설정하려는 충동"이 들 때마다, "ViewModel 프로퍼티를 바꾸면 되지"라고
> 스스로에게 말해 보세요. 이것이 WPF MVVM의 핵심 사고방식입니다.
