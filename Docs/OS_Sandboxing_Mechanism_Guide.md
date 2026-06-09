# [기술 가이드] 운영체제별(OS) 격리 및 권한 제한 메커니즘 비교

본 문서는 리눅스 환경의 핵심 보안 커널 기능인 **Landlock**과 **seccomp**에 대응하여, **macOS** 및 **Windows** 환경에서 애플리케이션 및 에이전트를 안전하게 격리(Sandboxing)하고 권한을 제한하기 위해 활용할 수 있는 운영체제별 네이티브 커널 기술을 정리합니다.

---

## 1. 운영체제별 보안 메커니즘 한눈에 보기

각 운영체제는 커널 아키텍처(Linux, XNU, NT 커널)가 완전히 다르기 때문에 동일한 명칭의 기술을 공유하지 않으며, 아래와 같이 고유의 네이티브 보안 프레임워크를 통해 유사한 목적을 달성합니다.

| 분류 | 🐧 Linux (리눅스) | 🍏 macOS (맥 OS) | 🪟 Windows (윈도우) |
| :--- | :--- | :--- | :--- |
| **커널 기반** | Linux Kernel (LSM) | XNU Kernel (MACF) | Windows NT Kernel |
| **파일/공간 격리**<br>*(Landlock 대응)* | **Landlock**<br>(AppArmor / SELinux) | **App Sandbox**<br>(Sandbox.kext / `.sb` 프로파일) | **AppContainer 파일 ACL**<br>(Windows Sandbox) |
| **행동/시스템콜 제한**<br>*(seccomp 대응)* | **seccomp**<br>(Secure Computing Mode) | **Endpoint Security API**<br>(Taskgated / 하위 시스템콜 제한) | **AppContainer SID 격리**<br>(Process Mitigation Policies) |

---

## 2. 운영체제별 상세 메커니즘 및 매핑

### 🐧 2.1. Linux (기준점)
* **Landlock (파일/공간 격리):** Linux Security Module(LSM) 기반 기술로, 프로세스가 최고 관리자(Root) 권한 유무와 상관없이 스스로 특정 디렉토리(예: `/sandbox`) 외의 파일 시스템 접근 권한을 포기(제한)하도록 커널이 강제합니다.
* **seccomp (행동/시스템콜 제한):** 프로세스가 커널에 요청할 수 있는 시스템 콜(Syscall)의 종류를 필터링합니다. 파일 읽기/쓰기 외에 시스템 리부팅, 네트워크 카드 조작 등 위험한 행동을 하는 시스템 콜 호출 시 즉각 프로세스를 종료하거나 거부합니다.

---

### 🍏 2.2. macOS (리눅스 기능 대응 방식)
macOS에서 Landlock과 seccomp 같은 강력한 격리 환경을 구현하려면 **App Sandbox** 프레임워크와 **Endpoint Security API**를 결합하여 사용해야 합니다.

#### ① 파일/공간 격리 (Landlock 대응) ──▶ **App Sandbox (`Sandbox.kext`)**
* **사용하는 기능:** macOS 커널 내장 모듈인 `Sandbox.kext`와 이를 제어하는 **App Sandbox** 메커니즘을 사용합니다.
* **구현 방법:** 개발자는 규칙 파일인 **샌드박스 프로파일(`.sb` 또는 `.entitlements`)**을 작성하여 프로세스에 바인딩합니다.
* **격리 수준:** 프로파일에 명시되지 않은 사용자 디렉토리(`~/Documents`, `~/Desktop`), 시스템 파일 영역에 대한 읽기/쓰기/실행 권한을 커널 레벨에서 원천 차단합니다. 허용된 임시 폴더(Container 디렉토리) 내부에서만 파일 작업이 가능해집니다.

#### ② 행동/권한 제한 (seccomp 대응) ──▶ **Entitlements 및 Endpoint Security API**
* **사용하는 기능:** macOS는 시스템콜을 개별적으로 촘촘히 막는 seccomp와 완벽히 1:1 매핑되는 기능은 없으나, **Entitlements(권한 선언)** 규격과 **Endpoint Security(ES) 프레임워크**를 통해 하드웨어 및 시스템 행동을 제한합니다.
* **격리 수준:** 프로세스가 네트워크 소켓을 생성하거나, 마이크로폰/카메라에 접근하거나, 다른 프로세스에 신호(Signal)를 보내는 등의 로우 레벨 행동을 전면 통제합니다.

---

### 🪟 2.3. Windows (리눅스 기능 대응 방식)
Windows 환경에서 Landlock과 seccomp 수준의 격리를 달성하기 위해 가장 권장되는 네이티브 기술은 **AppContainer**입니다.

#### ① 파일/공간 격리 (Landlock 대응) ──▶ **AppContainer 전용 하위 파일 시스템 ACL**
* **사용하는 기능:** Windows 커널의 격리 구획 기술인 **AppContainer**를 활용합니다.
* **구현 방법:** 격리하고자 하는 에이전트 프로세스를 AppContainer 토큰 보안 프로필 하위에서 실행합니다.
* **격리 수준:** AppContainer 환경에서 실행된 프로세스는 기본적으로 Windows의 모든 파일 시스템 및 레지스트리에 대해 접근 권한이 '0(Deny)'이 됩니다. 오직 해당 AppContainer의 전용 보안 식별자(SID)에 명시적으로 허용(ACL 설정)된 특정 작업 폴더만 읽고 쓸 수 있어 Landlock과 동일한 공간 격리 효과를 냅니다.

#### ② 행동/권한 제한 (seccomp 대응) ──▶ **AppContainer 권한 제한 및 Process Mitigation Policies**
* **사용하는 기능:** **Process Mitigation Policies(프로세스 완화 정책)** 및 AppContainer 기본 제약 조건을 사용합니다.
* **격리 수준:** Windows NT 커널은 시스템 콜 레벨의 필터링 대신, 프로세스 기능(Capability) 제한을 통해 자원을 통제합니다. 
    * 인터넷 접속 권한(`internetClient`), 로컬 네트워크 접속 권한 등을 차단할 수 있습니다.
    * 완화 정책을 통해 프로세스가 자체적으로 하위 자식 프로세스를 생성하는 행위(WMI 호환 공격 등)를 차단하고, 커널 모드 드라이버를 호출하거나 권한 상승을 유도하는 행위를 원천 봉쇄하여 seccomp와 유사한 행동 제어 효과를 달성합니다.

---

## 3. 개발자/아키텍트를 위한 요약 가이드

만약 크로스 플랫폼(Multi-OS)을 지원하는 AI 에이전트나 애플리케이션 보안 솔루션을 설계 중이고, Docker 없이 각 OS의 **네이티브 커널 기능으로 샌드박싱**을 구현하고자 한다면 아래의 API/기능을 타깃으로 코드를 분기해야 합니다.

1.  **Linux 타깃 빌드:** 리눅스 커널 표준 API인 `sys_landlock_create_ruleset` 및 `prctl(PR_SET_SECCOMP)` 호출 코드를 구현합니다.
2.  **macOS 타깃 빌드:** 맥 표준인 `sandbox_init` API를 호출하거나 애플리케이션 빌드 단계에서 `com.apple.security.app-sandbox` 자격을 코드 사인에 포함합니다.
3.  **Windows 타깃 빌드:** `CreateAppContainerProfile` 및 `CreateProcessAsUser` Win32 API를 호출하여 에이전트 프로세스에 AppContainer 보안 토큰을 강제로 주입해 실행합니다.