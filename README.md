# WPF & MVVM 패턴 튜토리얼

> **대상 독자**: C# WinForms / Python 데이터 분석 경험자
> **목표**: WPF + MVVM 패턴을 **처음부터** 이해하고, 실무 기술 스택과 함께 사용하는 법을 익힙니다.

---

## 기술 스택 요약

| 분류 | 기술 | 버전 | 용도 |
|------|------|------|------|
| 프레임워크 | .NET | 10.0 (LTS) | 런타임 |
| 언어 | C# | 13.0 | 주 개발 언어 |
| UI | WPF | .NET 10.0 | 데스크탑 클라이언트 UI |
| MVVM | CommunityToolkit.Mvvm | 8.4.0 | ViewModel 기반 패턴 (소스 제네레이터) |
| DB | PostgreSQL | 16+ | 주 데이터베이스 |
| ORM | Dapper | 2.1.44 | 경량 마이크로 ORM |
| DB 드라이버 | Npgsql | 9.0.3 | PostgreSQL ADO.NET 드라이버 |
| HTTP 복원력 | Polly | 8.5.2 | 재시도, 서킷브레이커, 타임아웃 |
| 메시징 | MQTTnet | 5.0.1 | MQTT v5 클라이언트 |
| 클라우드 | AWSSDK.S3 | 3.7.405.7 | S3 사진 저장소 |
| 로깅 | Serilog | 4.2.0 | 구조적 로깅 (파일 + 콘솔) |
| DI | Microsoft.Extensions.DependencyInjection | 10.0.0 | 의존성 주입 컨테이너 |
| 설정 | Microsoft.Extensions.Configuration | 10.0.0 | appsettings.json 기반 설정 |
| 트레이 아이콘 | Hardcodet.NotifyIcon.Wpf | 4.0.1 | 시스템 트레이 통합 |
| 장비 SDK | Hikvision HCNetSDK | - | 얼굴인식 단말기 P/Invoke |
| 직렬화 | System.Text.Json | 10.0.0 | JSON 직렬화/역직렬화 |

---

## 문서 구조

### 📂 [01-fundamentals](./01-fundamentals/) — WPF 기초
WinForms 개발자 관점에서 WPF의 핵심 개념을 설명합니다.

| # | 문서 | 내용 |
|---|------|------|
| 1 | [WPF 기초 개념](./01-fundamentals/01-wpf-basics.md) | WPF란? WinForms과의 차이점 |
| 2 | [XAML 기본 문법](./01-fundamentals/02-xaml-basics.md) | XAML 태그, 속성, 네임스페이스 |
| 3 | [데이터 바인딩](./01-fundamentals/03-data-binding.md) | 바인딩의 개념과 동작 원리 |
| 4 | [MVVM 패턴 이해](./01-fundamentals/04-mvvm-pattern.md) | Model-View-ViewModel이란? |

### 📂 [02-mvvm-comparison](./02-mvvm-comparison/) — MVVM 구현 방식 비교
일반(수동) MVVM 구현과 CommunityToolkit.Mvvm을 비교합니다.

| # | 문서 | 내용 |
|---|------|------|
| 1 | [일반 MVVM 구현](./02-mvvm-comparison/01-traditional-mvvm.md) | INotifyPropertyChanged 직접 구현 |
| 2 | [CommunityToolkit.Mvvm](./02-mvvm-comparison/02-communitytoolkit-mvvm.md) | 소스 제네레이터 기반 MVVM |
| 3 | [비교 요약](./02-mvvm-comparison/03-comparison-summary.md) | 두 방식의 장단점 총정리 |

### 📂 [03-project-setup](./03-project-setup/) — 프로젝트 구성
실제 프로젝트를 생성하고 기본 구조를 잡습니다.

| # | 문서 | 내용 |
|---|------|------|
| 1 | [프로젝트 생성](./03-project-setup/01-project-creation.md) | 솔루션 구조 및 NuGet 패키지 |
| 2 | [의존성 주입 (DI)](./03-project-setup/02-dependency-injection.md) | DI 컨테이너 설정 |
| 3 | [설정 관리](./03-project-setup/03-configuration.md) | appsettings.json 기반 설정 |

### 📂 [04-tech-stack-integration](./04-tech-stack-integration/) — 기술 스택 통합
각 기술을 WPF + MVVM 프로젝트에 통합하는 방법을 설명합니다.

| # | 문서 | 내용 |
|---|------|------|
| 1 | [PostgreSQL + Dapper](./04-tech-stack-integration/01-database-dapper-npgsql.md) | DB 연결과 CRUD |
| 2 | [Serilog 로깅](./04-tech-stack-integration/02-logging-serilog.md) | 구조적 로깅 설정 |
| 3 | [Polly HTTP 복원력](./04-tech-stack-integration/03-http-polly.md) | 재시도·서킷브레이커 |
| 4 | [MQTTnet 메시징](./04-tech-stack-integration/04-mqtt-mqttnet.md) | MQTT v5 통신 |
| 5 | [AWS S3 저장소](./04-tech-stack-integration/05-aws-s3.md) | 사진 업로드/다운로드 |
| 6 | [시스템 트레이](./04-tech-stack-integration/06-system-tray.md) | 트레이 아이콘 통합 |
| 7 | [Hikvision SDK](./04-tech-stack-integration/07-hikvision-sdk.md) | P/Invoke 연동 |
| 8 | [JSON 직렬화](./04-tech-stack-integration/08-json-serialization.md) | System.Text.Json 활용 |

### 📂 [05-practical-examples](./05-practical-examples/) — 실전 예제
배운 내용을 종합하여 실전 패턴을 구현합니다.

| # | 문서 | 내용 |
|---|------|------|
| 1 | [간단한 CRUD 예제](./05-practical-examples/01-simple-crud.md) | 직원 목록 관리 앱 |
| 2 | [화면 네비게이션](./05-practical-examples/02-navigation.md) | 페이지 전환 패턴 |
| 3 | [비동기 패턴](./05-practical-examples/03-async-patterns.md) | async/await와 UI 스레드 |

### 📂 [appendix](./appendix/) — 부록

| # | 문서 | 내용 |
|---|------|------|
| A | [WinForms → WPF 전환 가이드](./appendix/A-winforms-to-wpf.md) | 사고방식 전환 핵심 |
| B | [흔한 실수와 해결법](./appendix/B-common-mistakes.md) | FAQ 및 트러블슈팅 |

---

## 학습 순서 권장

```
1. WPF 기초 (01-fundamentals)
   → WinForms와 뭐가 다른지 감 잡기

2. MVVM 비교 (02-mvvm-comparison)
   → 왜 CommunityToolkit.Mvvm을 쓰는지 이해

3. 프로젝트 구성 (03-project-setup)
   → 실제 프로젝트 뼈대 만들기

4. 기술 스택 (04-tech-stack-integration)
   → 필요한 기술을 하나씩 붙이기

5. 실전 예제 (05-practical-examples)
   → 전체를 조합한 예제로 연습

6. 부록 (appendix)
   → 막히면 참고
```

---

## 사전 준비

- **Visual Studio 2022** (17.12+) 또는 **Rider** (2024.3+)
- **.NET 10.0 SDK** 설치
- **PostgreSQL 16+** 설치 및 실행
- **Git** (소스 관리용)
