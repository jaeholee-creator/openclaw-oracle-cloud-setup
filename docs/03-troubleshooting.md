# 문제 해결 가이드

OpenClaw 설정 중 발생할 수 있는 문제들과 해결 방법.

## 로그 확인 방법

```bash
# 오늘 로그 전체
cat /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# 에러만 필터링
grep ERROR /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# 실시간 모니터링
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# 특정 세션 ID로 필터링
grep "runId=YOUR_RUN_ID" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

## 자주 발생하는 문제

### 1. Config 유효성 에러

**증상**: 로그에 `Unrecognized key` 또는 `Invalid input` 에러

```
ERROR: commands.native: Invalid input
ERROR: Unrecognized key "permissions"
```

**해결**:
- `commands.native`와 `commands.nativeSkills`는 boolean 값(`true`/`false`) 사용
- 존재하지 않는 키(`permissions`, `autoApprove` 등) 제거
- `openclaw doctor --fix` 실행하여 설정 검증

**유효한 설정 값**:
```json
{
  "commands": {
    "native": true,
    "nativeSkills": true,
    "bash": true,
    "config": true,
    "restart": true
  }
}
```

### 2. Telegram 메시지 응답 없음

**증상**: 봇에 메시지를 보내도 응답이 오지 않음

**확인 사항**:

1. **서비스 실행 확인**:
```bash
systemctl --user status openclaw-gateway
```

2. **Telegram provider 시작 확인**:
```bash
grep "starting provider" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
# 출력: [default] starting provider (@your_bot)
```

3. **Bot Token 확인**: config.toml의 `channels.telegram.botToken`이 정확한지 확인

4. **allowFrom 확인**: `channels.telegram.allowFrom`이 `["*"]` 또는 본인 Telegram ID 포함

5. **dmPolicy 확인**: `"dmPolicy": "open"` 설정되어 있는지 확인

### 3. 2분 타임아웃 (Ollama 로컬 모델)

**증상**: 로그에 `typing TTL reached (2m); stopping typing indicator`

**원인**: CPU-only 환경에서 로컬 LLM 추론이 너무 느림

**해결**:
- 로컬 모델 대신 Cloud API (Groq) 사용
- GPU가 없는 환경에서는 8B 이상 모델 비추천

### 4. Ollama 프롬프트 Truncation

**증상**: `truncating input prompt: limit=4096 prompt=10873`

**원인**: Ollama 기본 context window가 4096 토큰

**해결** (로컬 모델 사용 시):
```bash
# 커스텀 Modelfile 생성
cat > Modelfile << 'EOF'
FROM llama3.1:8b
PARAMETER num_ctx 16384
EOF

# 커스텀 모델 생성
ollama create llama3.1:custom -f Modelfile
```

> 단, context window를 늘려도 CPU 환경에서는 타임아웃 문제 남음

### 5. Groq TPM 초과

**증상**: API 응답에 rate limit 에러

**원인**: Groq 무료 티어 TPM 제한 초과

**해결**:
- Llama 4 Scout 17B 사용 (30,000 TPM, Llama 3.3 70B보다 2.5배 높음)
- 대화가 길어지면 봇에 `/clear` 또는 새 대화 시작
- 짧은 시간에 많은 요청 자제

### 6. Gemini 404 에러

**증상**: `{"error":{"code":404,"status":"Not Found"}}`

**원인**: OpenClaw의 Gemini native provider 호환성 문제

**해결**:
- Gemini 대신 Groq 사용 권장
- `openai-completions` API 대신 native provider 사용 시에도 발생 가능
- 환경변수 `GEMINI_API_KEY`가 systemd 서비스에 설정되어 있는지 확인

### 7. Tool Calling이 JSON으로 출력

**증상**: 봇이 응답 대신 JSON을 텍스트로 출력

```
[{"name": "session_status", "arguments": {"sessionKey": "current"}}]
```

**원인**: 모델이 tool calling을 지원하지 않거나, 형식이 맞지 않음

**해결**:
- Tool calling을 정식 지원하는 모델 사용 (Groq Llama 4 Scout 권장)
- 작은 모델(3B) 대신 충분한 크기의 모델 사용

### 8. 환경변수가 적용되지 않음

**증상**: API 키를 설정했는데 인증 에러 발생

**원인**: `config.toml`의 `env` 섹션은 시스템 환경변수를 설정하지 않음

**해결**: systemd 서비스 파일에 직접 환경변수 추가

```ini
# ~/.config/systemd/user/openclaw-gateway.service 의 [Service] 섹션에 추가
Environment=GROQ_API_KEY=gsk_your_key_here
```

```bash
# 적용
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

### 9. preferences.json 오버라이드

**증상**: config.toml을 변경했는데 이전 모델이 계속 사용됨

**원인**: `~/.openclaw/agents/main/agent/preferences.json`이 config.toml을 오버라이드

**해결**:
```bash
# preferences.json 확인
cat ~/.openclaw/agents/main/agent/preferences.json

# 올바른 모델로 업데이트
cat > ~/.openclaw/agents/main/agent/preferences.json << 'EOF'
{
  "ai": {
    "provider": "groq",
    "model": "meta-llama/llama-4-scout-17b-16e-instruct"
  }
}
EOF

systemctl --user restart openclaw-gateway
```

## 디버깅 체크리스트

문제가 발생하면 순서대로 확인:

- [ ] 서비스 실행 중인가? (`systemctl --user status openclaw-gateway`)
- [ ] 로그에 에러가 있는가? (`grep ERROR /tmp/openclaw/...`)
- [ ] 모델이 올바르게 로드되었는가? (`grep "agent model" /tmp/openclaw/...`)
- [ ] Telegram provider가 시작되었는가? (`grep "starting provider" /tmp/openclaw/...`)
- [ ] 환경변수가 설정되었는가? (systemd 서비스 파일 확인)
- [ ] preferences.json이 config.toml과 일치하는가?
- [ ] Groq API 키가 유효한가? (curl로 직접 테스트)

```bash
# Groq API 직접 테스트
curl -s https://api.groq.com/openai/v1/models \
  -H "Authorization: Bearer $GROQ_API_KEY" | head -20
```
