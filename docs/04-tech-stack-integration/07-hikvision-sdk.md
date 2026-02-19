# 7. Hikvision HCNetSDK P/Invoke 연동

> **목표**: Hikvision HCNetSDK (비관리 C/C++ DLL)를 P/Invoke로 호출하여 얼굴인식 단말기와 연동하고, MVVM 패턴의 서비스로 래핑하여 ViewModel에서 이벤트를 수신하는 방법을 익힙니다.

---

## 목차

1. [P/Invoke란?](#1-pinvoke란)
2. [Hikvision HCNetSDK 개요](#2-hikvision-hcnetsdk-개요)
3. [DLL 준비](#3-dll-준비)
4. [P/Invoke 선언](#4-pinvoke-선언)
5. [서비스 패턴으로 래핑](#5-서비스-패턴으로-래핑)
6. [DI에 등록 (Singleton)](#6-di에-등록-singleton)
7. [ViewModel에서 얼굴인식 이벤트 수신](#7-viewmodel에서-얼굴인식-이벤트-수신)
8. [메모리 관리 주의사항](#8-메모리-관리-주의사항)
9. [전체 코드 예제](#9-전체-코드-예제)

---

## 1. P/Invoke란?

**P/Invoke (Platform Invocation Services)**는 C#(관리 코드)에서 **C/C++ DLL(비관리 코드)**의 함수를 호출하는 .NET 기술입니다.

### WinForms 경험이 있다면

WinForms에서 Win32 API를 호출한 경험이 있을 수 있습니다:

```csharp
// WinForms에서 흔히 볼 수 있는 P/Invoke 예제
[DllImport("user32.dll")]
static extern bool ShowWindow(IntPtr hWnd, int nCmdShow);
```

Hikvision SDK 연동도 **동일한 원리**입니다. 다만 호출할 함수와 구조체가 훨씬 많고 복잡합니다.

### 관리 코드 vs 비관리 코드

```
┌─────────────────────────────────┐
│        관리 코드 (C#/.NET)       │
│  ┌───────────────────────────┐  │
│  │    HikvisionService.cs    │  │
│  │    (우리가 만드는 서비스)   │  │
│  └─────────┬─────────────────┘  │
│            │ P/Invoke (DllImport)│
│  ┌─────────▼─────────────────┐  │
│  │   NativeMethods.cs        │  │
│  │   (DllImport 선언)        │  │
│  └─────────┬─────────────────┘  │
└────────────┼────────────────────┘
             │ 비관리 함수 호출
┌────────────▼────────────────────┐
│      비관리 코드 (C/C++)         │
│  ┌───────────────────────────┐  │
│  │     HCNetSDK.dll          │  │
│  │   (Hikvision 제공 DLL)    │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### P/Invoke의 핵심 요소

| 요소 | 설명 | C# 문법 |
|------|------|---------|
| **DllImport** | 호출할 DLL과 함수를 선언 | `[DllImport("HCNetSDK.dll")]` |
| **구조체 매핑** | C 구조체를 C# 구조체로 정의 | `[StructLayout(LayoutKind.Sequential)]` |
| **콜백 함수** | C에서 C#을 역호출 | `delegate`, `UnmanagedFunctionPointer` |
| **마샬링** | 관리↔비관리 간 데이터 변환 | `MarshalAs`, `Marshal.PtrToStructure` |

---

## 2. Hikvision HCNetSDK 개요

**HCNetSDK**는 Hikvision(하이크비전)에서 제공하는 C/C++ SDK로, NVR, DVR, IP 카메라, **얼굴인식 단말기** 등 Hikvision 장비와 통신하는 데 사용됩니다.

### 얼굴인식 단말기 연동 흐름

```
① SDK 초기화          NET_DVR_Init()
       │
② 장비 로그인          NET_DVR_Login_V40()
       │
③ 알람 채널 설정       NET_DVR_SetupAlarmChan_V41()
       │                   ↓
④ 콜백 등록            MSGCallBack_V31 (얼굴인식 이벤트 수신)
       │                   ↓
⑤ 이벤트 수신 대기     ← 장비에서 얼굴 인식 시 자동으로 콜백 호출
       │
⑥ 알람 채널 해제       NET_DVR_CloseAlarmChan_V30()
       │
⑦ 로그아웃             NET_DVR_Logout()
       │
⑧ SDK 정리             NET_DVR_Cleanup()
```

---

## 3. DLL 준비

### 3-1. HCNetSDK.dll 및 종속 DLL 배치

Hikvision SDK를 다운로드하면 다음과 같은 DLL 파일들이 포함됩니다:

```
HCNetSDK/
├── HCNetSDK.dll          ← 메인 SDK DLL
├── HCCore.dll            ← 코어 라이브러리
├── HCNetSDKCom/          ← 부가 컴포넌트
│   ├── HCAlarm.dll
│   ├── HCGeneralCfgMgr.dll
│   ├── HCPreview.dll
│   └── ...
├── libssl-1_1-x64.dll    ← OpenSSL (x64)
├── libcrypto-1_1-x64.dll
└── ...
```

### 3-2. x64/x86 구분

Hikvision SDK는 x64와 x86 버전이 따로 있습니다. **.NET 10.0에서는 기본적으로 x64**를 사용합니다.

```xml
<!-- .csproj에서 플랫폼 명시 -->
<PropertyGroup>
  <TargetFramework>net10.0-windows</TargetFramework>
  <UseWPF>true</UseWPF>
  <LangVersion>13.0</LangVersion>
  <!-- x64만 지원하도록 설정 -->
  <PlatformTarget>x64</PlatformTarget>
</PropertyGroup>
```

### 3-3. 프로젝트에 DLL 포함 방법

```xml
<!-- .csproj에 DLL 파일을 출력 디렉터리에 복사하도록 설정 -->
<ItemGroup>
  <!-- 메인 DLL -->
  <None Include="Libs\HCNetSDK\HCNetSDK.dll">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
  <None Include="Libs\HCNetSDK\HCCore.dll">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>

  <!-- 종속 DLL 일괄 복사 -->
  <None Include="Libs\HCNetSDK\HCNetSDKCom\*.*">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    <Link>HCNetSDKCom\%(Filename)%(Extension)</Link>
  </None>

  <!-- OpenSSL 등 종속 라이브러리 -->
  <None Include="Libs\HCNetSDK\libssl-1_1-x64.dll">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
  <None Include="Libs\HCNetSDK\libcrypto-1_1-x64.dll">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
</ItemGroup>
```

디렉터리 구조:

```
MyApp/
├── Libs/
│   └── HCNetSDK/              ← SDK DLL 파일들
│       ├── HCNetSDK.dll
│       ├── HCCore.dll
│       ├── HCNetSDKCom/
│       │   ├── HCAlarm.dll
│       │   └── ...
│       ├── libssl-1_1-x64.dll
│       └── libcrypto-1_1-x64.dll
├── Native/
│   └── NativeMethods.cs       ← P/Invoke 선언
├── Services/
│   ├── IHikvisionService.cs
│   └── HikvisionService.cs
└── ...
```

> **주의**: SDK DLL 파일들은 반드시 **실행 파일(.exe)과 같은 디렉터리**에 있어야 합니다. `CopyToOutputDirectory`를 설정하지 않으면 `DllNotFoundException`이 발생합니다.

---

## 4. P/Invoke 선언

### 4-1. NativeMethods 정적 클래스

P/Invoke 선언은 별도의 정적 클래스에 모아두는 것이 관례입니다.

```csharp
// Native/NativeMethods.cs
using System.Runtime.InteropServices;

namespace MyApp.Native;

/// <summary>
/// Hikvision HCNetSDK의 P/Invoke 선언 모음입니다.
/// 모든 네이티브(비관리) 함수 호출을 이 클래스에서 관리합니다.
///
/// 참고: HCNetSDK 프로그래밍 가이드의 함수 시그니처를
///       C#으로 변환한 것입니다.
/// </summary>
internal static partial class NativeMethods
{
    // DLL 이름 상수
    private const string SdkDll = "HCNetSDK.dll";

    // ──────────────────────────────────────────────
    // 콜백 함수 delegate 정의
    // ──────────────────────────────────────────────

    /// <summary>
    /// 알람/이벤트 수신 콜백 함수입니다.
    /// 장비에서 얼굴인식 등 이벤트가 발생하면 이 콜백이 호출됩니다.
    ///
    /// C 원형:
    /// typedef void(CALLBACK *MSGCallBack_V31)(
    ///     LONG lCommand, NET_DVR_ALARMER *pAlarmer,
    ///     char *pAlarmInfo, DWORD dwBufLen, void* pUser);
    /// </summary>
    [UnmanagedFunctionPointer(CallingConvention.StdCall)]
    internal delegate void MSGCallBack_V31(
        int lCommand,           // 명령 타입 (어떤 종류의 이벤트인지)
        ref NET_DVR_ALARMER pAlarmer,  // 알람 발생 장비 정보
        IntPtr pAlarmInfo,      // 알람 상세 정보 (명령 타입에 따라 다름)
        uint dwBufLen,          // 알람 정보 버퍼 길이
        IntPtr pUser);          // 사용자 데이터 포인터

    /// <summary>
    /// 예외 콜백 함수입니다.
    /// 네트워크 연결 끊김 등 비정상 상황 시 호출됩니다.
    /// </summary>
    [UnmanagedFunctionPointer(CallingConvention.StdCall)]
    internal delegate void ExceptionCallBack(
        uint dwType,            // 예외 타입
        int lUserID,            // 사용자(장비 로그인) ID
        int lHandle,            // 핸들
        IntPtr pUser);          // 사용자 데이터 포인터

    // ──────────────────────────────────────────────
    // SDK 초기화 / 정리 함수
    // ──────────────────────────────────────────────

    /// <summary>
    /// SDK를 초기화합니다. 모든 SDK 함수 호출 전에 반드시 호출해야 합니다.
    /// C 원형: BOOL NET_DVR_Init();
    /// </summary>
    [DllImport(SdkDll)]
    internal static extern bool NET_DVR_Init();

    /// <summary>
    /// SDK 리소스를 정리합니다. 앱 종료 시 호출합니다.
    /// C 원형: BOOL NET_DVR_Cleanup();
    /// </summary>
    [DllImport(SdkDll)]
    internal static extern bool NET_DVR_Cleanup();

    /// <summary>
    /// 마지막 에러 코드를 가져옵니다.
    /// C 원형: DWORD NET_DVR_GetLastError();
    /// </summary>
    [DllImport(SdkDll)]
    internal static extern uint NET_DVR_GetLastError();

    /// <summary>
    /// SDK 로그 파일을 설정합니다. 디버깅 시 유용합니다.
    /// C 원형: BOOL NET_DVR_SetLogToFile(
    ///     DWORD nLogLevel, char *strLogDir,
    ///     BOOL bAutoDel);
    /// </summary>
    [DllImport(SdkDll)]
    internal static extern bool NET_DVR_SetLogToFile(
        int nLogLevel,
        string strLogDir,
        bool bAutoDel);

    // ──────────────────────────────────────────────
    // 알람/이벤트 콜백 설정
    // ──────────────────────────────────────────────

    /// <summary>
    /// 알람/이벤트 콜백 함수를 등록합니다.
    /// C 원형: BOOL NET_DVR_SetDVRMessageCallBack_V31(
    ///     MSGCallBack_V31 fMessageCallBack, void* pUser);
    /// </summary>
    [DllImport(SdkDll)]
    internal static extern bool NET_DVR_SetDVRMessageCallBack_V31(
        MSGCallBack_V31 fMessageCallBack,
        IntPtr pUser);

    // ──────────────────────────────────────────────
    // 장비 로그인 / 로그아웃
    // ──────────────────────────────────────────────

    /// <summary>
    /// 장비에 로그인합니다 (V40 버전 - 비동기 지원).
    /// C 원형: LONG NET_DVR_Login_V40(
    ///     NET_DVR_USER_LOGIN_INFO *pLoginInfo,
    ///     NET_DVR_DEVICEINFO_V40 *lpDeviceInfo);
    /// </summary>
    [DllImport(SdkDll)]
    internal static extern int NET_DVR_Login_V40(
        ref NET_DVR_USER_LOGIN_INFO pLoginInfo,
        ref NET_DVR_DEVICEINFO_V40 lpDeviceInfo);

    /// <summary>
    /// 장비에서 로그아웃합니다.
    /// C 원형: BOOL NET_DVR_Logout(LONG lUserID);
    /// </summary>
    [DllImport(SdkDll)]
    internal static extern bool NET_DVR_Logout(int lUserID);

    // ──────────────────────────────────────────────
    // 알람 채널 설정 / 해제
    // ──────────────────────────────────────────────

    /// <summary>
    /// 알람 채널을 설정합니다 (V41 버전).
    /// 이 함수를 호출해야 장비의 이벤트(얼굴인식 등)를 수신할 수 있습니다.
    /// C 원형: LONG NET_DVR_SetupAlarmChan_V41(
    ///     NET_DVR_SETUPALARM_PARAM_V50 *lpSetupParam);
    /// </summary>
    [DllImport(SdkDll)]
    internal static extern int NET_DVR_SetupAlarmChan_V41(
        ref NET_DVR_SETUPALARM_PARAM_V50 lpSetupParam);

    /// <summary>
    /// 알람 채널을 해제합니다.
    /// C 원형: BOOL NET_DVR_CloseAlarmChan_V30(LONG lAlarmHandle);
    /// </summary>
    [DllImport(SdkDll)]
    internal static extern bool NET_DVR_CloseAlarmChan_V30(int lAlarmHandle);

    // ──────────────────────────────────────────────
    // 예외 콜백 설정
    // ──────────────────────────────────────────────

    /// <summary>
    /// 예외 콜백 함수를 등록합니다.
    /// </summary>
    [DllImport(SdkDll)]
    internal static extern bool NET_DVR_SetExceptionCallBack_V30(
        uint nMessage,
        IntPtr hWnd,
        ExceptionCallBack fExceptionCallBack,
        IntPtr pUser);
}
```

### 4-2. 구조체 정의 (StructLayout)

C의 구조체를 C#으로 변환합니다. **메모리 레이아웃을 정확히 맞추는 것이 핵심**입니다.

```csharp
// Native/NativeStructs.cs
using System.Runtime.InteropServices;

namespace MyApp.Native;

// ──────────────────────────────────────────────
// 상수 정의
// ──────────────────────────────────────────────

/// <summary>
/// HCNetSDK에서 사용하는 상수 모음입니다.
/// </summary>
internal static class HikConst
{
    // 이름/IP 주소 최대 길이
    public const int NET_DVR_DEV_ADDRESS_MAX_LEN = 129;
    public const int NET_DVR_LOGIN_USERNAME_MAX_LEN = 64;
    public const int NET_DVR_LOGIN_PASSWD_MAX_LEN = 64;

    // 알람 명령 타입
    public const int COMM_ALARM_ACS = 0x5002;           // 출입통제 알람
    public const int COMM_SNAP_MATCH_ALARM = 0x2902;    // 얼굴 비교 결과
    public const int COMM_FACE_SNAP = 0x1112;           // 얼굴 캡처 이벤트
    public const int COMM_FACESNAP_RESULT = 0x1112;     // 얼굴 캡처 결과

    // 예외 타입
    public const uint EXCEPTION_RECONNECT = 0x8005;     // 재연결 발생
    public const uint EXCEPTION_EXCHANGE = 0x800a;      // 사용자 교체
}

// ──────────────────────────────────────────────
// 로그인 관련 구조체
// ──────────────────────────────────────────────

/// <summary>
/// 장비 로그인 정보 구조체입니다.
/// C 원형: NET_DVR_USER_LOGIN_INFO
///
/// [StructLayout(LayoutKind.Sequential)] 은
/// C 구조체와 동일한 메모리 레이아웃을 보장합니다.
/// </summary>
[StructLayout(LayoutKind.Sequential)]
internal struct NET_DVR_USER_LOGIN_INFO
{
    /// <summary>장비 IP 주소 (최대 129바이트)</summary>
    [MarshalAs(UnmanagedType.ByValTStr,
        SizeConst = HikConst.NET_DVR_DEV_ADDRESS_MAX_LEN)]
    public string sDeviceAddress;

    /// <summary>포트 번호</summary>
    public ushort wPort;

    /// <summary>사용자 이름 (최대 64바이트)</summary>
    [MarshalAs(UnmanagedType.ByValTStr,
        SizeConst = HikConst.NET_DVR_LOGIN_USERNAME_MAX_LEN)]
    public string sUserName;

    /// <summary>비밀번호 (최대 64바이트)</summary>
    [MarshalAs(UnmanagedType.ByValTStr,
        SizeConst = HikConst.NET_DVR_LOGIN_PASSWD_MAX_LEN)]
    public string sPassword;

    /// <summary>로그인 모드: 0 = TCP, 1 = UDP</summary>
    public byte byLoginMode;

    /// <summary>HTTPS 포트 사용 여부</summary>
    public byte byHttps;

    /// <summary>프록시 타입: 0 = 직접 연결, 1 = 프록시</summary>
    public int iProxyID;

    /// <summary>TLS 사용 여부</summary>
    public byte byVerifyMode;
}

/// <summary>
/// 장비 정보 응답 구조체입니다 (V40).
/// 로그인 성공 시 장비 정보가 이 구조체에 채워집니다.
/// </summary>
[StructLayout(LayoutKind.Sequential)]
internal struct NET_DVR_DEVICEINFO_V40
{
    /// <summary>장비 기본 정보 (V30 호환)</summary>
    public NET_DVR_DEVICEINFO_V30 struDeviceV30;

    /// <summary>보안 상태</summary>
    public byte bySupportLock;

    /// <summary>남은 잠금 시도 횟수</summary>
    public byte byRetryLoginTime;

    /// <summary>비밀번호 리셋 기능 지원 여부</summary>
    public byte byPasswordLevel;

    /// <summary>프록시 타입</summary>
    public byte byProxyType;

    /// <summary>남은 잠금 시간 (초)</summary>
    public uint dwSurplusLockTime;

    /// <summary>문자 인코딩 방식</summary>
    public byte byCharEncodeType;

    /// <summary>이후 버전 호환을 위한 여유 공간</summary>
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 253)]
    public byte[] byRes2;
}

/// <summary>
/// 장비 기본 정보 구조체 (V30).
/// </summary>
[StructLayout(LayoutKind.Sequential)]
internal struct NET_DVR_DEVICEINFO_V30
{
    /// <summary>장비 시리얼 번호</summary>
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 48)]
    public byte[] sSerialNumber;

    /// <summary>알람 입력 수</summary>
    public byte byAlarmInPortNum;

    /// <summary>알람 출력 수</summary>
    public byte byAlarmOutPortNum;

    /// <summary>디스크 수</summary>
    public byte byDiskNum;

    /// <summary>장비 타입</summary>
    public byte byDVRType;

    /// <summary>채널 수</summary>
    public byte byChanNum;

    /// <summary>시작 채널 번호</summary>
    public byte byStartChan;

    /// <summary>오디오 채널 수</summary>
    public byte byAudioChanNum;

    /// <summary>IP 채널 수</summary>
    public byte byIPChanNum;

    /// <summary>0주 채널 수</summary>
    public byte byZeroChanNum;

    /// <summary>메인 프로토콜 버전 (예: V3.0 → byMainProto=3)</summary>
    public byte byMainProto;

    /// <summary>서브 프로토콜 버전</summary>
    public byte bySubProto;

    /// <summary>지원 기능</summary>
    public byte bySupport;

    /// <summary>추가 지원 기능</summary>
    public byte bySupport1;

    /// <summary>추가 지원 기능 2</summary>
    public byte bySupport2;

    /// <summary>장비 타입 (확장)</summary>
    public ushort wDevType;

    /// <summary>추가 지원 기능 3</summary>
    public byte bySupport3;

    /// <summary>다중 스트림 지원</summary>
    public byte byMultiStreamProto;

    /// <summary>시작 디지털 채널 번호</summary>
    public byte byStartDChan;

    /// <summary>시작 오디오 채널 번호</summary>
    public byte byStartDTalkChan;

    /// <summary>높은 자릿수 디지털 채널 수</summary>
    public byte byHighDChanNum;

    /// <summary>추가 지원 기능 4</summary>
    public byte bySupport4;

    /// <summary>언어 타입</summary>
    public byte byLanguageType;

    /// <summary>음성 대화 채널 수</summary>
    public byte byVoiceInNum;

    /// <summary>메인 채널 시작 번호</summary>
    public byte byStartMainChan;

    /// <summary>서브 채널 시작 번호</summary>
    public byte byStartSubChan;

    /// <summary>추가 지원 기능 5</summary>
    public byte bySupport5;

    /// <summary>추가 지원 기능 6</summary>
    public byte bySupport6;

    /// <summary>미러 채널 수</summary>
    public byte byMirrorChanNum;

    /// <summary>시작 미러 채널 번호</summary>
    public ushort wStartMirrorChanNo;

    /// <summary>추가 지원 기능 7</summary>
    public byte bySupport7;

    /// <summary>예약 필드</summary>
    public byte byRes2;
}

// ──────────────────────────────────────────────
// 알람 관련 구조체
// ──────────────────────────────────────────────

/// <summary>
/// 알람 설정 파라미터 구조체 (V50).
/// </summary>
[StructLayout(LayoutKind.Sequential)]
internal struct NET_DVR_SETUPALARM_PARAM_V50
{
    /// <summary>구조체 크기 (반드시 설정해야 함)</summary>
    public uint dwSize;

    /// <summary>알람 등급: 0 = 1등급, 1 = 2등급</summary>
    public byte byLevel;

    /// <summary>대역폭 수 (0 = 전체, 1~N = 선택)</summary>
    public byte byAlarmInfoType;

    /// <summary>배포 타입</summary>
    public byte byDeployType;

    /// <summary>예약</summary>
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 29)]
    public byte[] byRes;

    /// <summary>로그인 사용자 ID</summary>
    public int lUserID;

    /// <summary>알람 핸들</summary>
    public int lAlarmHandle;
}

/// <summary>
/// 알람 발생 장비 정보 구조체입니다.
/// 콜백 함수에서 어떤 장비가 알람을 발생시켰는지 확인할 때 사용합니다.
/// </summary>
[StructLayout(LayoutKind.Sequential)]
internal struct NET_DVR_ALARMER
{
    /// <summary>사용자 ID 유효 여부</summary>
    public byte byUserIDValid;

    /// <summary>시리얼 번호 유효 여부</summary>
    public byte bySerialValid;

    /// <summary>SDK 버전 유효 여부</summary>
    public byte byVersionValid;

    /// <summary>장비 이름 유효 여부</summary>
    public byte byDeviceNameValid;

    /// <summary>MAC 유효 여부</summary>
    public byte byMacAddrValid;

    /// <summary>로그인 사용자 ID</summary>
    public byte byLinkPortValid;

    /// <summary>장비 IP 유효 여부</summary>
    public byte byDeviceIPValid;

    /// <summary>소켓 IP 유효 여부</summary>
    public byte bySocketIPValid;

    /// <summary>사용자 ID</summary>
    public int lUserID;

    /// <summary>시리얼 번호</summary>
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 48)]
    public byte[] sSerialNumber;

    /// <summary>SDK 버전</summary>
    public uint dwDeviceVersion;

    /// <summary>장비 이름</summary>
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 32)]
    public string sDeviceName;

    /// <summary>MAC 주소</summary>
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 6)]
    public byte[] byMacAddr;

    /// <summary>링크 포트</summary>
    public ushort wLinkPort;

    /// <summary>장비 IP 주소</summary>
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 128)]
    public string sDeviceIP;

    /// <summary>소켓 IP 주소</summary>
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 128)]
    public string sSocketIP;

    /// <summary>HTTPS 포트</summary>
    public byte byIpProtocol;

    /// <summary>예약</summary>
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 11)]
    public byte[] byRes2;
}
```

> **핵심**: `[StructLayout(LayoutKind.Sequential)]`은 C# 컴파일러에게 "이 구조체의 필드 순서와 메모리 레이아웃을 C와 동일하게 유지하라"고 지시합니다. 이게 틀리면 데이터가 잘못 읽혀서 크래시가 발생합니다.

---

## 5. 서비스 패턴으로 래핑

P/Invoke를 직접 ViewModel에서 호출하면 코드가 복잡해지고 테스트가 어렵습니다. **서비스로 래핑**하여 깔끔하게 관리합니다.

### 5-1. 이벤트 데이터 모델

```csharp
// Models/FaceRecognitionEvent.cs
namespace MyApp.Models;

/// <summary>
/// 얼굴인식 이벤트 데이터입니다.
/// HCNetSDK의 콜백에서 받은 데이터를 C# 친화적으로 변환한 것입니다.
/// </summary>
public sealed class FaceRecognitionEvent
{
    /// <summary>이벤트 발생 시각</summary>
    public DateTime Timestamp { get; init; }

    /// <summary>장비 IP 주소</summary>
    public string DeviceIp { get; init; } = string.Empty;

    /// <summary>인식된 사람 이름 (등록된 경우)</summary>
    public string? PersonName { get; init; }

    /// <summary>사원 번호 (등록된 경우)</summary>
    public string? EmployeeId { get; init; }

    /// <summary>유사도 (0.0 ~ 1.0)</summary>
    public double Similarity { get; init; }

    /// <summary>캡처된 얼굴 이미지 데이터 (JPEG)</summary>
    public byte[]? CapturedFaceImage { get; init; }

    /// <summary>등록된 얼굴 이미지 데이터 (JPEG)</summary>
    public byte[]? RegisteredFaceImage { get; init; }

    /// <summary>이벤트 타입 (출입, 인식 실패 등)</summary>
    public FaceEventType EventType { get; init; }
}

/// <summary>얼굴인식 이벤트 타입</summary>
public enum FaceEventType
{
    /// <summary>얼굴 인식 성공 (등록된 사용자)</summary>
    Recognized,

    /// <summary>얼굴 감지되었으나 미등록</summary>
    Unknown,

    /// <summary>출입 허가</summary>
    AccessGranted,

    /// <summary>출입 거부</summary>
    AccessDenied
}
```

### 5-2. IHikvisionService 인터페이스

```csharp
// Services/IHikvisionService.cs
using MyApp.Models;

namespace MyApp.Services;

/// <summary>
/// Hikvision 장비 연동 서비스 인터페이스입니다.
/// 장비 로그인, 이벤트 수신 등의 기능을 추상화합니다.
/// </summary>
public interface IHikvisionService : IDisposable
{
    /// <summary>
    /// 얼굴인식 이벤트가 발생하면 호출되는 이벤트입니다.
    /// </summary>
    event EventHandler<FaceRecognitionEvent>? FaceRecognized;

    /// <summary>
    /// 장비 연결 상태가 변경되면 호출되는 이벤트입니다.
    /// </summary>
    event EventHandler<DeviceConnectionChangedEventArgs>? ConnectionChanged;

    /// <summary>SDK를 초기화합니다.</summary>
    void Initialize();

    /// <summary>장비에 로그인합니다.</summary>
    /// <param name="ip">장비 IP 주소</param>
    /// <param name="port">장비 포트 (기본: 8000)</param>
    /// <param name="username">사용자 이름</param>
    /// <param name="password">비밀번호</param>
    /// <returns>로그인 성공 여부</returns>
    bool Login(string ip, ushort port, string username, string password);

    /// <summary>장비에서 로그아웃합니다.</summary>
    void Logout();

    /// <summary>알람 채널을 설정하여 이벤트 수신을 시작합니다.</summary>
    bool StartListening();

    /// <summary>이벤트 수신을 중지합니다.</summary>
    void StopListening();

    /// <summary>현재 연결 상태를 반환합니다.</summary>
    bool IsConnected { get; }
}

/// <summary>
/// 장비 연결 상태 변경 이벤트 인자입니다.
/// </summary>
public sealed class DeviceConnectionChangedEventArgs : EventArgs
{
    /// <summary>장비 IP</summary>
    public required string DeviceIp { get; init; }

    /// <summary>연결 여부</summary>
    public required bool IsConnected { get; init; }

    /// <summary>변경 사유</summary>
    public string? Reason { get; init; }
}
```

### 5-3. HikvisionService 구현

```csharp
// Services/HikvisionService.cs
using System.Runtime.InteropServices;
using Microsoft.Extensions.Logging;
using MyApp.Models;
using MyApp.Native;

namespace MyApp.Services;

/// <summary>
/// Hikvision HCNetSDK를 래핑한 서비스 구현입니다.
/// P/Invoke를 통해 비관리 DLL과 통신하고,
/// 결과를 C# 이벤트로 변환합니다.
///
/// 반드시 Singleton으로 등록해야 합니다!
/// (SDK 초기화/정리가 앱 생명주기와 일치해야 함)
/// </summary>
public sealed class HikvisionService : IHikvisionService
{
    private readonly ILogger<HikvisionService> _logger;

    // 장비 로그인 ID (-1이면 미로그인)
    private int _userId = -1;

    // 알람 채널 핸들 (-1이면 미설정)
    private int _alarmHandle = -1;

    // SDK 초기화 여부
    private bool _isInitialized;

    // 연결된 장비 IP
    private string _deviceIp = string.Empty;

    // ──────────────────────────────────────────────
    // 중요: 콜백 delegate를 필드로 유지!
    // GC가 수집하지 못하도록 참조를 유지해야 합니다.
    // ──────────────────────────────────────────────
    private NativeMethods.MSGCallBack_V31? _alarmCallback;
    private NativeMethods.ExceptionCallBack? _exceptionCallback;

    /// <inheritdoc />
    public event EventHandler<FaceRecognitionEvent>? FaceRecognized;

    /// <inheritdoc />
    public event EventHandler<DeviceConnectionChangedEventArgs>? ConnectionChanged;

    /// <inheritdoc />
    public bool IsConnected => _userId >= 0;

    public HikvisionService(ILogger<HikvisionService> logger)
    {
        _logger = logger;
    }

    /// <inheritdoc />
    public void Initialize()
    {
        if (_isInitialized) return;

        // SDK 초기화
        bool result = NativeMethods.NET_DVR_Init();
        if (!result)
        {
            var errorCode = NativeMethods.NET_DVR_GetLastError();
            throw new InvalidOperationException(
                $"HCNetSDK 초기화 실패. 에러 코드: {errorCode}");
        }

        // SDK 로그 설정 (디버깅용)
        NativeMethods.NET_DVR_SetLogToFile(
            3,                    // 로그 레벨 (3 = 모두)
            "logs/hcnetsdk",     // 로그 디렉터리
            true);               // 자동 삭제

        // 예외 콜백 등록
        _exceptionCallback = OnException;
        NativeMethods.NET_DVR_SetExceptionCallBack_V30(
            0, IntPtr.Zero, _exceptionCallback, IntPtr.Zero);

        // 알람 콜백 등록
        _alarmCallback = OnAlarmReceived;
        NativeMethods.NET_DVR_SetDVRMessageCallBack_V31(
            _alarmCallback, IntPtr.Zero);

        _isInitialized = true;
        _logger.LogInformation("HCNetSDK 초기화 완료");
    }

    /// <inheritdoc />
    public bool Login(string ip, ushort port, string username, string password)
    {
        if (!_isInitialized)
            throw new InvalidOperationException(
                "SDK가 초기화되지 않았습니다. Initialize()를 먼저 호출하세요.");

        if (IsConnected)
        {
            _logger.LogWarning("이미 로그인된 상태입니다. 먼저 로그아웃하세요.");
            return false;
        }

        // 로그인 정보 구조체 설정
        var loginInfo = new NET_DVR_USER_LOGIN_INFO
        {
            sDeviceAddress = ip,
            wPort = port,
            sUserName = username,
            sPassword = password,
            byLoginMode = 0 // TCP
        };

        // 장비 정보를 받을 구조체 (out 파라미터처럼 사용)
        var deviceInfo = new NET_DVR_DEVICEINFO_V40();

        // 로그인 실행
        _userId = NativeMethods.NET_DVR_Login_V40(
            ref loginInfo, ref deviceInfo);

        if (_userId < 0)
        {
            var errorCode = NativeMethods.NET_DVR_GetLastError();
            _logger.LogError(
                "장비 로그인 실패. IP: {Ip}, 에러 코드: {ErrorCode}",
                ip, errorCode);
            return false;
        }

        _deviceIp = ip;
        _logger.LogInformation(
            "장비 로그인 성공. IP: {Ip}, UserID: {UserId}",
            ip, _userId);

        // 연결 상태 변경 이벤트 발생
        ConnectionChanged?.Invoke(this, new DeviceConnectionChangedEventArgs
        {
            DeviceIp = ip,
            IsConnected = true,
            Reason = "로그인 성공"
        });

        return true;
    }

    /// <inheritdoc />
    public void Logout()
    {
        if (!IsConnected) return;

        // 알람 수신 중지
        StopListening();

        // 로그아웃
        NativeMethods.NET_DVR_Logout(_userId);

        _logger.LogInformation(
            "장비 로그아웃. IP: {Ip}, UserID: {UserId}",
            _deviceIp, _userId);

        var ip = _deviceIp;
        _userId = -1;
        _deviceIp = string.Empty;

        // 연결 상태 변경 이벤트 발생
        ConnectionChanged?.Invoke(this, new DeviceConnectionChangedEventArgs
        {
            DeviceIp = ip,
            IsConnected = false,
            Reason = "로그아웃"
        });
    }

    /// <inheritdoc />
    public bool StartListening()
    {
        if (!IsConnected)
        {
            _logger.LogWarning("로그인하지 않은 상태에서 알람 수신을 시작할 수 없습니다.");
            return false;
        }

        if (_alarmHandle >= 0)
        {
            _logger.LogWarning("이미 알람 수신 중입니다.");
            return true;
        }

        // 알람 파라미터 설정
        var setupParam = new NET_DVR_SETUPALARM_PARAM_V50
        {
            dwSize = (uint)Marshal.SizeOf<NET_DVR_SETUPALARM_PARAM_V50>(),
            byLevel = 1,           // 2등급 (클라이언트)
            byAlarmInfoType = 1,   // 알람 상세 정보 포함
            byDeployType = 0,
            byRes = new byte[29],
            lUserID = _userId
        };

        // 알람 채널 설정
        _alarmHandle = NativeMethods.NET_DVR_SetupAlarmChan_V41(
            ref setupParam);

        if (_alarmHandle < 0)
        {
            var errorCode = NativeMethods.NET_DVR_GetLastError();
            _logger.LogError(
                "알람 채널 설정 실패. 에러 코드: {ErrorCode}", errorCode);
            return false;
        }

        _logger.LogInformation(
            "알람 수신 시작. AlarmHandle: {Handle}", _alarmHandle);
        return true;
    }

    /// <inheritdoc />
    public void StopListening()
    {
        if (_alarmHandle < 0) return;

        NativeMethods.NET_DVR_CloseAlarmChan_V30(_alarmHandle);
        _logger.LogInformation(
            "알람 수신 중지. AlarmHandle: {Handle}", _alarmHandle);
        _alarmHandle = -1;
    }

    // ──────────────────────────────────────────────
    // 콜백 핸들러
    // ──────────────────────────────────────────────

    /// <summary>
    /// 알람/이벤트 수신 콜백입니다.
    /// 이 메서드는 SDK의 백그라운드 스레드에서 호출됩니다!
    /// </summary>
    private void OnAlarmReceived(
        int lCommand,
        ref NET_DVR_ALARMER pAlarmer,
        IntPtr pAlarmInfo,
        uint dwBufLen,
        IntPtr pUser)
    {
        try
        {
            // 명령 타입에 따라 분기
            switch (lCommand)
            {
                case HikConst.COMM_SNAP_MATCH_ALARM:
                    // 얼굴 비교(매칭) 결과 이벤트
                    HandleFaceMatchEvent(pAlarmer, pAlarmInfo, dwBufLen);
                    break;

                case HikConst.COMM_ALARM_ACS:
                    // 출입통제 알람 이벤트
                    HandleAccessControlEvent(pAlarmer, pAlarmInfo, dwBufLen);
                    break;

                default:
                    _logger.LogDebug(
                        "처리하지 않는 알람 타입: 0x{Command:X}",
                        lCommand);
                    break;
            }
        }
        catch (Exception ex)
        {
            // 콜백 내에서 예외가 발생하면 SDK가 불안정해질 수 있으므로
            // 반드시 try-catch로 감싸야 합니다!
            _logger.LogError(ex,
                "알람 콜백 처리 중 오류 발생. Command: 0x{Command:X}",
                lCommand);
        }
    }

    /// <summary>
    /// 얼굴 매칭 이벤트를 처리합니다.
    /// </summary>
    private void HandleFaceMatchEvent(
        NET_DVR_ALARMER alarmer,
        IntPtr pAlarmInfo,
        uint bufLen)
    {
        // 비관리 메모리에서 구조체 데이터 읽기
        // pAlarmInfo는 NET_VCA_FACESNAP_MATCH_ALARM 구조체를 가리킵니다.
        // 여기서는 간소화하여 기본 정보만 추출합니다.

        var faceEvent = new FaceRecognitionEvent
        {
            Timestamp = DateTime.Now,
            DeviceIp = alarmer.sDeviceIP ?? _deviceIp,
            EventType = FaceEventType.Recognized,
            Similarity = 0.95, // 실제로는 구조체에서 읽어야 함
            PersonName = "홍길동", // 실제로는 구조체에서 읽어야 함
            EmployeeId = "EMP001"
        };

        // 캡처된 얼굴 이미지 추출 (비관리 메모리에서 바이트 배열로 복사)
        // 실제 구현에서는 구조체의 이미지 오프셋과 크기를 사용합니다.
        // faceEvent.CapturedFaceImage = ExtractImageFromBuffer(pAlarmInfo);

        _logger.LogInformation(
            "얼굴 인식 이벤트: {Name} ({Id}), 유사도: {Similarity:P0}",
            faceEvent.PersonName,
            faceEvent.EmployeeId,
            faceEvent.Similarity);

        // C# 이벤트로 전달
        // 이 시점에서 여전히 SDK의 백그라운드 스레드입니다.
        // UI 스레드 마샬링은 수신 측(ViewModel)에서 처리합니다.
        FaceRecognized?.Invoke(this, faceEvent);
    }

    /// <summary>
    /// 출입통제 이벤트를 처리합니다.
    /// </summary>
    private void HandleAccessControlEvent(
        NET_DVR_ALARMER alarmer,
        IntPtr pAlarmInfo,
        uint bufLen)
    {
        var faceEvent = new FaceRecognitionEvent
        {
            Timestamp = DateTime.Now,
            DeviceIp = alarmer.sDeviceIP ?? _deviceIp,
            EventType = FaceEventType.AccessGranted
        };

        _logger.LogInformation(
            "출입통제 이벤트: 장비 {DeviceIp}", faceEvent.DeviceIp);

        FaceRecognized?.Invoke(this, faceEvent);
    }

    /// <summary>
    /// 예외(연결 끊김 등) 콜백입니다.
    /// </summary>
    private void OnException(
        uint dwType, int lUserID, int lHandle, IntPtr pUser)
    {
        switch (dwType)
        {
            case HikConst.EXCEPTION_RECONNECT:
                _logger.LogWarning(
                    "장비 재연결 시도 중. UserID: {UserId}", lUserID);
                ConnectionChanged?.Invoke(this,
                    new DeviceConnectionChangedEventArgs
                    {
                        DeviceIp = _deviceIp,
                        IsConnected = false,
                        Reason = "연결 끊김 - 재연결 시도 중"
                    });
                break;

            default:
                _logger.LogWarning(
                    "SDK 예외 발생. 타입: 0x{Type:X}, UserID: {UserId}",
                    dwType, lUserID);
                break;
        }
    }

    // ──────────────────────────────────────────────
    // IDisposable 구현
    // ──────────────────────────────────────────────

    /// <summary>
    /// 모든 리소스를 정리합니다.
    /// </summary>
    public void Dispose()
    {
        // 알람 수신 중지
        StopListening();

        // 로그아웃
        if (IsConnected)
        {
            NativeMethods.NET_DVR_Logout(_userId);
            _userId = -1;
        }

        // SDK 정리
        if (_isInitialized)
        {
            NativeMethods.NET_DVR_Cleanup();
            _isInitialized = false;
        }

        // 콜백 참조 해제
        _alarmCallback = null;
        _exceptionCallback = null;

        _logger.LogInformation("HikvisionService 리소스 해제 완료");
    }
}
```

---

## 6. DI에 등록 (Singleton)

```csharp
// App.xaml.cs (DI 설정 부분)
using Microsoft.Extensions.DependencyInjection;
using MyApp.Services;
using MyApp.ViewModels;

// DI 컨테이너에 등록
var services = new ServiceCollection();

// ──────────────────────────────────────────────
// Hikvision 서비스: 반드시 Singleton!
// SDK 초기화/정리가 앱 생명주기와 일치해야 합니다.
// ──────────────────────────────────────────────
services.AddSingleton<IHikvisionService, HikvisionService>();

services.AddTransient<DeviceMonitorViewModel>();
services.AddLogging();

var serviceProvider = services.BuildServiceProvider();

// 앱 시작 시 SDK 초기화
var hikvisionService = serviceProvider
    .GetRequiredService<IHikvisionService>();
hikvisionService.Initialize();

// ... 앱 실행 ...

// 앱 종료 시 자동 Dispose (ServiceProvider.Dispose()에 의해)
```

> **왜 Singleton인가?** HCNetSDK의 `NET_DVR_Init()`과 `NET_DVR_Cleanup()`은 프로세스당 한 번만 호출해야 합니다. 여러 인스턴스가 각자 Init/Cleanup을 호출하면 SDK가 불안정해집니다.

---

## 7. ViewModel에서 얼굴인식 이벤트 수신

### 7-1. DeviceMonitorViewModel

```csharp
// ViewModels/DeviceMonitorViewModel.cs
using System.Collections.ObjectModel;
using System.Windows;
using System.Windows.Threading;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.Logging;
using MyApp.Models;
using MyApp.Services;

namespace MyApp.ViewModels;

/// <summary>
/// 장비 모니터링 ViewModel입니다.
/// 얼굴인식 단말기의 이벤트를 실시간으로 수신하고 표시합니다.
/// </summary>
public partial class DeviceMonitorViewModel : ObservableObject, IDisposable
{
    private readonly IHikvisionService _hikvisionService;
    private readonly ILogger<DeviceMonitorViewModel> _logger;

    // UI 스레드 디스패처 (백그라운드 스레드에서 UI 업데이트 시 필요)
    private readonly Dispatcher _dispatcher;

    public DeviceMonitorViewModel(
        IHikvisionService hikvisionService,
        ILogger<DeviceMonitorViewModel> logger)
    {
        _hikvisionService = hikvisionService;
        _logger = logger;

        // 현재 스레드(UI 스레드)의 Dispatcher 저장
        _dispatcher = Application.Current.Dispatcher;

        // 서비스 이벤트 구독
        _hikvisionService.FaceRecognized += OnFaceRecognized;
        _hikvisionService.ConnectionChanged += OnConnectionChanged;
    }

    // ──────────────────────────────────────────────
    // Observable 속성
    // ──────────────────────────────────────────────

    /// <summary>장비 IP 주소</summary>
    [ObservableProperty]
    private string _deviceIp = "192.168.1.100";

    /// <summary>장비 포트</summary>
    [ObservableProperty]
    private ushort _devicePort = 8000;

    /// <summary>사용자 이름</summary>
    [ObservableProperty]
    private string _username = "admin";

    /// <summary>비밀번호</summary>
    [ObservableProperty]
    private string _password = string.Empty;

    /// <summary>연결 상태</summary>
    [ObservableProperty]
    private bool _isConnected;

    /// <summary>상태 메시지</summary>
    [ObservableProperty]
    private string _statusMessage = "연결 대기 중";

    /// <summary>실시간 이벤트 목록 (최근 것이 위에)</summary>
    public ObservableCollection<FaceRecognitionEvent> Events { get; } = [];

    // ──────────────────────────────────────────────
    // 커맨드
    // ──────────────────────────────────────────────

    /// <summary>
    /// 장비에 연결합니다 (로그인 + 알람 수신 시작).
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanConnect))]
    private void Connect()
    {
        try
        {
            StatusMessage = "연결 중...";

            // 장비 로그인
            bool loginResult = _hikvisionService.Login(
                DeviceIp, DevicePort, Username, Password);

            if (!loginResult)
            {
                StatusMessage = "로그인 실패. IP 주소와 인증 정보를 확인하세요.";
                return;
            }

            // 알람(이벤트) 수신 시작
            bool listenResult = _hikvisionService.StartListening();
            if (!listenResult)
            {
                StatusMessage = "알람 채널 설정 실패.";
                return;
            }

            IsConnected = true;
            StatusMessage = $"연결됨: {DeviceIp}:{DevicePort}";
        }
        catch (Exception ex)
        {
            StatusMessage = $"연결 오류: {ex.Message}";
            _logger.LogError(ex, "장비 연결 실패");
        }
    }

    /// <summary>연결 가능 여부</summary>
    private bool CanConnect() =>
        !IsConnected && !string.IsNullOrEmpty(DeviceIp);

    /// <summary>
    /// 장비 연결을 해제합니다.
    /// </summary>
    [RelayCommand(CanExecute = nameof(CanDisconnect))]
    private void Disconnect()
    {
        _hikvisionService.Logout();
        IsConnected = false;
        StatusMessage = "연결 해제됨";
    }

    /// <summary>연결 해제 가능 여부</summary>
    private bool CanDisconnect() => IsConnected;

    /// <summary>
    /// 이벤트 목록을 초기화합니다.
    /// </summary>
    [RelayCommand]
    private void ClearEvents()
    {
        Events.Clear();
        _logger.LogDebug("이벤트 목록 초기화");
    }

    // ──────────────────────────────────────────────
    // 이벤트 핸들러
    // ──────────────────────────────────────────────

    /// <summary>
    /// 얼굴인식 이벤트를 수신합니다.
    /// 이 메서드는 SDK의 백그라운드 스레드에서 호출됩니다!
    /// 반드시 UI 스레드로 마샬링해야 합니다.
    /// </summary>
    private void OnFaceRecognized(object? sender, FaceRecognitionEvent e)
    {
        // UI 스레드로 마샬링
        // ObservableCollection은 UI 스레드에서만 수정할 수 있습니다!
        _dispatcher.Invoke(() =>
        {
            // 최근 이벤트를 맨 위에 추가
            Events.Insert(0, e);

            // 이벤트가 너무 많으면 오래된 것 제거 (메모리 관리)
            while (Events.Count > 100)
            {
                Events.RemoveAt(Events.Count - 1);
            }

            StatusMessage = $"최근 이벤트: {e.PersonName ?? "미등록"} " +
                            $"({e.EventType}) - {e.Timestamp:HH:mm:ss}";
        });
    }

    /// <summary>
    /// 연결 상태 변경 이벤트를 수신합니다.
    /// </summary>
    private void OnConnectionChanged(
        object? sender, DeviceConnectionChangedEventArgs e)
    {
        _dispatcher.Invoke(() =>
        {
            IsConnected = e.IsConnected;
            StatusMessage = e.IsConnected
                ? $"연결됨: {e.DeviceIp}"
                : $"연결 끊김: {e.Reason}";
        });
    }

    // ──────────────────────────────────────────────
    // IDisposable
    // ──────────────────────────────────────────────

    public void Dispose()
    {
        // 이벤트 구독 해제 (메모리 누수 방지)
        _hikvisionService.FaceRecognized -= OnFaceRecognized;
        _hikvisionService.ConnectionChanged -= OnConnectionChanged;
    }
}
```

### 7-2. View (XAML)

```xml
<!-- Views/DeviceMonitorView.xaml -->
<Window x:Class="MyApp.Views.DeviceMonitorView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        Title="장비 모니터링" Width="800" Height="600">

    <Grid Margin="16">
        <Grid.RowDefinitions>
            <!-- 연결 설정 -->
            <RowDefinition Height="Auto" />
            <!-- 이벤트 목록 -->
            <RowDefinition Height="*" />
            <!-- 상태 표시줄 -->
            <RowDefinition Height="Auto" />
        </Grid.RowDefinitions>

        <!-- ① 연결 설정 영역 -->
        <GroupBox Grid.Row="0" Header="장비 연결" Margin="0,0,0,8">
            <Grid Margin="8">
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto" />
                    <ColumnDefinition Width="*" />
                    <ColumnDefinition Width="Auto" />
                    <ColumnDefinition Width="80" />
                    <ColumnDefinition Width="Auto" />
                    <ColumnDefinition Width="*" />
                    <ColumnDefinition Width="Auto" />
                    <ColumnDefinition Width="*" />
                </Grid.ColumnDefinitions>

                <TextBlock Text="IP:" VerticalAlignment="Center"
                           Grid.Column="0" Margin="0,0,4,0" />
                <TextBox Text="{Binding DeviceIp}"
                         Grid.Column="1" Margin="0,0,8,0" />

                <TextBlock Text="포트:" VerticalAlignment="Center"
                           Grid.Column="2" Margin="0,0,4,0" />
                <TextBox Text="{Binding DevicePort}"
                         Grid.Column="3" Margin="0,0,8,0" />

                <TextBlock Text="사용자:" VerticalAlignment="Center"
                           Grid.Column="4" Margin="0,0,4,0" />
                <TextBox Text="{Binding Username}"
                         Grid.Column="5" Margin="0,0,8,0" />

                <TextBlock Text="비밀번호:" VerticalAlignment="Center"
                           Grid.Column="6" Margin="0,0,4,0" />
                <PasswordBox Grid.Column="7" />
            </Grid>
        </GroupBox>

        <!-- 연결/해제 버튼 -->
        <StackPanel Grid.Row="0"
                    Orientation="Horizontal"
                    HorizontalAlignment="Right"
                    VerticalAlignment="Bottom"
                    Margin="0,0,8,4">
            <Button Content="연결"
                    Command="{Binding ConnectCommand}"
                    Width="80" Height="30" Margin="0,0,4,0" />
            <Button Content="해제"
                    Command="{Binding DisconnectCommand}"
                    Width="80" Height="30" Margin="0,0,4,0" />
            <Button Content="초기화"
                    Command="{Binding ClearEventsCommand}"
                    Width="80" Height="30" />
        </StackPanel>

        <!-- ② 실시간 이벤트 목록 -->
        <DataGrid Grid.Row="1"
                  ItemsSource="{Binding Events}"
                  AutoGenerateColumns="False"
                  IsReadOnly="True"
                  CanUserAddRows="False"
                  Margin="0,0,0,8">
            <DataGrid.Columns>
                <!-- 시간 -->
                <DataGridTextColumn
                    Header="시간"
                    Binding="{Binding Timestamp, StringFormat=HH:mm:ss}"
                    Width="80" />

                <!-- 장비 IP -->
                <DataGridTextColumn
                    Header="장비 IP"
                    Binding="{Binding DeviceIp}"
                    Width="120" />

                <!-- 이벤트 타입 -->
                <DataGridTextColumn
                    Header="이벤트"
                    Binding="{Binding EventType}"
                    Width="100" />

                <!-- 이름 -->
                <DataGridTextColumn
                    Header="이름"
                    Binding="{Binding PersonName}"
                    Width="100" />

                <!-- 사원번호 -->
                <DataGridTextColumn
                    Header="사원번호"
                    Binding="{Binding EmployeeId}"
                    Width="100" />

                <!-- 유사도 -->
                <DataGridTextColumn
                    Header="유사도"
                    Binding="{Binding Similarity, StringFormat=P0}"
                    Width="80" />
            </DataGrid.Columns>
        </DataGrid>

        <!-- ③ 상태 표시줄 -->
        <StatusBar Grid.Row="2">
            <StatusBarItem>
                <!-- 연결 상태 표시 (동그라미) -->
                <Ellipse Width="10" Height="10"
                         Fill="{Binding IsConnected,
                             Converter={StaticResource BoolToColorConverter}}"
                         Margin="0,0,8,0" />
            </StatusBarItem>
            <StatusBarItem>
                <TextBlock Text="{Binding StatusMessage}" />
            </StatusBarItem>
            <StatusBarItem HorizontalAlignment="Right">
                <TextBlock>
                    <Run Text="이벤트 수: " />
                    <Run Text="{Binding Events.Count, Mode=OneWay}" />
                </TextBlock>
            </StatusBarItem>
        </StatusBar>
    </Grid>
</Window>
```

---

## 8. 메모리 관리 주의사항

P/Invoke에서 가장 중요하면서도 실수하기 쉬운 부분이 **메모리 관리**입니다.

### 8-1. 콜백 delegate를 필드로 유지 (GC 핀닝)

```csharp
// ❌ 잘못된 예: 로컬 변수로 delegate 생성
public void Initialize()
{
    // 이 delegate는 로컬 변수이므로 메서드가 끝나면
    // GC가 수집할 수 있습니다!
    // SDK가 나중에 이 콜백을 호출하면 크래시 발생!
    var callback = new NativeMethods.MSGCallBack_V31(OnAlarmReceived);
    NativeMethods.NET_DVR_SetDVRMessageCallBack_V31(callback, IntPtr.Zero);
}

// ✅ 올바른 예: 필드로 유지
private NativeMethods.MSGCallBack_V31? _alarmCallback;

public void Initialize()
{
    // 필드에 저장하면 HikvisionService 인스턴스가 살아있는 동안
    // GC가 수집하지 않습니다.
    _alarmCallback = new NativeMethods.MSGCallBack_V31(OnAlarmReceived);
    NativeMethods.NET_DVR_SetDVRMessageCallBack_V31(
        _alarmCallback, IntPtr.Zero);
}
```

### 8-2. Marshal.AllocHGlobal과 해제

```csharp
// 비관리 메모리를 할당하고 사용하는 예제
IntPtr pBuffer = IntPtr.Zero;

try
{
    // 비관리 메모리 할당 (C의 malloc과 유사)
    int bufferSize = Marshal.SizeOf<NET_DVR_USER_LOGIN_INFO>();
    pBuffer = Marshal.AllocHGlobal(bufferSize);

    // 구조체를 비관리 메모리에 복사
    var loginInfo = new NET_DVR_USER_LOGIN_INFO
    {
        sDeviceAddress = "192.168.1.100",
        wPort = 8000,
        sUserName = "admin",
        sPassword = "password123"
    };

    Marshal.StructureToPtr(loginInfo, pBuffer, false);

    // 비관리 메모리에서 구조체 읽기
    var readBack = Marshal.PtrToStructure<NET_DVR_USER_LOGIN_INFO>(pBuffer);
}
finally
{
    // 반드시 해제! (C의 free와 유사)
    // 해제하지 않으면 메모리 누수 발생!
    if (pBuffer != IntPtr.Zero)
    {
        Marshal.FreeHGlobal(pBuffer);
    }
}
```

### 8-3. 콜백에서 이미지 데이터 복사

```csharp
/// <summary>
/// 비관리 메모리에서 이미지 데이터를 안전하게 복사합니다.
/// 콜백 함수가 반환된 후에는 pAlarmInfo가 가리키는 메모리가
/// SDK에 의해 해제될 수 있으므로, 콜백 내에서 즉시 복사해야 합니다.
/// </summary>
private static byte[]? ExtractImageFromBuffer(
    IntPtr pImageData, uint imageSize)
{
    if (pImageData == IntPtr.Zero || imageSize == 0)
        return null;

    // 비관리 메모리에서 관리 바이트 배열로 복사
    var imageBytes = new byte[imageSize];
    Marshal.Copy(pImageData, imageBytes, 0, (int)imageSize);

    return imageBytes;
}
```

### 주요 메모리 관리 규칙 요약

| 규칙 | 설명 |
|------|------|
| **콜백 delegate는 필드로 유지** | GC가 수집하지 못하도록 참조를 유지 |
| **AllocHGlobal → FreeHGlobal** | 비관리 메모리는 반드시 쌍으로 할당/해제 |
| **콜백 내에서 데이터 즉시 복사** | 콜백 반환 후 SDK가 메모리를 해제할 수 있음 |
| **try-finally 패턴** | 예외가 발생해도 메모리가 해제되도록 보장 |
| **Dispose 패턴 구현** | SDK 정리를 보장 |
| **콜백 내에서 try-catch** | 예외가 비관리 코드로 전파되면 크래시 |

---

## 9. 전체 코드 예제

### 프로젝트 구조

```
MyApp/
├── Libs/
│   └── HCNetSDK/                    ← SDK DLL 파일들
│       ├── HCNetSDK.dll
│       ├── HCCore.dll
│       └── ...
├── Native/
│   ├── NativeMethods.cs             ← P/Invoke 함수 선언
│   └── NativeStructs.cs             ← 구조체·상수 정의
├── Models/
│   └── FaceRecognitionEvent.cs      ← 이벤트 데이터 모델
├── Services/
│   ├── IHikvisionService.cs         ← 서비스 인터페이스
│   └── HikvisionService.cs          ← 서비스 구현 (P/Invoke 래핑)
├── ViewModels/
│   └── DeviceMonitorViewModel.cs    ← 장비 모니터링 ViewModel
├── Views/
│   └── DeviceMonitorView.xaml       ← 장비 모니터링 화면
└── App.xaml.cs                       ← DI 설정 + SDK 초기화
```

### 데이터 흐름 요약

```
┌───────────────────────────────────────────────────────────┐
│                     Hikvision 장비                         │
│                (얼굴인식 단말기, NVR 등)                   │
└─────────────────────┬─────────────────────────────────────┘
                      │ TCP/IP (포트 8000)
┌─────────────────────▼─────────────────────────────────────┐
│                   HCNetSDK.dll                             │
│              (Hikvision 제공 비관리 DLL)                   │
└─────────────────────┬─────────────────────────────────────┘
                      │ P/Invoke (콜백)
┌─────────────────────▼─────────────────────────────────────┐
│               HikvisionService.cs                          │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  OnAlarmReceived() ← SDK 백그라운드 스레드에서 호출  │  │
│  │     ↓                                               │  │
│  │  HandleFaceMatchEvent() → C# 구조체로 변환          │  │
│  │     ↓                                               │  │
│  │  FaceRecognized?.Invoke() → C# 이벤트 발생         │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────┬─────────────────────────────────────┘
                      │ C# event (백그라운드 스레드)
┌─────────────────────▼─────────────────────────────────────┐
│            DeviceMonitorViewModel.cs                        │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  OnFaceRecognized()                                 │  │
│  │     ↓                                               │  │
│  │  _dispatcher.Invoke(() => {                         │  │
│  │     Events.Insert(0, faceEvent); ← UI 스레드       │  │
│  │  });                                                │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────┬─────────────────────────────────────┘
                      │ 데이터 바인딩
┌─────────────────────▼─────────────────────────────────────┐
│                DeviceMonitorView.xaml                       │
│  DataGrid ← Events (ObservableCollection)                  │
│  StatusBar ← StatusMessage, IsConnected                    │
└───────────────────────────────────────────────────────────┘
```

### 핵심 포인트 정리

| 항목 | 설명 |
|------|------|
| **P/Invoke 선언** | `NativeMethods` 클래스에 모든 DllImport를 모아둠 |
| **구조체 매핑** | `[StructLayout(LayoutKind.Sequential)]`로 C 구조체와 1:1 매핑 |
| **콜백 GC 방지** | delegate를 필드로 유지하여 GC 수집 방지 |
| **서비스 래핑** | P/Invoke를 `IHikvisionService`로 추상화 |
| **DI Singleton** | SDK 초기화/정리가 앱 생명주기와 일치 |
| **UI 스레드 마샬링** | `Dispatcher.Invoke()`로 백그라운드 → UI 스레드 전환 |
| **콜백 내 try-catch** | 비관리 코드로 예외가 전파되지 않도록 방어 |
| **메모리 관리** | `AllocHGlobal`/`FreeHGlobal`, `Marshal.Copy` 사용 |

> **다음 단계**: [System.Text.Json을 이용한 JSON 직렬화/역직렬화](./08-json-serialization.md)에서는 API 응답 처리, 로컬 파일 저장, MQTT 메시지 처리 등에 활용하는 JSON 직렬화 방법을 다룹니다.
