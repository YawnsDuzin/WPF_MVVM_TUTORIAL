# 흔한 실수와 해결법

> **목표**: WPF + MVVM을 처음 배울 때 자주 겪는 실수들을 모아,
> **잘못된 코드 → 올바른 코드 → 왜 그런지** 형태로 정리합니다.
> 막힐 때 이 문서를 참고하세요.

---

## 목차

1. [바인딩 관련 실수](#1-바인딩-관련-실수)
2. [MVVM 관련 실수](#2-mvvm-관련-실수)
3. [비동기 관련 실수](#3-비동기-관련-실수)
4. [DI 관련 실수](#4-di-관련-실수)
5. [성능 관련 실수](#5-성능-관련-실수)

---

## 1. 바인딩 관련 실수

### 실수 1-1: DataContext 설정 안 함

**증상**: 바인딩을 설정했는데 아무것도 표시되지 않는다. 에러도 없다.

```xml
<!-- ✗ 잘못된 코드: DataContext가 설정되지 않음 -->
<Window x:Class="MyApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation">
    <!-- 바인딩을 했지만 DataContext가 없어서 아무것도 안 나옴 -->
    <TextBox Text="{Binding Name}"/>
    <Button Content="저장" Command="{Binding SaveCommand}"/>
</Window>
```

```csharp
// ✗ 잘못된 코드 비하인드: DataContext 설정을 잊음
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        // DataContext = ??? ← 빠져있음!
    }
}
```

```csharp
// ○ 올바른 코드: DI를 통해 ViewModel을 주입받아 DataContext에 설정
public partial class MainWindow : Window
{
    public MainWindow(MainViewModel viewModel)
    {
        InitializeComponent();
        DataContext = viewModel;  // ← 이 한 줄이 핵심!
    }
}
```

**왜?**: DataContext가 없으면 `{Binding Name}`이 참조할 객체가 없습니다. WPF는 바인딩 실패를 **조용히** 무시하므로 에러 메시지 없이 빈 화면이 표시됩니다.

**디버깅 팁**: Visual Studio의 **출력(Output) 창**에서 바인딩 에러를 확인하세요.

```
// 출력 창에 이런 메시지가 나타남:
System.Windows.Data Error: 40 : BindingExpression path error:
'Name' property not found on 'object' ''  (HashCode=...).
```

---

### 실수 1-2: 프로퍼티명 오타 (바인딩 에러 확인법)

**증상**: 바인딩이 동작하지 않는다. Output 창에 바인딩 에러가 출력된다.

```xml
<!-- ✗ 잘못된 코드: 프로퍼티명 오타 (EmployeName → EmployeeName) -->
<TextBlock Text="{Binding EmployeName}"/>
<!-- 실제 ViewModel 프로퍼티는 EmployeeName (e가 빠짐) -->
```

```xml
<!-- ○ 올바른 코드: 정확한 프로퍼티명 -->
<TextBlock Text="{Binding EmployeeName}"/>
```

**바인딩 에러 확인법**:

```
1. Visual Studio → 보기(View) → 출력(Output) (Ctrl+Alt+O)
2. "출력 원본(Show output from)" 드롭다운에서 "디버그(Debug)" 선택
3. 앱 실행 후 바인딩 에러 메시지 확인:

   System.Windows.Data Error: 40 :
   BindingExpression path error: 'EmployeName' property not found
   on 'object' 'EmployeeListViewModel' (HashCode=12345678).

   → 'EmployeName'이 EmployeeListViewModel에 없다는 뜻
   → 오타를 수정하면 해결
```

**추가 팁: 바인딩 진단 강화**

```xml
<!-- App.xaml에서 바인딩 실패 시 예외를 발생시키도록 설정 -->
<!-- 개발 중에만 사용하고 배포 시에는 제거하세요 -->
<Application.Resources>
    <!-- PresentationTraceSources로 바인딩 디버깅 강화 -->
</Application.Resources>
```

```xml
<!-- 특정 바인딩에 디버그 수준 설정 -->
<TextBlock Text="{Binding EmployeeName,
           diag:PresentationTraceSources.TraceLevel=High}"
           xmlns:diag="clr-namespace:System.Diagnostics;assembly=WindowsBase"/>
```

---

### 실수 1-3: INotifyPropertyChanged 미구현

**증상**: 초기값은 표시되지만, 프로퍼티 값을 변경해도 UI가 갱신되지 않는다.

```csharp
// ✗ 잘못된 코드: 일반 프로퍼티 (변경 알림 없음)
public class EmployeeViewModel
{
    // 일반 프로퍼티는 값이 바뀌어도 UI에 통지하지 않음!
    public string Name { get; set; } = string.Empty;

    public void UpdateName()
    {
        Name = "김철수";  // 값은 바뀌지만 UI는 갱신되지 않음!
    }
}
```

```csharp
// ○ 올바른 코드: CommunityToolkit.Mvvm의 [ObservableProperty] 사용
public partial class EmployeeViewModel : ObservableObject
{
    // [ObservableProperty]가 INotifyPropertyChanged 구현을 자동 생성
    [ObservableProperty]
    private string _name = string.Empty;

    public void UpdateName()
    {
        Name = "김철수";  // UI가 자동으로 갱신됨!
        // 소스 제네레이터가 내부적으로 PropertyChanged 이벤트를 발생시킴
    }
}
```

**왜?**: WPF의 바인딩 엔진은 `INotifyPropertyChanged.PropertyChanged` 이벤트를 통해 프로퍼티 값이 변경된 것을 감지합니다. 이 인터페이스가 없으면 바인딩 엔진은 값이 바뀌었는지 알 수 없습니다.

---

### 실수 1-4: OneWay인데 TwoWay가 필요한 경우

**증상**: TextBox에 입력하는데 ViewModel의 프로퍼티가 갱신되지 않는다.

```xml
<!-- ✗ 잘못된 코드: Mode를 명시적으로 OneWay로 설정 -->
<TextBox Text="{Binding Name, Mode=OneWay}"/>
<!-- OneWay: ViewModel → View 방향만 (View에서 입력해도 ViewModel에 반영 안 됨) -->
```

```xml
<!-- ○ 올바른 코드: TwoWay 바인딩 (TextBox는 기본이 TwoWay이지만 명시 가능) -->
<TextBox Text="{Binding Name, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"/>
<!-- TwoWay: ViewModel ↔ View 양방향 -->
<!-- UpdateSourceTrigger=PropertyChanged: 매 키 입력마다 즉시 반영 -->
```

**컨트롤별 기본 바인딩 Mode**:

| 컨트롤 | 프로퍼티 | 기본 Mode |
|--------|---------|----------|
| TextBox | Text | TwoWay |
| CheckBox | IsChecked | TwoWay |
| ComboBox | SelectedItem | TwoWay |
| Slider | Value | TwoWay |
| TextBlock | Text | **OneWay** |
| Label | Content | **OneWay** |
| ProgressBar | Value | **OneWay** |
| DataGrid | SelectedItem | TwoWay |

> **팁**: 입력용 컨트롤(TextBox, CheckBox 등)은 기본 TwoWay, 표시용 컨트롤(TextBlock, Label 등)은 기본 OneWay입니다.

---

### 실수 1-5: ObservableCollection 대신 List 사용

**증상**: 목록에 항목을 추가/삭제해도 UI의 리스트가 갱신되지 않는다.

```csharp
// ✗ 잘못된 코드: List<T> 사용
public partial class EmployeeListViewModel : ObservableObject
{
    // List<T>는 항목 추가/삭제 시 UI에 통지하지 않음!
    public List<Employee> Employees { get; } = [];

    [RelayCommand]
    private async Task LoadAsync()
    {
        Employees.Clear();              // UI가 모름!
        var data = await _repo.GetAllAsync();
        foreach (var emp in data)
        {
            Employees.Add(emp);         // UI가 모름! → 화면에 안 나옴
        }
    }
}
```

```csharp
// ○ 올바른 코드: ObservableCollection<T> 사용
public partial class EmployeeListViewModel : ObservableObject
{
    // ObservableCollection<T>은 항목 변경 시 CollectionChanged 이벤트를 발생시킴
    // → UI가 자동으로 갱신됨
    public ObservableCollection<Employee> Employees { get; } = [];

    [RelayCommand]
    private async Task LoadAsync()
    {
        Employees.Clear();              // UI가 감지 → 리스트 비움
        var data = await _repo.GetAllAsync();
        foreach (var emp in data)
        {
            Employees.Add(emp);         // UI가 감지 → 행 추가
        }
    }
}
```

**왜?**: `List<T>`는 `INotifyCollectionChanged` 인터페이스를 구현하지 않습니다. `ObservableCollection<T>`은 이 인터페이스를 구현하여 항목 추가/삭제/이동 시 UI에 자동으로 통지합니다.

**주의**: `ObservableCollection`은 **항목의 추가/삭제**만 감지합니다. 항목 내부 프로퍼티 변경은 해당 항목이 `INotifyPropertyChanged`를 구현해야 감지됩니다.

```csharp
// 예: Employee 객체의 Name을 변경해도 DataGrid 셀은 자동 갱신되지 않음
// (Employee가 INotifyPropertyChanged를 구현하지 않은 경우)
Employees[0].Name = "변경된 이름";  // DataGrid에 반영 안 됨!

// 해결: Employee도 ObservableObject를 상속하거나,
// 목록을 통째로 교체하는 방식 사용
```

---

## 2. MVVM 관련 실수

### 실수 2-1: View의 코드 비하인드에 비즈니스 로직 작성

**증상**: 코드가 동작은 하지만 테스트 불가, 유지보수 어려움, View 교체 시 로직도 함께 이동해야 함.

```csharp
// ✗ 잘못된 코드: 코드 비하인드에 비즈니스 로직
public partial class EmployeeListView : UserControl
{
    private readonly NpgsqlConnection _connection;

    public EmployeeListView()
    {
        InitializeComponent();
        _connection = new NpgsqlConnection("Host=localhost;Database=mydb");
    }

    // ✗ DB 조회 로직이 View에 있음
    private async void btnLoad_Click(object sender, RoutedEventArgs e)
    {
        var employees = await _connection.QueryAsync<Employee>(
            "SELECT * FROM employees");

        dataGrid.ItemsSource = employees.ToList();
    }

    // ✗ 삭제 로직이 View에 있음
    private async void btnDelete_Click(object sender, RoutedEventArgs e)
    {
        if (dataGrid.SelectedItem is Employee emp)
        {
            await _connection.ExecuteAsync(
                "DELETE FROM employees WHERE id = @Id",
                new { emp.Id });

            btnLoad_Click(null, null);  // 목록 새로고침
        }
    }
}
```

```csharp
// ○ 올바른 코드: ViewModel에 비즈니스 로직, View는 바인딩만

// ViewModel (비즈니스 로직 담당)
public partial class EmployeeListViewModel : ObservableObject
{
    private readonly IEmployeeRepository _repository;

    public ObservableCollection<Employee> Employees { get; } = [];

    [ObservableProperty]
    private Employee? _selectedEmployee;

    [RelayCommand]
    private async Task LoadAsync()
    {
        var employees = await _repository.GetAllAsync();
        Employees.Clear();
        foreach (var emp in employees)
            Employees.Add(emp);
    }

    [RelayCommand(CanExecute = nameof(CanDelete))]
    private async Task DeleteAsync()
    {
        await _repository.DeleteAsync(SelectedEmployee!.Id);
        Employees.Remove(SelectedEmployee!);
    }

    private bool CanDelete() => SelectedEmployee is not null;
}
```

```xml
<!-- View (바인딩만 담당) -->
<UserControl>
    <DataGrid ItemsSource="{Binding Employees}"
              SelectedItem="{Binding SelectedEmployee}"/>
    <Button Content="로드" Command="{Binding LoadCommand}"/>
    <Button Content="삭제" Command="{Binding DeleteCommand}"/>
</UserControl>
```

```csharp
// View 코드 비하인드 (최소한의 코드만)
public partial class EmployeeListView : UserControl
{
    public EmployeeListView()
    {
        InitializeComponent();
        // 비즈니스 로직 없음!
    }
}
```

---

### 실수 2-2: ViewModel에서 View 직접 참조

**증상**: ViewModel이 특정 View에 종속되어 단위 테스트가 불가능하다.

```csharp
// ✗ 잘못된 코드: ViewModel이 View의 컨트롤을 직접 참조
public class EmployeeEditViewModel
{
    // ✗ View의 컨트롤을 멤버로 가지고 있음
    private readonly TextBox _nameTextBox;
    private readonly DataGrid _employeeGrid;

    public EmployeeEditViewModel(TextBox nameTextBox, DataGrid grid)
    {
        _nameTextBox = nameTextBox;
        _employeeGrid = grid;
    }

    public void Save()
    {
        // ✗ View의 컨트롤에서 직접 값 읽기
        var name = _nameTextBox.Text;

        // ✗ View의 컨트롤을 직접 조작
        _employeeGrid.Items.Refresh();
    }
}
```

```csharp
// ○ 올바른 코드: ViewModel은 View를 전혀 모름
public partial class EmployeeEditViewModel : ObservableObject
{
    // ○ View 참조 없음! 오직 데이터 프로퍼티만 가짐
    [ObservableProperty]
    private string _name = string.Empty;

    public ObservableCollection<Employee> Employees { get; } = [];

    [RelayCommand]
    private async Task SaveAsync()
    {
        // ○ 바인딩된 프로퍼티에서 값 읽기
        var employee = new Employee { Name = Name };
        await _repository.AddAsync(employee);

        // ○ ObservableCollection에 추가 (UI가 자동 갱신)
        Employees.Add(employee);
    }
}
```

---

### 실수 2-3: ViewModel에서 MessageBox.Show() 호출

**증상**: ViewModel이 System.Windows에 의존하여 테스트가 불가능하다.

```csharp
// ✗ 잘못된 코드: ViewModel에서 MessageBox 직접 호출
using System.Windows; // ✗ ViewModel이 WPF UI 네임스페이스에 의존!

public partial class EmployeeEditViewModel : ObservableObject
{
    [RelayCommand]
    private async Task DeleteAsync()
    {
        // ✗ MessageBox.Show()는 UI 요소!
        // → 단위 테스트 시 MessageBox가 떠서 테스트가 멈춤
        var result = MessageBox.Show(
            "정말 삭제하시겠습니까?",
            "확인",
            MessageBoxButton.YesNo);

        if (result == MessageBoxResult.Yes)
        {
            await _repository.DeleteAsync(SelectedEmployee!.Id);
        }
    }
}
```

```csharp
// ○ 올바른 코드: IDialogService를 통해 간접 호출
public partial class EmployeeEditViewModel : ObservableObject
{
    private readonly IDialogService _dialogService;  // ○ 인터페이스에 의존

    [RelayCommand]
    private async Task DeleteAsync()
    {
        // ○ 인터페이스를 통한 간접 호출
        // → 테스트 시 Mock으로 교체 가능
        var confirmed = await _dialogService.ShowConfirmAsync(
            "확인", "정말 삭제하시겠습니까?");

        if (confirmed)
        {
            await _repository.DeleteAsync(SelectedEmployee!.Id);
        }
    }
}
```

```csharp
// 테스트 코드: Mock으로 "예"를 자동 응답하도록 설정
[Fact]
public async Task Delete_WhenConfirmed_ShouldCallRepository()
{
    // Mock 다이얼로그: 항상 "예" 반환
    var mockDialog = new Mock<IDialogService>();
    mockDialog.Setup(d => d.ShowConfirmAsync(It.IsAny<string>(), It.IsAny<string>()))
              .ReturnsAsync(true);  // 자동으로 "예" 응답

    var vm = new EmployeeEditViewModel(
        mockRepo.Object, mockDialog.Object, /* ... */);

    await vm.DeleteCommand.ExecuteAsync(null);

    mockRepo.Verify(r => r.DeleteAsync(It.IsAny<int>()), Times.Once);
}
```

---

### 실수 2-4: 커맨드의 CanExecute 안 연결

**증상**: 버튼이 항상 활성화되어 있어서, 데이터가 없는 상태에서도 클릭할 수 있다. NullReferenceException 발생.

```csharp
// ✗ 잘못된 코드: CanExecute 없이 null 체크를 메서드 내부에서만 처리
public partial class EmployeeListViewModel : ObservableObject
{
    [ObservableProperty]
    private Employee? _selectedEmployee;

    // ✗ CanExecute 미설정 → 버튼이 항상 활성화
    [RelayCommand]
    private async Task DeleteAsync()
    {
        if (SelectedEmployee is null) return;  // null 체크는 있지만...
        // 사용자는 "버튼이 활성화되어 있으니 클릭 가능한 것"이라 생각함
        // → UX가 나쁨
        await _repository.DeleteAsync(SelectedEmployee.Id);
    }
}
```

```csharp
// ○ 올바른 코드: CanExecute를 연결하여 버튼 자동 비활성화
public partial class EmployeeListViewModel : ObservableObject
{
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(DeleteCommand))]  // ← 이것이 핵심!
    private Employee? _selectedEmployee;

    // ○ CanExecute를 연결하면 조건 불충족 시 버튼이 자동 비활성화
    [RelayCommand(CanExecute = nameof(CanDelete))]
    private async Task DeleteAsync()
    {
        // CanExecute가 true인 경우에만 이 메서드가 호출되므로
        // null 체크가 보장됨
        await _repository.DeleteAsync(SelectedEmployee!.Id);
        Employees.Remove(SelectedEmployee!);
    }

    // 활성화 조건: 직원이 선택되어 있을 때만
    private bool CanDelete() => SelectedEmployee is not null;
}
```

**`NotifyCanExecuteChangedFor`의 동작 원리**:

```
1. SelectedEmployee 프로퍼티 값이 변경됨
2. [NotifyCanExecuteChangedFor(nameof(DeleteCommand))]에 의해
   DeleteCommand.NotifyCanExecuteChanged() 자동 호출
3. WPF 바인딩 엔진이 CanDelete() 재평가
4. 결과에 따라 버튼 IsEnabled 자동 변경
```

---

## 3. 비동기 관련 실수

### 실수 3-1: async void 사용 (async Task가 아닌)

**증상**: 예외가 발생해도 catch되지 않아 앱이 갑자기 종료된다.

```csharp
// ✗ 잘못된 코드: async void
public partial class EmployeeListViewModel : ObservableObject
{
    // ✗ async void → 예외가 호출자에게 전파되지 않음!
    // → try-catch가 없으면 앱이 비정상 종료될 수 있음
    [RelayCommand]
    private async void LoadEmployees()  // ✗ void!
    {
        var data = await _repository.GetAllAsync();  // 여기서 예외 발생하면?
        // → 호출자가 catch할 수 없음
        // → 전역 UnhandledException으로 빠짐
        // → 앱 크래시!
    }
}
```

```csharp
// ○ 올바른 코드: async Task
public partial class EmployeeListViewModel : ObservableObject
{
    // ○ async Task → 예외가 Task를 통해 정상적으로 전파됨
    [RelayCommand]
    private async Task LoadEmployeesAsync()  // ○ Task!
    {
        try
        {
            var data = await _repository.GetAllAsync();
            // ... 데이터 처리
        }
        catch (Exception ex)
        {
            // ○ 예외를 정상적으로 catch 가능
            _logger.Error(ex, "데이터 로드 실패");
            await _dialogService.ShowErrorAsync("오류", ex.Message);
        }
    }
}
```

**`async void`를 써야 하는 유일한 경우**: 이벤트 핸들러

```csharp
// 이벤트 핸들러는 반환 타입이 void로 고정되어 있으므로 async void 사용
// 하지만 반드시 try-catch로 감싸야 합니다!
private async void Window_Loaded(object sender, RoutedEventArgs e)
{
    try
    {
        await InitializeAsync();
    }
    catch (Exception ex)
    {
        _logger.Error(ex, "초기화 실패");
    }
}
```

---

### 실수 3-2: UI 스레드에서 긴 작업 실행 (UI 멈춤)

**증상**: 버튼을 클릭하면 UI가 몇 초간 멈추고(Not Responding), 작업이 끝나면 다시 반응한다.

```csharp
// ✗ 잘못된 코드: 동기적으로 DB 조회 → UI 스레드 블로킹
[RelayCommand]
private void LoadEmployees()  // ✗ async가 아님!
{
    // ✗ .Result나 .Wait()는 UI 스레드를 블로킹함!
    var data = _repository.GetAllAsync().Result;  // ✗ UI 멈춤!

    // 또는
    var data2 = _repository.GetAllAsync().GetAwaiter().GetResult();  // ✗ 역시 UI 멈춤!

    // 또는 아예 동기 호출
    Thread.Sleep(5000);  // ✗ 5초간 UI 완전 멈춤!
}
```

```csharp
// ○ 올바른 코드: async/await 사용
[RelayCommand]
private async Task LoadEmployeesAsync()  // ○ async Task
{
    IsLoading = true;

    // ○ await: UI 스레드를 블로킹하지 않고 비동기 대기
    var data = await _repository.GetAllAsync();  // UI 계속 반응

    // CPU 바운드 작업은 Task.Run으로
    var processed = await Task.Run(() => HeavyComputation(data));

    Employees.Clear();
    foreach (var emp in processed)
        Employees.Add(emp);

    IsLoading = false;
}
```

**절대 하지 말 것**:

| 패턴 | 문제 |
|------|------|
| `.Result` | UI 스레드 블로킹, 데드락 위험 |
| `.Wait()` | UI 스레드 블로킹, 데드락 위험 |
| `.GetAwaiter().GetResult()` | UI 스레드 블로킹 |
| `Thread.Sleep()` | UI 스레드 블로킹 |
| 동기적 DB 조회 | UI 스레드 블로킹 |

---

### 실수 3-3: 백그라운드 스레드에서 UI 컨트롤 접근

**증상**: `System.InvalidOperationException: The calling thread cannot access this object because a different thread owns it.`

```csharp
// ✗ 잘못된 코드: Task.Run 내부에서 ObservableCollection 직접 조작
[RelayCommand]
private async Task LoadEmployeesAsync()
{
    await Task.Run(async () =>
    {
        var data = await _repository.GetAllAsync();

        // ✗ Task.Run 내부는 백그라운드 스레드!
        // ObservableCollection은 UI 스레드에서만 조작 가능!
        Employees.Clear();          // ✗ InvalidOperationException!
        foreach (var emp in data)
        {
            Employees.Add(emp);     // ✗ InvalidOperationException!
        }
    });
}
```

```csharp
// ○ 올바른 코드 (방법 1): Task.Run 밖에서 UI 조작
[RelayCommand]
private async Task LoadEmployeesAsync()
{
    // 데이터만 백그라운드에서 가져오기
    var data = await _repository.GetAllAsync();
    // → Repository 내부에서 이미 비동기 IO를 사용하므로
    //   실제로는 Task.Run조차 불필요

    // await 이후: UI 스레드에서 실행됨
    Employees.Clear();          // ○ UI 스레드에서 안전
    foreach (var emp in data)
    {
        Employees.Add(emp);     // ○ UI 스레드에서 안전
    }
}
```

```csharp
// ○ 올바른 코드 (방법 2): Dispatcher 사용 (MQTT 콜백 등)
private Task OnMqttMessageReceived(MqttApplicationMessageReceivedEventArgs e)
{
    var message = Encoding.UTF8.GetString(e.ApplicationMessage.PayloadSegment);

    // 이 콜백은 MQTT 라이브러리의 백그라운드 스레드에서 호출됨
    // Dispatcher를 통해 UI 스레드에서 실행
    Application.Current.Dispatcher.Invoke(() =>
    {
        Messages.Add(message);  // ○ Dispatcher.Invoke 안이므로 안전
    });

    return Task.CompletedTask;
}
```

---

### 실수 3-4: ConfigureAwait 오용

**증상**: `ConfigureAwait(false)` 이후 UI 컨트롤에 접근하면 예외 발생.

```csharp
// ✗ 잘못된 코드: ViewModel에서 ConfigureAwait(false) 사용
[RelayCommand]
private async Task LoadAsync()
{
    IsLoading = true;

    // ✗ ConfigureAwait(false): UI 스레드로 돌아가지 않음!
    var data = await _repository.GetAllAsync().ConfigureAwait(false);

    // ✗ 이 코드는 ThreadPool 스레드에서 실행될 수 있음!
    Employees.Clear();      // ✗ UI 스레드가 아닐 수 있음 → 예외 가능!
    IsLoading = false;      // ✗ 바인딩 갱신이 안 될 수 있음!
}
```

```csharp
// ○ 올바른 코드: ViewModel에서는 ConfigureAwait 생략 (기본값 = true)
[RelayCommand]
private async Task LoadAsync()
{
    IsLoading = true;

    // ○ ConfigureAwait 생략: await 후 UI 스레드로 복귀
    var data = await _repository.GetAllAsync();

    // ○ UI 스레드에서 안전하게 실행
    Employees.Clear();
    foreach (var emp in data)
        Employees.Add(emp);

    IsLoading = false;
}
```

**올바른 사용 위치**: Repository, Service 등 UI와 무관한 계층

```csharp
// ○ Repository에서 ConfigureAwait(false) 사용
public class EmployeeRepository : IEmployeeRepository
{
    public async Task<IEnumerable<Employee>> GetAllAsync()
    {
        await using var conn = CreateConnection();
        // ○ Repository는 UI와 무관하므로 ConfigureAwait(false) OK
        return await conn.QueryAsync<Employee>("SELECT * FROM employees")
            .ConfigureAwait(false);
    }
}
```

---

## 4. DI 관련 실수

### 실수 4-1: new로 직접 생성 (DI 무시)

**증상**: DI를 설정했는데도 `new`로 직접 생성하여 DI의 이점(자동 주입, 수명 관리)을 포기하게 된다.

```csharp
// ✗ 잘못된 코드: ViewModel에서 Repository를 직접 생성
public partial class EmployeeListViewModel : ObservableObject
{
    // ✗ new로 직접 생성 → DI 없이 모든 의존성을 직접 관리해야 함
    private readonly EmployeeRepository _repository
        = new EmployeeRepository("Host=localhost;Database=mydb",
                                  new LoggerConfiguration().CreateLogger());

    // ✗ NavigationService도 직접 생성 → 앱 전체와 공유되지 않음
    private readonly NavigationService _navigation = new NavigationService(/* ... */);
}
```

```csharp
// ○ 올바른 코드: 생성자 주입으로 DI 활용
public partial class EmployeeListViewModel : ObservableObject
{
    // ○ 인터페이스에 의존 (구현체를 모름)
    private readonly IEmployeeRepository _repository;
    private readonly INavigationService _navigation;
    private readonly ILogger _logger;

    // ○ DI 컨테이너가 자동으로 의존성 주입
    public EmployeeListViewModel(
        IEmployeeRepository repository,    // ← DI가 자동 주입
        INavigationService navigation,     // ← DI가 자동 주입
        ILogger logger)                    // ← DI가 자동 주입
    {
        _repository = repository;
        _navigation = navigation;
        _logger = logger;
    }
}
```

```csharp
// DI 등록 (App.xaml.cs)
services.AddSingleton<IEmployeeRepository, EmployeeRepository>();
services.AddSingleton<INavigationService, NavigationService>();
services.AddTransient<EmployeeListViewModel>();  // DI가 생성자 파라미터를 자동 주입
```

---

### 실수 4-2: Singleton vs Transient 잘못 선택

**증상**: 화면을 다시 열었는데 이전 데이터가 남아있거나, 반대로 공유되어야 할 상태가 각 화면에서 따로 관리된다.

```csharp
// ✗ 잘못된 코드: ViewModel을 Singleton으로 등록
services.AddSingleton<EmployeeEditViewModel>();
// 문제: 편집 화면을 닫고 다시 열어도 이전 입력값이 남아있음!
// → 새 직원 추가 시 이전 편집 데이터가 표시됨

// ✗ 잘못된 코드: NavigationService를 Transient로 등록
services.AddTransient<INavigationService, NavigationService>();
// 문제: 매번 새 인스턴스가 생성되어 네비게이션 스택이 공유되지 않음!
// → 뒤로 가기가 동작하지 않음
```

```csharp
// ○ 올바른 코드: 용도에 맞는 수명 선택
// Singleton: 앱 전체에서 하나만 있어야 하는 것
services.AddSingleton<INavigationService, NavigationService>();  // ○ 네비게이션 스택 공유
services.AddSingleton<IDialogService, DialogService>();          // ○ 다이얼로그 서비스
services.AddSingleton<MainViewModel>();                          // ○ 메인 윈도우 VM
services.AddSingleton(Log.Logger);                               // ○ 로거

// Transient: 매번 새로 생성해야 하는 것
services.AddTransient<EmployeeListViewModel>();    // ○ 화면 열 때마다 깨끗한 상태
services.AddTransient<EmployeeEditViewModel>();    // ○ 편집할 때마다 새 인스턴스
```

**수명 선택 가이드**:

| 수명 | 사용 대상 | 이유 |
|------|----------|------|
| **Singleton** | NavigationService, DialogService | 앱 전체에서 상태 공유 필요 |
| **Singleton** | MainViewModel | 앱 수명과 동일 |
| **Singleton** | Repository | DB 연결 풀 재사용 (상황에 따라) |
| **Singleton** | Logger | 하나의 로거 인스턴스 공유 |
| **Transient** | 화면 ViewModel | 매번 깨끗한 상태로 시작 |
| **Transient** | View (UserControl) | ViewModel과 함께 새로 생성 |
| **Scoped** | WPF에서는 거의 안 쓰임 | (ASP.NET의 HTTP 요청 범위용) |

---

### 실수 4-3: 순환 의존성

**증상**: `System.InvalidOperationException: A circular dependency was detected for the service of type 'ServiceA'.`

```csharp
// ✗ 잘못된 코드: 순환 의존성 (A가 B에 의존, B가 A에 의존)
public class ServiceA
{
    public ServiceA(ServiceB b) { }  // A는 B가 필요
}

public class ServiceB
{
    public ServiceB(ServiceA a) { }  // B는 A가 필요 → 순환!
}

// DI 컨테이너가 A를 생성하려면 B가 필요하고,
// B를 생성하려면 A가 필요하므로 무한 루프 → 예외 발생
```

```csharp
// ○ 해결 방법 1: 중간 인터페이스로 의존성 분리
public interface IServiceAFeature
{
    void DoSomething();
}

public class ServiceA : IServiceAFeature
{
    private readonly ServiceB _b;
    public ServiceA(ServiceB b) { _b = b; }
    public void DoSomething() { /* ... */ }
}

public class ServiceB
{
    // ○ 구체 타입 대신 인터페이스에 의존
    private readonly IServiceAFeature _aFeature;
    public ServiceB(IServiceAFeature aFeature) { _aFeature = aFeature; }
}
```

```csharp
// ○ 해결 방법 2: Messenger를 사용한 느슨한 결합
// 직접 참조 대신 메시지를 통해 간접 통신
public class ServiceA
{
    public ServiceA()
    {
        WeakReferenceMessenger.Default.Register<RequestFromBMessage>(this, (r, m) =>
        {
            // B의 요청에 응답
            m.Reply("A의 응답");
        });
    }
}

public class ServiceB
{
    public void NeedDataFromA()
    {
        // A를 직접 참조하지 않고 메시지로 요청
        var response = WeakReferenceMessenger.Default.Send(new RequestFromBMessage());
    }
}
```

```csharp
// ○ 해결 방법 3: Lazy<T>로 지연 초기화
public class ServiceB
{
    // Lazy<T>로 감싸면 실제 사용 시점에 생성됨
    // → DI 컨테이너 빌드 시점에는 순환이 발생하지 않음
    private readonly Lazy<ServiceA> _a;

    public ServiceB(Lazy<ServiceA> a)
    {
        _a = a;
    }

    public void DoWork()
    {
        _a.Value.DoSomething();  // 이 시점에 ServiceA 생성
    }
}
```

---

## 5. 성능 관련 실수

### 실수 5-1: DataGrid에 대량 데이터 직접 바인딩 (가상화 안 함)

**증상**: 데이터가 수천 건 이상이면 DataGrid가 매우 느려지고 메모리를 많이 사용한다.

```xml
<!-- ✗ 잘못된 코드: 가상화를 비활성화하거나 방해하는 설정 -->
<DataGrid ItemsSource="{Binding Employees}"
          EnableRowVirtualization="False"
          EnableColumnVirtualization="False">
    <!-- 또는 ScrollViewer로 감싸면 가상화가 무효화됨 -->
</DataGrid>

<!-- ✗ 잘못된 코드: ScrollViewer로 DataGrid를 감싸면 가상화 무효화 -->
<ScrollViewer>
    <!-- DataGrid가 모든 행을 한번에 렌더링함 → 수만 행이면 엄청 느림! -->
    <DataGrid ItemsSource="{Binding Employees}"/>
</ScrollViewer>
```

```xml
<!-- ○ 올바른 코드: 가상화 활성화 (기본값이지만 명시적으로) -->
<DataGrid ItemsSource="{Binding Employees}"
          EnableRowVirtualization="True"
          EnableColumnVirtualization="True"
          VirtualizingPanel.IsVirtualizing="True"
          VirtualizingPanel.VirtualizationMode="Recycling"
          VirtualizingPanel.ScrollUnit="Pixel"
          MaxHeight="600">
    <!-- 가상화: 화면에 보이는 행만 렌더링 → 10만 건이어도 빠름! -->
</DataGrid>
```

```csharp
// ○ 대량 데이터를 위한 페이징 패턴
public partial class EmployeeListViewModel : ObservableObject
{
    private const int PageSize = 50;  // 한 페이지에 50건씩

    [ObservableProperty]
    private int _currentPage = 1;

    [ObservableProperty]
    private int _totalPages;

    public ObservableCollection<Employee> Employees { get; } = [];

    [RelayCommand]
    private async Task LoadPageAsync()
    {
        // DB에서 현재 페이지의 데이터만 조회
        var (employees, totalCount) = await _repository.GetPageAsync(
            CurrentPage, PageSize);

        TotalPages = (int)Math.Ceiling((double)totalCount / PageSize);

        Employees.Clear();
        foreach (var emp in employees)
            Employees.Add(emp);
    }

    [RelayCommand]
    private async Task NextPageAsync()
    {
        if (CurrentPage < TotalPages)
        {
            CurrentPage++;
            await LoadPageAsync();
        }
    }

    [RelayCommand]
    private async Task PreviousPageAsync()
    {
        if (CurrentPage > 1)
        {
            CurrentPage--;
            await LoadPageAsync();
        }
    }
}
```

---

### 실수 5-2: 불필요한 PropertyChanged 이벤트 발생

**증상**: UI가 불필요하게 자주 갱신되어 성능이 저하된다.

```csharp
// ✗ 잘못된 코드: 같은 값을 설정해도 PropertyChanged가 발생
public class ManualViewModel : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    private string _name = string.Empty;

    public string Name
    {
        get => _name;
        set
        {
            // ✗ 같은 값이어도 무조건 이벤트 발생
            _name = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Name)));
            // → 값이 안 바뀌어도 UI가 다시 렌더링됨 → 성능 낭비
        }
    }
}
```

```csharp
// ○ 올바른 코드 (수동 구현 시): 값이 실제로 변경된 경우에만 이벤트 발생
public class ManualViewModel : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    private string _name = string.Empty;

    public string Name
    {
        get => _name;
        set
        {
            // ○ 같은 값이면 무시 (SetProperty 내부에서 이렇게 동작)
            if (_name == value) return;
            _name = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Name)));
        }
    }
}

// ○ 올바른 코드 (CommunityToolkit.Mvvm): [ObservableProperty]는 자동으로 처리
public partial class ToolkitViewModel : ObservableObject
{
    // ○ [ObservableProperty]는 값이 같으면 자동으로 이벤트를 발생시키지 않음
    [ObservableProperty]
    private string _name = string.Empty;
    // 소스 제네레이터가 if (value == _name) return; 코드를 자동 생성
}
```

---

### 실수 5-3: 이미지 메모리 누수

**증상**: 이미지를 많이 표시하는 화면에서 메모리가 계속 증가한다. 앱이 수백 MB의 메모리를 사용하게 된다.

```xml
<!-- ✗ 잘못된 코드: 이미지를 원본 크기로 로드 -->
<Image Source="{Binding PhotoPath}"/>
<!-- 5000x3000 해상도의 사진이 원본 크기로 메모리에 로드됨
     → 한 장에 수십 MB → 목록에 100장이면 수 GB! -->
```

```csharp
// ✗ 잘못된 코드: BitmapImage를 생성하고 해제하지 않음
public BitmapImage LoadImage(string path)
{
    // ✗ 파일을 계속 잡고 있음 + 원본 크기로 로드
    var bitmap = new BitmapImage(new Uri(path));
    return bitmap;
}
```

```csharp
// ○ 올바른 코드: 적절한 크기로 디코딩하고 캐시 옵션 설정
public static BitmapImage LoadImageOptimized(string path, int maxWidth = 200)
{
    var bitmap = new BitmapImage();
    bitmap.BeginInit();

    // ○ 캐시 옵션: 메모리에 로드 후 파일 핸들 해제
    bitmap.CacheOption = BitmapCacheOption.OnLoad;

    // ○ 스트림으로 읽어서 파일 잠금 방지
    bitmap.UriSource = new Uri(path, UriKind.Absolute);

    // ○ 표시 크기에 맞게 디코딩 (원본 5000px → 200px로 축소)
    // → 메모리 사용량 대폭 감소
    bitmap.DecodePixelWidth = maxWidth;

    bitmap.EndInit();

    // ○ 객체를 동결하여 다른 스레드에서도 접근 가능하게 하고
    //    변경 불가능하게 만들어 성능 향상
    bitmap.Freeze();

    return bitmap;
}
```

```xml
<!-- ○ XAML에서 디코딩 크기 직접 지정하는 방법 -->
<Image Width="200" Height="150">
    <Image.Source>
        <BitmapImage UriSource="{Binding PhotoPath}"
                     DecodePixelWidth="200"
                     CacheOption="OnLoad"/>
    </Image.Source>
</Image>
```

```csharp
// ○ 목록에 이미지를 표시할 때의 ViewModel 패턴
public partial class EmployeeWithPhotoViewModel : ObservableObject
{
    [ObservableProperty]
    private BitmapImage? _thumbnail;

    /// <summary>
    /// 썸네일을 비동기적으로 로드 (UI 스레드 블로킹 방지)
    /// </summary>
    public async Task LoadThumbnailAsync(string photoPath)
    {
        if (!File.Exists(photoPath))
        {
            Thumbnail = null;
            return;
        }

        // Task.Run으로 파일 읽기를 백그라운드에서 실행
        Thumbnail = await Task.Run(() =>
        {
            var bitmap = new BitmapImage();
            bitmap.BeginInit();
            bitmap.CacheOption = BitmapCacheOption.OnLoad;
            bitmap.DecodePixelWidth = 80;  // 썸네일 크기
            bitmap.UriSource = new Uri(photoPath, UriKind.Absolute);
            bitmap.EndInit();
            bitmap.Freeze();  // ← Freeze() 필수! (다른 스레드에서 UI로 전달)
            return bitmap;
        });
    }
}
```

---

## 실수 총정리 치트시트

### 바인딩

| 실수 | 증상 | 해결 |
|------|------|------|
| DataContext 미설정 | 아무것도 표시 안 됨 | `DataContext = viewModel` 설정 |
| 프로퍼티명 오타 | 바인딩 실패 | Output 창에서 에러 확인 |
| INotifyPropertyChanged 미구현 | 초기값만 표시, 갱신 안 됨 | `ObservableObject` 상속 + `[ObservableProperty]` |
| 잘못된 바인딩 Mode | 입력이 ViewModel에 반영 안 됨 | `Mode=TwoWay` 명시 |
| List 대신 ObservableCollection | 항목 추가/삭제 시 UI 갱신 안 됨 | `ObservableCollection<T>` 사용 |

### MVVM

| 실수 | 증상 | 해결 |
|------|------|------|
| 코드 비하인드에 비즈니스 로직 | 테스트 불가, 유지보수 어려움 | ViewModel로 이동 |
| ViewModel에서 View 참조 | 강한 결합, 테스트 불가 | 바인딩 + 서비스 인터페이스 |
| ViewModel에서 MessageBox | UI 종속, 테스트 불가 | `IDialogService` 사용 |
| CanExecute 미연결 | 버튼 항상 활성화 | `[RelayCommand(CanExecute = ...)]` |

### 비동기

| 실수 | 증상 | 해결 |
|------|------|------|
| `async void` | 예외 미처리, 앱 크래시 | `async Task` 사용 |
| `.Result` / `.Wait()` | UI 멈춤, 데드락 | `await` 사용 |
| 백그라운드에서 UI 접근 | InvalidOperationException | `await` 후 처리 또는 `Dispatcher.Invoke` |
| ViewModel에서 `ConfigureAwait(false)` | UI 갱신 실패 | ViewModel에서는 생략 (기본 true) |

### DI

| 실수 | 증상 | 해결 |
|------|------|------|
| `new`로 직접 생성 | DI 이점 포기 | 생성자 주입 사용 |
| Singleton/Transient 혼동 | 상태 오염 또는 상태 유실 | 용도에 맞는 수명 선택 |
| 순환 의존성 | 앱 시작 시 예외 | 인터페이스 분리 또는 Messenger |

### 성능

| 실수 | 증상 | 해결 |
|------|------|------|
| DataGrid 가상화 비활성화 | 대량 데이터 시 느림 | 가상화 활성화 + 페이징 |
| 불필요한 PropertyChanged | UI 과도한 렌더링 | `[ObservableProperty]` (자동 중복 체크) |
| 이미지 메모리 누수 | 메모리 지속 증가 | `DecodePixelWidth` + `Freeze()` |
