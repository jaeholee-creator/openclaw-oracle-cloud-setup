# OpenClaw 고급 기능 설정 가이드

## 브라우저 자동화

### 구성
- **Chromium**: snap으로 설치 (/snap/bin/chromium)
- **systemd 서비스**: chromium-headless.service (자동 시작)
- **CDP 포트**: 18801
- **OpenClaw 프로필**: cdp-headless (기본값)

### 사용법 (Telegram에서)
```
"example.com 스크린샷 찍어줘"
"네이버 검색 결과 확인해줘"
"이 URL에서 데이터 수집해줘"
```

### CLI 사용법
```bash
openclaw browser navigate https://example.com
openclaw browser screenshot
openclaw browser snapshot
openclaw browser click <ref>
openclaw browser type <ref> "text"
```

## MCP (Model Context Protocol) 서버

### 설치된 MCP 서버
| 서버 | 패키지 | 용도 |
|------|--------|------|
| filesystem | @modelcontextprotocol/server-filesystem | 파일 시스템 접근 |
| memory | @modelcontextprotocol/server-memory | 지식 그래프 메모리 |
| sequential-thinking | @modelcontextprotocol/server-sequential-thinking | 단계별 추론 |

### 설정 파일 위치
- `/home/ubuntu/config/mcporter.json` (프로젝트)
- `~/.mcporter/mcporter.json` (시스템)
- `~/.openclaw/agents/main/agent/mcp.json` (에이전트)

### 테스트
```bash
mcporter call filesystem.list_directory path=/home/ubuntu
mcporter call memory.create_entities entities='[{"name":"test","entityType":"test","observations":["test"]}]'
```

## 크론잡

### 현재 설정 (5개)
| 이름 | 스케줄 | 설명 |
|------|--------|------|
| morning-briefing | 매일 07:00 KST | 아침 브리핑 (서버 상태 + 날씨) |
| server-healthcheck | 매 6시간 | 서버 헬스체크 |
| weekly-security-audit | 매주 월 09:00 KST | 보안 감사 |
| daily-summary | 매일 18:00 KST | 일일 서버 활동 요약 |
| weekly-report | 매주 금 17:00 KST | 주간 운영 리포트 |

### 관리 명령어
```bash
openclaw cron list           # 전체 목록
openclaw cron run <id>       # 즉시 실행
openclaw cron disable <id>   # 비활성화
openclaw cron enable <id>    # 활성화
```

## 설치된 스킬 (14/54)

### 준비 완료
- clawhub: 스킬 검색/설치
- github: GitHub CLI 통합
- healthcheck: 서버 보안 점검
- mcporter: MCP 서버 관리
- session-logs: 세션 로그 분석
- skill-creator: 스킬 제작
- tmux: 터미널 세션 관리
- video-frames: 비디오 프레임 추출
- weather: 날씨 정보
- skill-scanner: 악성 스킬 스캔
- brave-api-setup: 웹 검색 설정
- ipo-alert: 한국 공모주 알림
- security-monitor: 보안 모니터링
- system_monitor: 시스템 상태

### 추가 스킬 설치 (미래)
```bash
# Notion 통합 (NOTION_API_KEY 필요)
export NOTION_API_KEY=your_key
# notion 스킬이 자동 활성화됨

# 이메일 통합
sudo apt install himalaya
# himalaya 스킬이 자동 활성화됨
```
