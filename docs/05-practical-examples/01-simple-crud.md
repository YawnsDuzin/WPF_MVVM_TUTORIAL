# 실전 예제: 직원 관리 CRUD 애플리케이션

> **목표**: Model → Repository → ViewModel → View 전 계층을 아우르는 **완전한 CRUD 예제**를 만들며,
> 지금까지 배운 데이터 바인딩, CommunityToolkit.Mvvm, Dapper + Npgsql, DI를 하나로 통합합니다.

---

## 목차

1. [전체 아키텍처](#1-전체-아키텍처)
2. [PostgreSQL 테이블 준비](#2-postgresql-테이블-준비)
3. [Model 계층](#3-model-계층)
4. [Repository 계층](#4-repository-계층)
5. [ViewModel 계층](#5-viewmodel-계층)
6. [View 계층](#6-view-계층)
7. [DI 등록 및 연결](#7-di-등록-및-연결)
8. [전체 동작 흐름](#8-전체-동작-흐름)

---

## 1. 전체 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                      View 계층                        │
│  ┌─────────────────────┐  ┌────────────────────────┐ │
│  │  EmployeeListView   │  │  EmployeeEditView      │ │
│  │  (DataGrid, 검색바,  │  │  (TextBox, DatePicker, │ │
│  │   추가/삭제 버튼)     │  │   저장/취소 버튼)       │ │
│  └────────┬────────────┘  └────────┬───────────────┘ │
│           │ DataContext              │ DataContext      │
├───────────┼──────────────────────────┼─────────────────┤
│           ▼                          ▼                 │
│      ViewModel 계층                                    │
│  ┌─────────────────────┐  ┌────────────────────────┐ │
│  │ EmployeeListVM      │  │ EmployeeEditVM         │ │
│  │ - Employees (목록)   │  │ - Name, Dept, Email…  │ │
│  │ - SearchText         │  │ - SaveCommand          │ │
│  │ - LoadCommand        │  │ - CancelCommand        │ │
│  │ - DeleteCommand      │  │ - 유효성 검사           │ │
│  └────────┬────────────┘  └────────┬───────────────┘ │
│           │ 의존성 주입               │ 의존성 주입      │
├───────────┼──────────────────────────┼─────────────────┤
│           ▼                          ▼                 │
│      Repository 계층                                   │
│  ┌──────────────────────────────────────────────────┐ │
│  │  IEmployeeRepository  ←  EmployeeRepository      │ │
│  │  (인터페이스)              (Dapper + Npgsql 구현)   │ │
│  └────────────────────────┬─────────────────────────┘ │
│                           │                            │
├───────────────────────────┼────────────────────────────┤
│                           ▼                            │
│      Database (PostgreSQL)                             │
│  ┌──────────────────────────────────────────────────┐ │
│  │  employees 테이블                                  │ │
│  └──────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

**핵심 원칙**:
- **View**는 XAML과 최소한의 코드 비하인드만 가짐
- **ViewModel**은 View를 전혀 모름 (View의 타입을 참조하지 않음)
- **Repository**는 DB 접근 로직을 캡슐화
- 모든 계층은 **DI(의존성 주입)**로 연결

---

## 2. PostgreSQL 테이블 준비

```sql
-- 직원 테이블 생성
CREATE TABLE IF NOT EXISTS employees (
    id          SERIAL       PRIMARY KEY,       -- 자동 증가 기본 키
    name        VARCHAR(100) NOT NULL,           -- 직원 이름
    department  VARCHAR(100) NOT NULL,           -- 부서명
    email       VARCHAR(200) NOT NULL UNIQUE,    -- 이메일 (고유)
    hire_date   DATE         NOT NULL,           -- 입사일
    created_at  TIMESTAMP    NOT NULL DEFAULT NOW(),  -- 생성 시각
    updated_at  TIMESTAMP    NOT NULL DEFAULT NOW()   -- 수정 시각
);

-- 부서별 조회 인덱스
CREATE INDEX IF NOT EXISTS idx_employees_department
    ON employees (department);

-- 이름 검색용 인덱스
CREATE INDEX IF NOT EXISTS idx_employees_name
    ON employees (name);

-- 테스트 데이터 삽입
INSERT INTO employees (name, department, email, hire_date)
VALUES
    ('김철수', '개발팀', 'chulsoo.kim@example.com', '2020-03-15'),
    ('이영희', '기획팀', 'younghee.lee@example.com', '2019-07-01'),
    ('박민수', '개발팀', 'minsoo.park@example.com', '2021-11-20'),
    ('정수진', '디자인팀', 'soojin.jung@example.com', '2022-01-10'),
    ('한지원', '인사팀', 'jiwon.han@example.com', '2018-05-25');
```

---

## 3. Model 계층

### Employee.cs

```csharp
// 파일: Models/Employee.cs
namespace MyApp.Models;

/// <summary>
/// 직원 엔티티 — DB의 employees 테이블과 1:1 매핑
/// </summary>
public class Employee
{
    /// <summary>직원 고유 ID (DB 자동 생성)</summary>
    public int Id { get; set; }

    /// <summary>직원 이름</summary>
    public string Name { get; set; } = string.Empty;

    /// <summary>소속 부서</summary>
    public string Department { get; set; } = string.Empty;

    /// <summary>이메일 주소</summary>
    public string Email { get; set; } = string.Empty;

    /// <summary>입사일</summary>
    public DateTime HireDate { get; set; }

    /// <summary>레코드 생성 시각</summary>
    public DateTime CreatedAt { get; set; }

    /// <summary>레코드 수정 시각</summary>
    public DateTime UpdatedAt { get; set; }
}
```

> **WinForms 비교**: WinForms에서는 DataTable/DataRow를 직접 다루는 경우가 많았지만,
> WPF + MVVM에서는 **강타입 클래스(Model)**를 정의하고 바인딩합니다.
> Dapper가 SQL 결과를 이 클래스로 자동 매핑해 줍니다.

---

## 4. Repository 계층

### IEmployeeRepository.cs (인터페이스)

```csharp
// 파일: Repositories/IEmployeeRepository.cs
namespace MyApp.Repositories;

using MyApp.Models;

/// <summary>
/// 직원 데이터 접근 인터페이스
/// ViewModel은 이 인터페이스에만 의존합니다 (구현체를 직접 참조하지 않음).
/// </summary>
public interface IEmployeeRepository
{
    /// <summary>모든 직원 목록 조회</summary>
    Task<IEnumerable<Employee>> GetAllAsync();

    /// <summary>ID로 직원 1명 조회</summary>
    Task<Employee?> GetByIdAsync(int id);

    /// <summary>이름 또는 부서로 검색</summary>
    Task<IEnumerable<Employee>> SearchAsync(string keyword);

    /// <summary>직원 추가 (INSERT) — 생성된 ID 반환</summary>
    Task<int> AddAsync(Employee employee);

    /// <summary>직원 정보 수정 (UPDATE)</summary>
    Task<bool> UpdateAsync(Employee employee);

    /// <summary>직원 삭제 (DELETE)</summary>
    Task<bool> DeleteAsync(int id);
}
```

### EmployeeRepository.cs (Dapper + Npgsql 구현)

```csharp
// 파일: Repositories/EmployeeRepository.cs
namespace MyApp.Repositories;

using Dapper;
using MyApp.Models;
using Npgsql;
using Serilog;

/// <summary>
/// 직원 Repository 구현체 — Dapper + Npgsql 사용
/// DI 컨테이너에 등록하여 ViewModel에 주입합니다.
/// </summary>
public class EmployeeRepository : IEmployeeRepository
{
    // DB 연결 문자열 (DI로 주입받음)
    private readonly string _connectionString;
    private readonly ILogger _logger;

    public EmployeeRepository(string connectionString, ILogger logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    /// <summary>
    /// Npgsql 커넥션 생성 헬퍼
    /// 매 요청마다 새 커넥션을 열고 닫는 것이 Dapper의 권장 패턴입니다.
    /// </summary>
    private NpgsqlConnection CreateConnection()
    {
        return new NpgsqlConnection(_connectionString);
    }

    /// <summary>전체 직원 목록 조회</summary>
    public async Task<IEnumerable<Employee>> GetAllAsync()
    {
        const string sql = """
            SELECT id, name, department, email,
                   hire_date AS HireDate,
                   created_at AS CreatedAt,
                   updated_at AS UpdatedAt
            FROM employees
            ORDER BY name
            """;

        _logger.Information("전체 직원 목록 조회 시작");

        await using var connection = CreateConnection();
        var employees = await connection.QueryAsync<Employee>(sql);

        _logger.Information("직원 {Count}명 조회 완료", employees.Count());
        return employees;
    }

    /// <summary>ID로 직원 1명 조회</summary>
    public async Task<Employee?> GetByIdAsync(int id)
    {
        const string sql = """
            SELECT id, name, department, email,
                   hire_date AS HireDate,
                   created_at AS CreatedAt,
                   updated_at AS UpdatedAt
            FROM employees
            WHERE id = @Id
            """;

        _logger.Information("직원 조회: ID={Id}", id);

        await using var connection = CreateConnection();
        return await connection.QuerySingleOrDefaultAsync<Employee>(sql, new { Id = id });
    }

    /// <summary>키워드로 이름/부서 검색 (ILIKE = 대소문자 무시)</summary>
    public async Task<IEnumerable<Employee>> SearchAsync(string keyword)
    {
        const string sql = """
            SELECT id, name, department, email,
                   hire_date AS HireDate,
                   created_at AS CreatedAt,
                   updated_at AS UpdatedAt
            FROM employees
            WHERE name ILIKE @Keyword OR department ILIKE @Keyword
            ORDER BY name
            """;

        // '%' 와일드카드를 양쪽에 추가하여 부분 일치 검색
        var parameter = new { Keyword = $"%{keyword}%" };

        _logger.Information("직원 검색: 키워드={Keyword}", keyword);

        await using var connection = CreateConnection();
        return await connection.QueryAsync<Employee>(sql, parameter);
    }

    /// <summary>새 직원 추가 — RETURNING id로 생성된 ID를 바로 반환</summary>
    public async Task<int> AddAsync(Employee employee)
    {
        const string sql = """
            INSERT INTO employees (name, department, email, hire_date)
            VALUES (@Name, @Department, @Email, @HireDate)
            RETURNING id
            """;

        _logger.Information("직원 추가: {Name}, {Department}", employee.Name, employee.Department);

        await using var connection = CreateConnection();
        var newId = await connection.ExecuteScalarAsync<int>(sql, employee);

        _logger.Information("직원 추가 완료: ID={Id}", newId);
        return newId;
    }

    /// <summary>직원 정보 수정</summary>
    public async Task<bool> UpdateAsync(Employee employee)
    {
        const string sql = """
            UPDATE employees
            SET name       = @Name,
                department = @Department,
                email      = @Email,
                hire_date  = @HireDate,
                updated_at = NOW()
            WHERE id = @Id
            """;

        _logger.Information("직원 수정: ID={Id}", employee.Id);

        await using var connection = CreateConnection();
        var affected = await connection.ExecuteAsync(sql, employee);

        // affected > 0이면 수정 성공
        return affected > 0;
    }

    /// <summary>직원 삭제</summary>
    public async Task<bool> DeleteAsync(int id)
    {
        const string sql = "DELETE FROM employees WHERE id = @Id";

        _logger.Information("직원 삭제: ID={Id}", id);

        await using var connection = CreateConnection();
        var affected = await connection.ExecuteAsync(sql, new { Id = id });

        return affected > 0;
    }
}
```

> **Dapper 포인트 요약**:
> - `QueryAsync<T>()` : SELECT → 여러 행을 `T` 컬렉션으로 매핑
> - `QuerySingleOrDefaultAsync<T>()` : SELECT → 1행 또는 null
> - `ExecuteScalarAsync<T>()` : INSERT ... RETURNING → 단일 값 반환
> - `ExecuteAsync()` : INSERT/UPDATE/DELETE → 영향받은 행 수 반환
> - SQL 컬럼명과 C# 프로퍼티명이 다르면 `AS` 별칭으로 맞춤 (예: `hire_date AS HireDate`)

---

## 5. ViewModel 계층

### EmployeeListViewModel (목록 화면)

```csharp
// 파일: ViewModels/EmployeeListViewModel.cs
namespace MyApp.ViewModels;

using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using CommunityToolkit.Mvvm.Messaging;
using MyApp.Models;
using MyApp.Repositories;
using MyApp.Services;
using Serilog;

/// <summary>
/// 직원 목록 화면의 ViewModel
/// - 직원 목록 조회, 검색, 삭제 기능 담당
/// - CommunityToolkit.Mvvm 소스 제네레이터 사용
/// </summary>
public partial class EmployeeListViewModel : ObservableObject
{
    // === DI로 주입받는 의존성 ===
    private readonly IEmployeeRepository _employeeRepository;
    private readonly INavigationService _navigationService;
    private readonly IDialogService _dialogService;
    private readonly ILogger _logger;

    public EmployeeListViewModel(
        IEmployeeRepository employeeRepository,
        INavigationService navigationService,
        IDialogService dialogService,
        ILogger logger)
    {
        _employeeRepository = employeeRepository;
        _navigationService = navigationService;
        _dialogService = dialogService;
        _logger = logger;

        // Messenger를 통해 편집 화면에서 "저장 완료" 메시지를 받으면 목록을 새로고침
        WeakReferenceMessenger.Default.Register<EmployeeSavedMessage>(this, async (r, m) =>
        {
            _logger.Information("직원 저장 메시지 수신 — 목록 새로고침");
            await LoadEmployeesAsync();
        });
    }

    // === Observable 속성들 (소스 제네레이터가 프로퍼티 자동 생성) ===

    /// <summary>직원 목록 — DataGrid에 바인딩</summary>
    public ObservableCollection<Employee> Employees { get; } = [];

    /// <summary>검색어 — 검색 TextBox에 바인딩</summary>
    [ObservableProperty]
    private string _searchText = string.Empty;

    /// <summary>현재 선택된 직원 — DataGrid.SelectedItem에 바인딩</summary>
    [ObservableProperty]
    private Employee? _selectedEmployee;

    /// <summary>로딩 중 여부 — 프로그레스 바에 바인딩</summary>
    [ObservableProperty]
    private bool _isLoading;

    // === 커맨드들 (소스 제네레이터가 IRelayCommand 프로퍼티 자동 생성) ===

    /// <summary>
    /// 직원 목록 조회 커맨드
    /// [RelayCommand]는 LoadEmployeesCommand 프로퍼티를 자동 생성합니다.
    /// </summary>
    [RelayCommand]
    private async Task LoadEmployeesAsync()
    {
        try
        {
            IsLoading = true;

            IEnumerable<Employee> employees;

            // 검색어가 있으면 검색, 없으면 전체 조회
            if (string.IsNullOrWhiteSpace(SearchText))
            {
                employees = await _employeeRepository.GetAllAsync();
            }
            else
            {
                employees = await _employeeRepository.SearchAsync(SearchText);
            }

            // ObservableCollection 내용 교체
            // Clear() + 반복 Add()로 UI에 변경 알림
            Employees.Clear();
            foreach (var employee in employees)
            {
                Employees.Add(employee);
            }

            _logger.Information("직원 목록 로드 완료: {Count}명", Employees.Count);
        }
        catch (Exception ex)
        {
            _logger.Error(ex, "직원 목록 로드 실패");
            await _dialogService.ShowErrorAsync("오류", "직원 목록을 불러오는 중 오류가 발생했습니다.");
        }
        finally
        {
            IsLoading = false;
        }
    }

    /// <summary>
    /// 검색 실행 커맨드
    /// SearchText가 변경될 때마다 호출할 수도 있고,
    /// 검색 버튼 클릭 시에만 호출할 수도 있습니다.
    /// </summary>
    [RelayCommand]
    private async Task SearchAsync()
    {
        _logger.Information("검색 실행: {SearchText}", SearchText);
        await LoadEmployeesAsync();
    }

    /// <summary>
    /// 새 직원 추가 화면으로 이동
    /// </summary>
    [RelayCommand]
    private void AddEmployee()
    {
        _logger.Information("새 직원 추가 화면으로 이동");
        // 네비게이션 서비스를 통해 편집 화면으로 전환 (신규 모드)
        _navigationService.NavigateTo<EmployeeEditViewModel>();
    }

    /// <summary>
    /// 선택된 직원 편집 화면으로 이동
    /// CanExecute: 직원이 선택된 경우에만 활성화
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanEditOrDeleteEmployee))]
    private void EditEmployee()
    {
        if (SelectedEmployee is null) return;

        _logger.Information("직원 편집 화면으로 이동: ID={Id}", SelectedEmployee.Id);
        // 선택된 직원의 ID를 파라미터로 전달
        _navigationService.NavigateTo<EmployeeEditViewModel>(
            parameter: SelectedEmployee.Id);
    }

    /// <summary>
    /// 선택된 직원 삭제
    /// CanExecute: 직원이 선택된 경우에만 활성화
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanEditOrDeleteEmployee))]
    private async Task DeleteEmployeeAsync()
    {
        if (SelectedEmployee is null) return;

        // 삭제 전 사용자 확인
        var confirmed = await _dialogService.ShowConfirmAsync(
            "삭제 확인",
            $"'{SelectedEmployee.Name}' 직원을 정말 삭제하시겠습니까?");

        if (!confirmed) return;

        try
        {
            IsLoading = true;

            var success = await _employeeRepository.DeleteAsync(SelectedEmployee.Id);

            if (success)
            {
                _logger.Information("직원 삭제 완료: {Name}", SelectedEmployee.Name);
                // 목록에서도 즉시 제거 (DB 재조회 없이 UI 즉시 반영)
                Employees.Remove(SelectedEmployee);
                SelectedEmployee = null;
            }
            else
            {
                await _dialogService.ShowErrorAsync("오류", "직원 삭제에 실패했습니다.");
            }
        }
        catch (Exception ex)
        {
            _logger.Error(ex, "직원 삭제 실패");
            await _dialogService.ShowErrorAsync("오류", "직원 삭제 중 오류가 발생했습니다.");
        }
        finally
        {
            IsLoading = false;
        }
    }

    /// <summary>
    /// 편집/삭제 버튼 활성화 조건: 직원이 선택되어 있어야 함
    /// </summary>
    private bool CanEditOrDeleteEmployee()
    {
        return SelectedEmployee is not null;
    }

    /// <summary>
    /// SelectedEmployee가 변경되면 편집/삭제 커맨드의 CanExecute를 다시 평가
    /// partial 메서드: [ObservableProperty]가 자동 호출하는 콜백
    /// </summary>
    partial void OnSelectedEmployeeChanged(Employee? value)
    {
        EditEmployeeCommand.NotifyCanExecuteChanged();
        DeleteEmployeeCommand.NotifyCanExecuteChanged();
    }
}

/// <summary>
/// 직원 저장 완료 메시지 — Messenger로 ViewModel 간 통신
/// </summary>
public class EmployeeSavedMessage
{
    /// <summary>저장된 직원의 ID</summary>
    public int EmployeeId { get; init; }
}
```

### EmployeeEditViewModel (편집 화면)

```csharp
// 파일: ViewModels/EmployeeEditViewModel.cs
namespace MyApp.ViewModels;

using System.ComponentModel.DataAnnotations;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using CommunityToolkit.Mvvm.Messaging;
using MyApp.Models;
using MyApp.Repositories;
using MyApp.Services;
using Serilog;

/// <summary>
/// 직원 편집(추가/수정) 화면의 ViewModel
/// ObservableValidator를 상속하여 유효성 검사 기능을 사용합니다.
///
/// ObservableValidator는 ObservableObject를 상속하면서
/// DataAnnotations 기반 유효성 검사를 추가로 제공합니다.
/// </summary>
public partial class EmployeeEditViewModel : ObservableValidator
{
    // === DI로 주입받는 의존성 ===
    private readonly IEmployeeRepository _employeeRepository;
    private readonly INavigationService _navigationService;
    private readonly IDialogService _dialogService;
    private readonly ILogger _logger;

    // 편집 중인 직원의 ID (0이면 신규)
    private int _employeeId;

    /// <summary>신규 등록 모드인지 여부</summary>
    public bool IsNewEmployee => _employeeId == 0;

    /// <summary>화면 제목 — 신규/편집에 따라 달라짐</summary>
    public string Title => IsNewEmployee ? "새 직원 등록" : "직원 정보 수정";

    public EmployeeEditViewModel(
        IEmployeeRepository employeeRepository,
        INavigationService navigationService,
        IDialogService dialogService,
        ILogger logger)
    {
        _employeeRepository = employeeRepository;
        _navigationService = navigationService;
        _dialogService = dialogService;
        _logger = logger;

        // 기본값: 오늘 날짜
        _hireDate = DateTime.Today;
    }

    // === Observable 속성 + 유효성 검사 ===

    /// <summary>직원 이름 (필수, 2~50자)</summary>
    [ObservableProperty]
    [NotifyDataErrorInfo]          // 유효성 검사 에러를 UI에 자동 통지
    [NotifyCanExecuteChangedFor(nameof(SaveCommand))]  // 값 변경 시 SaveCommand 활성화 재평가
    [Required(ErrorMessage = "이름은 필수 입력입니다.")]
    [MinLength(2, ErrorMessage = "이름은 2자 이상이어야 합니다.")]
    [MaxLength(50, ErrorMessage = "이름은 50자 이하여야 합니다.")]
    private string _name = string.Empty;

    /// <summary>소속 부서 (필수)</summary>
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [NotifyCanExecuteChangedFor(nameof(SaveCommand))]
    [Required(ErrorMessage = "부서는 필수 입력입니다.")]
    private string _department = string.Empty;

    /// <summary>이메일 (필수, 이메일 형식)</summary>
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [NotifyCanExecuteChangedFor(nameof(SaveCommand))]
    [Required(ErrorMessage = "이메일은 필수 입력입니다.")]
    [EmailAddress(ErrorMessage = "올바른 이메일 형식이 아닙니다.")]
    private string _email = string.Empty;

    /// <summary>입사일</summary>
    [ObservableProperty]
    [NotifyDataErrorInfo]
    [NotifyCanExecuteChangedFor(nameof(SaveCommand))]
    [Required(ErrorMessage = "입사일은 필수 입력입니다.")]
    private DateTime _hireDate;

    /// <summary>로딩 상태</summary>
    [ObservableProperty]
    private bool _isLoading;

    // === 부서 목록 (콤보박스용) ===

    /// <summary>드롭다운에 표시할 부서 목록</summary>
    public List<string> Departments { get; } =
    [
        "개발팀", "기획팀", "디자인팀", "인사팀", "영업팀", "마케팅팀"
    ];

    // === 기존 직원 데이터 로드 (편집 모드) ===

    /// <summary>
    /// 편집할 직원의 ID를 설정하고 데이터를 로드합니다.
    /// 네비게이션 시 파라미터로 호출됩니다.
    /// </summary>
    public async Task LoadEmployeeAsync(int employeeId)
    {
        _employeeId = employeeId;

        if (IsNewEmployee)
        {
            _logger.Information("새 직원 등록 모드");
            return;
        }

        try
        {
            IsLoading = true;
            _logger.Information("직원 데이터 로드: ID={Id}", employeeId);

            var employee = await _employeeRepository.GetByIdAsync(employeeId);
            if (employee is null)
            {
                await _dialogService.ShowErrorAsync("오류", "직원 정보를 찾을 수 없습니다.");
                _navigationService.GoBack();
                return;
            }

            // 조회한 데이터를 각 프로퍼티에 설정
            Name = employee.Name;
            Department = employee.Department;
            Email = employee.Email;
            HireDate = employee.HireDate;

            // 프로퍼티 변경 알림 (Title이 바뀌므로)
            OnPropertyChanged(nameof(Title));
            OnPropertyChanged(nameof(IsNewEmployee));
        }
        catch (Exception ex)
        {
            _logger.Error(ex, "직원 데이터 로드 실패: ID={Id}", employeeId);
            await _dialogService.ShowErrorAsync("오류", "직원 정보를 불러오는 중 오류가 발생했습니다.");
        }
        finally
        {
            IsLoading = false;
        }
    }

    // === 커맨드 ===

    /// <summary>
    /// 저장 커맨드
    /// CanExecute: 유효성 검사를 통과해야 활성화
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanSave))]
    private async Task SaveAsync()
    {
        // 전체 유효성 검사 실행
        ValidateAllProperties();

        if (HasErrors)
        {
            _logger.Warning("유효성 검사 실패 — 저장 중단");
            return;
        }

        try
        {
            IsLoading = true;

            // Model 객체 생성
            var employee = new Employee
            {
                Id = _employeeId,
                Name = Name,
                Department = Department,
                Email = Email,
                HireDate = HireDate
            };

            if (IsNewEmployee)
            {
                // 신규 등록
                var newId = await _employeeRepository.AddAsync(employee);
                _logger.Information("새 직원 등록 완료: ID={Id}, Name={Name}", newId, Name);
                await _dialogService.ShowInfoAsync("완료", $"'{Name}' 직원이 등록되었습니다.");
                _employeeId = newId;
            }
            else
            {
                // 기존 수정
                var success = await _employeeRepository.UpdateAsync(employee);
                if (success)
                {
                    _logger.Information("직원 수정 완료: ID={Id}", _employeeId);
                    await _dialogService.ShowInfoAsync("완료", "직원 정보가 수정되었습니다.");
                }
                else
                {
                    await _dialogService.ShowErrorAsync("오류", "직원 정보 수정에 실패했습니다.");
                    return;
                }
            }

            // 목록 화면에 "저장 완료" 메시지 전송
            WeakReferenceMessenger.Default.Send(new EmployeeSavedMessage
            {
                EmployeeId = _employeeId
            });

            // 목록 화면으로 돌아가기
            _navigationService.GoBack();
        }
        catch (Exception ex)
        {
            _logger.Error(ex, "직원 저장 실패");
            await _dialogService.ShowErrorAsync("오류", "저장 중 오류가 발생했습니다.");
        }
        finally
        {
            IsLoading = false;
        }
    }

    /// <summary>
    /// Save 커맨드 활성화 조건: 필수 필드가 비어있지 않고 에러가 없을 때
    /// </summary>
    private bool CanSave()
    {
        return !string.IsNullOrWhiteSpace(Name)
            && !string.IsNullOrWhiteSpace(Department)
            && !string.IsNullOrWhiteSpace(Email)
            && !HasErrors;
    }

    /// <summary>
    /// 취소 커맨드 — 편집을 취소하고 목록 화면으로 돌아감
    /// </summary>
    [RelayCommand]
    private async Task CancelAsync()
    {
        // 변경 사항이 있으면 확인 다이얼로그 표시
        var confirmed = await _dialogService.ShowConfirmAsync(
            "취소 확인",
            "변경 사항이 저장되지 않습니다. 정말 취소하시겠습니까?");

        if (confirmed)
        {
            _logger.Information("편집 취소");
            _navigationService.GoBack();
        }
    }
}
```

> **핵심 포인트 — ObservableValidator**
>
> `ObservableValidator`는 `ObservableObject`를 상속하면서 DataAnnotations 유효성 검사 기능을 추가합니다.
>
> | 특성 | 역할 |
> |------|------|
> | `[Required]` | 필수 입력 검사 |
> | `[MinLength]`, `[MaxLength]` | 길이 제한 |
> | `[EmailAddress]` | 이메일 형식 검사 |
> | `[NotifyDataErrorInfo]` | 검사 에러를 UI에 자동 통지 |
> | `[NotifyCanExecuteChangedFor]` | 값 변경 시 지정된 커맨드의 CanExecute 재평가 |
> | `ValidateAllProperties()` | 모든 프로퍼티 유효성 검사 실행 |
> | `HasErrors` | 에러 존재 여부 |

---

## 6. View 계층

### EmployeeListView.xaml (목록 화면)

```xml
<!-- 파일: Views/EmployeeListView.xaml -->
<UserControl x:Class="MyApp.Views.EmployeeListView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             mc:Ignorable="d"
             d:DataContext="{d:DesignInstance vm:EmployeeListViewModel, IsDesignTimeCreatable=False}"
             d:DesignHeight="600" d:DesignWidth="800">

    <Grid Margin="16">
        <Grid.RowDefinitions>
            <!-- 행 0: 제목 -->
            <RowDefinition Height="Auto"/>
            <!-- 행 1: 검색바 -->
            <RowDefinition Height="Auto"/>
            <!-- 행 2: DataGrid (남은 공간 전체) -->
            <RowDefinition Height="*"/>
            <!-- 행 3: 하단 버튼 -->
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- === 제목 === -->
        <TextBlock Grid.Row="0"
                   Text="직원 관리"
                   FontSize="24"
                   FontWeight="Bold"
                   Margin="0,0,0,16"/>

        <!-- === 검색바 === -->
        <Grid Grid.Row="1" Margin="0,0,0,12">
            <Grid.ColumnDefinitions>
                <!-- 검색 입력란 (남는 공간 차지) -->
                <ColumnDefinition Width="*"/>
                <!-- 검색 버튼 -->
                <ColumnDefinition Width="Auto"/>
            </Grid.ColumnDefinitions>

            <!-- 검색어 입력 — ViewModel.SearchText에 양방향 바인딩 -->
            <TextBox Grid.Column="0"
                     Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
                     Padding="8,6"
                     FontSize="14"
                     VerticalContentAlignment="Center">
                <!-- 엔터 키로 검색 실행을 위한 InputBinding -->
                <TextBox.InputBindings>
                    <KeyBinding Key="Enter"
                                Command="{Binding SearchCommand}"/>
                </TextBox.InputBindings>
            </TextBox>

            <!-- 검색 버튼 -->
            <Button Grid.Column="1"
                    Content="검색"
                    Command="{Binding SearchCommand}"
                    Padding="16,6"
                    Margin="8,0,0,0"
                    FontSize="14"/>
        </Grid>

        <!-- === 직원 목록 DataGrid === -->
        <DataGrid Grid.Row="2"
                  ItemsSource="{Binding Employees}"
                  SelectedItem="{Binding SelectedEmployee}"
                  AutoGenerateColumns="False"
                  IsReadOnly="True"
                  SelectionMode="Single"
                  CanUserAddRows="False"
                  CanUserDeleteRows="False"
                  GridLinesVisibility="Horizontal"
                  AlternatingRowBackground="#F5F5F5"
                  HeadersVisibility="Column"
                  FontSize="14"
                  RowHeight="36">

            <DataGrid.Columns>
                <!-- ID 열 -->
                <DataGridTextColumn Header="ID"
                                    Binding="{Binding Id}"
                                    Width="60"/>
                <!-- 이름 열 -->
                <DataGridTextColumn Header="이름"
                                    Binding="{Binding Name}"
                                    Width="120"/>
                <!-- 부서 열 -->
                <DataGridTextColumn Header="부서"
                                    Binding="{Binding Department}"
                                    Width="120"/>
                <!-- 이메일 열 -->
                <DataGridTextColumn Header="이메일"
                                    Binding="{Binding Email}"
                                    Width="*"/>
                <!-- 입사일 열 (날짜 포맷 지정) -->
                <DataGridTextColumn Header="입사일"
                                    Binding="{Binding HireDate, StringFormat=yyyy-MM-dd}"
                                    Width="120"/>
            </DataGrid.Columns>

            <!-- 행 더블클릭 시 편집 화면 이동 -->
            <DataGrid.InputBindings>
                <MouseBinding MouseAction="LeftDoubleClick"
                              Command="{Binding EditEmployeeCommand}"/>
            </DataGrid.InputBindings>
        </DataGrid>

        <!-- === 로딩 오버레이 === -->
        <!-- IsLoading이 true일 때 DataGrid 위에 반투명 오버레이 표시 -->
        <Border Grid.Row="2"
                Background="#80FFFFFF"
                Visibility="{Binding IsLoading, Converter={StaticResource BooleanToVisibilityConverter}}">
            <StackPanel HorizontalAlignment="Center"
                        VerticalAlignment="Center">
                <!-- 회전 애니메이션이 적용된 프로그레스 링 -->
                <ProgressBar IsIndeterminate="True"
                             Width="200"
                             Height="4"
                             Margin="0,0,0,8"/>
                <TextBlock Text="데이터를 불러오는 중..."
                           HorizontalAlignment="Center"
                           FontSize="14"
                           Foreground="Gray"/>
            </StackPanel>
        </Border>

        <!-- === 하단 버튼 영역 === -->
        <StackPanel Grid.Row="3"
                    Orientation="Horizontal"
                    HorizontalAlignment="Right"
                    Margin="0,12,0,0">

            <!-- 새 직원 추가 버튼 -->
            <Button Content="새 직원 추가"
                    Command="{Binding AddEmployeeCommand}"
                    Padding="16,8"
                    FontSize="14"
                    Margin="0,0,8,0"
                    Background="#4CAF50"
                    Foreground="White"/>

            <!-- 편집 버튼 (직원 선택 시에만 활성화) -->
            <Button Content="편집"
                    Command="{Binding EditEmployeeCommand}"
                    Padding="16,8"
                    FontSize="14"
                    Margin="0,0,8,0"/>

            <!-- 삭제 버튼 (직원 선택 시에만 활성화) -->
            <Button Content="삭제"
                    Command="{Binding DeleteEmployeeCommand}"
                    Padding="16,8"
                    FontSize="14"
                    Background="#F44336"
                    Foreground="White"/>
        </StackPanel>
    </Grid>
</UserControl>
```

### EmployeeListView.xaml.cs (코드 비하인드)

```csharp
// 파일: Views/EmployeeListView.xaml.cs
namespace MyApp.Views;

using System.Windows.Controls;

/// <summary>
/// 직원 목록 화면의 코드 비하인드
/// MVVM 패턴에서는 코드 비하인드를 최소화합니다.
/// DataContext(ViewModel)는 DI를 통해 외부에서 설정됩니다.
/// </summary>
public partial class EmployeeListView : UserControl
{
    public EmployeeListView()
    {
        InitializeComponent();
    }
}
```

### EmployeeEditView.xaml (편집 화면)

```xml
<!-- 파일: Views/EmployeeEditView.xaml -->
<UserControl x:Class="MyApp.Views.EmployeeEditView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             mc:Ignorable="d"
             d:DataContext="{d:DesignInstance vm:EmployeeEditViewModel, IsDesignTimeCreatable=False}"
             d:DesignHeight="500" d:DesignWidth="500">

    <Grid Margin="24">
        <Grid.RowDefinitions>
            <!-- 행 0: 제목 -->
            <RowDefinition Height="Auto"/>
            <!-- 행 1: 폼 영역 -->
            <RowDefinition Height="Auto"/>
            <!-- 행 2: 빈 공간 -->
            <RowDefinition Height="*"/>
            <!-- 행 3: 버튼 영역 -->
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- === 제목 (신규 등록 / 직원 정보 수정) === -->
        <TextBlock Grid.Row="0"
                   Text="{Binding Title}"
                   FontSize="24"
                   FontWeight="Bold"
                   Margin="0,0,0,24"/>

        <!-- === 폼 입력 영역 === -->
        <StackPanel Grid.Row="1" MaxWidth="400" HorizontalAlignment="Left">

            <!-- 이름 입력 -->
            <TextBlock Text="이름 *" FontSize="14" Margin="0,0,0,4"/>
            <TextBox Text="{Binding Name,
                            UpdateSourceTrigger=PropertyChanged,
                            ValidatesOnDataErrors=True,
                            NotifyOnValidationError=True}"
                     Padding="8,6"
                     FontSize="14"
                     Margin="0,0,0,4"/>
            <!-- 유효성 검사 에러 메시지 표시 -->
            <TextBlock Text="{Binding (Validation.Errors)[0].ErrorContent,
                              RelativeSource={RelativeSource PreviousData}}"
                       Foreground="Red"
                       FontSize="12"
                       Margin="0,0,0,12"/>

            <!-- 부서 선택 (콤보박스) -->
            <TextBlock Text="부서 *" FontSize="14" Margin="0,0,0,4"/>
            <ComboBox ItemsSource="{Binding Departments}"
                      SelectedItem="{Binding Department,
                                     UpdateSourceTrigger=PropertyChanged,
                                     ValidatesOnDataErrors=True}"
                      Padding="8,6"
                      FontSize="14"
                      Margin="0,0,0,16"/>

            <!-- 이메일 입력 -->
            <TextBlock Text="이메일 *" FontSize="14" Margin="0,0,0,4"/>
            <TextBox Text="{Binding Email,
                            UpdateSourceTrigger=PropertyChanged,
                            ValidatesOnDataErrors=True,
                            NotifyOnValidationError=True}"
                     Padding="8,6"
                     FontSize="14"
                     Margin="0,0,0,16"/>

            <!-- 입사일 선택 (DatePicker) -->
            <TextBlock Text="입사일 *" FontSize="14" Margin="0,0,0,4"/>
            <DatePicker SelectedDate="{Binding HireDate,
                                       UpdateSourceTrigger=PropertyChanged,
                                       ValidatesOnDataErrors=True}"
                        Padding="8,6"
                        FontSize="14"
                        Margin="0,0,0,16"/>

        </StackPanel>

        <!-- === 하단 버튼 === -->
        <StackPanel Grid.Row="3"
                    Orientation="Horizontal"
                    HorizontalAlignment="Right"
                    Margin="0,12,0,0">

            <!-- 저장 버튼 (유효성 검사 통과 시에만 활성화) -->
            <Button Content="저장"
                    Command="{Binding SaveCommand}"
                    Padding="24,10"
                    FontSize="14"
                    Margin="0,0,8,0"
                    Background="#4CAF50"
                    Foreground="White"
                    IsDefault="True"/>

            <!-- 취소 버튼 -->
            <Button Content="취소"
                    Command="{Binding CancelCommand}"
                    Padding="24,10"
                    FontSize="14"
                    IsCancel="True"/>
        </StackPanel>

        <!-- === 로딩 오버레이 === -->
        <Border Grid.Row="0" Grid.RowSpan="4"
                Background="#80FFFFFF"
                Visibility="{Binding IsLoading, Converter={StaticResource BooleanToVisibilityConverter}}">
            <ProgressBar IsIndeterminate="True"
                         Width="200"
                         Height="4"
                         HorizontalAlignment="Center"
                         VerticalAlignment="Center"/>
        </Border>
    </Grid>
</UserControl>
```

### EmployeeEditView.xaml.cs (코드 비하인드)

```csharp
// 파일: Views/EmployeeEditView.xaml.cs
namespace MyApp.Views;

using System.Windows.Controls;

/// <summary>
/// 직원 편집 화면의 코드 비하인드
/// MVVM 패턴에서는 코드 비하인드를 최소화합니다.
/// </summary>
public partial class EmployeeEditView : UserControl
{
    public EmployeeEditView()
    {
        InitializeComponent();
    }
}
```

---

## 7. DI 등록 및 연결

### App.xaml.cs에서 DI 등록

```csharp
// 파일: App.xaml.cs (관련 부분 발췌)
namespace MyApp;

using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Configuration;
using MyApp.Repositories;
using MyApp.ViewModels;
using MyApp.Views;
using MyApp.Services;
using Serilog;

public partial class App : Application
{
    /// <summary>DI 서비스 프로바이더</summary>
    public IServiceProvider Services { get; private set; } = null!;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // 설정 파일 로드
        var configuration = new ConfigurationBuilder()
            .AddJsonFile("appsettings.json", optional: false)
            .Build();

        // Serilog 설정
        Log.Logger = new LoggerConfiguration()
            .ReadFrom.Configuration(configuration)
            .CreateLogger();

        // DI 컨테이너 구성
        var services = new ServiceCollection();

        // --- 로깅 ---
        services.AddSingleton(Log.Logger);

        // --- Repository 등록 ---
        // DB 연결 문자열을 설정에서 읽어 Repository에 전달
        var connectionString = configuration.GetConnectionString("DefaultConnection")!;
        services.AddSingleton<IEmployeeRepository>(provider =>
            new EmployeeRepository(connectionString, provider.GetRequiredService<ILogger>()));

        // --- 서비스 등록 ---
        services.AddSingleton<INavigationService, NavigationService>();
        services.AddSingleton<IDialogService, DialogService>();

        // --- ViewModel 등록 ---
        // Transient: 매번 새 인스턴스 (화면 전환 시마다 새로 생성)
        services.AddTransient<EmployeeListViewModel>();
        services.AddTransient<EmployeeEditViewModel>();

        // --- View 등록 ---
        services.AddTransient<EmployeeListView>();
        services.AddTransient<EmployeeEditView>();

        // --- MainWindow ---
        services.AddSingleton<MainViewModel>();
        services.AddSingleton<MainWindow>();

        // 서비스 프로바이더 생성
        Services = services.BuildServiceProvider();

        // MainWindow 표시
        var mainWindow = Services.GetRequiredService<MainWindow>();
        mainWindow.Show();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        // Serilog 플러시 (남은 로그 기록 보장)
        Log.CloseAndFlush();
        base.OnExit(e);
    }
}
```

### DataTemplate 매핑 (App.xaml)

```xml
<!-- 파일: App.xaml -->
<Application x:Class="MyApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             xmlns:views="clr-namespace:MyApp.Views">

    <Application.Resources>
        <!-- Boolean → Visibility 변환기 (여러 곳에서 사용) -->
        <BooleanToVisibilityConverter x:Key="BooleanToVisibilityConverter"/>

        <!-- ViewModel → View 자동 매핑 DataTemplate -->
        <!-- ContentControl에 ViewModel이 바인딩되면, 해당 DataTemplate의 View가 자동으로 표시됨 -->
        <DataTemplate DataType="{x:Type vm:EmployeeListViewModel}">
            <views:EmployeeListView/>
        </DataTemplate>

        <DataTemplate DataType="{x:Type vm:EmployeeEditViewModel}">
            <views:EmployeeEditView/>
        </DataTemplate>
    </Application.Resources>
</Application>
```

---

## 8. 전체 동작 흐름

### 시나리오 1: 앱 시작 → 직원 목록 조회

```
1. App.OnStartup() 실행
   → DI 컨테이너 구성
   → MainWindow 생성 & 표시

2. MainWindow의 ContentControl에 EmployeeListViewModel 설정
   → DataTemplate에 의해 EmployeeListView가 자동 렌더링

3. EmployeeListView 로드 시 LoadEmployeesCommand 실행
   → EmployeeListViewModel.LoadEmployeesAsync() 호출
   → IsLoading = true (로딩 오버레이 표시)
   → IEmployeeRepository.GetAllAsync() 호출
   → Dapper가 PostgreSQL에서 데이터 조회
   → ObservableCollection에 데이터 추가
   → IsLoading = false (로딩 오버레이 숨김)
   → DataGrid에 직원 목록 표시
```

### 시나리오 2: 직원 검색

```
1. 사용자가 검색 TextBox에 "개발" 입력
   → SearchText 프로퍼티 자동 업데이트 (TwoWay 바인딩)

2. 엔터 키 또는 검색 버튼 클릭
   → SearchCommand 실행
   → LoadEmployeesAsync() 호출
   → SearchText가 비어있지 않으므로 SearchAsync("개발") 호출
   → SQL: WHERE name ILIKE '%개발%' OR department ILIKE '%개발%'
   → 결과로 ObservableCollection 교체
   → DataGrid 자동 갱신
```

### 시나리오 3: 새 직원 추가

```
1. "새 직원 추가" 버튼 클릭
   → AddEmployeeCommand 실행
   → NavigationService.NavigateTo<EmployeeEditViewModel>() 호출
   → MainViewModel.CurrentViewModel이 EmployeeEditViewModel로 변경
   → DataTemplate에 의해 EmployeeEditView 표시

2. 사용자가 폼 입력 (이름, 부서, 이메일, 입사일)
   → 각 프로퍼티에 TwoWay 바인딩
   → 입력할 때마다 유효성 검사 실행 ([NotifyDataErrorInfo])
   → 유효하지 않으면 에러 메시지 표시 + 저장 버튼 비활성화

3. "저장" 버튼 클릭
   → SaveCommand 실행
   → ValidateAllProperties()로 최종 검사
   → IEmployeeRepository.AddAsync() 호출
   → DB에 INSERT 실행
   → 성공 다이얼로그 표시
   → WeakReferenceMessenger로 EmployeeSavedMessage 전송
   → NavigationService.GoBack()으로 목록 화면 복귀

4. 목록 화면에서 메시지 수신
   → LoadEmployeesAsync() 자동 호출
   → 새 직원이 포함된 목록 표시
```

### 시나리오 4: 직원 삭제

```
1. DataGrid에서 직원 행 클릭 (선택)
   → SelectedEmployee 프로퍼티 업데이트
   → OnSelectedEmployeeChanged() 호출
   → EditEmployeeCommand, DeleteEmployeeCommand의 CanExecute 재평가
   → 편집/삭제 버튼 활성화

2. "삭제" 버튼 클릭
   → DeleteEmployeeCommand 실행
   → 확인 다이얼로그: "정말 삭제하시겠습니까?"

3. "예" 클릭
   → IEmployeeRepository.DeleteAsync(id) 호출
   → DB에서 DELETE 실행
   → ObservableCollection에서 해당 항목 제거
   → DataGrid 자동 갱신
```

---

## WinForms 개발자를 위한 비교 정리

| 항목 | WinForms 방식 | WPF MVVM 방식 (이 예제) |
|------|--------------|------------------------|
| 데이터 표시 | `dataGridView1.DataSource = dataTable` | `ItemsSource="{Binding Employees}"` |
| 행 선택 | `dataGridView1.SelectedRows[0]` | `SelectedItem="{Binding SelectedEmployee}"` |
| 버튼 클릭 | `button1_Click` 이벤트 핸들러 | `Command="{Binding DeleteEmployeeCommand}"` |
| 버튼 활성화 | `button1.Enabled = (selected != null)` | `CanExecute = nameof(CanEditOrDeleteEmployee)` |
| 입력 검증 | `if (textBox1.Text == "") { ... }` | `[Required]`, `[EmailAddress]` 어트리뷰트 |
| 화면 전환 | `var form = new EditForm(); form.Show();` | `NavigationService.NavigateTo<EditVM>()` |
| DB 조회 | `SqlCommand` + `DataReader` 직접 작성 | `Dapper QueryAsync<T>()` + Repository 패턴 |
| 화면 간 통신 | 이벤트, public 메서드 직접 호출 | `WeakReferenceMessenger` 메시지 패턴 |

> **핵심**: WinForms에서는 "컨트롤을 직접 조작"했지만, WPF MVVM에서는 "데이터(ViewModel)만 변경하면 UI가 자동으로 따라온다"는 것이 가장 큰 차이입니다.
