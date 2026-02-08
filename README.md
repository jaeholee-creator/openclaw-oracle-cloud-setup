# OpenClaw on Oracle Cloud - Telegram Bot Setup

Oracle Cloud ARM 인스턴스에서 OpenClaw AI 어시스턴트를 Telegram 봇으로 운영하는 완전한 가이드.

## 아키텍처

```
iPhone (Telegram)
    ↓ Telegram Bot API
Oracle Cloud (ARM, 12GB RAM)
    ├── OpenClaw Gateway (systemd)
    ├── Groq API (Llama 4 Scout 17B)
    └── Tailscale VPN (관리용)
```

## 최종 구성

| 항목 | 값 |
|------|-----|
| **서버** | Oracle Cloud ARM (Ampere A1, 4 OCPU, 12GB RAM) |
| **OS** | Ubuntu 24.04 (aarch64) |
| **OpenClaw** | v2026.2.6-3 |
| **LLM** | Groq Llama 4 Scout 17B (무료 API) |
| **채널** | Telegram (@horam_claw_bot) |
| **네트워크** | Tailscale VPN |
| **권한** | Full (bash, filesystem, elevated tools) |

## 주요 특징

- **완전 무료**: Groq 무료 티어 (30,000 TPM) 사용
- **24/7 운영**: Oracle Cloud Always Free + systemd 자동 재시작
- **전체 권한**: 셸 명령 실행, 파일 시스템 접근, 서버 완전 제어
- **빠른 응답**: 1.7초 ~ 16.7초 응답 시간
- **Tool Calling**: 함수 호출 지원으로 실제 작업 수행 가능

## 디렉토리 구조

```
.
├── README.md                           # 이 파일
├── configs/
│   ├── config.toml.example             # OpenClaw 메인 설정 (마스킹)
│   └── openclaw-gateway.service.example # systemd 서비스 파일 (마스킹)
└── docs/
    ├── 01-setup-guide.md               # 처음부터 설치하는 가이드
    ├── 02-model-comparison.md          # 테스트한 모델 비교표
    ├── 03-troubleshooting.md           # 문제 해결 가이드
    └── 04-permissions-guide.md         # 권한 설정 가이드
```

## 빠른 시작

### 1. 사전 요구사항

- Oracle Cloud ARM 인스턴스 (Always Free)
- Telegram Bot Token ([@BotFather](https://t.me/BotFather))
- Groq API Key ([console.groq.com](https://console.groq.com))
- Tailscale 계정

### 2. OpenClaw 설치

```bash
# Node.js 설치 (nvm)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 22

# OpenClaw 설치
npm install -g openclaw

# 초기 설정
openclaw init
```

### 3. 설정 적용

```bash
# config.toml 복사 후 API 키/토큰 교체
cp configs/config.toml.example ~/.openclaw/config.toml
# 에디터로 YOUR_* 부분을 실제 값으로 교체

# systemd 서비스 등록
cp configs/openclaw-gateway.service.example ~/.config/systemd/user/openclaw-gateway.service
# 에디터로 YOUR_* 부분을 실제 값으로 교체

# 서비스 시작
systemctl --user daemon-reload
systemctl --user enable openclaw-gateway
systemctl --user start openclaw-gateway

# 상태 확인
systemctl --user status openclaw-gateway
```

### 4. 검증

```bash
# 로그 확인
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep -E '(agent model|listening|ERROR)'

# 정상이면 다음이 보여야 함:
# agent model: groq/meta-llama/llama-4-scout-17b-16e-instruct
# listening on ws://0.0.0.0:18789
```

Telegram에서 봇에 메시지를 보내 응답을 확인합니다.

## 자세한 가이드

- [설치 가이드](docs/01-setup-guide.md) - 처음부터 끝까지 단계별 설정
- [모델 비교](docs/02-model-comparison.md) - 테스트한 6개 모델의 결과 비교
- [문제 해결](docs/03-troubleshooting.md) - 흔한 에러와 해결 방법
- [권한 설정](docs/04-permissions-guide.md) - 전체 권한 부여 방법

## License

MIT
