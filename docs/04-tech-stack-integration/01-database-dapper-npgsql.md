# PostgreSQL + Dapper + Npgsql 데이터베이스 연동

> **목표**: WPF MVVM 프로젝트에서 PostgreSQL 데이터베이스에 Dapper + Npgsql을 사용하여 연결하고, Repository 패턴으로 CRUD를 구현합니다.
> **대상 독자**: WinForms에서 SqlConnection/SqlCommand로 DB를 직접 다뤄본 경험이 있는 개발자

---

## 목차

1. [WinForms의 DB 접근 방식 vs WPF MVVM 방식](#1-winforms의-db-접근-방식-vs-wpf-mvvm-방식)
2. [Npgsql — PostgreSQL ADO.NET 드라이버](#2-npgsql--postgresql-adonet-드라이버)
3. [Dapper — 마이크로 ORM](#3-dapper--마이크로-orm)
4. [연결 문자열 관리 (appsettings.json → IOptions)](#4-연결-문자열-관리)
5. [Repository 패턴 구현](#5-repository-패턴-구현)
6. [DI에 Repository 등록](#6-di에-repository-등록)
7. [ViewModel에서 Repository 사용](#7-viewmodel에서-repository-사용)
8. [전체 CRUD 흐름](#8-전체-crud-흐름)
9. [Connection Pooling](#9-connection-pooling)
10. [주의사항: UI 스레드에서 DB 호출 금지](#10-주의사항-ui-스레드에서-db-호출-금지)

---

## 1. WinForms의 DB 접근 방식 vs WPF MVVM 방식

### WinForms에서 흔히 사용하던 방식

WinForms 프로젝트에서는 보통 버튼 클릭 이벤트 핸들러 안에서 직접 DB에 접근했습니다.

```csharp
// ❌ WinForms 방식 — 이벤트 핸들러에서 직접 DB 접근
private void btnSave_Click(object sender, EventArgs e)
{
    // UI 코드와 DB 코드가 한 곳에 섞여 있음
    using var conn = new NpgsqlConnection("Host=localhost;Database=mydb;...");
    conn.Open();

    using var cmd = new NpgsqlCommand(
        "INSERT INTO employees (name, department) VALUES (@name, @dept)", conn);
    cmd.Parameters.AddWithValue("@name", txtName.Text);
    cmd.Parameters.AddWithValue("@dept", cboDepartment.Text);
    cmd.ExecuteNonQuery();

    MessageBox.Show("저장 완료!");
    LoadEmployeeGrid(); // 그리드 새로고침도 직접 호출
}
```

**이 방식의 문제점:**

| 문제 | 설명 |
|------|------|
| UI와 로직 결합 | 버튼 클릭 이벤트에 SQL이 직접 들어감 |
| 테스트 불가 | Form을 띄우지 않으면 로직 테스트가 불가능 |
| 동기 호출 | `ExecuteNonQuery()`가 끝날 때까지 UI가 멈춤 (프리징) |
| 연결 문자열 하드코딩 | 배포 환경마다 코드를 수정해야 함 |
| 코드 재사용 불가 | 같은 SQL을 여러 Form에서 복붙하게 됨 |

### WPF MVVM 방식

```
View (XAML)  →  ViewModel  →  Repository  →  Database
   바인딩          명령           Dapper         PostgreSQL
```

WPF MVVM에서는 각 계층이 명확하게 분리됩니다:

- **View (XAML)**: 화면 표시만 담당. DB에 대해 아무것도 모름
- **ViewModel**: UI 로직을 담당. Repository를 통해 데이터를 요청
- **Repository**: DB 접근을 캡슐화. SQL 쿼리 실행
- **Model**: 데이터 구조를 정의

```csharp
// ✅ WPF MVVM 방식 — ViewModel에서 Repository를 통해 DB 접근
public partial class EmployeeViewModel : ObservableObject
{
    private readonly IEmployeeRepository _repository;

    // 생성자 주입으로 Repository를 받음
    public EmployeeViewModel(IEmployeeRepository repository)
    {
        _repository = repository;
    }

    [RelayCommand]
    private async Task SaveEmployeeAsync()
    {
        // UI 스레드를 블로킹하지 않는 비동기 호출
        await _repository.CreateAsync(new Employee
        {
            Name = _name,
            Department = _department
        });

        // 목록 새로고침
        await LoadEmployeesAsync();
    }
}
```

**MVVM 방식의 장점:**

| 장점 | 설명 |
|------|------|
| 관심사 분리 | UI, 비즈니스 로직, DB 접근이 각각 독립적 |
| 테스트 용이 | Repository를 Mock으로 교체하여 ViewModel 단독 테스트 가능 |
| 비동기 지원 | async/await로 UI가 절대 멈추지 않음 |
| 설정 외부화 | 연결 문자열을 appsettings.json에서 관리 |
| 코드 재사용 | Repository를 여러 ViewModel에서 공유 |

---

## 2. Npgsql — PostgreSQL ADO.NET 드라이버

### Npgsql이란?

Npgsql은 PostgreSQL 데이터베이스에 접근하기 위한 **.NET 공식 ADO.NET 드라이버**입니다. SQL Server에서 `SqlConnection`을 쓰듯이, PostgreSQL에서는 `NpgsqlConnection`을 사용합니다.

### NuGet 패키지 설치

```bash
# Npgsql 드라이버
dotnet add package Npgsql --version 9.0.3

# Dapper (마이크로 ORM)
dotnet add package Dapper --version 2.1.44
```

### ADO.NET과의 관계

```
ADO.NET (추상 인터페이스)     Npgsql (PostgreSQL 구현)
─────────────────────────    ─────────────────────────
DbConnection           →    NpgsqlConnection
DbCommand              →    NpgsqlCommand
DbDataReader           →    NpgsqlDataReader
DbParameter            →    NpgsqlParameter
```

WinForms에서 `SqlConnection`을 사용했다면, 거의 동일한 방식으로 `NpgsqlConnection`을 사용할 수 있습니다. 차이점은 연결 문자열 형식뿐입니다.

```csharp
// SQL Server 연결 문자열
"Server=localhost;Database=mydb;Trusted_Connection=True;"

// PostgreSQL 연결 문자열 (Npgsql)
"Host=localhost;Port=5432;Database=mydb;Username=postgres;Password=secret;"
```

---

## 3. Dapper — 마이크로 ORM

### Dapper란?

Dapper는 Stack Overflow 팀이 만든 **마이크로 ORM(Object-Relational Mapper)**입니다. ADO.NET의 `IDbConnection` 위에 얇은 확장 메서드 레이어를 추가하여, SQL 결과를 C# 객체로 자동 매핑해 줍니다.

### Entity Framework vs Dapper 비교

| 항목 | Entity Framework Core | Dapper |
|------|----------------------|--------|
| 종류 | 풀 ORM | 마이크로 ORM |
| SQL 작성 | 자동 생성 (LINQ → SQL) | 직접 작성 |
| 학습 곡선 | 높음 (마이그레이션, DbContext, LINQ 등) | 낮음 (SQL만 알면 됨) |
| 성능 | 상대적으로 느림 | ADO.NET에 가까운 성능 |
| 매핑 | 자동 (Convention 기반) | 자동 (컬럼명 ↔ 속성명 매핑) |
| 변경 추적 | 있음 (Change Tracker) | 없음 |
| 마이그레이션 | 내장 | 없음 (별도 도구 필요) |
| NuGet 크기 | 크다 | 매우 작다 |

### 왜 Dapper를 선택하는가?

1. **SQL을 직접 제어**: 복잡한 JOIN, 서브쿼리, CTE 등을 자유롭게 작성
2. **성능**: ORM 오버헤드가 거의 없어 ADO.NET 수준의 속도
3. **낮은 학습 비용**: WinForms에서 SQL을 직접 썼다면 바로 적응 가능
4. **간결함**: 확장 메서드 몇 개만 익히면 됨

### Dapper 핵심 확장 메서드

```csharp
// 여러 행 조회 — SQL 결과를 IEnumerable<T>로 변환
IEnumerable<Employee> employees = await connection.QueryAsync<Employee>(
    "SELECT * FROM employees");

// 단일 행 조회 — 첫 번째 행 또는 null 반환
Employee? employee = await connection.QueryFirstOrDefaultAsync<Employee>(
    "SELECT * FROM employees WHERE id = @Id", new { Id = 1 });

// INSERT, UPDATE, DELETE — 영향받은 행 수 반환
int affectedRows = await connection.ExecuteAsync(
    "INSERT INTO employees (name) VALUES (@Name)", new { Name = "홍길동" });

// 스칼라 값 조회 — 단일 값 반환
int count = await connection.ExecuteScalarAsync<int>(
    "SELECT COUNT(*) FROM employees");
```

---

## 4. 연결 문자열 관리

### 연결 문자열을 하드코딩하면 안 되는 이유

```csharp
// ❌ 절대 이렇게 하지 마세요
var conn = new NpgsqlConnection("Host=localhost;Database=mydb;Username=postgres;Password=1234");
```

- 비밀번호가 소스 코드에 노출됨
- 개발/스테이징/운영 환경마다 코드를 수정해야 함
- Git에 비밀번호가 커밋됨

### appsettings.json에 연결 문자열 저장

```jsonc
// appsettings.json
{
  "DatabaseSettings": {
    "ConnectionString": "Host=localhost;Port=5432;Database=employee_db;Username=postgres;Password=yourpassword;",
    "CommandTimeout": 30
  }
}
```

### 설정 클래스 정의

```csharp
// Models/Settings/DatabaseSettings.cs

namespace MyApp.Models.Settings;

/// <summary>
/// 데이터베이스 연결 설정을 담는 클래스.
/// appsettings.json의 "DatabaseSettings" 섹션과 매핑됩니다.
/// </summary>
public sealed class DatabaseSettings
{
    /// <summary>
    /// 설정 파일에서 이 섹션의 키 이름
    /// </summary>
    public const string SectionName = "DatabaseSettings";

    /// <summary>
    /// PostgreSQL 연결 문자열
    /// </summary>
    public string ConnectionString { get; set; } = string.Empty;

    /// <summary>
    /// SQL 명령 타임아웃 (초 단위, 기본값 30초)
    /// </summary>
    public int CommandTimeout { get; set; } = 30;
}
```

### DI에 설정 등록 (App.xaml.cs)

```csharp
// App.xaml.cs (DI 설정 부분)

// appsettings.json에서 DatabaseSettings 섹션을 읽어 IOptions<DatabaseSettings>로 등록
services.Configure<DatabaseSettings>(
    configuration.GetSection(DatabaseSettings.SectionName));
```

이렇게 하면 어디서든 `IOptions<DatabaseSettings>`를 주입받아 연결 문자열을 사용할 수 있습니다.

---

## 5. Repository 패턴 구현

Repository 패턴은 **데이터 접근 로직을 캡슐화**하는 디자인 패턴입니다. ViewModel은 Repository 인터페이스만 알면 되고, 실제 SQL이 어떻게 실행되는지 몰라도 됩니다.

### 5.1 Model 정의

```csharp
// Models/Employee.cs

namespace MyApp.Models;

/// <summary>
/// 직원 정보를 나타내는 모델 클래스.
/// DB의 employees 테이블과 매핑됩니다.
/// </summary>
public sealed class Employee
{
    /// <summary>직원 고유 ID (PK, 자동 증가)</summary>
    public int Id { get; set; }

    /// <summary>직원 이름</summary>
    public string Name { get; set; } = string.Empty;

    /// <summary>부서명</summary>
    public string Department { get; set; } = string.Empty;

    /// <summary>이메일 주소</summary>
    public string Email { get; set; } = string.Empty;

    /// <summary>입사일</summary>
    public DateTime HireDate { get; set; }

    /// <summary>활성 상태 여부</summary>
    public bool IsActive { get; set; } = true;

    /// <summary>레코드 생성 시각</summary>
    public DateTime CreatedAt { get; set; }

    /// <summary>레코드 수정 시각</summary>
    public DateTime? UpdatedAt { get; set; }
}
```

### 5.2 PostgreSQL 테이블 생성 SQL

```sql
-- PostgreSQL에서 employees 테이블 생성
CREATE TABLE IF NOT EXISTS employees (
    id          SERIAL PRIMARY KEY,                    -- 자동 증가 PK
    name        VARCHAR(100) NOT NULL,                 -- 직원 이름
    department  VARCHAR(50)  NOT NULL,                 -- 부서명
    email       VARCHAR(200) NOT NULL UNIQUE,          -- 이메일 (유니크)
    hire_date   DATE         NOT NULL DEFAULT CURRENT_DATE, -- 입사일
    is_active   BOOLEAN      NOT NULL DEFAULT TRUE,    -- 활성 상태
    created_at  TIMESTAMP    NOT NULL DEFAULT NOW(),   -- 생성 시각
    updated_at  TIMESTAMP                              -- 수정 시각
);

-- 자주 검색하는 컬럼에 인덱스 추가
CREATE INDEX idx_employees_department ON employees(department);
CREATE INDEX idx_employees_is_active ON employees(is_active);
```

### 5.3 IEmployeeRepository 인터페이스 정의

```csharp
// Repositories/IEmployeeRepository.cs

namespace MyApp.Repositories;

/// <summary>
/// 직원 데이터 접근을 위한 Repository 인터페이스.
/// ViewModel은 이 인터페이스에만 의존합니다.
/// 실제 구현(PostgreSQL, Mock 등)은 DI에서 주입됩니다.
/// </summary>
public interface IEmployeeRepository
{
    /// <summary>
    /// 모든 직원 목록을 조회합니다.
    /// </summary>
    /// <param name="cancellationToken">취소 토큰</param>
    /// <returns>직원 목록</returns>
    Task<IEnumerable<Employee>> GetAllAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// ID로 직원 한 명을 조회합니다.
    /// </summary>
    /// <param name="id">직원 ID</param>
    /// <param name="cancellationToken">취소 토큰</param>
    /// <returns>직원 정보. 없으면 null</returns>
    Task<Employee?> GetByIdAsync(int id, CancellationToken cancellationToken = default);

    /// <summary>
    /// 새 직원을 생성합니다.
    /// </summary>
    /// <param name="employee">생성할 직원 정보</param>
    /// <param name="cancellationToken">취소 토큰</param>
    /// <returns>생성된 직원의 ID</returns>
    Task<int> CreateAsync(Employee employee, CancellationToken cancellationToken = default);

    /// <summary>
    /// 기존 직원 정보를 수정합니다.
    /// </summary>
    /// <param name="employee">수정할 직원 정보 (Id 필수)</param>
    /// <param name="cancellationToken">취소 토큰</param>
    /// <returns>수정 성공 여부</returns>
    Task<bool> UpdateAsync(Employee employee, CancellationToken cancellationToken = default);

    /// <summary>
    /// 직원을 삭제합니다.
    /// </summary>
    /// <param name="id">삭제할 직원 ID</param>
    /// <param name="cancellationToken">취소 토큰</param>
    /// <returns>삭제 성공 여부</returns>
    Task<bool> DeleteAsync(int id, CancellationToken cancellationToken = default);
}
```

### 5.4 EmployeeRepository 구현 (Dapper 사용)

```csharp
// Repositories/EmployeeRepository.cs

using System.Data;
using Dapper;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using Npgsql;

namespace MyApp.Repositories;

/// <summary>
/// Dapper + Npgsql을 사용한 직원 Repository 구현.
/// 모든 메서드가 비동기이며, 연결은 매번 새로 생성합니다 (Connection Pooling 활용).
/// </summary>
public sealed class EmployeeRepository : IEmployeeRepository
{
    private readonly string _connectionString;
    private readonly int _commandTimeout;
    private readonly ILogger<EmployeeRepository> _logger;

    /// <summary>
    /// 생성자 — DI를 통해 설정과 로거를 주입받습니다.
    /// </summary>
    /// <param name="options">DB 설정 (연결 문자열, 타임아웃)</param>
    /// <param name="logger">로거</param>
    public EmployeeRepository(
        IOptions<DatabaseSettings> options,
        ILogger<EmployeeRepository> logger)
    {
        // IOptions<T>에서 실제 설정 값을 꺼냄
        _connectionString = options.Value.ConnectionString;
        _commandTimeout = options.Value.CommandTimeout;
        _logger = logger;
    }

    /// <summary>
    /// 새로운 DB 연결을 생성합니다.
    /// Npgsql은 내부적으로 Connection Pooling을 관리하므로,
    /// 매번 new NpgsqlConnection()을 호출해도 성능 문제가 없습니다.
    /// </summary>
    private NpgsqlConnection CreateConnection()
    {
        return new NpgsqlConnection(_connectionString);
    }

    // ─────────────────────────────────────────────
    // 전체 조회
    // ─────────────────────────────────────────────

    /// <inheritdoc />
    public async Task<IEnumerable<Employee>> GetAllAsync(
        CancellationToken cancellationToken = default)
    {
        // SQL 쿼리 — PostgreSQL의 snake_case 컬럼을 C#의 PascalCase 속성에 매핑
        // Dapper는 기본적으로 대소문자를 무시하고 매핑하지만,
        // 명시적으로 AS 별칭을 사용하면 더 안전합니다.
        const string sql = """
            SELECT
                id          AS Id,
                name        AS Name,
                department  AS Department,
                email       AS Email,
                hire_date   AS HireDate,
                is_active   AS IsActive,
                created_at  AS CreatedAt,
                updated_at  AS UpdatedAt
            FROM employees
            ORDER BY name
            """;

        _logger.LogDebug("직원 전체 목록 조회 시작");

        // using 선언 — 메서드 끝에서 자동으로 연결이 닫힘
        await using var connection = CreateConnection();

        // Dapper의 QueryAsync: SQL 결과를 Employee 객체 컬렉션으로 변환
        var employees = await connection.QueryAsync<Employee>(
            new CommandDefinition(sql, commandTimeout: _commandTimeout,
                cancellationToken: cancellationToken));

        _logger.LogInformation("직원 {Count}명 조회 완료", employees.AsList().Count);

        return employees;
    }

    // ─────────────────────────────────────────────
    // 단건 조회
    // ─────────────────────────────────────────────

    /// <inheritdoc />
    public async Task<Employee?> GetByIdAsync(
        int id,
        CancellationToken cancellationToken = default)
    {
        const string sql = """
            SELECT
                id          AS Id,
                name        AS Name,
                department  AS Department,
                email       AS Email,
                hire_date   AS HireDate,
                is_active   AS IsActive,
                created_at  AS CreatedAt,
                updated_at  AS UpdatedAt
            FROM employees
            WHERE id = @Id
            """;

        _logger.LogDebug("직원 조회: Id={EmployeeId}", id);

        await using var connection = CreateConnection();

        // QueryFirstOrDefaultAsync: 결과가 없으면 null 반환
        // 익명 객체 new { Id = id }로 파라미터 바인딩
        var employee = await connection.QueryFirstOrDefaultAsync<Employee>(
            new CommandDefinition(sql, new { Id = id },
                commandTimeout: _commandTimeout,
                cancellationToken: cancellationToken));

        if (employee is null)
        {
            _logger.LogWarning("직원을 찾을 수 없음: Id={EmployeeId}", id);
        }

        return employee;
    }

    // ─────────────────────────────────────────────
    // 생성
    // ─────────────────────────────────────────────

    /// <inheritdoc />
    public async Task<int> CreateAsync(
        Employee employee,
        CancellationToken cancellationToken = default)
    {
        // RETURNING id — PostgreSQL에서 INSERT 후 생성된 ID를 바로 받을 수 있음
        const string sql = """
            INSERT INTO employees (name, department, email, hire_date, is_active, created_at)
            VALUES (@Name, @Department, @Email, @HireDate, @IsActive, NOW())
            RETURNING id
            """;

        _logger.LogDebug("직원 생성: Name={EmployeeName}", employee.Name);

        await using var connection = CreateConnection();

        // ExecuteScalarAsync: INSERT 결과로 반환된 id 값을 가져옴
        // Dapper는 employee 객체의 속성명과 @파라미터명을 자동 매핑
        var newId = await connection.ExecuteScalarAsync<int>(
            new CommandDefinition(sql, employee,
                commandTimeout: _commandTimeout,
                cancellationToken: cancellationToken));

        _logger.LogInformation("직원 생성 완료: Id={EmployeeId}, Name={EmployeeName}",
            newId, employee.Name);

        return newId;
    }

    // ─────────────────────────────────────────────
    // 수정
    // ─────────────────────────────────────────────

    /// <inheritdoc />
    public async Task<bool> UpdateAsync(
        Employee employee,
        CancellationToken cancellationToken = default)
    {
        const string sql = """
            UPDATE employees
            SET
                name       = @Name,
                department = @Department,
                email      = @Email,
                hire_date  = @HireDate,
                is_active  = @IsActive,
                updated_at = NOW()
            WHERE id = @Id
            """;

        _logger.LogDebug("직원 수정: Id={EmployeeId}", employee.Id);

        await using var connection = CreateConnection();

        // ExecuteAsync: 영향받은 행 수를 반환
        var affectedRows = await connection.ExecuteAsync(
            new CommandDefinition(sql, employee,
                commandTimeout: _commandTimeout,
                cancellationToken: cancellationToken));

        // 영향받은 행이 1이면 성공
        var success = affectedRows > 0;

        if (success)
        {
            _logger.LogInformation("직원 수정 완료: Id={EmployeeId}", employee.Id);
        }
        else
        {
            _logger.LogWarning("직원 수정 실패 (존재하지 않음): Id={EmployeeId}", employee.Id);
        }

        return success;
    }

    // ─────────────────────────────────────────────
    // 삭제
    // ─────────────────────────────────────────────

    /// <inheritdoc />
    public async Task<bool> DeleteAsync(
        int id,
        CancellationToken cancellationToken = default)
    {
        const string sql = "DELETE FROM employees WHERE id = @Id";

        _logger.LogDebug("직원 삭제: Id={EmployeeId}", id);

        await using var connection = CreateConnection();

        var affectedRows = await connection.ExecuteAsync(
            new CommandDefinition(sql, new { Id = id },
                commandTimeout: _commandTimeout,
                cancellationToken: cancellationToken));

        var success = affectedRows > 0;

        if (success)
        {
            _logger.LogInformation("직원 삭제 완료: Id={EmployeeId}", id);
        }
        else
        {
            _logger.LogWarning("직원 삭제 실패 (존재하지 않음): Id={EmployeeId}", id);
        }

        return success;
    }
}
```

### 5.5 파라미터 바인딩 상세 설명

Dapper의 파라미터 바인딩은 **익명 객체** 또는 **모델 객체**의 속성명을 SQL의 `@파라미터명`과 자동으로 매핑합니다.

```csharp
// 방법 1: 익명 객체 — 간단한 쿼리에 적합
await connection.QueryAsync<Employee>(
    "SELECT * FROM employees WHERE department = @Dept AND is_active = @Active",
    new { Dept = "개발팀", Active = true }); // @Dept, @Active에 값이 바인딩됨

// 방법 2: 모델 객체 — CRUD에서 모델을 그대로 전달
var employee = new Employee { Id = 1, Name = "홍길동", Department = "개발팀" };
await connection.ExecuteAsync(
    "UPDATE employees SET name = @Name, department = @Department WHERE id = @Id",
    employee); // employee.Name → @Name, employee.Department → @Department, employee.Id → @Id

// 방법 3: DynamicParameters — 동적 파라미터가 필요할 때
var parameters = new DynamicParameters();
parameters.Add("@Name", "홍길동");
parameters.Add("@MinDate", DateTime.Today.AddYears(-1));
await connection.QueryAsync<Employee>(
    "SELECT * FROM employees WHERE name LIKE @Name AND hire_date >= @MinDate",
    parameters);
```

> **SQL 인젝션 방지**: Dapper는 항상 파라미터화된 쿼리를 사용하므로, SQL 인젝션이 자동으로 방지됩니다. 절대 문자열 연결(`$"... WHERE name = '{name}'"`)로 SQL을 만들지 마세요.

### 5.6 트랜잭션 처리

여러 SQL 문을 하나의 트랜잭션으로 묶어야 하는 경우:

```csharp
// Repositories/EmployeeRepository.cs — 트랜잭션 예제 메서드

/// <summary>
/// 직원을 다른 부서로 전환합니다 (트랜잭션 사용).
/// 부서 이동 이력도 함께 기록합니다.
/// 두 작업이 모두 성공해야 커밋되고, 하나라도 실패하면 롤백됩니다.
/// </summary>
public async Task<bool> TransferDepartmentAsync(
    int employeeId,
    string newDepartment,
    CancellationToken cancellationToken = default)
{
    await using var connection = CreateConnection();
    await connection.OpenAsync(cancellationToken);

    // 트랜잭션 시작
    await using var transaction = await connection.BeginTransactionAsync(cancellationToken);

    try
    {
        // 1단계: 현재 부서 조회
        const string selectSql = """
            SELECT department AS Department
            FROM employees
            WHERE id = @Id
            """;

        var currentDepartment = await connection.ExecuteScalarAsync<string>(
            new CommandDefinition(selectSql, new { Id = employeeId },
                transaction: transaction,
                cancellationToken: cancellationToken));

        if (currentDepartment is null)
        {
            _logger.LogWarning("부서 전환 실패: 직원 Id={EmployeeId} 없음", employeeId);
            return false;
        }

        // 2단계: 부서 변경
        const string updateSql = """
            UPDATE employees
            SET department = @NewDept, updated_at = NOW()
            WHERE id = @Id
            """;

        await connection.ExecuteAsync(
            new CommandDefinition(updateSql,
                new { Id = employeeId, NewDept = newDepartment },
                transaction: transaction,
                cancellationToken: cancellationToken));

        // 3단계: 이력 기록
        const string historySql = """
            INSERT INTO department_transfer_history
                (employee_id, from_department, to_department, transferred_at)
            VALUES
                (@EmpId, @FromDept, @ToDept, NOW())
            """;

        await connection.ExecuteAsync(
            new CommandDefinition(historySql,
                new { EmpId = employeeId, FromDept = currentDepartment, ToDept = newDepartment },
                transaction: transaction,
                cancellationToken: cancellationToken));

        // 모든 작업이 성공하면 커밋
        await transaction.CommitAsync(cancellationToken);

        _logger.LogInformation(
            "부서 전환 완료: Id={EmployeeId}, {FromDept} → {ToDept}",
            employeeId, currentDepartment, newDepartment);

        return true;
    }
    catch (Exception ex)
    {
        // 오류 발생 시 롤백 — 모든 변경이 취소됨
        await transaction.RollbackAsync(cancellationToken);
        _logger.LogError(ex, "부서 전환 실패: Id={EmployeeId}", employeeId);
        throw; // 예외를 다시 던져 상위에서 처리
    }
}
```

> **트랜잭션 사용 시 주의**: `connection.OpenAsync()`를 직접 호출해야 합니다. Dapper의 기본 메서드는 연결이 닫혀 있으면 자동으로 열고 닫지만, 트랜잭션 안에서는 연결을 직접 관리해야 합니다.

---

## 6. DI에 Repository 등록

### App.xaml.cs에 등록

```csharp
// App.xaml.cs

using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace MyApp;

public partial class App : Application
{
    private readonly IHost _host;

    public App()
    {
        _host = Host.CreateDefaultBuilder()
            .ConfigureAppConfiguration((context, config) =>
            {
                // appsettings.json 로드
                config.SetBasePath(AppContext.BaseDirectory);
                config.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);
            })
            .ConfigureServices((context, services) =>
            {
                // ── 설정 등록 ──
                services.Configure<DatabaseSettings>(
                    context.Configuration.GetSection(DatabaseSettings.SectionName));

                // ── Repository 등록 ──
                // Transient: 요청할 때마다 새 인스턴스 생성
                // Repository는 상태를 갖지 않으므로 Transient가 적합
                services.AddTransient<IEmployeeRepository, EmployeeRepository>();

                // ── ViewModel 등록 ──
                services.AddTransient<EmployeeViewModel>();

                // ── View (Window) 등록 ──
                services.AddTransient<EmployeeWindow>();
            })
            .Build();
    }

    protected override async void OnStartup(StartupEventArgs e)
    {
        await _host.StartAsync();

        // DI에서 메인 윈도우를 가져와서 표시
        var mainWindow = _host.Services.GetRequiredService<EmployeeWindow>();
        mainWindow.Show();

        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        await _host.StopAsync();
        _host.Dispose();
        base.OnExit(e);
    }
}
```

### Repository의 Lifetime 선택 가이드

| Lifetime | 설명 | Repository에 적합? |
|----------|------|---------------------|
| **Transient** | 요청할 때마다 새 인스턴스 | **적합** — 상태가 없고, 연결은 매번 새로 생성 |
| Scoped | 요청(scope) 당 하나 | 웹에서 주로 사용, WPF에서는 드묾 |
| Singleton | 앱 전체에서 하나 | 가능하지만 주의 필요 (스레드 안전성) |

---

## 7. ViewModel에서 Repository 사용

### 7.1 ViewModel 전체 구현

```csharp
// ViewModels/EmployeeViewModel.cs

using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.Logging;

namespace MyApp.ViewModels;

/// <summary>
/// 직원 관리 화면의 ViewModel.
/// CommunityToolkit.Mvvm의 소스 제네레이터를 활용합니다.
/// </summary>
public partial class EmployeeViewModel : ObservableObject
{
    // ── 의존성 (생성자 주입) ──
    private readonly IEmployeeRepository _repository;
    private readonly ILogger<EmployeeViewModel> _logger;

    /// <summary>
    /// 생성자 — DI에서 Repository와 Logger를 주입받습니다.
    /// WinForms에서는 Form 안에서 직접 DB를 호출했지만,
    /// MVVM에서는 Repository를 통해 간접적으로 접근합니다.
    /// </summary>
    public EmployeeViewModel(
        IEmployeeRepository repository,
        ILogger<EmployeeViewModel> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    // ── 직원 목록 ──

    /// <summary>
    /// 직원 목록. ObservableCollection이므로 항목이 추가/삭제되면
    /// UI(ListView, DataGrid 등)가 자동으로 업데이트됩니다.
    /// </summary>
    public ObservableCollection<Employee> Employees { get; } = [];

    // ── 선택된 직원 ──

    /// <summary>
    /// 현재 선택된 직원. [ObservableProperty]로 자동 변경 알림 생성.
    /// </summary>
    [ObservableProperty]
    private Employee? _selectedEmployee;

    // ── 입력 필드 ──

    /// <summary>직원 이름 입력</summary>
    [ObservableProperty]
    private string _name = string.Empty;

    /// <summary>부서 입력</summary>
    [ObservableProperty]
    private string _department = string.Empty;

    /// <summary>이메일 입력</summary>
    [ObservableProperty]
    private string _email = string.Empty;

    /// <summary>입사일 입력</summary>
    [ObservableProperty]
    private DateTime _hireDate = DateTime.Today;

    // ── 상태 표시 ──

    /// <summary>데이터 로딩 중 여부 (로딩 스피너 표시용)</summary>
    [ObservableProperty]
    private bool _isLoading;

    /// <summary>상태 메시지 (화면 하단에 표시)</summary>
    [ObservableProperty]
    private string _statusMessage = string.Empty;

    // ─────────────────────────────────────────────
    // 명령 (Commands)
    // ─────────────────────────────────────────────

    /// <summary>
    /// 직원 목록을 DB에서 로드합니다.
    /// [RelayCommand]는 LoadEmployeesCommand 속성을 자동 생성합니다.
    /// async Task를 반환하므로 비동기 명령이 됩니다.
    /// </summary>
    [RelayCommand]
    private async Task LoadEmployeesAsync()
    {
        try
        {
            IsLoading = true;
            StatusMessage = "직원 목록을 불러오는 중...";

            // 비동기로 DB에서 데이터를 가져옴 — UI 스레드를 블로킹하지 않음
            var employees = await _repository.GetAllAsync();

            // ObservableCollection은 Clear() + Add()로 갱신
            Employees.Clear();
            foreach (var emp in employees)
            {
                Employees.Add(emp);
            }

            StatusMessage = $"직원 {Employees.Count}명 로드 완료";
            _logger.LogInformation("직원 목록 로드 완료: {Count}명", Employees.Count);
        }
        catch (Exception ex)
        {
            StatusMessage = $"오류: {ex.Message}";
            _logger.LogError(ex, "직원 목록 로드 실패");
        }
        finally
        {
            IsLoading = false;
        }
    }

    /// <summary>
    /// 새 직원을 생성합니다.
    /// </summary>
    [RelayCommand]
    private async Task CreateEmployeeAsync()
    {
        try
        {
            // 입력값 검증
            if (string.IsNullOrWhiteSpace(Name) || string.IsNullOrWhiteSpace(Email))
            {
                StatusMessage = "이름과 이메일은 필수 입력 항목입니다.";
                return;
            }

            IsLoading = true;
            StatusMessage = "직원 생성 중...";

            var newEmployee = new Employee
            {
                Name = Name,
                Department = Department,
                Email = Email,
                HireDate = HireDate,
                IsActive = true
            };

            // DB에 저장하고 생성된 ID를 받음
            var newId = await _repository.CreateAsync(newEmployee);

            StatusMessage = $"직원 '{Name}' 생성 완료 (ID: {newId})";

            // 입력 필드 초기화
            ClearInputFields();

            // 목록 새로고침
            await LoadEmployeesAsync();
        }
        catch (Exception ex)
        {
            StatusMessage = $"직원 생성 실패: {ex.Message}";
            _logger.LogError(ex, "직원 생성 실패: {EmployeeName}", Name);
        }
        finally
        {
            IsLoading = false;
        }
    }

    /// <summary>
    /// 선택된 직원 정보를 수정합니다.
    /// </summary>
    [RelayCommand]
    private async Task UpdateEmployeeAsync()
    {
        try
        {
            if (SelectedEmployee is null)
            {
                StatusMessage = "수정할 직원을 선택하세요.";
                return;
            }

            IsLoading = true;

            // 선택된 직원 객체에 수정된 값을 반영
            SelectedEmployee.Name = Name;
            SelectedEmployee.Department = Department;
            SelectedEmployee.Email = Email;
            SelectedEmployee.HireDate = HireDate;

            var success = await _repository.UpdateAsync(SelectedEmployee);

            StatusMessage = success
                ? $"직원 '{Name}' 수정 완료"
                : "수정할 직원을 찾을 수 없습니다.";

            if (success)
            {
                await LoadEmployeesAsync();
            }
        }
        catch (Exception ex)
        {
            StatusMessage = $"직원 수정 실패: {ex.Message}";
            _logger.LogError(ex, "직원 수정 실패: Id={EmployeeId}", SelectedEmployee?.Id);
        }
        finally
        {
            IsLoading = false;
        }
    }

    /// <summary>
    /// 선택된 직원을 삭제합니다.
    /// </summary>
    [RelayCommand]
    private async Task DeleteEmployeeAsync()
    {
        try
        {
            if (SelectedEmployee is null)
            {
                StatusMessage = "삭제할 직원을 선택하세요.";
                return;
            }

            IsLoading = true;

            var success = await _repository.DeleteAsync(SelectedEmployee.Id);

            StatusMessage = success
                ? $"직원 '{SelectedEmployee.Name}' 삭제 완료"
                : "삭제할 직원을 찾을 수 없습니다.";

            if (success)
            {
                SelectedEmployee = null;
                ClearInputFields();
                await LoadEmployeesAsync();
            }
        }
        catch (Exception ex)
        {
            StatusMessage = $"직원 삭제 실패: {ex.Message}";
            _logger.LogError(ex, "직원 삭제 실패: Id={EmployeeId}", SelectedEmployee?.Id);
        }
        finally
        {
            IsLoading = false;
        }
    }

    // ─────────────────────────────────────────────
    // 속성 변경 처리
    // ─────────────────────────────────────────────

    /// <summary>
    /// SelectedEmployee가 변경될 때 자동 호출됩니다.
    /// (CommunityToolkit.Mvvm의 partial 메서드 활용)
    /// 선택된 직원의 정보를 입력 필드에 채워줍니다.
    /// </summary>
    partial void OnSelectedEmployeeChanged(Employee? value)
    {
        if (value is not null)
        {
            Name = value.Name;
            Department = value.Department;
            Email = value.Email;
            HireDate = value.HireDate;
        }
    }

    // ─────────────────────────────────────────────
    // 유틸리티
    // ─────────────────────────────────────────────

    /// <summary>
    /// 입력 필드를 초기화합니다.
    /// </summary>
    private void ClearInputFields()
    {
        Name = string.Empty;
        Department = string.Empty;
        Email = string.Empty;
        HireDate = DateTime.Today;
    }
}
```

### 7.2 View (XAML) 구현

```xml
<!-- Views/EmployeeWindow.xaml -->
<Window x:Class="MyApp.Views.EmployeeWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        Title="직원 관리" Width="800" Height="600">

    <!--
        Window의 DataContext는 코드비하인드에서 DI를 통해 설정됩니다.
        d:DataContext는 디자인 타임에만 사용됩니다.
    -->

    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>  <!-- 입력 폼 -->
            <RowDefinition Height="Auto"/>  <!-- 버튼 영역 -->
            <RowDefinition Height="*"/>     <!-- 직원 목록 -->
            <RowDefinition Height="Auto"/>  <!-- 상태 바 -->
        </Grid.RowDefinitions>

        <!-- ═══════ 입력 폼 ═══════ -->
        <GroupBox Grid.Row="0" Header="직원 정보 입력" Margin="0,0,0,8">
            <Grid Margin="8">
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="80"/>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="80"/>
                    <ColumnDefinition Width="*"/>
                </Grid.ColumnDefinitions>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                </Grid.RowDefinitions>

                <!-- 이름 -->
                <TextBlock Grid.Row="0" Grid.Column="0" Text="이름:"
                           VerticalAlignment="Center" Margin="0,0,8,4"/>
                <TextBox Grid.Row="0" Grid.Column="1"
                         Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}"
                         Margin="0,0,16,4"/>

                <!-- 부서 -->
                <TextBlock Grid.Row="0" Grid.Column="2" Text="부서:"
                           VerticalAlignment="Center" Margin="0,0,8,4"/>
                <TextBox Grid.Row="0" Grid.Column="3"
                         Text="{Binding Department, UpdateSourceTrigger=PropertyChanged}"
                         Margin="0,0,0,4"/>

                <!-- 이메일 -->
                <TextBlock Grid.Row="1" Grid.Column="0" Text="이메일:"
                           VerticalAlignment="Center" Margin="0,0,8,4"/>
                <TextBox Grid.Row="1" Grid.Column="1"
                         Text="{Binding Email, UpdateSourceTrigger=PropertyChanged}"
                         Margin="0,0,16,4"/>

                <!-- 입사일 -->
                <TextBlock Grid.Row="1" Grid.Column="2" Text="입사일:"
                           VerticalAlignment="Center" Margin="0,0,8,4"/>
                <DatePicker Grid.Row="1" Grid.Column="3"
                            SelectedDate="{Binding HireDate}"
                            Margin="0,0,0,4"/>
            </Grid>
        </GroupBox>

        <!-- ═══════ 버튼 영역 ═══════ -->
        <StackPanel Grid.Row="1" Orientation="Horizontal" Margin="0,0,0,8">
            <!--
                Command 바인딩: ViewModel의 [RelayCommand]가 자동 생성한
                LoadEmployeesCommand, CreateEmployeeCommand 등에 바인딩
            -->
            <Button Content="조회" Command="{Binding LoadEmployeesCommand}"
                    Width="80" Margin="0,0,8,0"/>
            <Button Content="생성" Command="{Binding CreateEmployeeCommand}"
                    Width="80" Margin="0,0,8,0"/>
            <Button Content="수정" Command="{Binding UpdateEmployeeCommand}"
                    Width="80" Margin="0,0,8,0"/>
            <Button Content="삭제" Command="{Binding DeleteEmployeeCommand}"
                    Width="80" Margin="0,0,8,0"/>
        </StackPanel>

        <!-- ═══════ 직원 목록 (DataGrid) ═══════ -->
        <DataGrid Grid.Row="2"
                  ItemsSource="{Binding Employees}"
                  SelectedItem="{Binding SelectedEmployee}"
                  AutoGenerateColumns="False"
                  IsReadOnly="True"
                  SelectionMode="Single"
                  CanUserAddRows="False">
            <DataGrid.Columns>
                <DataGridTextColumn Header="ID" Binding="{Binding Id}" Width="50"/>
                <DataGridTextColumn Header="이름" Binding="{Binding Name}" Width="*"/>
                <DataGridTextColumn Header="부서" Binding="{Binding Department}" Width="100"/>
                <DataGridTextColumn Header="이메일" Binding="{Binding Email}" Width="*"/>
                <DataGridTextColumn Header="입사일"
                                    Binding="{Binding HireDate, StringFormat={}{0:yyyy-MM-dd}}"
                                    Width="100"/>
                <DataGridCheckBoxColumn Header="활성" Binding="{Binding IsActive}" Width="50"/>
            </DataGrid.Columns>
        </DataGrid>

        <!-- ═══════ 상태 바 ═══════ -->
        <StatusBar Grid.Row="3" Margin="0,8,0,0">
            <!-- 로딩 중일 때 ProgressBar 표시 -->
            <ProgressBar IsIndeterminate="{Binding IsLoading}"
                         Width="100" Height="16"
                         Visibility="{Binding IsLoading,
                             Converter={StaticResource BooleanToVisibilityConverter}}"/>
            <TextBlock Text="{Binding StatusMessage}" Margin="8,0,0,0"/>
        </StatusBar>
    </Grid>
</Window>
```

### 7.3 View 코드비하인드

```csharp
// Views/EmployeeWindow.xaml.cs

namespace MyApp.Views;

/// <summary>
/// 직원 관리 화면의 코드비하인드.
/// MVVM에서는 코드비하인드를 최소화하지만,
/// DataContext 설정과 초기 로드는 여기서 합니다.
/// </summary>
public partial class EmployeeWindow : Window
{
    public EmployeeWindow(EmployeeViewModel viewModel)
    {
        InitializeComponent();

        // DI에서 주입받은 ViewModel을 DataContext에 설정
        DataContext = viewModel;

        // 윈도우가 로드된 후 직원 목록을 자동으로 불러옴
        Loaded += async (_, _) =>
        {
            await viewModel.LoadEmployeesCommand.ExecuteAsync(null);
        };
    }
}
```

---

## 8. 전체 CRUD 흐름

### 데이터 흐름 다이어그램

```
┌──────────────────────────────────────────────────────────────────────┐
│                        CREATE (생성) 흐름                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [View]                    [ViewModel]              [Repository]     │
│                                                                      │
│  버튼 클릭               CreateEmployeeCommand       CreateAsync()  │
│     │                         │                         │            │
│     └── Command 바인딩 ──→   입력값 검증               │            │
│                               │                         │            │
│                               └── Employee 객체 생성 ──→ SQL 실행   │
│                                                         │            │
│                               ┌── newId 반환 ──────────┘            │
│                               │                                      │
│                               └── LoadEmployeesAsync() 호출         │
│                                      │                               │
│  DataGrid 자동 갱신 ←───── ObservableCollection 갱신               │
│  StatusMessage 표시 ←───── "생성 완료" 메시지                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                         READ (조회) 흐름                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [View]                    [ViewModel]              [Repository]     │
│                                                                      │
│  조회 버튼 또는          LoadEmployeesCommand       GetAllAsync()   │
│  Window.Loaded                 │                         │           │
│     │                          │                         │           │
│     └── Command 실행 ──→  IsLoading = true              │           │
│                               │                          │           │
│                               └── await 호출 ──────────→ SQL 실행   │
│                                                          │           │
│                               ┌── IEnumerable<Employee> ┘           │
│                               │                                      │
│                               └── Employees.Clear()                  │
│                                   Employees.Add(각 직원)            │
│                                   IsLoading = false                  │
│                                                                      │
│  DataGrid 자동 업데이트 ←── ObservableCollection 변경 알림          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       UPDATE (수정) 흐름                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [View]                    [ViewModel]              [Repository]     │
│                                                                      │
│  DataGrid에서 행 선택   SelectedEmployee 변경             │          │
│     │                         │                           │          │
│     └── 바인딩 ──────→  OnSelectedEmployeeChanged()       │          │
│                          → 입력 필드에 값 채움            │          │
│                                                           │          │
│  입력 필드 수정 후                                        │          │
│  수정 버튼 클릭        UpdateEmployeeCommand               │          │
│     │                         │                           │          │
│     └── Command 실행 ──→  수정된 값을 Employee에 반영     │          │
│                               │                           │          │
│                               └── UpdateAsync() ────────→ SQL 실행  │
│                                                           │          │
│                               ┌── bool (성공 여부) ──────┘          │
│                               │                                      │
│                               └── LoadEmployeesAsync()               │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       DELETE (삭제) 흐름                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [View]                    [ViewModel]              [Repository]     │
│                                                                      │
│  DataGrid에서 행 선택   SelectedEmployee 설정              │         │
│     │                         │                            │         │
│  삭제 버튼 클릭        DeleteEmployeeCommand                │         │
│     │                         │                            │         │
│     └── Command 실행 ──→  SelectedEmployee.Id 확인         │         │
│                               │                            │         │
│                               └── DeleteAsync(id) ───────→ SQL 실행 │
│                                                            │         │
│                               ┌── bool (성공 여부) ───────┘         │
│                               │                                      │
│                               └── SelectedEmployee = null            │
│                                   ClearInputFields()                 │
│                                   LoadEmployeesAsync()               │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### WinForms vs WPF MVVM — CRUD 흐름 비교

| 단계 | WinForms | WPF MVVM |
|------|----------|----------|
| 사용자 입력 | `txtName.Text`로 직접 읽음 | 바인딩으로 자동 동기화 |
| DB 호출 | 이벤트 핸들러에서 직접 | ViewModel → Repository |
| 결과 표시 | `dataGridView.DataSource = dt` | ObservableCollection 자동 반영 |
| 에러 처리 | `MessageBox.Show()` | StatusMessage 바인딩 |
| UI 갱신 | 수동 Refresh | 자동 (변경 알림) |

---

## 9. Connection Pooling

### Connection Pooling이란?

DB 연결을 매번 새로 만들고 닫는 것은 비용이 많이 드는 작업입니다. Connection Pooling은 한번 생성된 연결을 풀(Pool)에 보관했다가, 다음 요청 시 재사용하는 방식입니다.

```
앱 코드                    Connection Pool              PostgreSQL
─────────                  ───────────────              ──────────

new NpgsqlConnection()  →  풀에서 사용 가능한 연결 확인
                            ├─ 있으면 → 기존 연결 재사용
                            └─ 없으면 → 새 연결 생성    → 실제 TCP 연결

connection.Dispose()    →  연결을 풀에 반환 (닫지 않음!)
                            └─ 풀이 가득 차면 → 실제로 닫음
```

### Npgsql의 Connection Pooling

Npgsql은 **기본적으로 Connection Pooling이 활성화**되어 있습니다. 별도 설정 없이 자동으로 동작합니다.

```csharp
// 이 코드가 Connection Pooling을 자동으로 활용합니다
await using var connection = new NpgsqlConnection(connectionString);
// connection이 Dispose되면 실제로 닫히는 게 아니라 풀에 반환됨
```

### 연결 문자열에서 풀 설정

```
Host=localhost;Database=mydb;Username=postgres;Password=secret;
Pooling=true;           // 풀링 활성화 (기본값: true)
MinPoolSize=1;          // 최소 유지 연결 수 (기본값: 0)
MaxPoolSize=100;        // 최대 연결 수 (기본값: 100)
ConnectionIdleLifetime=300;  // 유휴 연결 유지 시간(초) (기본값: 300)
ConnectionPruningInterval=10; // 유휴 연결 정리 주기(초) (기본값: 10)
```

### 주의사항

```csharp
// ❌ 연결을 Dispose하지 않으면 풀에 반환되지 않아 연결 누수 발생!
var conn = new NpgsqlConnection(connStr);
conn.Open();
// ... 사용
// conn.Dispose()를 호출하지 않으면 연결이 풀에 반환되지 않음

// ✅ using 또는 await using을 사용하면 자동으로 Dispose됨
await using var conn = new NpgsqlConnection(connStr);
// 블록이 끝나면 자동으로 풀에 반환
```

---

## 10. 주의사항: UI 스레드에서 DB 호출 금지

### 문제: UI 프리징

WPF(그리고 WinForms도 마찬가지로)에서 모든 UI 작업은 **단일 UI 스레드(메인 스레드)**에서 실행됩니다. UI 스레드에서 시간이 오래 걸리는 작업(DB 쿼리, 파일 I/O, HTTP 호출 등)을 동기적으로 실행하면, 그 작업이 끝날 때까지 **화면이 멈춥니다(프리징)**.

```csharp
// ❌ 동기 호출 — UI가 멈춤!
[RelayCommand]
private void LoadEmployees()
{
    // 이 줄이 실행되는 동안 창을 드래그하거나 버튼을 클릭할 수 없음
    var employees = _repository.GetAll(); // 동기 호출 — UI 스레드 블로킹
    // ...
}

// ❌ .Result나 .Wait()도 마찬가지 — 데드락 위험까지 있음!
[RelayCommand]
private void LoadEmployees()
{
    var employees = _repository.GetAllAsync().Result; // 데드락 가능성!
    // ...
}
```

### 해결: async/await 사용

```csharp
// ✅ 비동기 호출 — UI가 반응하는 상태 유지
[RelayCommand]
private async Task LoadEmployeesAsync()
{
    // await 키워드를 만나면:
    // 1. DB 쿼리를 백그라운드 스레드에서 실행
    // 2. UI 스레드는 해방되어 사용자 입력을 처리
    // 3. 쿼리가 끝나면 UI 스레드로 돌아와서 나머지 코드 실행
    var employees = await _repository.GetAllAsync();

    // await 이후의 코드는 UI 스레드에서 실행되므로
    // ObservableCollection 조작이 안전함
    Employees.Clear();
    foreach (var emp in employees)
    {
        Employees.Add(emp);
    }
}
```

### 동작 원리 다이어그램

```
시간축 →

UI 스레드:    [Command 실행] → [await] ─── 해방 ─── [결과 처리] → [UI 갱신]
                                  │                       ↑
                                  ↓                       │
백그라운드:                   [DB 쿼리 실행.............] ─┘
                              (Npgsql이 비동기 I/O 사용)

사용자 경험:   [버튼 클릭] → [로딩 스피너 표시] → [결과가 DataGrid에 나타남]
                              (UI가 계속 반응함)
```

### CommunityToolkit.Mvvm의 비동기 명령 지원

`[RelayCommand]`를 `async Task` 메서드에 적용하면, CommunityToolkit.Mvvm이 자동으로:

1. `AsyncRelayCommand`를 생성 (기존 `RelayCommand` 대신)
2. 실행 중에는 같은 명령을 다시 실행할 수 없게 함 (이중 클릭 방지)
3. `IsRunning` 속성을 제공하여 로딩 상태 표시에 활용 가능

```csharp
// [RelayCommand]가 자동 생성하는 코드 (참고용)
public IAsyncRelayCommand LoadEmployeesCommand { get; }

// XAML에서 사용
// <Button Command="{Binding LoadEmployeesCommand}" />
```

---

## 요약

| 항목 | 핵심 |
|------|------|
| **Npgsql** | PostgreSQL용 ADO.NET 드라이버. NpgsqlConnection을 사용 |
| **Dapper** | SQL 결과를 C# 객체로 자동 매핑하는 마이크로 ORM |
| **연결 문자열** | appsettings.json + IOptions<T>로 관리 |
| **Repository 패턴** | DB 접근을 인터페이스 뒤에 캡슐화 |
| **DI 등록** | Transient로 Repository를 등록하고 ViewModel에 주입 |
| **비동기** | 모든 DB 호출은 async/await로 수행 |
| **Connection Pooling** | Npgsql이 자동 관리. using으로 연결을 반드시 반환 |
| **UI 스레드** | DB 호출을 동기로 하면 UI가 멈춤. 반드시 async 사용 |

---

> **다음 문서**: [Serilog 구조적 로깅](./02-logging-serilog.md) — 앱의 동작을 추적하고 문제를 진단하는 로깅을 설정합니다.
