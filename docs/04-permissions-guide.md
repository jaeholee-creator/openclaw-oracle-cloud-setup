# OpenClaw 권한 설정 가이드

Telegram에서 서버를 완전히 제어하기 위한 권한 설정.

## 권한 수준

### 기본 (Default)
- 대화만 가능
- 도구 사용 제한
- 파일 시스템 접근 불가

### 전체 (Full)
- 셸 명령 실행
- 파일 시스템 읽기/쓰기
- 브라우저 제어
- 서버 관리 (재시작, 설정 변경)
- 네이티브 명령어/스킬 사용

## 전체 권한 설정 방법

### config.toml에 추가할 섹션

#### 1. 도구(Tools) 권한

```json
{
  "tools": {
    "profile": "full",
    "allow": ["*"],
    "deny": [],
    "elevated": {
      "enabled": true,
      "allowFrom": {
        "telegram": ["*"]
      }
    }
  }
}
```

| 키 | 설명 |
|----|------|
| `profile` | 도구 프로필. `"full"`은 모든 도구 활성화 |
| `allow` | 허용할 도구 목록. `["*"]`은 전체 허용 |
| `deny` | 차단할 도구 목록. `[]`은 차단 없음 |
| `elevated.enabled` | 상위 권한 도구 (위험한 작업 포함) 활성화 |
| `elevated.allowFrom.telegram` | Telegram에서 상위 권한 허용 사용자. `["*"]`은 모든 사용자 |

> **보안 주의**: 프로덕션 환경에서는 `allowFrom`에 특정 Telegram ID만 지정하세요.

#### 2. 명령어(Commands) 권한

```json
{
  "commands": {
    "native": true,
    "nativeSkills": true,
    "bash": true,
    "config": true,
    "restart": true,
    "useAccessGroups": false
  }
}
```

| 키 | 설명 |
|----|------|
| `native` | 네이티브 명령어 사용 |
| `nativeSkills` | 네이티브 스킬 사용 |
| `bash` | 셸 명령 실행 (`bash`, `sh` 등) |
| `config` | 설정 변경 명령 |
| `restart` | 서비스 재시작 명령 |
| `useAccessGroups` | 접근 그룹 사용 여부 |

> **주의**: `native`와 `nativeSkills`는 반드시 boolean 값(`true`/`false`)을 사용. `"auto"`나 `"all"`은 Invalid input 에러 발생.

#### 3. 샌드박스(Sandbox) 비활성화

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "off"
      }
    }
  }
}
```

| 모드 | 설명 |
|------|------|
| `"off"` | 샌드박스 비활성화 (전체 접근) |
| `"permissive"` | 느슨한 샌드박스 |
| `"strict"` | 엄격한 샌드박스 |

## 특정 사용자만 허용

보안을 강화하려면 Telegram ID를 지정:

```json
{
  "tools": {
    "elevated": {
      "enabled": true,
      "allowFrom": {
        "telegram": ["YOUR_TELEGRAM_ID"]
      }
    }
  },
  "channels": {
    "telegram": {
      "allowFrom": ["YOUR_TELEGRAM_ID"]
    }
  }
}
```

Telegram ID 확인 방법:
1. [@userinfobot](https://t.me/userinfobot) 에 메시지 전송
2. 반환된 숫자가 Telegram ID

## 테스트 명령어

권한이 제대로 설정되었는지 확인하는 테스트:

### 셸 명령 실행
```
"서버 디스크 사용량 확인해줘"
"현재 실행 중인 프로세스 목록 보여줘"
"서버 uptime 알려줘"
```

### 파일 시스템 접근
```
"/home/ubuntu 디렉토리 파일 목록 보여줘"
"~/.openclaw/config.toml 내용 보여줘"
```

### 시스템 정보
```
"서버 메모리 사용량 알려줘"
"네트워크 인터페이스 정보 보여줘"
"CPU 정보 확인해줘"
```

### 서비스 관리
```
"OpenClaw 서비스 상태 확인해줘"
"/restart"
```

## 보안 권장사항

1. **Telegram ID 제한**: `allowFrom`에 본인 ID만 지정
2. **주기적 키 로테이션**: Groq API 키 주기적 변경
3. **로그 모니터링**: 비인가 접근 시도 모니터링
4. **방화벽**: Oracle Cloud Security List에서 필요한 포트만 개방
5. **Tailscale**: 공개 IP 대신 Tailscale VPN으로 관리 접근
