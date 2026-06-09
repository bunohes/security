# `src/nemoclaw.ts` 및 CLI 코어 모듈 상세 분석

`src/nemoclaw.ts`는 NemoClaw CLI 프로그램의 **최상위 메인 진입점(Entry Point)**입니다. 과거에는 수천 줄에 달하는 방대한 모놀리식(Monolithic) 라우터 역할을 수행했으나, 현재는 대대적인 리팩토링을 거쳐 **매우 얇은 호환성 프론트 컨트롤러(Front Controller)** 역할만을 수행합니다.

## 1. 주요 역할 및 책임

`src/nemoclaw.ts`는 의도적으로 코드 사이즈가 작게 유지되며, CLI 명령어를 파싱하고 분기하는 무거운 작업은 하위 모듈(`oclif` 프레임워크 및 `src/lib/cli/public-dispatch.ts`)로 완전히 위임합니다.

### 1) CLI 퍼블릭 디스패처 호출
사용자가 터미널에서 `nemoclaw` 명령어를 실행하면, 내부적으로 `src/lib/cli/public-dispatch.ts`에 위치한 `dispatchCli` 함수를 호출합니다.
```typescript
import { dispatchCli } from "./lib/cli/public-dispatch";

exports.main = dispatchCli;
module.exports.dispatchCli = dispatchCli;
```

### 2) 테스트 및 인-프로세스(In-process) 호환성 제공
자동화 테스트나 외부 모듈에서 CLI를 직접 가져다 쓸 때를 대비하여 `mainPromise`를 반환합니다. `NEMOCLAW_DISABLE_AUTO_DISPATCH` 환경 변수를 통해 자동 실행을 제어할 수 있습니다.

## 2. 명령어 탐색 및 실행 구조 (Oclif 프레임워크)

NemoClaw CLI의 실제 명령어 파싱, 도움말 렌더링, 명령어 실행 로직은 **`src/commands/`** 하위 디렉토리에 전담되어 있습니다.

### 주요 커맨드 디렉토리 구성
* **`src/commands/root/`**: 전역 기본 명령어들.
* **`src/commands/sandbox/`**: 샌드박스의 생성, 시작, 정지 등 생애주기 관리 명령어.
* **`src/commands/credentials/`**: 자격 증명(API Key 등) 관리.
* **`src/commands/inference/`**: 추론(LLM) 공급자 및 라우팅 설정 변경 명령어.
* **`src/commands/tunnel/`**: 네트워크 포트 터널링 관리.
* **`src/commands/internal/`**: 내부 디버깅 및 시스템 호출용 은닉(Hidden) 명령어.

## 3. 핵심 비즈니스 로직 (`src/lib/`)

CLI 명령어가 파싱된 이후 실제 동작을 수행하는 핵심 도메인 로직은 `src/lib/` 하위에 철저히 모듈화되어 있습니다.

* **`src/lib/core/`, `domain/`, `state/`**: 샌드박스의 런타임 상태 구조 및 핵심 비즈니스 모델을 관리합니다.
* **`src/lib/sandbox/`**: Docker 데몬이나 OpenShell과 통신하여 컨테이너를 직접 구동하고 제어하는 인프라 엔진입니다.
* **`src/lib/onboard/`, `deploy/`**: 사용자가 최초로 샌드박스를 구축할 때 거치는 설정 마법사 로직입니다.
* **`src/lib/policy/`, `shields/`, `security/`**: 네트워크 통신 제어, 파일 접근 권한, 로컬 네트워크 탐색(SSRF) 차단 등 보안의 핵심 뼈대를 담당합니다.
* **`src/lib/inference/`**: 외부 LLM 프로바이더나 로컬 NIM, vLLM과의 연결 및 라우팅을 조율합니다.
* **`src/lib/messaging/`**: 메시징 채널(Telegram, Discord, Slack, WeChat 등) 연동 설정 및 충돌 감지를 관리합니다.
* **`src/lib/gateway-runtime-action.ts`, `gateway-token-command.ts`**: 추론 게이트웨이 런타임 및 토큰 관리를 수행합니다.
* **`src/lib/share-command.ts`**: 샌드박스 공유 기능을 제공합니다.
* **`src/lib/trace.ts`**: 런타임 트레이싱 및 디버그 로그 수집을 담당합니다.
* **`src/lib/oauth-device-code.ts`**: OAuth 디바이스 코드 플로우를 통한 인증을 처리합니다.
* **`src/lib/skill-install.ts`**: 에이전트 스킬 설치 기능을 관리합니다.

## 요약
NemoClaw CLI 아키텍처는 이제 `src/nemoclaw.ts` 하나에 모든 것을 담지 않고, **명령어 선언부(`src/commands/`)**와 **비즈니스 실행부(`src/lib/`)**로 완전히 분리되어 뛰어난 확장성과 유지보수성을 가지게 되었습니다.
