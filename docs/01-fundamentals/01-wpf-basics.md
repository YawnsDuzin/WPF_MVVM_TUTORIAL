# WPF 기초 개념 — WinForms 개발자를 위한 가이드

> 이 문서는 WinForms나 Python GUI 개발 경험이 있는 개발자를 대상으로,
> WPF(Windows Presentation Foundation)의 핵심 개념을 처음부터 설명합니다.

---

## 목차

1. [WPF란 무엇인가](#1-wpf란-무엇인가)
2. [WinForms vs WPF 핵심 차이점](#2-winforms-vs-wpf-핵심-차이점)
3. [WPF의 핵심 아키텍처](#3-wpf의-핵심-아키텍처)
4. [WPF 프로젝트의 기본 파일 구조](#4-wpf-프로젝트의-기본-파일-구조)
5. [WinForms 코드를 WPF에서는 어떻게 하는가](#5-winforms-코드를-wpf에서는-어떻게-하는가)
6. [Hello World 예제](#6-hello-world-예제)

---

## 1. WPF란 무엇인가

**WPF(Windows Presentation Foundation)** 는 Microsoft가 .NET Framework 3.0(2006년)부터 제공한 데스크톱 UI 프레임워크입니다. WinForms의 후속 기술로, 다음과 같은 목표를 가지고 설계되었습니다:

- **선언적 UI**: XAML이라는 XML 기반 마크업 언어로 화면을 정의합니다.
- **해상도 독립적 렌더링**: 모니터 DPI에 상관없이 선명한 UI를 제공합니다.
- **풍부한 그래픽**: DirectX 기반 렌더링으로 애니메이션, 3D, 벡터 그래픽을 기본 지원합니다.
- **데이터 바인딩**: UI와 데이터를 자동으로 동기화하는 강력한 바인딩 시스템을 갖추고 있습니다.

### Python 개발자를 위한 비유

Python에서 Tkinter나 PyQt를 써봤다면, WPF는 PyQt의 QML과 비슷한 개념입니다.
UI를 코드가 아닌 **마크업(XAML)** 으로 정의하고, 로직은 C# 코드에서 처리합니다.

```
┌─────────────────────────────────┐
│  Python 세계        C# 세계      │
│                                 │
│  Tkinter  ────────  WinForms    │  ← 코드로 UI를 직접 구성
│  PyQt/QML ────────  WPF/XAML    │  ← 마크업으로 UI를 선언
└─────────────────────────────────┘
```

---

## 2. WinForms vs WPF 핵심 차이점

### 비교 테이블

| 비교 항목 | WinForms | WPF |
|---|---|---|
| **UI 정의 방식** | 이벤트 드리븐 (코드 중심) | 선언적 UI (XAML 마크업) |
| **렌더링 엔진** | GDI+ (CPU 기반) | DirectX (GPU 가속) |
| **데이터 표시** | 코드 비하인드에서 직접 대입 | 데이터 바인딩 (자동 동기화) |
| **좌표 시스템** | 픽셀(px) 기반 고정 크기 | 장치 독립적 단위 (DIP, 1/96인치) |
| **레이아웃** | 절대 좌표 배치 (Anchor/Dock) | 유연한 레이아웃 패널 (Grid, StackPanel 등) |
| **스타일링** | 컨트롤별 속성 개별 설정 | Style, Template으로 일괄 적용 |
| **디자인 패턴** | 이벤트 핸들러 (코드 비하인드) | MVVM 패턴 권장 |
| **컨트롤 커스터마이징** | 제한적 (Owner Draw) | ControlTemplate으로 완전 재정의 가능 |
| **데이터 그리드** | DataGridView | DataGrid (바인딩 기반) |
| **학습 곡선** | 낮음 (직관적) | 높음 (새로운 개념 다수) |

### 이벤트 드리븐 vs 선언적 UI

**WinForms (이벤트 드리븐)**:

UI를 코드로 조작합니다. "버튼을 클릭하면 → 텍스트박스에 값을 넣어라"처럼
모든 동작을 **명령형(imperative)** 으로 작성합니다.

```csharp
// WinForms: 버튼 클릭하면 라벨에 텍스트 설정
private void button1_Click(object sender, EventArgs e)
{
    label1.Text = "안녕하세요!";           // 직접 컨트롤을 참조해서 값 대입
    textBox1.BackColor = Color.Yellow;     // 직접 속성 변경
    listBox1.Items.Add("새 항목");          // 직접 아이템 추가
}
```

**WPF (선언적 UI)**:

UI의 **상태(state)** 를 정의하고, 데이터가 바뀌면 UI가 **자동으로** 갱신됩니다.
"이 텍스트블록은 Name 속성의 값을 보여준다"처럼 **선언적(declarative)** 으로 작성합니다.

```xml
<!-- WPF: XAML에서 바인딩을 선언 -->
<TextBlock Text="{Binding Name}" />           <!-- Name 속성이 바뀌면 자동 갱신 -->
<TextBox Background="{Binding BgColor}" />    <!-- BgColor 속성에 따라 자동 변경 -->
<ListBox ItemsSource="{Binding Items}" />     <!-- Items 컬렉션이 바뀌면 자동 반영 -->
```

### GDI+ vs DirectX 렌더링

```
┌─────────── WinForms 렌더링 ───────────┐
│                                        │
│  C# 코드 → GDI+ (CPU) → 화면 출력     │
│                                        │
│  - CPU가 모든 그리기를 담당             │
│  - 복잡한 그래픽에서 느림              │
│  - 해상도 변경 시 깨짐 발생            │
└────────────────────────────────────────┘

┌─────────── WPF 렌더링 ────────────────┐
│                                        │
│  XAML → MIL Core → DirectX (GPU) → 화면│
│                                        │
│  - GPU가 렌더링을 담당 (하드웨어 가속)  │
│  - 복잡한 애니메이션도 부드러움         │
│  - 벡터 기반이라 해상도에 무관          │
└────────────────────────────────────────┘
```

### 픽셀 기반 vs 해상도 독립적

WinForms에서 `button1.Size = new Size(100, 30)`이라고 하면 **100픽셀 x 30픽셀**입니다.
고해상도(4K) 모니터에서는 버튼이 아주 작게 보이게 됩니다.

WPF에서 `Width="100" Height="30"`이라고 하면 **100 DIP x 30 DIP**입니다.
DIP(Device Independent Pixel)는 96 DPI 기준의 논리적 단위이므로,
모니터의 DPI 설정에 따라 자동으로 크기가 조정됩니다.

```
96 DPI 모니터:   100 DIP = 100 물리적 픽셀
144 DPI 모니터:  100 DIP = 150 물리적 픽셀  (150% 배율)
192 DPI 모니터:  100 DIP = 200 물리적 픽셀  (200% 배율)
```

---

## 3. WPF의 핵심 아키텍처

### Logical Tree (논리적 트리)

Logical Tree는 XAML에 작성한 **논리적 계층 구조**를 그대로 반영합니다.
개발자가 직접 작성한 요소들의 부모-자식 관계입니다.

```xml
<!-- 이 XAML의 Logical Tree -->
<Window>                          <!-- 루트 -->
    <Grid>                        <!--   └─ Grid -->
        <StackPanel>              <!--       └─ StackPanel -->
            <TextBlock Text="이름:" />  <!--       ├─ TextBlock -->
            <TextBox />                 <!--       └─ TextBox -->
        </StackPanel>
    </Grid>
</Window>
```

```
Logical Tree:

Window
  └─ Grid
       └─ StackPanel
            ├─ TextBlock ("이름:")
            └─ TextBox
```

### Visual Tree (시각적 트리)

Visual Tree는 **실제 렌더링되는 모든 시각적 요소**를 포함합니다.
예를 들어 `Button` 하나도 내부적으로 `Border`, `ContentPresenter`, `TextBlock` 등
여러 시각적 요소로 구성됩니다.

```
Visual Tree (Button 하나의 내부 구조):

Button
  └─ ButtonChrome (테두리, 배경)
       └─ ContentPresenter
            └─ TextBlock ("클릭하세요")
```

### 왜 이 구분이 중요한가?

| 용도 | Logical Tree 사용 | Visual Tree 사용 |
|---|---|---|
| **리소스 탐색** | O (부모를 따라 올라가며 리소스 검색) | |
| **데이터 바인딩** | O (DataContext 상속) | |
| **이벤트 라우팅** | | O (터널링/버블링) |
| **렌더링** | | O (화면에 그릴 요소 결정) |
| **Hit Testing** | | O (마우스 클릭 위치 판단) |

WinForms에서는 `Controls` 컬렉션 하나만 있었지만,
WPF에서는 이 두 트리를 구분해서 이해해야 합니다.

---

## 4. WPF 프로젝트의 기본 파일 구조

Visual Studio에서 "WPF 앱(.NET)" 프로젝트를 생성하면 다음과 같은 파일들이 만들어집니다.

```
MyWpfApp/
├── MyWpfApp.csproj          ← 프로젝트 파일 (.NET SDK 스타일)
├── App.xaml                 ← 애플리케이션 전체 설정 (WinForms의 Program.cs 역할)
├── App.xaml.cs              ← App.xaml의 코드 비하인드
├── MainWindow.xaml          ← 메인 창 UI 정의 (WinForms의 Form1.Designer.cs 역할)
├── MainWindow.xaml.cs       ← 메인 창 코드 비하인드 (WinForms의 Form1.cs 역할)
└── AssemblyInfo.cs          ← 어셈블리 정보
```

### App.xaml — 애플리케이션 진입점

WinForms에서 `Program.cs`의 `Main()` 메서드가 애플리케이션 시작점이었다면,
WPF에서는 `App.xaml`이 그 역할을 합니다.

```xml
<!-- App.xaml -->
<Application x:Class="MyWpfApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             StartupUri="MainWindow.xaml">
    <!--
        StartupUri: 앱이 시작할 때 보여줄 첫 번째 창을 지정
        WinForms의 Application.Run(new Form1())과 같은 역할
    -->
    <Application.Resources>
        <!-- 앱 전체에서 사용할 공통 리소스 (스타일, 브러시 등)를 여기에 정의 -->
    </Application.Resources>
</Application>
```

```csharp
// App.xaml.cs — App.xaml의 코드 비하인드
namespace MyWpfApp
{
    public partial class App : Application
    {
        // 필요한 경우 앱 시작/종료 이벤트를 여기서 처리
        // 예: 전역 예외 처리, 로깅 초기화 등
    }
}
```

### MainWindow.xaml — 메인 창

```xml
<!-- MainWindow.xaml -->
<Window x:Class="MyWpfApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="내 첫 WPF 앱" Height="350" Width="500">
    <!--
        Window: WinForms의 Form에 해당
        Title: WinForms의 Form.Text와 같은 역할
        Height/Width: 창의 크기 (DIP 단위)
    -->
    <Grid>
        <!-- 여기에 UI 요소를 배치 -->
    </Grid>
</Window>
```

```csharp
// MainWindow.xaml.cs — MainWindow.xaml의 코드 비하인드
namespace MyWpfApp
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
            // WinForms의 InitializeComponent()와 동일한 역할
            // XAML에서 정의한 UI를 실제로 생성하고 초기화
        }
    }
}
```

### WinForms와 파일 구조 비교

| 역할 | WinForms | WPF |
|---|---|---|
| 앱 시작점 | `Program.cs` | `App.xaml` |
| 앱 설정/리소스 | `Program.cs` + 설정 파일 | `App.xaml` + `App.xaml.cs` |
| 폼/창 UI 디자인 | `Form1.Designer.cs` (자동 생성 코드) | `MainWindow.xaml` (XAML 마크업) |
| 폼/창 로직 | `Form1.cs` (이벤트 핸들러) | `MainWindow.xaml.cs` (코드 비하인드) |

### .csproj 파일 (프로젝트 파일)

```xml
<!-- MyWpfApp.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <Nullable>enable</Nullable>
    <UseWPF>true</UseWPF>          <!-- WPF 프로젝트임을 명시 -->
  </PropertyGroup>
</Project>
```

---

## 5. WinForms 코드를 WPF에서는 어떻게 하는가

WinForms 개발자가 가장 먼저 궁금해하는 것:
"**`textBox1.Text = "Hello"` 같은 코드를 WPF에서는 어떻게 쓰지?**"

답은 **두 가지 방법**이 있습니다.

### 방법 1: 코드 비하인드 (WinForms 스타일 — 비권장)

WPF에서도 WinForms처럼 컨트롤에 이름을 주고 직접 접근할 수 있습니다.
하지만 이 방식은 **MVVM 패턴에서는 사용하지 않습니다.**

```xml
<!-- MainWindow.xaml -->
<Window x:Class="MyWpfApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="코드 비하인드 예제" Height="200" Width="400">
    <StackPanel Margin="20">
        <!-- x:Name으로 이름을 붙이면 코드 비하인드에서 참조 가능 -->
        <!-- WinForms에서 컨트롤의 Name 속성을 설정하는 것과 동일 -->
        <TextBox x:Name="txtInput" Margin="0,0,0,10" />
        <Button Content="인사하기" Click="Button_Click" />
        <TextBlock x:Name="txtOutput" Margin="0,10,0,0" FontSize="16" />
    </StackPanel>
</Window>
```

```csharp
// MainWindow.xaml.cs
namespace MyWpfApp
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }

        // WinForms의 button1_Click과 완전히 같은 방식!
        private void Button_Click(object sender, RoutedEventArgs e)
        {
            // WinForms: label1.Text = "안녕하세요, " + textBox1.Text;
            // WPF 코드 비하인드: 거의 동일
            txtOutput.Text = "안녕하세요, " + txtInput.Text + "!";
        }
    }
}
```

### 방법 2: 데이터 바인딩 (WPF 권장 방식 — MVVM)

UI 요소가 **데이터 객체의 속성**에 자동으로 연결됩니다.
데이터가 바뀌면 UI가 자동 갱신되고, UI에서 입력하면 데이터도 자동 갱신됩니다.

```xml
<!-- MainWindow.xaml -->
<Window x:Class="MyWpfApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="데이터 바인딩 예제" Height="200" Width="400">
    <StackPanel Margin="20">
        <!-- x:Name이 없습니다! 대신 {Binding ...}으로 데이터에 연결 -->
        <TextBox Text="{Binding InputName, UpdateSourceTrigger=PropertyChanged}"
                 Margin="0,0,0,10" />
        <Button Content="인사하기" Command="{Binding GreetCommand}" />
        <TextBlock Text="{Binding GreetingMessage}" Margin="0,10,0,0" FontSize="16" />
    </StackPanel>
</Window>
```

```csharp
// MainWindow.xaml.cs — 바인딩 방식에서는 매우 깔끔!
namespace MyWpfApp
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
            // DataContext를 설정하면 XAML의 {Binding ...}이 이 객체에서 속성을 찾음
            DataContext = new MainViewModel();
        }
    }
}
```

```csharp
// MainViewModel.cs — UI 로직을 담당하는 새로운 클래스
// (이 코드는 나중에 MVVM 패턴 문서에서 자세히 설명합니다)
public class MainViewModel : INotifyPropertyChanged
{
    private string _inputName = "";
    private string _greetingMessage = "";

    // UI의 TextBox와 연결된 속성
    public string InputName
    {
        get => _inputName;
        set
        {
            _inputName = value;
            OnPropertyChanged(nameof(InputName));  // "값이 바뀌었어!" 알림
        }
    }

    // UI의 TextBlock과 연결된 속성
    public string GreetingMessage
    {
        get => _greetingMessage;
        set
        {
            _greetingMessage = value;
            OnPropertyChanged(nameof(GreetingMessage));
        }
    }

    // UI의 Button과 연결된 커맨드
    public ICommand GreetCommand { get; }

    public MainViewModel()
    {
        GreetCommand = new RelayCommand(() =>
        {
            GreetingMessage = $"안녕하세요, {InputName}!";
        });
    }

    // INotifyPropertyChanged 구현
    public event PropertyChangedEventHandler? PropertyChanged;
    protected void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

### 두 방식의 비교

```
┌─────────── 코드 비하인드 방식 ──────────────────────────────┐
│                                                              │
│  [XAML]  ──x:Name──→  [코드 비하인드]                        │
│  txtOutput.Text = "안녕하세요";  ← 직접 접근                 │
│                                                              │
│  장점: 간단하고 직관적                                       │
│  단점: UI와 로직이 섞임, 테스트 어려움                       │
└──────────────────────────────────────────────────────────────┘

┌─────────── 데이터 바인딩 방식 ──────────────────────────────┐
│                                                              │
│  [XAML]  ──{Binding}──→  [ViewModel]                         │
│  GreetingMessage = "안녕하세요";  ← 속성 변경만 하면 자동 반영│
│                                                              │
│  장점: UI와 로직 분리, 테스트 가능, 재사용 가능              │
│  단점: 초기 학습 곡선이 있음                                 │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. Hello World 예제

이 섹션에서는 WinForms 개발자에게 익숙한 **코드 비하인드 방식**으로
간단한 Hello World 앱을 만듭니다.
(이후 문서에서 이 코드를 점진적으로 MVVM 패턴으로 개선합니다.)

### 프로젝트 생성

1. Visual Studio에서 **"WPF 응용 프로그램 (.NET)"** 템플릿 선택
2. 프로젝트 이름: `HelloWpfApp`
3. 대상 프레임워크: `.NET 8.0` (또는 최신 LTS)

### MainWindow.xaml

```xml
<Window x:Class="HelloWpfApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Hello WPF - WinForms 개발자를 위한 첫 번째 앱"
        Height="300" Width="450"
        WindowStartupLocation="CenterScreen">
    <!--
        WindowStartupLocation="CenterScreen"
        → WinForms의 StartPosition = FormStartPosition.CenterScreen 과 동일
    -->

    <!-- Grid: WPF에서 가장 많이 쓰는 레이아웃 패널 -->
    <Grid Margin="20">
        <!-- 행(Row) 3개 정의 -->
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />   <!-- 첫 번째 행: 내용에 맞게 -->
            <RowDefinition Height="Auto" />   <!-- 두 번째 행: 내용에 맞게 -->
            <RowDefinition Height="*" />      <!-- 세 번째 행: 나머지 공간 전부 -->
        </Grid.RowDefinitions>

        <!-- 열(Column) 2개 정의 -->
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto" />  <!-- 첫 번째 열: 라벨 너비만큼 -->
            <ColumnDefinition Width="*" />     <!-- 두 번째 열: 나머지 공간 전부 -->
        </Grid.ColumnDefinitions>

        <!-- 1행: 이름 입력 -->
        <TextBlock Grid.Row="0" Grid.Column="0"
                   Text="이름:"
                   VerticalAlignment="Center"
                   Margin="0,0,10,10"
                   FontSize="14" />
        <!--
            TextBlock: WinForms의 Label에 해당
            Grid.Row, Grid.Column: Grid 안에서의 위치 지정
            VerticalAlignment: 세로 정렬 (Top, Center, Bottom, Stretch)
        -->

        <TextBox Grid.Row="0" Grid.Column="1"
                 x:Name="txtName"
                 Margin="0,0,0,10"
                 FontSize="14"
                 Padding="5,3" />
        <!--
            TextBox: WinForms의 TextBox와 동일
            Padding: 텍스트와 테두리 사이의 내부 여백
        -->

        <!-- 2행: 버튼 -->
        <Button Grid.Row="1" Grid.Column="0" Grid.ColumnSpan="2"
                Content="인사하기"
                x:Name="btnGreet"
                Click="BtnGreet_Click"
                Height="35"
                FontSize="14"
                Margin="0,0,0,10" />
        <!--
            Content: WinForms의 Button.Text에 해당
            (WPF Button은 텍스트뿐 아니라 어떤 요소든 Content로 넣을 수 있음!)
            Grid.ColumnSpan="2": 2개 열에 걸쳐서 배치
        -->

        <!-- 3행: 결과 표시 -->
        <TextBlock Grid.Row="2" Grid.Column="0" Grid.ColumnSpan="2"
                   x:Name="txtResult"
                   FontSize="20"
                   Foreground="DarkBlue"
                   HorizontalAlignment="Center"
                   VerticalAlignment="Center" />
    </Grid>
</Window>
```

### MainWindow.xaml.cs

```csharp
using System.Windows;

namespace HelloWpfApp
{
    /// <summary>
    /// MainWindow의 코드 비하인드
    /// WinForms의 Form1.cs와 같은 역할
    /// </summary>
    public partial class MainWindow : Window
    {
        // 인사 횟수를 추적하는 카운터
        private int _greetCount = 0;

        public MainWindow()
        {
            InitializeComponent();

            // 창이 로드된 후 실행할 코드
            // WinForms의 Form1_Load 이벤트와 유사
            this.Loaded += MainWindow_Loaded;
        }

        private void MainWindow_Loaded(object sender, RoutedEventArgs e)
        {
            // 앱 시작 시 텍스트박스에 포커스 설정
            // WinForms: txtName.Focus();
            txtName.Focus();
        }

        private void BtnGreet_Click(object sender, RoutedEventArgs e)
        {
            // 입력 검증 — WinForms에서 하던 것과 동일
            if (string.IsNullOrWhiteSpace(txtName.Text))
            {
                // WinForms: MessageBox.Show("이름을 입력해주세요.");
                // WPF: MessageBox.Show()는 동일하게 사용 가능!
                MessageBox.Show("이름을 입력해주세요.",
                               "입력 오류",
                               MessageBoxButton.OK,
                               MessageBoxImage.Warning);
                txtName.Focus();
                return;
            }

            _greetCount++;

            // 결과 텍스트 설정
            // WinForms: label1.Text = "...";
            // WPF: TextBlock.Text = "..."; (동일!)
            txtResult.Text = $"안녕하세요, {txtName.Text}님! (인사 {_greetCount}회)";

            // 텍스트 선택 및 포커스
            txtName.SelectAll();
            txtName.Focus();
        }
    }
}
```

### 실행 결과

```
┌──────────────────────────────────────────────┐
│  Hello WPF - WinForms 개발자를 위한 첫 번째 앱  │
├──────────────────────────────────────────────┤
│                                              │
│  이름:  [홍길동                        ]     │
│                                              │
│  [            인사하기              ]        │
│                                              │
│        안녕하세요, 홍길동님! (인사 1회)        │
│                                              │
└──────────────────────────────────────────────┘
```

### 이 예제에서 배울 점

1. **XAML에서 UI를 선언적으로 정의**합니다 (WinForms의 Designer 코드 대신)
2. **Grid로 레이아웃**을 구성합니다 (WinForms의 절대 좌표 대신)
3. **코드 비하인드에서 이벤트를 처리**합니다 (WinForms와 동일한 패턴)
4. `Content`(Button), `Text`(TextBlock/TextBox) 등 **속성 이름이 WinForms과 약간 다릅니다**

### 다음 단계 예고

이 코드의 문제점:
- `txtName`, `txtResult` 같은 **컨트롤을 이름으로 직접 참조**
- **UI와 로직이 코드 비하인드에 섞여** 있음
- **단위 테스트 불가능** (Window 객체 없이는 테스트할 수 없음)

다음 문서들에서 이 코드를 점차 개선합니다:
1. **02-xaml-basics.md** → XAML 문법을 더 자세히 배웁니다
2. **03-data-binding.md** → 데이터 바인딩으로 코드 비하인드를 제거합니다
3. **04-mvvm-pattern.md** → MVVM 패턴으로 완전히 리팩토링합니다

---

> **핵심 요약**
>
> | WinForms에서 하던 것 | WPF에서는 이렇게 |
> |---|---|
> | `textBox1.Text = "값"` | `{Binding PropertyName}` 또는 `x:Name` 참조 |
> | `Form1.Designer.cs` | `MainWindow.xaml` (XAML 마크업) |
> | `Application.Run(new Form1())` | `App.xaml`의 `StartupUri` |
> | `label1.ForeColor = Color.Blue` | `Foreground="Blue"` (XAML 속성) |
> | `Anchor`, `Dock` | `Grid`, `StackPanel` 등 레이아웃 패널 |
> | `button1_Click` 이벤트 핸들러 | `Command` 바인딩 (MVVM) |
