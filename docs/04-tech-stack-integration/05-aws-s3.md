# 5. AWS S3를 이용한 사진 저장소

> **목표**: AWSSDK.S3를 사용하여 WPF + MVVM 프로젝트에서 AWS S3에 사진을 업로드/다운로드/삭제하고, 임시 URL을 생성하는 방법을 익힙니다.

---

## 목차

1. [AWS S3란?](#1-aws-s3란)
2. [AWSSDK.S3 NuGet 패키지](#2-awssdks3-nuget-패키지)
3. [AWS 자격 증명 설정](#3-aws-자격-증명-설정)
4. [S3 서비스 구현](#4-s3-서비스-구현)
5. [DI에 등록](#5-di에-등록)
6. [ViewModel에서 사진 업로드/다운로드](#6-viewmodel에서-사진-업로드다운로드)
7. [전체 코드 예제](#7-전체-코드-예제)

---

## 1. AWS S3란?

**Amazon S3 (Simple Storage Service)**는 AWS에서 제공하는 **오브젝트 스토리지** 서비스입니다.

### 오브젝트 스토리지란?

일반 파일 시스템과 달리, 오브젝트 스토리지는 **키(Key)-값(Value)** 구조로 데이터를 저장합니다.

| 개념 | 설명 | 비유 |
|------|------|------|
| **Bucket** | 파일을 담는 최상위 컨테이너 | 드라이브 (C:, D:) |
| **Object** | 저장되는 파일 하나 | 파일 |
| **Key** | 오브젝트의 고유 경로 | 파일 경로 (`photos/2024/image.jpg`) |

### 왜 S3를 쓰나?

- **무제한 저장**: 용량 제한 없음
- **높은 내구성**: 99.999999999% (11 nine) 내구성
- **CDN 연계**: CloudFront로 글로벌 배포 가능
- **비용 효율**: 사용한 만큼만 과금
- **Presigned URL**: 임시 다운로드/업로드 링크 생성 가능

> **WinForms/Python 경험자를 위한 참고**: Python에서 `boto3`를 쓴 경험이 있다면, AWSSDK.S3는 C# 버전의 boto3라고 생각하면 됩니다. `boto3.client('s3').upload_file()`처럼 `AmazonS3Client.PutObjectAsync()`를 호출합니다.

---

## 2. AWSSDK.S3 NuGet 패키지

### 패키지 설치

```bash
# 프로젝트에 AWSSDK.S3 추가
dotnet add package AWSSDK.S3 --version 3.7.405.7

# 설정 파일 바인딩을 위한 패키지 (이미 설치되어 있을 수 있음)
dotnet add package Microsoft.Extensions.Configuration.Binder
```

### .csproj 확인

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0-windows</TargetFramework>
    <UseWPF>true</UseWPF>
    <LangVersion>13.0</LangVersion>
  </PropertyGroup>

  <ItemGroup>
    <!-- AWS S3 SDK -->
    <PackageReference Include="AWSSDK.S3" Version="3.7.405.7" />

    <!-- MVVM 툴킷 -->
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.4.0" />

    <!-- DI 및 설정 -->
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="10.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="10.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="10.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Binder" Version="10.0.0" />
  </ItemGroup>
</Project>
```

---

## 3. AWS 자격 증명 설정

S3에 접근하려면 **자격 증명(Credentials)**이 필요합니다. 방법은 여러 가지가 있습니다.

### 3-1. appsettings.json에 설정

가장 간단한 방법입니다. 개발 환경에서 주로 사용합니다.

```jsonc
// appsettings.json
{
  "AWS": {
    "Region": "ap-northeast-2",          // 서울 리전
    "BucketName": "my-app-photos",       // S3 버킷 이름
    "AccessKey": "AKIAIOSFODNN7EXAMPLE", // IAM 액세스 키
    "SecretKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY", // IAM 시크릿 키
    "Prefix": "photos/"                  // 오브젝트 키 접두사 (폴더처럼 사용)
  }
}
```

> **주의**: `AccessKey`와 `SecretKey`를 소스 코드에 직접 넣거나 Git에 커밋하면 **절대 안 됩니다**. `appsettings.json`은 개발용이며, 배포 시에는 환경 변수나 IAM 역할을 사용하세요. `.gitignore`에 `appsettings.*.json`을 추가하는 것을 권장합니다.

### 3-2. 설정 클래스 정의

```csharp
// Models/AwsSettings.cs
namespace MyApp.Models;

/// <summary>
/// AWS S3 관련 설정을 담는 클래스입니다.
/// appsettings.json의 "AWS" 섹션에 바인딩됩니다.
/// </summary>
public sealed class AwsSettings
{
    /// <summary>AWS 리전 (예: ap-northeast-2)</summary>
    public required string Region { get; init; }

    /// <summary>S3 버킷 이름</summary>
    public required string BucketName { get; init; }

    /// <summary>IAM 액세스 키 (환경 변수 대체 가능)</summary>
    public string? AccessKey { get; init; }

    /// <summary>IAM 시크릿 키 (환경 변수 대체 가능)</summary>
    public string? SecretKey { get; init; }

    /// <summary>오브젝트 키 접두사 (예: "photos/")</summary>
    public string Prefix { get; init; } = string.Empty;
}
```

### 3-3. IAM 사용자/역할

실제 운영 환경에서는 IAM을 사용합니다.

**IAM 사용자 생성 (AWS Console)**:
1. AWS Console → IAM → 사용자 → 사용자 추가
2. 프로그래매틱 액세스 선택
3. 정책 연결: `AmazonS3FullAccess` (또는 최소 권한 커스텀 정책)
4. 액세스 키/시크릿 키 저장

**최소 권한 IAM 정책 예시**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-photos",
        "arn:aws:s3:::my-app-photos/*"
      ]
    }
  ]
}
```

### 3-4. 환경 변수 방식

배포 환경에서 가장 안전한 방법입니다.

```bash
# Windows 환경 변수 설정
set AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
set AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
set AWS_DEFAULT_REGION=ap-northeast-2
```

코드에서 환경 변수를 읽어 자격 증명을 생성하는 방식은 아래 서비스 구현에서 다룹니다.

---

## 4. S3 서비스 구현

### 4-1. IS3StorageService 인터페이스

```csharp
// Services/IS3StorageService.cs
namespace MyApp.Services;

/// <summary>
/// AWS S3 저장소 서비스 인터페이스입니다.
/// 사진 업로드, 다운로드, 삭제, 임시 URL 생성 기능을 정의합니다.
/// </summary>
public interface IS3StorageService
{
    /// <summary>
    /// 파일을 S3에 업로드합니다.
    /// </summary>
    /// <param name="localFilePath">로컬 파일 경로</param>
    /// <param name="s3Key">S3 오브젝트 키 (경로)</param>
    /// <param name="contentType">MIME 타입 (예: "image/jpeg")</param>
    /// <param name="progressCallback">진행률 콜백 (0~100)</param>
    /// <param name="cancellationToken">취소 토큰</param>
    /// <returns>업로드된 오브젝트의 S3 키</returns>
    Task<string> UploadFileAsync(
        string localFilePath,
        string s3Key,
        string contentType = "application/octet-stream",
        Action<int>? progressCallback = null,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// S3에서 파일을 다운로드합니다.
    /// </summary>
    /// <param name="s3Key">S3 오브젝트 키</param>
    /// <param name="localFilePath">저장할 로컬 경로</param>
    /// <param name="progressCallback">진행률 콜백 (0~100)</param>
    /// <param name="cancellationToken">취소 토큰</param>
    Task DownloadFileAsync(
        string s3Key,
        string localFilePath,
        Action<int>? progressCallback = null,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// S3에서 파일을 삭제합니다.
    /// </summary>
    /// <param name="s3Key">삭제할 오브젝트 키</param>
    /// <param name="cancellationToken">취소 토큰</param>
    Task DeleteFileAsync(
        string s3Key,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// 임시(Presigned) URL을 생성합니다.
    /// 이 URL로 인증 없이 일정 시간 동안 파일에 접근할 수 있습니다.
    /// </summary>
    /// <param name="s3Key">오브젝트 키</param>
    /// <param name="expiration">URL 만료 시간</param>
    /// <returns>Presigned URL 문자열</returns>
    string GetPresignedUrl(string s3Key, TimeSpan expiration);
}
```

### 4-2. S3StorageService 구현

```csharp
// Services/S3StorageService.cs
using Amazon;
using Amazon.Runtime;
using Amazon.S3;
using Amazon.S3.Model;
using Amazon.S3.Transfer;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using MyApp.Models;

namespace MyApp.Services;

/// <summary>
/// AWS S3 저장소 서비스 구현입니다.
/// AWSSDK.S3를 사용하여 파일 업로드, 다운로드, 삭제, Presigned URL 생성을 수행합니다.
/// </summary>
public sealed class S3StorageService : IS3StorageService, IDisposable
{
    private readonly IAmazonS3 _s3Client;        // S3 클라이언트
    private readonly AwsSettings _settings;      // AWS 설정
    private readonly ILogger<S3StorageService> _logger;

    public S3StorageService(
        IOptions<AwsSettings> settings,
        ILogger<S3StorageService> logger)
    {
        _settings = settings.Value;
        _logger = logger;

        // S3 클라이언트 생성
        _s3Client = CreateS3Client();

        _logger.LogInformation(
            "S3StorageService 초기화 완료. 버킷: {Bucket}, 리전: {Region}",
            _settings.BucketName, _settings.Region);
    }

    /// <summary>
    /// 자격 증명 방식에 따라 S3 클라이언트를 생성합니다.
    /// AccessKey가 설정되어 있으면 명시적 자격 증명을 사용하고,
    /// 그렇지 않으면 환경 변수 또는 IAM 역할에서 자동으로 자격 증명을 가져옵니다.
    /// </summary>
    private IAmazonS3 CreateS3Client()
    {
        // 리전 설정
        var region = RegionEndpoint.GetBySystemName(_settings.Region);

        // AccessKey가 명시적으로 설정된 경우: 직접 자격 증명 사용
        if (!string.IsNullOrEmpty(_settings.AccessKey) &&
            !string.IsNullOrEmpty(_settings.SecretKey))
        {
            var credentials = new BasicAWSCredentials(
                _settings.AccessKey,
                _settings.SecretKey);

            _logger.LogDebug("명시적 자격 증명으로 S3 클라이언트 생성");
            return new AmazonS3Client(credentials, region);
        }

        // 환경 변수 또는 IAM 역할에서 자동으로 자격 증명을 가져옴
        // (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY 환경 변수 또는
        //  EC2 인스턴스 IAM 역할)
        _logger.LogDebug("기본 자격 증명 체인으로 S3 클라이언트 생성");
        return new AmazonS3Client(region);
    }

    /// <inheritdoc />
    public async Task<string> UploadFileAsync(
        string localFilePath,
        string s3Key,
        string contentType = "application/octet-stream",
        Action<int>? progressCallback = null,
        CancellationToken cancellationToken = default)
    {
        // 전체 키 = 접두사 + 오브젝트 키
        var fullKey = $"{_settings.Prefix}{s3Key}";

        _logger.LogInformation(
            "S3 업로드 시작: {LocalPath} → s3://{Bucket}/{Key}",
            localFilePath, _settings.BucketName, fullKey);

        // TransferUtility는 대용량 파일의 멀티파트 업로드를 자동 처리합니다.
        // 파일 크기가 16MB 이상이면 자동으로 멀티파트 업로드를 사용합니다.
        using var transferUtility = new TransferUtility(_s3Client);

        // 업로드 요청 생성
        var uploadRequest = new TransferUtilityUploadRequest
        {
            FilePath = localFilePath,          // 로컬 파일 경로
            BucketName = _settings.BucketName, // 대상 버킷
            Key = fullKey,                     // S3 오브젝트 키
            ContentType = contentType,         // MIME 타입
            // 서버 측 암호화 설정 (선택사항)
            ServerSideEncryptionMethod = ServerSideEncryptionMethod.AES256
        };

        // 진행률 콜백 등록
        if (progressCallback is not null)
        {
            uploadRequest.UploadProgressEvent += (sender, args) =>
            {
                // TransferredBytes / TotalBytes 비율을 퍼센트로 변환
                var percent = (int)((double)args.TransferredBytes /
                                     args.TotalBytes * 100);
                progressCallback(percent);
            };
        }

        // 실제 업로드 수행
        await transferUtility.UploadAsync(uploadRequest, cancellationToken);

        _logger.LogInformation(
            "S3 업로드 완료: s3://{Bucket}/{Key}",
            _settings.BucketName, fullKey);

        return fullKey;
    }

    /// <inheritdoc />
    public async Task DownloadFileAsync(
        string s3Key,
        string localFilePath,
        Action<int>? progressCallback = null,
        CancellationToken cancellationToken = default)
    {
        var fullKey = $"{_settings.Prefix}{s3Key}";

        _logger.LogInformation(
            "S3 다운로드 시작: s3://{Bucket}/{Key} → {LocalPath}",
            _settings.BucketName, fullKey, localFilePath);

        // 저장할 디렉터리가 없으면 생성
        var directory = Path.GetDirectoryName(localFilePath);
        if (!string.IsNullOrEmpty(directory))
        {
            Directory.CreateDirectory(directory);
        }

        using var transferUtility = new TransferUtility(_s3Client);

        // 다운로드 요청 생성
        var downloadRequest = new TransferUtilityDownloadRequest
        {
            BucketName = _settings.BucketName,
            Key = fullKey,
            FilePath = localFilePath
        };

        // 진행률 콜백 등록
        if (progressCallback is not null)
        {
            downloadRequest.WriteObjectProgressEvent += (sender, args) =>
            {
                var percent = (int)((double)args.TransferredBytes /
                                     args.TotalBytes * 100);
                progressCallback(percent);
            };
        }

        await transferUtility.DownloadAsync(downloadRequest, cancellationToken);

        _logger.LogInformation(
            "S3 다운로드 완료: {LocalPath}",
            localFilePath);
    }

    /// <inheritdoc />
    public async Task DeleteFileAsync(
        string s3Key,
        CancellationToken cancellationToken = default)
    {
        var fullKey = $"{_settings.Prefix}{s3Key}";

        _logger.LogInformation(
            "S3 삭제 시작: s3://{Bucket}/{Key}",
            _settings.BucketName, fullKey);

        // 삭제 요청
        var deleteRequest = new DeleteObjectRequest
        {
            BucketName = _settings.BucketName,
            Key = fullKey
        };

        await _s3Client.DeleteObjectAsync(deleteRequest, cancellationToken);

        _logger.LogInformation(
            "S3 삭제 완료: s3://{Bucket}/{Key}",
            _settings.BucketName, fullKey);
    }

    /// <inheritdoc />
    public string GetPresignedUrl(string s3Key, TimeSpan expiration)
    {
        var fullKey = $"{_settings.Prefix}{s3Key}";

        // Presigned URL 요청 생성
        // 이 URL은 인증 정보가 쿼리 파라미터에 포함되어 있어서
        // URL만 있으면 누구나 파일에 접근할 수 있습니다.
        var request = new GetPreSignedUrlRequest
        {
            BucketName = _settings.BucketName,
            Key = fullKey,
            Expires = DateTime.UtcNow.Add(expiration), // 만료 시간
            Verb = HttpVerb.GET                        // GET 요청용 URL
        };

        var url = _s3Client.GetPreSignedURL(request);

        _logger.LogDebug(
            "Presigned URL 생성: {Key}, 만료: {Expiration}",
            fullKey, expiration);

        return url;
    }

    /// <summary>
    /// S3 클라이언트 리소스를 해제합니다.
    /// </summary>
    public void Dispose()
    {
        _s3Client.Dispose();
        _logger.LogDebug("S3StorageService 리소스 해제 완료");
    }
}
```

### 4-3. 진행률 콜백 상세 설명

```
┌──────────────┐    UploadProgressEvent    ┌──────────────┐
│              │  ─────────────────────────→│              │
│   TransferU  │   TransferredBytes: 5MB   │  ViewModel   │
│   tility     │   TotalBytes: 20MB        │  (콜백 수신) │
│              │   → 25%                   │              │
│              │                            │  UploadProg- │
│  .UploadAs-  │  ─────────────────────────→│  ress = 25   │
│  ync()       │   TransferredBytes: 20MB  │              │
│              │   TotalBytes: 20MB        │  UploadProg- │
│              │   → 100%                  │  ress = 100  │
└──────────────┘                            └──────────────┘
```

진행률 콜백은 **TransferUtility** 내부에서 멀티파트 업로드/다운로드 진행 시 주기적으로 호출됩니다.

> **참고**: 작은 파일(수 KB)은 진행률 이벤트가 한 번만 발생할 수 있습니다. 진행률 표시는 대용량 파일(수 MB 이상)에서 의미가 있습니다.

---

## 5. DI에 등록

```csharp
// App.xaml.cs (또는 DI 설정 파일)
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using MyApp.Models;
using MyApp.Services;
using MyApp.ViewModels;

namespace MyApp;

public partial class App : Application
{
    private readonly ServiceProvider _serviceProvider;

    public App()
    {
        // 설정 파일 로드
        var configuration = new ConfigurationBuilder()
            .SetBasePath(AppContext.BaseDirectory)
            .AddJsonFile("appsettings.json", optional: false)
            .AddJsonFile("appsettings.Development.json", optional: true)
            .AddEnvironmentVariables() // 환경 변수도 읽기
            .Build();

        var services = new ServiceCollection();

        // ──────────────────────────────────────────────
        // AWS 설정을 IOptions<AwsSettings>로 바인딩
        // ──────────────────────────────────────────────
        services.Configure<AwsSettings>(
            configuration.GetSection("AWS"));

        // ──────────────────────────────────────────────
        // S3 서비스 등록 (Singleton: S3 클라이언트를 재사용)
        // ──────────────────────────────────────────────
        services.AddSingleton<IS3StorageService, S3StorageService>();

        // ──────────────────────────────────────────────
        // ViewModel 등록
        // ──────────────────────────────────────────────
        services.AddTransient<PhotoViewModel>();

        // 로깅 등록 (Serilog 등)
        services.AddLogging();

        _serviceProvider = services.BuildServiceProvider();
    }

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        var mainWindow = new MainWindow
        {
            DataContext = _serviceProvider.GetRequiredService<PhotoViewModel>()
        };
        mainWindow.Show();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        // ServiceProvider를 Dispose하면
        // S3StorageService의 Dispose()도 자동 호출됩니다.
        _serviceProvider.Dispose();
        base.OnExit(e);
    }
}
```

---

## 6. ViewModel에서 사진 업로드/다운로드

### 6-1. PhotoViewModel

```csharp
// ViewModels/PhotoViewModel.cs
using System.IO;
using System.Windows.Media.Imaging;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.Logging;
using Microsoft.Win32;
using MyApp.Services;

namespace MyApp.ViewModels;

/// <summary>
/// 사진 업로드/다운로드를 관리하는 ViewModel입니다.
/// CommunityToolkit.Mvvm의 소스 제네레이터를 사용합니다.
/// </summary>
public partial class PhotoViewModel : ObservableObject
{
    private readonly IS3StorageService _s3Service;
    private readonly ILogger<PhotoViewModel> _logger;

    public PhotoViewModel(
        IS3StorageService s3Service,
        ILogger<PhotoViewModel> logger)
    {
        _s3Service = s3Service;
        _logger = logger;
    }

    // ──────────────────────────────────────────────
    // Observable 속성들 (소스 제네레이터가 자동 생성)
    // ──────────────────────────────────────────────

    /// <summary>선택된 로컬 파일 경로</summary>
    [ObservableProperty]
    private string _selectedFilePath = string.Empty;

    /// <summary>업로드 진행률 (0~100)</summary>
    [ObservableProperty]
    private int _uploadProgress;

    /// <summary>다운로드 진행률 (0~100)</summary>
    [ObservableProperty]
    private int _downloadProgress;

    /// <summary>업로드/다운로드 중인지 여부</summary>
    [ObservableProperty]
    private bool _isBusy;

    /// <summary>상태 메시지</summary>
    [ObservableProperty]
    private string _statusMessage = "준비됨";

    /// <summary>다운로드한 이미지 (바인딩용)</summary>
    [ObservableProperty]
    private BitmapImage? _downloadedImage;

    /// <summary>S3에 저장된 오브젝트 키</summary>
    [ObservableProperty]
    private string _currentS3Key = string.Empty;

    /// <summary>Presigned URL</summary>
    [ObservableProperty]
    private string _presignedUrl = string.Empty;

    // ──────────────────────────────────────────────
    // 파일 선택 커맨드
    // ──────────────────────────────────────────────

    /// <summary>
    /// 파일 선택 다이얼로그를 열어 사진 파일을 선택합니다.
    /// </summary>
    [RelayCommand]
    private void SelectFile()
    {
        // WPF의 OpenFileDialog 사용
        // (WinForms의 OpenFileDialog와 사용법이 거의 동일합니다)
        var dialog = new OpenFileDialog
        {
            Title = "사진 파일 선택",
            Filter = "이미지 파일|*.jpg;*.jpeg;*.png;*.bmp;*.gif|모든 파일|*.*",
            InitialDirectory = Environment.GetFolderPath(
                Environment.SpecialFolder.MyPictures)
        };

        // 다이얼로그 표시 (true이면 파일 선택 완료)
        if (dialog.ShowDialog() == true)
        {
            SelectedFilePath = dialog.FileName;
            StatusMessage = $"파일 선택됨: {Path.GetFileName(dialog.FileName)}";
        }
    }

    // ──────────────────────────────────────────────
    // 업로드 커맨드
    // ──────────────────────────────────────────────

    /// <summary>
    /// 선택된 파일을 S3에 업로드합니다.
    /// IsBusy가 아닐 때, 파일이 선택되어 있을 때만 실행 가능합니다.
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanUpload))]
    private async Task UploadPhotoAsync(CancellationToken cancellationToken)
    {
        try
        {
            IsBusy = true;
            UploadProgress = 0;
            StatusMessage = "업로드 중...";

            // 파일명으로 S3 키 생성 (예: "2024/01/15/photo_abc123.jpg")
            var fileName = Path.GetFileName(SelectedFilePath);
            var s3Key = $"{DateTime.Now:yyyy/MM/dd}/{Guid.NewGuid():N}_{fileName}";

            // MIME 타입 결정
            var contentType = GetContentType(fileName);

            // S3에 업로드 (진행률 콜백 전달)
            CurrentS3Key = await _s3Service.UploadFileAsync(
                SelectedFilePath,
                s3Key,
                contentType,
                progressCallback: percent =>
                {
                    // 진행률 콜백은 백그라운드 스레드에서 호출될 수 있으므로
                    // UI 스레드로 디스패치합니다.
                    App.Current.Dispatcher.Invoke(() =>
                    {
                        UploadProgress = percent;
                    });
                },
                cancellationToken);

            StatusMessage = $"업로드 완료: {CurrentS3Key}";
            _logger.LogInformation("사진 업로드 성공: {Key}", CurrentS3Key);
        }
        catch (OperationCanceledException)
        {
            StatusMessage = "업로드가 취소되었습니다.";
            _logger.LogWarning("사진 업로드 취소됨");
        }
        catch (Exception ex)
        {
            StatusMessage = $"업로드 실패: {ex.Message}";
            _logger.LogError(ex, "사진 업로드 실패");
        }
        finally
        {
            IsBusy = false;
        }
    }

    /// <summary>업로드 가능 여부</summary>
    private bool CanUpload() =>
        !IsBusy && !string.IsNullOrEmpty(SelectedFilePath);

    // ──────────────────────────────────────────────
    // 다운로드 커맨드
    // ──────────────────────────────────────────────

    /// <summary>
    /// S3에서 사진을 다운로드하고 이미지를 표시합니다.
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanDownload))]
    private async Task DownloadPhotoAsync(CancellationToken cancellationToken)
    {
        try
        {
            IsBusy = true;
            DownloadProgress = 0;
            StatusMessage = "다운로드 중...";

            // 임시 파일 경로 생성
            var tempPath = Path.Combine(
                Path.GetTempPath(),
                $"s3_download_{Guid.NewGuid():N}{Path.GetExtension(CurrentS3Key)}");

            // S3에서 다운로드 (진행률 콜백 전달)
            await _s3Service.DownloadFileAsync(
                CurrentS3Key,
                tempPath,
                progressCallback: percent =>
                {
                    App.Current.Dispatcher.Invoke(() =>
                    {
                        DownloadProgress = percent;
                    });
                },
                cancellationToken);

            // 다운로드한 파일을 BitmapImage로 로드
            // BitmapImage는 반드시 UI 스레드에서 생성해야 합니다.
            App.Current.Dispatcher.Invoke(() =>
            {
                DownloadedImage = LoadBitmapImage(tempPath);
            });

            StatusMessage = "다운로드 완료! 이미지가 표시됩니다.";
            _logger.LogInformation("사진 다운로드 성공: {Key}", CurrentS3Key);
        }
        catch (OperationCanceledException)
        {
            StatusMessage = "다운로드가 취소되었습니다.";
        }
        catch (Exception ex)
        {
            StatusMessage = $"다운로드 실패: {ex.Message}";
            _logger.LogError(ex, "사진 다운로드 실패");
        }
        finally
        {
            IsBusy = false;
        }
    }

    /// <summary>다운로드 가능 여부</summary>
    private bool CanDownload() =>
        !IsBusy && !string.IsNullOrEmpty(CurrentS3Key);

    // ──────────────────────────────────────────────
    // 삭제 커맨드
    // ──────────────────────────────────────────────

    /// <summary>
    /// S3에서 사진을 삭제합니다.
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanDownload))]
    private async Task DeletePhotoAsync(CancellationToken cancellationToken)
    {
        try
        {
            IsBusy = true;
            StatusMessage = "삭제 중...";

            await _s3Service.DeleteFileAsync(CurrentS3Key, cancellationToken);

            StatusMessage = $"삭제 완료: {CurrentS3Key}";
            CurrentS3Key = string.Empty;
            DownloadedImage = null;
        }
        catch (Exception ex)
        {
            StatusMessage = $"삭제 실패: {ex.Message}";
            _logger.LogError(ex, "사진 삭제 실패");
        }
        finally
        {
            IsBusy = false;
        }
    }

    // ──────────────────────────────────────────────
    // Presigned URL 생성 커맨드
    // ──────────────────────────────────────────────

    /// <summary>
    /// 현재 S3 키에 대한 1시간짜리 임시 URL을 생성합니다.
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanDownload))]
    private void GeneratePresignedUrl()
    {
        try
        {
            // 1시간 동안 유효한 Presigned URL 생성
            PresignedUrl = _s3Service.GetPresignedUrl(
                CurrentS3Key,
                TimeSpan.FromHours(1));

            StatusMessage = "Presigned URL이 생성되었습니다. (1시간 유효)";
        }
        catch (Exception ex)
        {
            StatusMessage = $"URL 생성 실패: {ex.Message}";
            _logger.LogError(ex, "Presigned URL 생성 실패");
        }
    }

    // ──────────────────────────────────────────────
    // 헬퍼 메서드
    // ──────────────────────────────────────────────

    /// <summary>
    /// 파일을 BitmapImage로 로드합니다.
    /// 파일 잠금을 방지하기 위해 스트림을 사용합니다.
    /// </summary>
    private static BitmapImage LoadBitmapImage(string filePath)
    {
        var bitmap = new BitmapImage();
        bitmap.BeginInit();

        // CacheOption을 OnLoad로 설정하면
        // 파일을 메모리로 모두 읽은 후 파일 잠금을 해제합니다.
        bitmap.CacheOption = BitmapCacheOption.OnLoad;

        // 파일 스트림에서 직접 읽기
        bitmap.UriSource = new Uri(filePath, UriKind.Absolute);

        bitmap.EndInit();

        // Freeze()하면 다른 스레드에서도 접근 가능합니다.
        // (WPF에서 스레드 간 공유를 위해 필수)
        bitmap.Freeze();

        return bitmap;
    }

    /// <summary>
    /// 파일 확장자에 따른 MIME 타입을 반환합니다.
    /// </summary>
    private static string GetContentType(string fileName)
    {
        var extension = Path.GetExtension(fileName).ToLowerInvariant();
        return extension switch
        {
            ".jpg" or ".jpeg" => "image/jpeg",
            ".png" => "image/png",
            ".gif" => "image/gif",
            ".bmp" => "image/bmp",
            ".webp" => "image/webp",
            _ => "application/octet-stream" // 기본값
        };
    }
}
```

### 6-2. View (XAML)

```xml
<!-- Views/PhotoView.xaml -->
<Window x:Class="MyApp.Views.PhotoView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        Title="S3 사진 관리"
        Width="600" Height="700"
        WindowStartupLocation="CenterScreen">

    <!-- 디자인 타임에 ViewModel 타입 힌트 (인텔리센스용) -->
    <d:Window.DataContext>
        <vm:PhotoViewModel />
    </d:Window.DataContext>

    <Grid Margin="16">
        <Grid.RowDefinitions>
            <!-- 파일 선택 영역 -->
            <RowDefinition Height="Auto" />
            <!-- 버튼 영역 -->
            <RowDefinition Height="Auto" />
            <!-- 진행률 표시 영역 -->
            <RowDefinition Height="Auto" />
            <!-- 이미지 표시 영역 -->
            <RowDefinition Height="*" />
            <!-- S3 키/URL 정보 -->
            <RowDefinition Height="Auto" />
            <!-- 상태 표시줄 -->
            <RowDefinition Height="Auto" />
        </Grid.RowDefinitions>

        <!-- ① 파일 선택 -->
        <StackPanel Grid.Row="0" Orientation="Horizontal" Margin="0,0,0,8">
            <Button Content="파일 선택..."
                    Command="{Binding SelectFileCommand}"
                    Width="100" Height="32" />
            <TextBlock Text="{Binding SelectedFilePath}"
                       VerticalAlignment="Center"
                       Margin="8,0,0,0"
                       TextTrimming="CharacterEllipsis"
                       MaxWidth="400" />
        </StackPanel>

        <!-- ② 액션 버튼들 -->
        <StackPanel Grid.Row="1" Orientation="Horizontal" Margin="0,0,0,8">
            <!-- 업로드 버튼: CanExecute에 의해 자동 비활성화 -->
            <Button Content="업로드"
                    Command="{Binding UploadPhotoCommand}"
                    Width="80" Height="32" Margin="0,0,8,0" />

            <!-- 다운로드 버튼 -->
            <Button Content="다운로드"
                    Command="{Binding DownloadPhotoCommand}"
                    Width="80" Height="32" Margin="0,0,8,0" />

            <!-- 삭제 버튼 -->
            <Button Content="삭제"
                    Command="{Binding DeletePhotoCommand}"
                    Width="80" Height="32" Margin="0,0,8,0" />

            <!-- Presigned URL 생성 -->
            <Button Content="임시 URL 생성"
                    Command="{Binding GeneratePresignedUrlCommand}"
                    Width="120" Height="32" />
        </StackPanel>

        <!-- ③ 진행률 표시 -->
        <StackPanel Grid.Row="2" Margin="0,0,0,8">
            <!-- 업로드 진행률 -->
            <TextBlock Text="업로드:" Margin="0,0,0,2" />
            <ProgressBar Value="{Binding UploadProgress}"
                         Minimum="0" Maximum="100"
                         Height="18" Margin="0,0,0,4" />

            <!-- 다운로드 진행률 -->
            <TextBlock Text="다운로드:" Margin="0,4,0,2" />
            <ProgressBar Value="{Binding DownloadProgress}"
                         Minimum="0" Maximum="100"
                         Height="18" />
        </StackPanel>

        <!-- ④ 다운로드한 이미지 표시 -->
        <Border Grid.Row="3"
                BorderBrush="Gray" BorderThickness="1"
                Margin="0,0,0,8">
            <!-- 이미지가 null이면 빈 영역으로 표시됩니다 -->
            <Image Source="{Binding DownloadedImage}"
                   Stretch="Uniform"
                   RenderOptions.BitmapScalingMode="HighQuality" />
        </Border>

        <!-- ⑤ S3 키 및 Presigned URL -->
        <StackPanel Grid.Row="4" Margin="0,0,0,8">
            <TextBlock>
                <Run Text="S3 Key: " FontWeight="Bold" />
                <Run Text="{Binding CurrentS3Key, Mode=OneWay}" />
            </TextBlock>
            <TextBlock TextWrapping="Wrap" Margin="0,4,0,0">
                <Run Text="Presigned URL: " FontWeight="Bold" />
                <Run Text="{Binding PresignedUrl, Mode=OneWay}" />
            </TextBlock>
        </StackPanel>

        <!-- ⑥ 상태 표시줄 -->
        <StatusBar Grid.Row="5">
            <StatusBarItem>
                <TextBlock Text="{Binding StatusMessage}" />
            </StatusBarItem>
            <!-- IsBusy일 때 로딩 인디케이터 표시 -->
            <StatusBarItem HorizontalAlignment="Right"
                           Visibility="{Binding IsBusy,
                               Converter={StaticResource BoolToVisibilityConverter}}">
                <ProgressBar IsIndeterminate="True"
                             Width="100" Height="14" />
            </StatusBarItem>
        </StatusBar>
    </Grid>
</Window>
```

> **WinForms 경험자를 위한 비교**: WinForms에서는 `pictureBox1.Image = Image.FromFile(path)`로 이미지를 표시했지만, WPF에서는 `BitmapImage`를 ViewModel의 속성에 바인딩합니다. View의 `<Image Source="{Binding DownloadedImage}" />`가 자동으로 이미지를 표시합니다.

---

## 7. 전체 코드 예제

아래는 전체 흐름을 한눈에 보여주는 통합 예제입니다.

### 프로젝트 구조

```
MyApp/
├── Models/
│   └── AwsSettings.cs          ← AWS 설정 클래스
├── Services/
│   ├── IS3StorageService.cs    ← 인터페이스
│   └── S3StorageService.cs     ← S3 서비스 구현
├── ViewModels/
│   └── PhotoViewModel.cs       ← 사진 관리 ViewModel
├── Views/
│   └── PhotoView.xaml          ← 사진 관리 화면
├── App.xaml.cs                 ← DI 설정
└── appsettings.json            ← AWS 자격 증명 설정
```

### 데이터 흐름 요약

```
┌─────────────────────────────────────────────────────────────┐
│                          View (XAML)                         │
│  [파일 선택] → OpenFileDialog → SelectedFilePath 업데이트    │
│  [업로드]    → UploadPhotoCommand 실행                       │
│  [다운로드]  → DownloadPhotoCommand 실행                     │
│  ProgressBar ← UploadProgress / DownloadProgress 바인딩     │
│  Image       ← DownloadedImage 바인딩                       │
└───────────────────────┬─────────────────────────────────────┘
                        │ 데이터 바인딩 (양방향)
┌───────────────────────▼─────────────────────────────────────┐
│                     PhotoViewModel                           │
│  UploadPhotoAsync()  → _s3Service.UploadFileAsync()         │
│  DownloadPhotoAsync()→ _s3Service.DownloadFileAsync()       │
│  진행률 콜백         → Dispatcher.Invoke → UploadProgress    │
└───────────────────────┬─────────────────────────────────────┘
                        │ DI (생성자 주입)
┌───────────────────────▼─────────────────────────────────────┐
│                    S3StorageService                           │
│  TransferUtility     → AmazonS3Client                        │
│  UploadAsync()       → S3 PutObject (멀티파트)               │
│  DownloadAsync()     → S3 GetObject                          │
│  GetPreSignedURL()   → Presigned URL 생성                    │
└───────────────────────┬─────────────────────────────────────┘
                        │ HTTPS
┌───────────────────────▼─────────────────────────────────────┐
│                      AWS S3                                  │
│  Bucket: my-app-photos                                       │
│  Object: photos/2024/01/15/abc123_photo.jpg                  │
└─────────────────────────────────────────────────────────────┘
```

### 핵심 포인트 정리

| 항목 | 설명 |
|------|------|
| **인터페이스 분리** | `IS3StorageService`로 추상화하여 테스트 용이성 확보 |
| **DI 등록** | `Singleton`으로 등록 (S3 클라이언트를 재사용) |
| **진행률 콜백** | `TransferUtility`의 이벤트를 `Action<int>`로 래핑 |
| **UI 스레드** | `Dispatcher.Invoke()`로 진행률을 UI 스레드에서 업데이트 |
| **BitmapImage** | `Freeze()`로 스레드 간 공유 가능하게 처리 |
| **Presigned URL** | 인증 없이 임시로 파일에 접근할 수 있는 URL |
| **환경별 자격 증명** | 개발(`appsettings.json`) / 운영(환경 변수·IAM 역할) 분리 |

> **다음 단계**: [시스템 트레이 아이콘 통합](./06-system-tray.md)에서는 앱을 시스템 트레이에 최소화하고, 풍선 팝업으로 알림을 표시하는 방법을 다룹니다.
