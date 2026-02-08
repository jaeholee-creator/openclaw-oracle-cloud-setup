# LLM 모델 비교 (Oracle Cloud ARM 환경)

Oracle Cloud ARM 인스턴스(4 OCPU, 12GB RAM, CPU Only)에서 실제 테스트한 결과.

## 테스트 결과 요약

| 모델 | 유형 | 결과 | 응답시간 | Tool Calling | 비용 |
|------|------|------|---------|-------------|------|
| **Groq Llama 4 Scout 17B** | Cloud API | **성공** | 1.7~16.7초 | **정상** | 무료 |
| Groq Llama 3.3 70B | Cloud API | 실패 | - | - | 무료 |
| Gemini 2.0 Flash | Cloud API | 실패 (404) | - | - | 무료 |
| Ollama Llama 3.2 3B | Local | 실패 | - | 미지원 | 무료 |
| Ollama Llama 3.1 8B | Local | 실패 | 2분+ 타임아웃 | 불완전 | 무료 |
| Ollama Hermes 3 8B | Local | 실패 | 2분+ 타임아웃 | 부분적 | 무료 |

## 상세 분석

### Groq Llama 4 Scout 17B (최종 선택)

```
모델: groq/meta-llama/llama-4-scout-17b-16e-instruct
상태: 정상 작동
응답 시간: 1.7초 (간단), 16.7초 (복잡)
Tool Calling: 정상
TPM: 30,000 (무료 티어)
```

**장점**:
- 무료 API, 충분한 TPM
- 빠른 응답 시간
- Tool calling 완벽 지원
- 서버 리소스 소모 없음

**단점**:
- Groq 서버 의존 (가용성 리스크)
- 무료 티어 Rate limit 존재

### Groq Llama 3.3 70B

```
모델: groq/meta-llama/llama-3.3-70b-versatile
상태: 실패
원인: TPM 12,000 부족 (OpenClaw 시스템 프롬프트가 큼)
```

OpenClaw의 시스템 프롬프트 + 도구 정의가 상당한 토큰을 소모하여, 12,000 TPM으로는 부족.

### Gemini 2.0 Flash

```
모델: google/gemini-2.0-flash
상태: 실패 (404 에러)
시도 1: openai-completions API → 400 에러
시도 2: Native provider → 404 에러
```

**원인 분석**:
- `openai-completions` API로 설정 시 호환성 문제 (400)
- Native Google provider 사용 시에도 404 발생
- 환경변수 `GEMINI_API_KEY` 설정 후에도 동일
- OpenClaw의 Gemini native provider 호환성 문제로 추정

### Ollama Llama 3.2 3B

```
모델: ollama/llama3.2:3b
상태: 실패
원인: Tool calling 미지원, 모델 성능 부족
```

3B 모델은 OpenClaw의 복잡한 시스템 프롬프트를 이해하지 못하고, tool calling 형식의 JSON을 텍스트로 출력.

### Ollama Llama 3.1 8B

```
모델: ollama/llama3.1:8b (커스텀 num_ctx 16384)
상태: 실패
원인: CPU-only 추론으로 2분 타임아웃 초과
```

**시도한 해결**:
1. 기본 context window 4096 → 프롬프트 truncation 발생
2. 커스텀 Modelfile로 `num_ctx 16384` 설정
3. 프롬프트 처리는 되었으나, CPU 추론이 너무 느려 2분 타임아웃

```dockerfile
# 시도한 Modelfile
FROM llama3.1:8b
PARAMETER num_ctx 16384
```

### Ollama Hermes 3 8B

```
모델: ollama/hermes3:8b
상태: 실패
원인: CPU-only 추론으로 2분 타임아웃, 메모리 5.2GB 사용
```

Tool calling 전문 모델이라 부분적으로 동작했으나, CPU-only 환경에서 8B 모델 추론이 너무 느림.

## 결론

### Oracle Cloud ARM (CPU-only) 환경에서의 교훈

1. **로컬 LLM은 비현실적**: GPU 없이 8B 이상 모델은 tool calling과 함께 사용하기엔 너무 느림
2. **3B 모델은 성능 부족**: 작은 모델은 OpenClaw의 복잡한 시스템 프롬프트/도구 정의를 처리하지 못함
3. **Cloud API가 정답**: Groq 무료 티어로 충분한 성능과 속도 확보
4. **TPM이 중요**: OpenClaw는 시스템 프롬프트가 크므로 최소 20,000+ TPM 권장
5. **Llama 4 Scout**: 최신 모델로 30,000 TPM 무료, tool calling 우수

### 대안 모델 (미테스트)

| 모델 | 제공자 | TPM | 비고 |
|------|--------|-----|------|
| Cerebras Llama 3.3 70B | Cerebras | - | 초고속 추론 |
| DeepSeek V3 | DeepSeek | - | 중국산 고성능 |
| Mistral Large | Mistral | - | 유럽산 |
