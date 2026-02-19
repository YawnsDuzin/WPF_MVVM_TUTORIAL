# 프로젝트 생성 및 구조

> **목표**: dotnet CLI를 사용하여 WPF 프로젝트를 생성하고, MVVM 패턴에 맞는 폴더 구조를 잡으며, 필요한 모든 NuGet 패키지를 설치합니다.

---

## 목차

1. [dotnet CLI로 WPF 프로젝트 생성](#1-dotnet-cli로-wpf-프로젝트-생성)
2. [솔루션 구조 권장 (폴더별 역할)](#2-솔루션-구조-권장-폴더별-역할)
3. [.csproj 파일 설정](#3-csproj-파일-설정)
4. [dotnet CLI로 NuGet 패키지 추가하기](#4-dotnet-cli로-nuget-패키지-추가하기)
5. [WinForms 프로젝트와의 파일 구조 차이점](#5-winforms-프로젝트와의-파일-구조-차이점)

---

## 1. dotnet CLI로 WPF 프로젝트 생성

### 1.1 프로젝트 생성 명령어

터미널(명령 프롬프트, PowerShell, 또는 Linux/Mac의 터미널)을 열고 다음 명령어를 실행합니다.

```bash
# WPF 프로젝트 생성
# -n : 프로젝트 이름
# -f : 대상 프레임워크 (.NET 10.0)
dotnet new wpf -n MyApp -f net10.0
```

이 명령어를 실행하면 다음과 같은 기본 파일들이 생성됩니다:

```
MyApp/
├── App.xaml           # 애플리케이션 시작점 (XAML)
├── App.xaml.cs        # 애플리케이션 코드 비하인드
├── AssemblyInfo.cs    # 어셈블리 정보
├── MainWindow.xaml    # 메인 윈도우 (XAML)
├── MainWindow.xaml.cs # 메인 윈도우 코드 비하인드
└── MyApp.csproj       # 프로젝트 설정 파일
```

### 1.2 솔루션 파일 생성 (선택 사항)

프로젝트가 여러 개이거나 Visual Studio에서 관리하려면 솔루션 파일을 만드는 것이 좋습니다.

```bash
# 솔루션 파일 생성
dotnet new sln -n MyApp

# 프로젝트를 솔루션에 추가
dotnet sln MyApp.sln add MyApp/MyApp.csproj
```

### 1.3 프로젝트가 제대로 생성되었는지 확인

```bash
# 프로젝트 폴더로 이동
cd MyApp

# 빌드 확인 (에러 없이 성공해야 함)
dotnet build

# 실행 확인 (빈 윈도우가 뜨면 성공)
dotnet run
```

> **WinForms 개발자 참고**: WinForms에서는 `dotnet new winforms`를 사용하지만, WPF에서는 `dotnet new wpf`를 사용합니다. 프로젝트 생성 방식은 거의 동일합니다.

> **Python 개발자 참고**: Python에서 `pip install`로 패키지를 설치하고 `requirements.txt`로 관리하듯이, .NET에서는 `dotnet add package`로 패키지를 설치하고 `.csproj` 파일이 `requirements.txt` 역할을 합니다.

---

## 2. 솔루션 구조 권장 (폴더별 역할)

기본 생성된 프로젝트는 폴더 구조가 없습니다. MVVM 패턴에 맞게 폴더를 직접 만들어야 합니다.

### 2.1 권장 폴더 구조

```
MyApp/
├── Models/                    # 데이터 모델 (DB 테이블과 매핑되는 클래스)
│   ├── Employee.cs
│   └── Department.cs
│
├── ViewModels/                # 뷰모델 (UI 로직, 데이터 바인딩 대상)
│   ├── MainViewModel.cs
│   ├── EmployeeListViewModel.cs
│   └── EmployeeDetailViewModel.cs
│
├── Views/                     # XAML 뷰 (사용자에게 보이는 화면)
│   ├── MainWindow.xaml
│   ├── MainWindow.xaml.cs
│   ├── EmployeeListView.xaml
│   ├── EmployeeListView.xaml.cs
│   ├── EmployeeDetailView.xaml
│   └── EmployeeDetailView.xaml.cs
│
├── Services/                  # 비즈니스 로직, 외부 시스템 연동
│   ├── Interfaces/            # 서비스 인터페이스 (계약 정의)
│   │   ├── IEmployeeService.cs
│   │   ├── IDatabaseService.cs
│   │   └── IMqttService.cs
│   └── Implementations/      # 서비스 구현체
│       ├── EmployeeService.cs
│       ├── DatabaseService.cs
│       └── MqttService.cs
│
├── Helpers/                   # 유틸리티 클래스
│   └── DateTimeHelper.cs
│
├── Converters/                # XAML 값 변환기 (IValueConverter 구현)
│   ├── BoolToVisibilityConverter.cs
│   └── DateFormatConverter.cs
│
├── Resources/                 # 리소스 파일 (스타일, 이미지, 아이콘 등)
│   ├── Styles/
│   │   └── CommonStyles.xaml
│   └── Images/
│       └── logo.png
│
├── App.xaml                   # 앱 시작점 (리소스 딕셔너리 정의)
├── App.xaml.cs                # 앱 초기화 로직 (DI 컨테이너 설정)
├── appsettings.json           # 설정 파일 (DB 연결 문자열, 앱 설정 등)
├── AssemblyInfo.cs            # 어셈블리 메타데이터
└── MyApp.csproj               # 프로젝트 파일 (NuGet 패키지, 빌드 옵션)
```

### 2.2 폴더를 CLI로 한 번에 만들기

```bash
# 프로젝트 루트(MyApp/)에서 실행
mkdir -p Models ViewModels Views Services/Interfaces Services/Implementations Helpers Converters Resources/Styles Resources/Images
```

> **Windows PowerShell이라면:**
> ```powershell
> New-Item -ItemType Directory -Path Models, ViewModels, Views, Services\Interfaces, Services\Implementations, Helpers, Converters, Resources\Styles, Resources\Images
> ```

### 2.3 각 폴더의 역할 상세 설명

| 폴더 | 역할 | 예시 |
|------|------|------|
| `Models/` | 데이터 구조를 정의하는 순수 클래스. DB 테이블의 행 하나를 C# 클래스로 표현 | `Employee`, `Department` |
| `ViewModels/` | View와 Model 사이의 중개자. UI에 표시할 데이터와 명령(Command)을 정의 | `EmployeeListViewModel` |
| `Views/` | 사용자에게 보이는 화면(XAML). 코드 비하인드는 최소한으로 유지 | `EmployeeListView.xaml` |
| `Services/Interfaces/` | 서비스의 "계약"만 정의 (구현 내용 없음). DI에서 이 인터페이스를 기반으로 주입 | `IEmployeeService` |
| `Services/Implementations/` | 인터페이스의 실제 구현. DB 접근, API 호출 등의 로직 포함 | `EmployeeService` |
| `Helpers/` | 여러 곳에서 재사용하는 유틸리티 클래스/메서드 | `DateTimeHelper` |
| `Converters/` | XAML에서 값을 변환하는 데 사용 (예: bool → Visibility) | `BoolToVisibilityConverter` |
| `Resources/` | 공용 스타일, 이미지, 아이콘 등의 리소스 파일 | `CommonStyles.xaml`, `logo.png` |

### 2.4 왜 이런 구조를 사용하는가?

```
[핵심 원칙: 관심사의 분리 (Separation of Concerns)]

┌─────────────┐     바인딩      ┌──────────────┐     호출      ┌─────────────┐
│    Views     │ ◄────────────► │  ViewModels  │ ────────────► │  Services   │
│  (화면 표시)  │               │ (UI 로직)     │               │ (비즈니스    │
│  XAML 파일   │               │ C# 클래스     │               │  로직/DB)    │
└─────────────┘               └──────────────┘               └──────┬──────┘
                                                                     │
                                                                     ▼
                                                              ┌─────────────┐
                                                              │   Models    │
                                                              │ (데이터 구조) │
                                                              └─────────────┘
```

**WinForms와의 비교:**
- WinForms에서는 `Form1.cs` 하나에 UI 코드, 데이터 처리, DB 접근이 모두 섞여 있는 경우가 많습니다.
- WPF + MVVM에서는 각 역할을 명확히 분리하여, 코드 재사용성과 테스트 용이성을 높입니다.

---

## 3. .csproj 파일 설정

`.csproj` 파일은 프로젝트의 핵심 설정 파일입니다. Python의 `pyproject.toml`, WinForms의 `.csproj`와 동일한 역할을 합니다.

### 3.1 전체 .csproj 파일

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <!-- 출력 타입: WPF 윈도우 앱 -->
    <OutputType>WinExe</OutputType>

    <!-- 대상 프레임워크: .NET 10.0 -->
    <TargetFramework>net10.0-windows</TargetFramework>

    <!-- Nullable 참조 타입 활성화 (null 안전성 강화) -->
    <Nullable>enable</Nullable>

    <!-- 글로벌 using 자동 추가 (System, System.Linq 등) -->
    <ImplicitUsings>enable</ImplicitUsings>

    <!-- WPF 사용 활성화 -->
    <UseWPF>true</UseWPF>
  </PropertyGroup>

  <ItemGroup>
    <!-- ===== MVVM 프레임워크 ===== -->
    <!-- CommunityToolkit.Mvvm: 소스 제네레이터 기반 MVVM 도우미 -->
    <!-- [ObservableProperty], [RelayCommand] 등의 어트리뷰트 제공 -->
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.4.0" />

    <!-- ===== 데이터베이스 ===== -->
    <!-- Dapper: 경량 마이크로 ORM (SQL 직접 작성, 빠른 매핑) -->
    <PackageReference Include="Dapper" Version="2.1.44" />

    <!-- Npgsql: PostgreSQL용 ADO.NET 드라이버 -->
    <PackageReference Include="Npgsql" Version="9.0.3" />

    <!-- ===== HTTP/네트워크 복원력 ===== -->
    <!-- Polly: 재시도, 서킷브레이커, 타임아웃 등의 복원력 패턴 -->
    <PackageReference Include="Polly" Version="8.5.2" />

    <!-- ===== MQTT 메시징 ===== -->
    <!-- MQTTnet: MQTT v5 프로토콜 클라이언트/서버 -->
    <PackageReference Include="MQTTnet" Version="5.0.1" />

    <!-- ===== AWS 클라우드 ===== -->
    <!-- AWSSDK.S3: Amazon S3 파일 저장소 SDK -->
    <PackageReference Include="AWSSDK.S3" Version="3.7.405.7" />

    <!-- ===== 로깅 ===== -->
    <!-- Serilog: 구조적 로깅 라이브러리 (메인 패키지) -->
    <PackageReference Include="Serilog" Version="4.2.0" />

    <!-- Serilog 콘솔 출력 (개발 시 디버깅용) -->
    <PackageReference Include="Serilog.Sinks.Console" Version="6.0.0" />

    <!-- Serilog 파일 출력 (운영 환경 로그 저장) -->
    <PackageReference Include="Serilog.Sinks.File" Version="6.0.0" />

    <!-- ===== 의존성 주입 (DI) ===== -->
    <!-- Microsoft.Extensions.DependencyInjection: DI 컨테이너 -->
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="10.0.0" />

    <!-- ===== 설정 관리 ===== -->
    <!-- Microsoft.Extensions.Configuration: 설정 시스템 기본 패키지 -->
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="10.0.0" />

    <!-- JSON 파일에서 설정 읽기 (appsettings.json) -->
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="10.0.0" />

    <!-- ===== 시스템 트레이 아이콘 ===== -->
    <!-- Hardcodet.NotifyIcon.Wpf: WPF용 시스템 트레이 아이콘 라이브러리 -->
    <PackageReference Include="Hardcodet.NotifyIcon.Wpf" Version="4.0.1" />

    <!-- ===== JSON 직렬화 ===== -->
    <!-- System.Text.Json: 고성능 JSON 직렬화/역직렬화 -->
    <PackageReference Include="System.Text.Json" Version="10.0.0" />
  </ItemGroup>

  <ItemGroup>
    <!-- appsettings.json을 빌드 출력 디렉토리에 복사 -->
    <!-- PreserveNewest: 파일이 변경된 경우에만 복사 -->
    <None Update="appsettings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>

    <!-- 개발 환경 전용 설정 파일 (있는 경우) -->
    <None Update="appsettings.Development.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
      <!-- 파일이 없어도 빌드 에러가 나지 않도록 Condition 추가 가능 -->
    </None>
  </ItemGroup>

</Project>
```

### 3.2 주요 설정 항목 설명

| 항목 | 값 | 설명 |
|------|-----|------|
| `OutputType` | `WinExe` | Windows 실행 파일 (.exe). `Exe`로 하면 콘솔 창이 같이 뜸 |
| `TargetFramework` | `net10.0-windows` | .NET 10.0, Windows 전용 API 사용 가능 |
| `Nullable` | `enable` | null 참조 경고 활성화. 런타임 NullReferenceException 예방 |
| `ImplicitUsings` | `enable` | `System`, `System.Linq`, `System.Collections.Generic` 등 자동 using |
| `UseWPF` | `true` | WPF 관련 빌드 설정 활성화 (XAML 컴파일러 등) |

### 3.3 NuGet 패키지 요약 표

| 패키지 | 버전 | 용도 | Python 비유 |
|--------|------|------|-------------|
| CommunityToolkit.Mvvm | 8.4.0 | MVVM 도우미 (소스 제네레이터) | - |
| Dapper | 2.1.44 | 경량 ORM (SQL → 객체 매핑) | `records` 라이브러리 |
| Npgsql | 9.0.3 | PostgreSQL 드라이버 | `psycopg2` |
| Polly | 8.5.2 | 재시도/서킷브레이커 | `tenacity` |
| MQTTnet | 5.0.1 | MQTT 클라이언트 | `paho-mqtt` |
| AWSSDK.S3 | 3.7.405.7 | AWS S3 접근 | `boto3` |
| Serilog | 4.2.0 | 구조적 로깅 | `loguru` / `structlog` |
| Serilog.Sinks.Console | 6.0.0 | 로그 → 콘솔 출력 | (loguru 기본 기능) |
| Serilog.Sinks.File | 6.0.0 | 로그 → 파일 출력 | (loguru 기본 기능) |
| Microsoft.Extensions.DependencyInjection | 10.0.0 | DI 컨테이너 | `dependency-injector` |
| Microsoft.Extensions.Configuration | 10.0.0 | 설정 관리 | `python-dotenv` |
| Microsoft.Extensions.Configuration.Json | 10.0.0 | JSON 설정 읽기 | `json.load()` |
| Hardcodet.NotifyIcon.Wpf | 4.0.1 | 시스템 트레이 아이콘 | `pystray` |
| System.Text.Json | 10.0.0 | JSON 직렬화 | `json` (내장) |

---

## 4. dotnet CLI로 NuGet 패키지 추가하기

`.csproj` 파일을 직접 편집해도 되지만, CLI로 패키지를 추가하는 방법도 알아두면 편리합니다.

### 4.1 패키지 추가 명령어

```bash
# 기본 형식
# dotnet add package [패키지이름] --version [버전]

# ===== MVVM =====
dotnet add package CommunityToolkit.Mvvm --version 8.4.0

# ===== 데이터베이스 =====
dotnet add package Dapper --version 2.1.44
dotnet add package Npgsql --version 9.0.3

# ===== HTTP 복원력 =====
dotnet add package Polly --version 8.5.2

# ===== MQTT =====
dotnet add package MQTTnet --version 5.0.1

# ===== AWS =====
dotnet add package AWSSDK.S3 --version 3.7.405.7

# ===== 로깅 =====
dotnet add package Serilog --version 4.2.0
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File

# ===== DI =====
dotnet add package Microsoft.Extensions.DependencyInjection --version 10.0.0

# ===== 설정 =====
dotnet add package Microsoft.Extensions.Configuration --version 10.0.0
dotnet add package Microsoft.Extensions.Configuration.Json

# ===== 트레이 아이콘 =====
dotnet add package Hardcodet.NotifyIcon.Wpf --version 4.0.1

# ===== JSON =====
dotnet add package System.Text.Json --version 10.0.0
```

### 4.2 유용한 패키지 관리 명령어

```bash
# 설치된 패키지 목록 확인
dotnet list package

# 특정 패키지 제거
dotnet remove package [패키지이름]

# 패키지 업데이트 확인 (최신 버전 있는지)
dotnet list package --outdated

# 패키지 복원 (다른 PC에서 프로젝트를 받았을 때)
# Python의 pip install -r requirements.txt 와 비슷
dotnet restore
```

> **Python 개발자 참고**: `dotnet restore`는 Python의 `pip install -r requirements.txt`와 같은 역할입니다. `.csproj` 파일에 정의된 모든 패키지를 자동으로 다운로드합니다. Git 저장소에서 프로젝트를 클론한 후 처음 할 일이 `dotnet restore`입니다 (사실 `dotnet build` 시 자동으로 실행됩니다).

---

## 5. WinForms 프로젝트와의 파일 구조 차이점

### 5.1 전체 비교 표

| 항목 | WinForms | WPF (MVVM) |
|------|----------|------------|
| **UI 정의** | `Form1.Designer.cs` (C# 코드로 컨트롤 배치) | `MainWindow.xaml` (XAML 마크업으로 선언적 정의) |
| **UI 로직** | `Form1.cs` (이벤트 핸들러에 직접 코드 작성) | `MainViewModel.cs` (ViewModel에 로직 분리) |
| **데이터 표시** | `textBox1.Text = "값";` (직접 대입) | `{Binding Name}` (데이터 바인딩 자동 연결) |
| **설정 파일** | `Settings.settings` (IDE 전용 도구) | `appsettings.json` (표준 JSON, 범용적) |
| **리소스** | `Resources.resx` (바이너리 리소스) | `Resources/` 폴더 + `ResourceDictionary` |
| **프로젝트 파일** | `.csproj` (동일) | `.csproj` (동일하지만 `UseWPF` 추가) |
| **코드 구조** | 폴더 없이 Form 파일들이 나열 | `Models/`, `ViewModels/`, `Views/`, `Services/` 등으로 분리 |

### 5.2 파일별 상세 비교

#### Form1.cs (WinForms) vs View + ViewModel (WPF)

**WinForms 방식 - 모든 것이 하나의 파일에:**

```csharp
// WinForms: Form1.cs
// UI 이벤트 핸들러에 모든 로직이 섞여 있음
public partial class Form1 : Form
{
    public Form1()
    {
        InitializeComponent();
    }

    // 버튼 클릭 이벤트에 DB 접근 코드가 직접 들어감
    private void btnSave_Click(object sender, EventArgs e)
    {
        // UI에서 값 가져오기
        string name = txtName.Text;
        string email = txtEmail.Text;

        // DB 저장 (UI 코드에 DB 로직이 섞임!)
        using var conn = new NpgsqlConnection(connectionString);
        conn.Open();
        conn.Execute("INSERT INTO employees (name, email) VALUES (@Name, @Email)",
            new { Name = name, Email = email });

        // UI 업데이트
        MessageBox.Show("저장 완료!");
        txtName.Text = "";
        txtEmail.Text = "";
    }
}
```

**WPF + MVVM 방식 - 역할이 명확히 분리됨:**

```xml
<!-- WPF: Views/EmployeeDetailView.xaml -->
<!-- UI는 XAML로 선언적으로 정의 -->
<UserControl x:Class="MyApp.Views.EmployeeDetailView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <StackPanel Margin="20">
        <!-- 텍스트박스가 ViewModel의 Name 속성에 자동으로 연결됨 -->
        <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />
        <TextBox Text="{Binding Email, UpdateSourceTrigger=PropertyChanged}" />

        <!-- 버튼이 ViewModel의 SaveCommand에 자동으로 연결됨 -->
        <Button Content="저장" Command="{Binding SaveCommand}" />
    </StackPanel>
</UserControl>
```

```csharp
// WPF: ViewModels/EmployeeDetailViewModel.cs
// UI 로직만 담당 (DB 접근은 Service에 위임)
public partial class EmployeeDetailViewModel : ObservableObject
{
    // 서비스 주입 (DB 접근은 서비스가 담당)
    private readonly IEmployeeService _employeeService;

    public EmployeeDetailViewModel(IEmployeeService employeeService)
    {
        _employeeService = employeeService;
    }

    // 바인딩 속성 (UI의 TextBox와 자동 연결)
    [ObservableProperty]
    private string _name = string.Empty;

    [ObservableProperty]
    private string _email = string.Empty;

    // 커맨드 (UI의 Button과 자동 연결)
    [RelayCommand]
    private async Task SaveAsync()
    {
        // DB 저장은 서비스에 위임 (ViewModel은 DB를 모름!)
        await _employeeService.SaveEmployeeAsync(
            new Employee { Name = Name, Email = Email });

        // UI 초기화
        Name = string.Empty;
        Email = string.Empty;
    }
}
```

```csharp
// WPF: Services/Implementations/EmployeeService.cs
// 실제 DB 접근 로직
public class EmployeeService : IEmployeeService
{
    private readonly string _connectionString;

    public EmployeeService(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task SaveEmployeeAsync(Employee employee)
    {
        using var conn = new NpgsqlConnection(_connectionString);
        await conn.ExecuteAsync(
            "INSERT INTO employees (name, email) VALUES (@Name, @Email)",
            employee);
    }
}
```

### 5.3 핵심 차이점 요약

```
WinForms:
┌──────────────────────────┐
│        Form1.cs          │  ← 하나의 파일에 전부 다 있음
│  ┌─────────────────────┐ │
│  │ UI (Designer.cs)    │ │
│  │ 이벤트 핸들러        │ │
│  │ 비즈니스 로직        │ │
│  │ DB 접근 코드         │ │
│  └─────────────────────┘ │
└──────────────────────────┘

WPF + MVVM:
┌──────────┐  ┌──────────────┐  ┌────────────────┐  ┌──────────┐
│  View    │→ │  ViewModel   │→ │    Service     │→ │  Model   │
│ (XAML)   │  │ (UI 로직)     │  │ (비즈니스 로직)  │  │(데이터)   │
│ 화면 표시 │  │ 바인딩 속성    │  │ DB, API 접근   │  │ 순수 클래스│
└──────────┘  └──────────────┘  └────────────────┘  └──────────┘
     ↑ 각 역할이 별도의 파일/폴더에 분리되어 있음
```

**분리의 장점:**
1. **테스트 가능**: ViewModel과 Service를 UI 없이 단위 테스트할 수 있음
2. **재사용성**: ViewModel의 로직을 다른 View에서도 재사용 가능
3. **유지보수**: UI 변경이 비즈니스 로직에 영향을 주지 않음
4. **팀 협업**: UI 담당과 로직 담당이 독립적으로 작업 가능

### 5.4 Designer 파일 비교

WinForms에서는 `Form1.Designer.cs`에 컨트롤 배치 코드가 자동 생성됩니다.
WPF에서는 이 역할을 XAML이 대신합니다.

```csharp
// WinForms: Form1.Designer.cs (자동 생성된 코드)
// C# 코드로 UI를 구성 - 직접 수정하면 안 됨!
private void InitializeComponent()
{
    this.txtName = new System.Windows.Forms.TextBox();
    this.txtName.Location = new System.Drawing.Point(100, 50); // 절대 좌표!
    this.txtName.Size = new System.Drawing.Size(200, 23);
    this.Controls.Add(this.txtName);

    this.btnSave = new System.Windows.Forms.Button();
    this.btnSave.Location = new System.Drawing.Point(100, 100);
    this.btnSave.Text = "저장";
    this.btnSave.Click += new System.EventHandler(this.btnSave_Click);
    this.Controls.Add(this.btnSave);
}
```

```xml
<!-- WPF: Views/EmployeeDetailView.xaml -->
<!-- XAML로 선언적으로 UI 구성 - 직접 편집 가능! -->
<StackPanel Margin="20">
    <!-- 레이아웃 패널이 컨트롤 배치를 자동 관리 (절대 좌표 불필요) -->
    <TextBox Text="{Binding Name}" Margin="0,0,0,10" />
    <Button Content="저장" Command="{Binding SaveCommand}" />
</StackPanel>
```

> **핵심**: WinForms는 **절대 좌표** (Location)로 컨트롤을 배치하지만, WPF는 **레이아웃 패널** (StackPanel, Grid 등)을 사용하여 **상대적으로** 배치합니다. 이 덕분에 창 크기가 바뀌어도 자동으로 재배치됩니다.

---

## 다음 단계

프로젝트가 생성되고 폴더 구조가 잡혔으면, 다음은 **의존성 주입(DI)** 을 설정하여 서비스들을 연결하는 단계입니다.

> [다음 문서: 의존성 주입 (Dependency Injection) 설정 →](./02-dependency-injection.md)
