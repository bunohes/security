# NemoClaw Blueprint 상세 분석 (`nemoclaw-blueprint/`)

`nemoclaw-blueprint/` 디렉토리는 OpenShell 환경 내에서 OpenClaw 에이전트가 격리될 **'샌드박스(Sandbox)의 형태와 규칙'**을 정의하는 설계도 모음입니다. 시스템 레벨의 네트워크 격리, 제로 트러스트(Zero-Trust) 기반의 접근 제어 등 가장 강력한 하부 보안 통제가 여기서 이루어집니다.

## 1. `blueprint.yaml` (샌드박스 총괄 도면)

이 파일은 샌드박스의 버저닝 요구사항, 베이스 컨테이너 이미지, 추론(Inference) 라우팅 프로필 등을 선언적으로 정의합니다.

### 1.1 공급망 공격(Supply Chain Attack) 방어 및 버전 통제
누군가 컨테이너 레지스트리를 해킹하여 악성 `latest` 이미지를 업로드하는 것을 막기 위해, 특정 시점의 이미지 해시(`sha256`)를 명시적으로 고정(Pinning)합니다.
또한, 호환성을 보장하기 위해 `min_openshell_version: "0.0.44"`, `max_openshell_version: "0.0.44"`, `min_openclaw_version: "2026.3.11"` 와 같이 엄격한 버전 제약을 적용합니다.

```yaml
version: "0.1.0"
min_openshell_version: "0.0.44"
max_openshell_version: "0.0.44"
min_openclaw_version: "2026.3.11"
digest: "sha256:b3d832b596ab6b7184a9dcb4ae93337ca32851a4f93b00765cc12de26baa3a9a"

components:
  sandbox:
    image: "ghcr.io/nvidia/openshell-community/sandboxes/openclaw@sha256:b3d832b..."
```

### 1.2 다중 추론 라우팅(Inference Routing) 프로필
로컬 하드웨어나 클라우드 환경에 맞춰 다양한 LLM 연결 설정을 미리 정의해 둡니다. (`default`, `ncp`, `nim-local`, `vllm`, `routed` 등)
라우팅 기능(`router` 컴포넌트)을 켜면 포트 터널링과 풀 설정(`pool_config_path`)을 통해 트래픽을 유연하게 제어할 수 있습니다.

---

## 2. `policies/openclaw-sandbox.yaml` (방화벽 기본 정책)

에이전트가 갇혀있는 철창의 물리적 규격을 정의하는 **실질적인 보안 통제 센터**입니다. "명시적으로 허용된 것 외에는 전부 차단한다(Default Deny)"는 원칙을 따릅니다.

### 2.1 파일 시스템 통제 (Filesystem Policy)
에이전트가 시스템 코어 파일을 변조하지 못하도록 강력하게 강제합니다.
* **`read_only`**: 시스템 영역(`/usr`, `/lib`, `/proc`, `/app`, `/etc`, `/var/log` 등)을 읽기 전용으로 잠급니다.
* **`read_write`**: 샌드박스의 임시 폴더(`/tmp`, `/dev/null`)와 에이전트 설정 및 상태 보관 장소(`/sandbox/.openclaw`, `/sandbox/.nemoclaw`)에만 제한적으로 쓰기를 허용합니다.

```yaml
filesystem_policy:
  include_workdir: true
  read_only:
    - /usr
    - /lib
    - /proc
    # ...
  read_write:
    - /tmp
    - /sandbox/.openclaw
    - /sandbox/.nemoclaw
```

### 2.2 네트워크 통제 (Network Policies)

**"어떤 프로세스(Binary)가 어떤 URL로, 어떤 HTTP Method(GET/POST)로 나가는지"**를 바이너리 레벨에서 매우 세밀하게 통제합니다. 

* **기본 허용 (Baseline)**: 모델 추론 서버(`integrate.api.nvidia.com`), 관리형 추론 게이트웨이(`inference.local`), OpenClaw 플러그인 레지스트리 및 인증 허브(`openclaw.ai`, `clawhub.ai`), NPM 레지스트리 등 동작에 꼭 필요한 엔드포인트만 열려 있습니다.
* **바이너리 차단**: NPM 레지스트리(`registry.npmjs.org`) 접근은 오직 `openclaw` 바이너리에만 허용되어, 에이전트가 몰래 NPM을 직접 사용해 악성 패키지를 땡겨오는 것을 방어합니다.

```yaml
  managed_inference:
    endpoints:
      - host: inference.local
        rules:
          - allow: { method: GET, path: "/**" }
          - allow: { method: POST, path: "/**" }
    binaries:
      - { path: /usr/local/bin/openclaw }
      - { path: /usr/local/bin/node }
      - { path: /usr/bin/curl }
      # ...
```

### 2.3 Opt-in 기반 서드파티 통신 (Presets)

**중요 변경사항:** 과거에는 Telegram, Discord, Slack 등의 메신저 API와 Github API 허용 정책이 베이스라인에 모두 포함되어 있어 에이전트가 임의로 외부 서버와 통신할 위험이 있었습니다.
현재는 보안 강화를 위해 이 룰들을 베이스라인에서 **완전히 제거**(`#1705`)하고, `presets/` 디렉토리 하위의 독립된 프리셋(Preset) 정책 파일들(`telegram.yaml`, `discord.yaml`, `github.yaml` 등)로 분리했습니다.
사용자가 온보딩 과정에서 특정 채널을 **명시적으로 승인(Opt-in)**해야만 해당 정책이 추가로 오버레이(Overlay) 됩니다.

현재 `presets/`에는 20개의 프리셋이 존재합니다: `brave`, `brew`, `discord`, `github`, `huggingface`, `jira`, `local-inference`, `nous-audio`, `nous-browser`, `nous-code`, `nous-image`, `nous-web`, `npm`, `openclaw-pricing`, `outlook`, `pypi`, `slack`, `telegram`, `wechat`, `whatsapp`.

### 2.4 정책 티어 (Policy Tiers)
`policies/tiers.yaml`는 보안 정책의 단계별 엄격도를 정의합니다. 기본(Base) 정책과 완화(Permissive) 정책 사이의 점진적 보안 수준을 사전 정의하여, 사용자가 요구 사항에 맞는 보안 수준을 선택할 수 있게 합니다.

### 2.5 완화(Permissive) 정책
`policies/openclaw-sandbox-permissive.yaml`는 개발/테스트 환경을 위한 완화된 정책을 제공합니다. 프로덕션 환경에서는 사용을 권장하지 않습니다.

---

## 3. `scripts/` (블루프린트 런타임 보안 가드)

`nemoclaw-blueprint/scripts/` 디렉토리에는 샌드박스 내부에서 동작하는 런타임 보안 가드 스크립트들이 위치합니다.

* **`sandbox-safety-net.js`**: 샌드박스 전반의 안전장치 런타임 가드.
* **`seccomp-guard.js`**: Seccomp 시스템 콜 필터링 가드.
* **`ciao-network-guard.js`**: 네트워크 격리 가드.
* **`http-proxy-fix.js`**: HTTP 프록시 호환성 패치.
* **`nemotron-inference-fix.js`**: Nemotron 모델 추론 호환성 패치.
* **`slack-channel-guard.js`**: Slack 채널 접근 가드.
* **`telegram-diagnostics.js`**: Telegram 연동 진단 도구.
* **`wechat-diagnostics.js`**: WeChat 연동 진단 도구.

---

## 4. `provider-profiles/` (프로바이더 프로필)

`nemoclaw-blueprint/provider-profiles/` 디렉토리에는 추론 프로바이더별 추가 프로필 설정이 위치합니다.
* `brave.yaml`: Brave Search 프로바이더 프로필.

## 요약
NemoClaw 블루프린트는 샌드박스 안에서 활동하는 AI 에이전트를 원천적으로 믿지 않고, **"허용된 프로그램이, 허용된 도메인으로만 통신하고, 허용된 폴더에만 글을 쓸 수 있게"** 통제하는 완벽한 제로 트러스트(Zero-Trust) 구조를 띄고 있으며, 최근 메신저 채널 등 서드파티 룰을 제거하며 보안을 한층 더 강화했습니다.
