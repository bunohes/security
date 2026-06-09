# NemoClaw OpenShell 보안 모듈 재개발 명세서

> **버전:** 1.0  
> **작성일:** 2026-05-19  
> **대상:** NemoClaw OpenShell 샌드박스 보안 레이어 (OpenClaw·Hermes 제외)

---

## 1. 프로젝트 개요

### 1.1 목적

NemoClaw의 OpenShell 샌드박스 보안 구현체를 처음부터 재설계한다. 기존 코드베이스의 구조와 설계 결정을 참조하되, 전체를 새로 구현하여 기술 부채 해소와 보안 경계 명확화를 달성한다.

### 1.2 범위

| 포함 | 제외 |
|------|------|
| OpenShell 샌드박스 격리 | OpenClaw 에이전트 런타임 |
| 네트워크 정책 엔진 | Hermes 에이전트 |
| 자격증명·시크릿 관리 | 외부 클라우드 인프라 |
| SSRF 방어 레이어 | 모델 추론 엔진 |
| 컨테이너 보안 하드닝 | UI/프론트엔드 |
| 사전 점검(Preflight) | — |

### 1.3 기존 코드 분석 요약

기존 구현에서 식별된 핵심 보안 컴포넌트:

| 컴포넌트 | 경로 | 역할 |
|----------|------|------|
| SSRF 방어 | `nemoclaw/src/blueprint/ssrf.ts` | DNS 핀닝·사설망 차단 |
| 사설망 차단 목록 | `nemoclaw-blueprint/private-networks.yaml` | IPv4/IPv6 CIDR 블록리스트 |
| 비밀 패턴 탐지 | `src/lib/security/secret-patterns.ts` | 30+ 토큰 패턴 |
| 출력 리댁션 | `src/lib/security/redact.ts` | 스트림 실시간 치환 |
| 자격증명 필터 | `src/lib/security/credential-filter.ts` | 백업·마이그레이션 전 정제 |
| 네트워크 정책 | `nemoclaw-blueprint/policies/` | 기본 차단·선택적 허용 |
| 컨테이너 하드닝 | `Dockerfile`, `Dockerfile.base` | 비루트·고정 의존성 |
| 스냅샷 보안 | `nemoclaw/src/blueprint/snapshot.ts` | 심링크 공격 방지 |
| 사전 점검 | `src/lib/onboard/preflight.ts` | 호스트 환경 검증 |

---

## 2. 팀 구성 및 역할 분담

### 2.1 팀원 구성 (4명)

```
┌─────────────────────────────────────────────────────────────┐
│                     보안 모듈 재개발 팀                      │
├──────────────┬──────────────┬──────────────┬───────────────┤
│   팀원 A      │   팀원 B      │   팀원 C      │   팀원 D      │
│ 네트워크 보안  │ 자격증명 보안  │ 컨테이너 보안  │ 정책 & 통합   │
│   리드        │   리드        │   리드        │   리드        │
└──────────────┴──────────────┴──────────────┴───────────────┘
```

---

### 2.2 팀원 A — 네트워크 보안 리드

**주요 책임 영역:**

- SSRF 방어 엔진 (완전 재구현)
- 사설망 CIDR 블록리스트 관리
- DNS 핀닝 및 TOCTOU 방어
- L7 프록시 연동 인터페이스

**담당 구현 모듈:**

| 모듈 | 설명 |
|------|------|
| `security/network/ssrf-validator.ts` | URL 유효성 검사·DNS 핀닝 |
| `security/network/private-networks.ts` | BlockList 로드·조회 API |
| `security/network/proxy-enforcer.ts` | 샌드박스 이그레스 프록시 제어 |
| `nemoclaw-blueprint/private-networks.yaml` | CIDR 블록리스트 (갱신) |

**핵심 설계 요구사항:**

```typescript
// 필수 API 시그니처
interface SsrfValidator {
  validateUrl(url: string): Promise<ValidatedUrl>;
  isPinnedIpPrivate(hostname: string): Promise<boolean>;
}

interface ValidatedUrl {
  originalUrl: string;
  pinnedUrl: string;   // DNS 고정 URL (TOCTOU 방지)
  scheme: 'https' | 'http';
}
```

---

### 2.3 팀원 B — 자격증명 보안 리드

**주요 책임 영역:**

- 시크릿 패턴 탐지 엔진 (단일 진실 소스)
- 실시간 출력 리댁션
- 자격증명 필터링 및 마이그레이션 정제
- 워크스페이스 메모리 시크릿 스캐너

**담당 구현 모듈:**

| 모듈 | 설명 |
|------|------|
| `security/credentials/secret-patterns.ts` | 토큰·API 키 패턴 30+ |
| `security/credentials/redact.ts` | 부분·전체·URL 리댁션 |
| `security/credentials/credential-filter.ts` | 백업 전 자격증명 제거 |
| `security/credentials/credential-hash.ts` | SHA-256 정규화 해시 |
| `security/credentials/secret-scanner.ts` | 메모리 쓰기 전 스캔 |

**핵심 설계 요구사항:**

```typescript
// 필수 API 시그니처
interface SecretRedactor {
  redactPartial(text: string): string;     // 앞 4자 유지
  redactFull(text: string): string;        // 전체 <REDACTED>
  redactUrl(url: string): string;          // 자격증명·쿼리 제거
  redactError(err: unknown): SafeError;    // 에러 객체 정제
}

interface SecretScanner {
  scan(content: string): ScanResult;
  isMemoryPath(path: string): boolean;     // 영속 경로 감지
}
```

---

### 2.4 팀원 C — 컨테이너 보안 리드

**주요 책임 영역:**

- Dockerfile 멀티스테이지 빌드 설계
- 비루트 사용자·그룹 격리
- 파일시스템 권한 및 불변성 설계
- Landlock 파일시스템 정책 통합
- 사전 점검(Preflight) 시스템

**담당 구현 모듈:**

| 모듈 | 설명 |
|------|------|
| `Dockerfile` | 런타임 이미지 (보안 패치 적용) |
| `Dockerfile.base` | 고정 의존성 베이스 이미지 |
| `security/container/preflight.ts` | 호스트 환경 검증 |
| `security/container/filesystem-policy.ts` | Landlock 정책 생성 |
| `security/container/capability-drop.ts` | Linux 권한 최소화 |

**핵심 설계 요구사항:**

```
# 컨테이너 보안 제약 (필수)
- 실행 사용자: sandbox (비루트)
- 게이트웨이 사용자: gateway (별도 분리)
- 쓰기 불가 경로: /usr, /lib, /proc, /dev, /app, /etc (Landlock)
- 삭제 대상 도구: gcc, g++, make, netcat (런타임 이미지)
- 설정 파일 해시: SHA-256 빌드 타임 고정
- NPM 오프라인: 이미지 빌드 완료 후 레지스트리 차단
```

---

### 2.5 팀원 D — 정책 & 통합 리드

**주요 책임 영역:**

- 네트워크 정책 YAML 스키마 설계
- 정책 티어 시스템 (restricted/balanced/open)
- 정책 프리셋 관리 (메시징, 패키지, 검색)
- 스냅샷 보안 (심링크 공격 방지)
- 전체 모듈 통합 및 인터페이스 정의

**담당 구현 모듈:**

| 모듈 | 설명 |
|------|------|
| `nemoclaw-blueprint/policies/` | 기본·허용 정책 YAML |
| `nemoclaw-blueprint/policies/presets/` | 채널별 프리셋 |
| `security/policy/policy-engine.ts` | 정책 파싱·검증 |
| `security/policy/snapshot-security.ts` | 심링크 방지·복원 |
| `security/policy/initial-policy.ts` | 온보딩 정책 빌드 |

**핵심 설계 요구사항:**

```yaml
# 정책 기본 원칙
- 기본값: deny-all (명시적 허용 목록만)
- 서드파티: 프리셋 명시적 선택으로만 활성화
- 각 엔드포인트: 바이너리·HTTP 메서드·경로 제한
- 프리셋 목적 필드: 감사 추적 필수
```

---

## 3. 핵심 보안 구현 요구사항

### 3.1 SSRF 방어 (우선순위: Critical)

**요구사항:**

1. **URL 스킴 제한:** `https://`, `http://` 외 모든 스킴 거부
2. **사설망 차단:** 14개 IPv4 + 14개 IPv6 CIDR 범위 전면 차단
3. **DNS 핀닝:** 검증 후 실제 연결 전 IP 재확인으로 TOCTOU 방어
4. **호스트명 차단:** `localhost`, `local`, `invalid` 및 서브도메인 일치
5. **IPv6 매핑 IPv4:** `::ffff:` 형식의 우회 시도 차단

**수용 기준:**
- 모든 사설 IP 범위에 대한 단위 테스트 통과
- DNS 리바인딩 시뮬레이션 테스트 통과
- CLI와 플러그인 구현체 간 동등성(parity) 테스트 통과

---

### 3.2 비밀 관리 (우선순위: Critical)

**요구사항:**

1. **패턴 단일 소스:** `secret-patterns.ts` 하나만 존재, 모든 모듈이 참조
2. **실시간 리댁션:** stdout/stderr 스트림에서 토큰 즉시 치환
3. **부분 리댁션:** CLI 출력은 앞 4자 보존 (`nvapi-XXXX...`)
4. **전체 리댁션:** 진단 덤프·로그는 완전 대체 (`<REDACTED>`)
5. **URL 자격증명 제거:** username, password, 민감 쿼리 파라미터 제거
6. **에러 객체 정제:** message, cmd, stdout, stderr, stack 전체 스캔
7. **순환 참조 감지:** 재귀 리댁션 시 무한 루프 방지

**커버리지 요구사항 (최소):**

| 토큰 유형 | 패턴 수 |
|-----------|---------|
| NVIDIA (nvapi-, nvcf-) | 2 |
| GitHub (ghp_, gho_, ghu_, ghs_, ghr_, github_pat_) | 6 |
| OpenAI / Anthropic (sk-proj-, sk-ant-, sk-) | 3 |
| AWS (AKIA*, ASIA*) | 2 |
| Slack (xox[bpas]-, xapp-) | 5 |
| HuggingFace, GitLab, Groq, PyPI | 4 |
| Telegram Bot, Discord Bot | 2 |
| PEM 개인키 | 1 |
| Bearer Authorization | 1 |
| **합계** | **26+** |

---

### 3.3 네트워크 정책 엔진 (우선순위: High)

**요구사항:**

1. **기본 차단:** 정책 미정의 모든 이그레스 트래픽 차단
2. **명시적 허용:** 각 엔드포인트에 바이너리·메서드·경로 제한 명시
3. **프리셋 격리:** 메시징/패키지/검색 플랫폼은 기본 정책에 미포함
4. **티어 시스템:**
   - `restricted`: 핵심 에이전트 도구만 허용
   - `balanced`: 개발 도구 + 웹 검색 추가
   - `open`: 광범위한 서드파티 접근 허용
5. **감사 필드:** 모든 정책 항목에 `purpose` 필드 필수

**허용 프리셋 목록 (구현 필수):**

| 카테고리 | 프리셋 |
|----------|--------|
| 메시징 | slack, discord, telegram, whatsapp, wechat |
| 개발 | github, npm, pypi, brew, huggingface |
| 검색 | brave |
| 추론 | local-inference |
| 생산성 | jira, outlook |

---

### 3.4 컨테이너 보안 하드닝 (우선순위: High)

**요구사항:**

1. **멀티스테이지 빌드:** 빌더 스테이지와 런타임 스테이지 분리
2. **빌드 도구 제거:** gcc, g++, make, netcat 런타임 이미지에 미포함
3. **비루트 실행:** 에이전트(`sandbox`), 게이트웨이(`gateway`) 별도 사용자
4. **의존성 고정:** 베이스 이미지 sha256, apt 패키지 버전 고정
5. **설정 무결성:** openclaw-config.json SHA-256 빌드 타임 핀
6. **NPM 오프라인:** 설치 완료 후 `NPM_CONFIG_OFFLINE=true` 강제
7. **파일시스템 권한:**
   - `.openclaw/`: `g+w` + setgid (그룹 상속)
   - `.nemoclaw/`: sticky bit (1755) 루트 소유
   - 블루프린트: `root:root 755` 불변
   - 홈 rc 파일: 루트 소유 444 (쓰기 불가)
8. **환경변수 주입:** Python 인터폴레이션 우회 방지 (ENV 지시어 사용)

---

### 3.5 심링크 공격 방지 (우선순위: High)

**요구사항:**

1. **경로 검증:** 소스·대상 경로 및 모든 상위 디렉터리($HOME까지) 심링크 검사
2. **거부 시 오류:** 심링크 발견 즉시 작업 중단
3. **레거시 처리:** `.openclaw-data` 심링크 탐지 후 실제 디렉터리로 구체화
4. **샌드박스명 검증:** `^[a-z][a-z0-9-]{0,62}$` 형식 강제

---

### 3.6 사전 점검(Preflight) (우선순위: Medium)

**요구사항:**

1. **Docker 상태 확인:** 설치 여부·접근 가능성·cgroup 버전
2. **DNS 프로브:** 컨테이너 내부에서 DNS 해석 가능 여부 확인 (RFC 6761 `.invalid` TLD 사용)
3. **포트 충돌 감지:** lsof 기반 + Node.js net 폴백 이중 확인
4. **스토리지 드라이버 충돌:** 중첩 오버레이 감지 (Docker 26+)
5. **수정 계획:** 조치 항목을 `info/manual/auto/sudo` 카테고리로 분류
6. **GPU 확인:** nvidia-smi, CDI 스펙, CUDA cuInit 단계적 검증

---

## 4. 개발 로드맵

```
Week 1    Week 2    Week 3    Week 4    Week 5    Week 6    Week 7    Week 8    Week 9    Week 10
   │         │         │         │         │         │         │         │         │         │
Phase 1: 기반 설계 (2주)
   ├─────────┤
   A: SSRF 인터페이스·CIDR 목록 설계
   B: 시크릿 패턴 카탈로그·리댁션 API 설계
   C: 컨테이너 아키텍처·사용자 모델 설계
   D: 정책 YAML 스키마·티어 설계
             │
Phase 2: 핵심 모듈 구현 (3주)
             ├───────────────────┤
             A: ssrf-validator, private-networks, proxy-enforcer
             B: secret-patterns, redact, credential-filter, credential-hash
             C: Dockerfile.base, preflight, capability-drop
             D: policy-engine, 기본 정책 YAML, 프리셋 9종
                               │
Phase 3: 통합 & 고급 기능 (2주)
                               ├───────────────┤
                               A: CLI↔플러그인 동등성 테스트
                               B: secret-scanner, 스트림 리댁션 통합
                               C: Dockerfile 런타임 이미지, Landlock 통합
                               D: snapshot-security, initial-policy, 티어 완성
                                             │
Phase 4: 테스트·감사·릴리스 (3주)
                                             ├───────────────────────┤
                                             전체: 크로스팀 보안 리뷰
                                             전체: 커버리지 임계값 달성
                                             전체: 침투 테스트 시나리오
                                             전체: 문서화 및 최종 배포
```

### 4.1 Phase 1 — 기반 설계 (Week 1–2)

**목표:** 모든 팀원이 인터페이스 계약과 데이터 모델에 합의

| 팀원 | 주요 산출물 |
|------|-------------|
| A | `SsrfValidator` 인터페이스, CIDR 목록 구조 확정 |
| B | `SecretPattern[]` 카탈로그 초안, 리댁션 레벨 정의 |
| C | 컨테이너 사용자 모델, 파일시스템 트리 설계 |
| D | 정책 YAML 스키마 v1, 티어 정의 문서 |

**완료 기준:** 인터페이스 문서 리뷰 완료, 팀 합의

---

### 4.2 Phase 2 — 핵심 모듈 구현 (Week 3–5)

**목표:** 각 담당 모듈의 핵심 기능 구현 완료

| 팀원 | 주요 구현 | 완료 기준 |
|------|-----------|-----------|
| A | `ssrf-validator.ts`, `private-networks.ts`, `proxy-enforcer.ts` | 사설 IP 전범위 단위 테스트 통과 |
| B | `secret-patterns.ts`, `redact.ts`, `credential-filter.ts`, `credential-hash.ts` | 26+ 패턴 모두 탐지, 리댁션 단위 테스트 통과 |
| C | `Dockerfile.base`, `preflight.ts`, `capability-drop.ts` | 이미지 빌드 성공, 비루트 실행 검증 |
| D | `policy-engine.ts`, `openclaw-sandbox.yaml`, 프리셋 9종 | 정책 파싱 테스트, 프리셋 활성화 테스트 통과 |

---

### 4.3 Phase 3 — 통합 & 고급 기능 (Week 6–7)

**목표:** 모듈 간 통합, 고급 보안 기능 구현

| 팀원 | 주요 작업 |
|------|-----------|
| A | CLI↔플러그인 parity 테스트, DNS 리바인딩 시뮬레이션 |
| B | `secret-scanner.ts` (메모리 경로), 스트림 리댁션 통합 |
| C | 런타임 `Dockerfile`, Landlock 정책 연동, `filesystem-policy.ts` |
| D | `snapshot-security.ts` (심링크 방지), `initial-policy.ts`, `tiers.yaml` 완성 |

---

### 4.4 Phase 4 — 테스트·감사·릴리스 (Week 8–10)

**목표:** 출시 가능한 보안 품질 달성

**공통 작업:**
- 크로스팀 코드 리뷰 (각 팀원 타팀 모듈 리뷰)
- 침투 테스트 시나리오 실행 (SSRF 우회, 시크릿 유출, 정책 이탈)
- 커버리지 임계값 달성 (CLI: 기존 이상, 플러그인: 기존 이상)
- 보안 감사 문서 작성
- `docs/` 사용자 문서 업데이트
- CHANGELOG 작성 및 릴리스 태그

---

## 5. 기술 스택 및 제약사항

### 5.1 언어 및 런타임

| 레이어 | 언어 | 규칙 |
|--------|------|------|
| CLI (`src/`) | TypeScript (→ `dist/`) | `tsconfig.cli.json` |
| 플러그인 (`nemoclaw/src/`) | TypeScript | `nemoclaw/tsconfig.json` |
| 런처 (`bin/`) | JavaScript CJS | Node.js 22.16+ |
| 테스트 (`test/`) | JavaScript ESM | Vitest |
| 블루프린트 | YAML | OpenShell 스키마 |

### 5.2 코드 스타일

- **Biome** 린터·포매터 (루트 `biome.json`)
- **ShellCheck + shfmt** (셸 스크립트)
- **Conventional Commits** (commitlint, prek 훅 강제)
- **SPDX 헤더** 자동 삽입 (pre-commit)

### 5.3 테스트 전략

```
test/                   # CLI 통합 테스트 (ESM)
nemoclaw/src/**/*.test.ts  # 플러그인 단위 테스트 (소스 코-로케이션)
test/e2e/               # E2E 테스트 (Brev 임시 인스턴스)
```

**보안 전용 테스트 요구사항:**

| 테스트 유형 | 담당 | 최소 케이스 수 |
|-------------|------|----------------|
| SSRF 단위 (CIDR 전범위) | A | 28+ |
| DNS 핀닝·리바인딩 | A | 5+ |
| 시크릿 패턴 탐지 | B | 26+ |
| 리댁션 레벨별 | B | 15+ |
| 컨테이너 권한 검증 | C | 10+ |
| 정책 파싱·검증 | D | 12+ |
| 심링크 공격 방지 | D | 8+ |

---

## 6. 보안 설계 원칙

1. **기본 차단 (Deny-by-default):** 허용 목록에 없는 모든 트래픽·접근 차단
2. **명시적 선택 확대 (Explicit opt-in):** 서드파티 연동은 사용자 명시 선택
3. **단일 진실 소스 (Single source of truth):** 패턴·정책·CIDR은 중복 없이 한 곳에
4. **심층 방어 (Defense in depth):** SSRF + Landlock + 프록시 + 정책 다중 레이어
5. **최소 권한 (Least privilege):** 비루트, Linux Capability 최소화
6. **불변성 (Immutability):** 블루프린트·설정 해시 빌드 타임 고정
7. **감사 가능성 (Auditability):** 모든 정책 항목에 `purpose` 필드 필수

---

## 7. 인터페이스 계약 (팀 간 합의 필수)

### 7.1 팀 A ↔ 팀 D

```typescript
// policy-engine이 ssrf-validator를 호출하는 API
interface SsrfValidator {
  validateUrl(url: string, options?: { allowHttp?: boolean }): Promise<ValidatedUrl>;
}
```

### 7.2 팀 B ↔ 팀 C (컨테이너 런타임)

```typescript
// runner가 redact를 호출하는 API (스트림 파이프)
interface StreamRedactor {
  createRedactTransform(): Transform;  // Node.js Transform 스트림
}
```

### 7.3 팀 B ↔ 팀 D (스냅샷)

```typescript
// snapshot이 credential-filter를 호출하는 API
interface CredentialFilter {
  sanitizeConfigFile(filePath: string): Promise<void>;
  filterObject(obj: unknown): unknown;
}
```

### 7.4 팀 C ↔ 팀 D (정책 적용)

```typescript
// preflight이 policy-engine을 호출하는 API
interface PolicyEngine {
  loadPolicy(policyPath: string): Promise<SandboxPolicy>;
  applyPresets(base: SandboxPolicy, presets: string[]): SandboxPolicy;
  validatePolicy(policy: SandboxPolicy): ValidationResult;
}
```

---

## 8. 완료 기준 (Definition of Done)

각 Phase 완료 시 다음 조건을 모두 만족해야 한다:

- [ ] `make check` (전체 린터) 통과
- [ ] `npm test` (CLI + 플러그인 테스트) 통과
- [ ] 커버리지 임계값 이상 유지
- [ ] 새 코드에 SSRF·시크릿 유출 취약점 없음
- [ ] 모든 새 파일에 SPDX 헤더 포함
- [ ] PR 템플릿 체크리스트 완료
- [ ] 관련 docs/ 업데이트 완료

---

---

## 9. 전체 디렉터리 구조 및 팀원 검토 매핑

> 범례: **A** = 네트워크 보안 | **B** = 자격증명 보안 | **C** = 컨테이너 보안 | **D** = 정책 & 통합 | `[제외]` = 보안 범위 외

```
NemoClaw/
├── .agents/                          [제외] 에이전트 스킬 (자동생성 문서)
│   └── skills/
│       ├── nemoclaw-maintainer-security-code-review/  [D] 보안 리뷰 워크플로우 참조
│       ├── nemoclaw-user-configure-security/          [D] 사용자 보안 가이드 참조
│       └── nemoclaw-user-manage-policy/               [D] 정책 관리 가이드 참조
│
├── .github/                          [D] CI/CD 파이프라인
│   ├── actions/
│   │   ├── basic-checks/             [D] 기본 검사 액션
│   │   ├── resolve-hermes-base-image/ [제외]
│   │   └── resolve-sandbox-base-image/ [C] 샌드박스 이미지 리졸버
│   └── workflows/
│       ├── code-scanning.yaml        [C] 코드 보안 스캐닝 설정
│       ├── docker-pin-check.yaml     [C] 도커 이미지 핀 검증
│       ├── main.yaml                 [D] 메인 CI 파이프라인
│       ├── pr.yaml                   [D] PR 검사
│       ├── sandbox-images-and-e2e.yaml [C] 샌드박스 이미지 빌드
│       └── (기타 워크플로우)             [D]
│
├── agents/                           [제외] 에이전트 런타임 (Hermes/OpenClaw)
│   ├── hermes/                       [제외]
│   └── openclaw/                     [제외]
│
├── bin/                              [B] CLI 런처 (시크릿 노출 경로 확인)
│   ├── nemoclaw.js                   [B] 메인 CLI 엔트리포인트 - 시크릿 로깅 확인
│   ├── nemohermes.js                 [제외]
│   └── lib/
│       ├── credentials.js            [B] 자격증명 처리 (CJS 레이어)
│       ├── tiers.js                  [D] 티어 정의 참조
│       ├── ports.js                  [C] 포트 관리
│       └── (기타)                     [D]
│
├── ci/                               [D] CI 설정
│   ├── coverage-threshold-cli.json   [D] CLI 커버리지 임계값 관리
│   ├── coverage-threshold-plugin.json [D] 플러그인 커버리지 임계값 관리
│   └── env-var-doc-allowlist.json    [B] 환경변수 문서화 허용목록
│
├── docs/                             [제외] 사용자 문서
│   ├── security/                     [D] 보안 관련 문서 (정책 변경 시 동기화)
│   │   ├── best-practices.md         [D]
│   │   ├── credential-storage.md     [B]
│   │   └── openclaw-controls.md      [D]
│   ├── network-policy/               [A] 네트워크 정책 문서 (코드 변경 시 동기화)
│   └── deployment/
│       └── sandbox-hardening.md      [C] 컨테이너 하드닝 문서
│
├── nemoclaw/                         ★ 플러그인 핵심 소스
│   └── src/
│       ├── blueprint/                [A][D] 블루프린트 런타임
│       │   ├── ssrf.ts               [A] SSRF 방어 핵심
│       │   ├── private-networks.ts   [A] 사설망 블록리스트
│       │   ├── snapshot.ts           [D] 스냅샷 보안 (심링크 방지)
│       │   ├── runner.ts             [B][C] 샌드박스 실행 제어
│       │   └── state.ts              [D] 상태 관리
│       ├── commands/                 [D] 슬래시 커맨드
│       ├── lib/                      [D] 공유 라이브러리
│       ├── onboard/                  [D][C] 온보딩 설정
│       └── security/                 ★ 보안 전담 디렉터리
│           └── secret-scanner.ts     [B] 메모리 시크릿 스캐너
│
├── nemoclaw-blueprint/               ★ 샌드박스 정책 핵심
│   ├── policies/                     [D][A] 네트워크 정책
│   │   ├── openclaw-sandbox.yaml     [D] 기본 정책 (deny-all)
│   │   ├── openclaw-sandbox-permissive.yaml [D] 개발용 허용 정책
│   │   ├── tiers.yaml                [D] 티어 정의
│   │   └── presets/                  [D] 채널별 프리셋
│   │       ├── slack.yaml            [D]
│   │       ├── discord.yaml          [D]
│   │       ├── telegram.yaml         [D]
│   │       ├── whatsapp.yaml         [D]
│   │       ├── wechat.yaml           [D]
│   │       ├── github.yaml           [D]
│   │       ├── npm.yaml              [D]
│   │       ├── pypi.yaml             [D]
│   │       ├── brew.yaml             [D]
│   │       ├── huggingface.yaml      [D]
│   │       ├── brave.yaml            [D]
│   │       ├── jira.yaml             [D]
│   │       ├── outlook.yaml          [D]
│   │       └── local-inference.yaml  [D]
│   ├── private-networks.yaml         [A] IPv4/IPv6 CIDR 블록리스트
│   ├── model-specific-setup/         [제외] 모델 호환성 레지스트리
│   ├── openclaw-plugins/             [제외] OpenClaw 플러그인
│   ├── router/                       [A] LLM 라우터 (프록시 경로)
│   └── scripts/                      [D] 블루프린트 스크립트
│
├── schemas/                          [D] JSON 스키마
│
├── scripts/                          [C][D] 설치·자동화 스크립트
│   ├── checks/                       [D] 검사 스크립트
│   ├── e2e/                          [D] E2E 테스트 스크립트
│   └── lib/                          [C] 공유 스크립트 라이브러리
│
├── src/                              ★ CLI 핵심 소스
│   ├── commands/                     [각 담당별 연관 커맨드]
│   │   ├── credentials/              [B] 자격증명 커맨드
│   │   ├── sandbox/
│   │   │   ├── channels/             [D] 메시징 채널
│   │   │   ├── gateway/              [A] 게이트웨이 커맨드
│   │   │   ├── policy/               [D] 정책 커맨드
│   │   │   ├── shields/              [D] 방어막(shields) 커맨드
│   │   │   └── snapshot/             [D] 스냅샷 커맨드
│   │   └── (기타)                     [D]
│   └── lib/                          ★ 핵심 라이브러리
│       ├── security/                 ★ 보안 전담 디렉터리
│       │   ├── secret-patterns.ts    [B] 단일 시크릿 패턴 소스
│       │   ├── redact.ts             [B] 리댁션 엔진
│       │   ├── credential-filter.ts  [B] 자격증명 필터
│       │   ├── credential-hash.ts    [B] 자격증명 해시
│       │   └── README.md             [B] 보안 모듈 문서
│       ├── onboard/
│       │   ├── preflight.ts          [C] 사전 점검
│       │   └── initial-policy.ts     [D] 초기 정책 생성
│       ├── private-networks.ts       [A] CLI측 사설망 검증 (parity)
│       ├── policy/                   [D] 정책 로직
│       ├── credentials/              [B] 자격증명 저장·검증
│       ├── sandbox/                  [C] 샌드박스 제어
│       ├── shields/                  [D] 방어막 로직
│       ├── adapters/
│       │   ├── docker/               [C] Docker 어댑터
│       │   ├── http/                 [A] HTTP 클라이언트
│       │   └── openshell/            [C] OpenShell 어댑터
│       ├── diagnostics/              [B][C] 진단 (시크릿 리댁션 적용 필수)
│       ├── runner.ts                 [B] 명령 실행 (리댁션 파이프라인)
│       └── (기타 lib)                 [D]
│
└── test/                             ★ 테스트
    ├── (*.test.js)                   [각 담당 모듈별]
    └── e2e/
        ├── validation_suites/
        │   ├── security/             [A][B] 보안 E2E 테스트 (핵심)
        │   ├── platform/             [C] 플랫폼 검증
        │   └── (기타)                 [D]
        └── nemoclaw_scenarios/       [D] 시나리오 테스트
```

---

### 9.1 팀원별 주요 검토 파일 요약

#### 팀원 A — 네트워크 보안

| 우선순위 | 파일 |
|----------|------|
| P0 | `nemoclaw/src/blueprint/ssrf.ts` |
| P0 | `nemoclaw/src/blueprint/private-networks.ts` |
| P0 | `src/lib/private-networks.ts` |
| P0 | `nemoclaw-blueprint/private-networks.yaml` |
| P1 | `nemoclaw-blueprint/router/` |
| P1 | `src/lib/adapters/http/` |
| P2 | `.github/workflows/docker-pin-check.yaml` |

#### 팀원 B — 자격증명 보안

| 우선순위 | 파일 |
|----------|------|
| P0 | `src/lib/security/secret-patterns.ts` |
| P0 | `src/lib/security/redact.ts` |
| P0 | `src/lib/security/credential-filter.ts` |
| P0 | `src/lib/security/credential-hash.ts` |
| P0 | `nemoclaw/src/security/secret-scanner.ts` |
| P1 | `src/lib/credentials/` |
| P1 | `src/commands/credentials/` |
| P1 | `src/lib/runner.ts` (리댁션 파이프라인) |
| P2 | `src/lib/diagnostics/` (시크릿 노출 확인) |
| P2 | `bin/lib/credentials.js` |

#### 팀원 C — 컨테이너 보안

| 우선순위 | 파일 |
|----------|------|
| P0 | `Dockerfile` |
| P0 | `Dockerfile.base` |
| P0 | `src/lib/onboard/preflight.ts` |
| P1 | `src/lib/adapters/docker/` |
| P1 | `src/lib/sandbox/` |
| P1 | `.github/workflows/code-scanning.yaml` |
| P2 | `scripts/` (설치 스크립트 보안) |
| P2 | `.github/actions/resolve-sandbox-base-image/` |

#### 팀원 D — 정책 & 통합

| 우선순위 | 파일 |
|----------|------|
| P0 | `nemoclaw-blueprint/policies/openclaw-sandbox.yaml` |
| P0 | `nemoclaw-blueprint/policies/tiers.yaml` |
| P0 | `nemoclaw-blueprint/policies/presets/` (전체) |
| P0 | `nemoclaw/src/blueprint/snapshot.ts` |
| P0 | `src/lib/onboard/initial-policy.ts` |
| P1 | `src/lib/policy/` |
| P1 | `src/lib/shields/` |
| P1 | `src/commands/sandbox/policy/` |
| P1 | `src/commands/sandbox/shields/` |
| P1 | `ci/coverage-threshold-*.json` |
| P2 | `schemas/` |
| P2 | `test/e2e/validation_suites/security/` |

---

### 9.2 공동 검토 구역

다음 파일들은 **보안 경계를 가로지르므로 최소 2명이 리뷰**해야 한다:

| 파일 | 검토자 | 이유 |
|------|--------|------|
| `src/lib/runner.ts` | B + C | 프로세스 실행 + 출력 리댁션 교차 |
| `nemoclaw/src/blueprint/runner.ts` | A + D | 프록시 연동 + 정책 적용 교차 |
| `src/lib/onboard/initial-policy.ts` | C + D | 컨테이너 기동 + 정책 생성 교차 |
| `src/lib/diagnostics/` | B + C | 진단 덤프 = 시크릿 유출 위험 |
| `test/e2e/validation_suites/security/` | A + B | 보안 테스트 전범위 |
| `Dockerfile` 패치 섹션 | A + C | L7 프록시 패치 + 컨테이너 경계 |

---

*이 문서는 `docs/CONTRIBUTING.md` 스타일 가이드를 따르며, 구현 진행에 따라 갱신된다.*
