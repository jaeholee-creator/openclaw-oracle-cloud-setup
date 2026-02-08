# OpenClaw 설치 및 설정 가이드

Oracle Cloud ARM 인스턴스에 OpenClaw를 설치하고 Telegram 봇으로 연결하는 전체 과정.

## 1. Oracle Cloud 인스턴스 준비

### 인스턴스 사양

| 항목 | 권장 사양 |
|------|----------|
| Shape | VM.Standard.A1.Flex (ARM) |
| OCPU | 4 |
| Memory | 12 GB |
| OS | Ubuntu 24.04 |
| Storage | 50 GB |
| 비용 | Always Free (무료) |

### 보안 규칙

Oracle Cloud 콘솔에서 Security List에 다음 인바운드 규칙 추가:

| 프로토콜 | 포트 | 소스 | 용도 |
|---------|------|------|------|
| TCP | 22 | 0.0.0.0/0 | SSH |
| TCP | 18789 | Tailscale CIDR | OpenClaw Gateway |

> Tailscale을 사용하면 18789 포트를 공개할 필요 없음

## 2. 서버 기본 설정

```bash
# SSH 접속
ssh ubuntu@YOUR_INSTANCE_IP

# 시스템 업데이트
sudo apt update && sudo apt upgrade -y

# 필수 패키지
sudo apt install -y build-essential curl git
```

## 3. Tailscale 설치

iPhone에서 직접 서버에 접근하기 위한 VPN 메시 네트워크.

```bash
# Tailscale 설치
curl -fsSL https://tailscale.com/install.sh | sh

# 로그인 (링크가 출력됨, 브라우저에서 인증)
sudo tailscale up

# IP 확인
tailscale ip -4
# 예: 100.87.94.10
```

iPhone에도 Tailscale 앱을 설치하고 같은 계정으로 로그인.

## 4. Node.js 설치

```bash
# nvm 설치
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc

# Node.js 22 LTS 설치
nvm install 22
nvm use 22
node --version  # v22.x.x 확인
```

## 5. OpenClaw 설치

```bash
# 전역 설치
npm install -g openclaw

# 버전 확인
openclaw --version

# 초기 설정 (대화형)
openclaw init
```

`openclaw init` 실행 시:
- Gateway mode 선택
- Telegram 채널 설정
- Bot token 입력

## 6. Telegram Bot 생성

1. Telegram에서 [@BotFather](https://t.me/BotFather) 에 `/newbot` 전송
2. 봇 이름 입력 (예: "My OpenClaw Bot")
3. 봇 사용자명 입력 (예: "my_openclaw_bot")
4. 발급된 Bot Token 저장

## 7. Groq API 키 발급

1. [console.groq.com](https://console.groq.com) 접속
2. 계정 생성/로그인
3. API Keys 메뉴에서 키 생성
4. `gsk_` 로 시작하는 키 저장

### Groq 무료 티어 제한

| 모델 | TPM | RPM | RPD |
|------|-----|-----|-----|
| Llama 4 Scout 17B | 30,000 | 30 | 14,400 |
| Llama 3.3 70B | 12,000 | 30 | 14,400 |

> Llama 4 Scout 17B를 권장 (TPM이 2.5배 높음)

## 8. 설정 파일 작성

### config.toml

```bash
nano ~/.openclaw/config.toml
```

[config.toml.example](../configs/config.toml.example)을 참고하여 작성.

주요 설정 포인트:
- `agents.defaults.model.primary`: 사용할 LLM 모델
- `channels.telegram.botToken`: Telegram Bot Token
- `tools.profile`: 도구 접근 수준 ("full" 권장)
- `sandbox.mode`: 샌드박스 ("off"로 전체 접근)

### systemd 서비스 등록

```bash
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/openclaw-gateway.service
```

[openclaw-gateway.service.example](../configs/openclaw-gateway.service.example)을 참고하여 작성.

**중요**: 환경변수로 API 키를 설정해야 합니다:
```
Environment=GROQ_API_KEY=gsk_your_actual_key_here
```

### 서비스 활성화

```bash
# systemd 사용자 서비스가 로그아웃 후에도 유지되도록
loginctl enable-linger ubuntu

# 데몬 리로드 및 서비스 시작
systemctl --user daemon-reload
systemctl --user enable openclaw-gateway
systemctl --user start openclaw-gateway

# 상태 확인
systemctl --user status openclaw-gateway
```

## 9. 검증

### 로그 확인

```bash
# 실시간 로그 모니터링
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# 정상 출력 예시:
# agent model: groq/meta-llama/llama-4-scout-17b-16e-instruct
# listening on ws://0.0.0.0:18789
# [default] starting provider (@your_bot)
```

### Telegram 테스트

1. Telegram에서 봇을 검색하여 대화 시작
2. "안녕하세요" 전송
3. 1~2초 내 응답 확인

### 에러가 있다면

```bash
# 에러만 필터링
grep ERROR /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

## 10. 유지보수

### 서비스 관리 명령

```bash
# 재시작
systemctl --user restart openclaw-gateway

# 중지
systemctl --user stop openclaw-gateway

# 로그 확인
journalctl --user -u openclaw-gateway -f
```

### OpenClaw 업데이트

```bash
npm update -g openclaw
systemctl --user restart openclaw-gateway
```
