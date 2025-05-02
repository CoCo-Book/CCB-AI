# 🧠 꼬꼬북 - AI 설계 개요

**꼬꼬북**은 아이들이 상상한 이야기를 실시간으로 AI와 함께 동화책으로 만들어주는 EduTech 기반 프로젝트입니다.  
이 디렉터리는 해당 서비스의 **AI 처리 파트**로, 음성 수집부터 스토리 구성, 이미지 생성, 음성 합성까지 전 과정을 포함합니다.

---

## 🧩 AI 시스템 구성

### 🤖 Chat-bot A (이야기 시작 챗봇 - "부기")
- **역할**: 아이의 음성을 실시간 수집하고, 줄거리를 요약, 그리고 아이의 이야기를 만들기 위해 대화를 통해 이야기 유도
- **모델**: `gpt-4o-mini`
- **대화 프롬프트 특징**:
  - 연령별(4-5세, 6-7세, 8-9세) 맞춤형 언어 사용
  - 아이의 관심사 기반 자연스러운 대화 유도
  - 이야기 단계별(캐릭터, 배경, 문제, 해결) 수집 전략
  - 격려와 자연스러운 후속 질문으로 상상력 확장
  - 교육적 가치를 담은 내용으로 유도
- **기술 요소**:
       - 실시간 음성 스트리밍 수신 (WebSocket 기반)
       - `RNNoise`로 노이즈 제거
       - `Whisper`를 이용한 음성 → 텍스트 변환
       - GPT-4o-mini로 대화 요약 및 줄거리 생성
       - `ElevenLabs` API를 통해 음성 클로닝 요청
       - **엔드포인트**: `/ws/audio`
       - **프로토콜**: WebSocket (ws/wss)
       - **인증**: Query 파라미터로 토큰 전달 (example: `?token=valid_token`)
       - **사용자 정보**: Query 파라미터로 전달 (example: `?child_name=민준&age=5&interests=공룡,우주,동물`)
       - **오디오 전송**: chunk 단위 바이너리(16kHz, mono, wav/opus 등)
       - **chunk 기준**: 2초 또는 128KB마다 서버가 처리
       - **응답**: 항상 JSON 패킷
              - `type`: "ai_response"
              - `text`: AI 텍스트 응답
              - `audio`: base64 인코딩된 mp3(음성)
              - `status`: "ok", "partial", "error"
              - `user_text`: 인식된 사용자 텍스트 (STT 결과)
              - `error_message`, `error_code`: (에러 발생 시)
       - **에러**: type이 "error"인 패킷으로 안내
       - **보안**: 운영 환경에서는 HTTPS/WSS, 인증 필수
       - **모니터링**: 서버 로그(logging) 기반 에러 추적
       - **대화 저장**: 연결 종료 시 자동 저장 (output/conversations 폴더)

### 🐢 Chat-bot B (스토리 완성 챗봇 - "꼬기")
- **역할**: 부기가 만든 줄거리와 음성 클론을 바탕으로 전체 동화 구성
- **모델**: `GPT-4o, DALL·E 3, ElevenLabs API`
- **기술 요소**:
  - `GPT-4o`로 상세 스토리 및 대사 생성
  - `DALL·E 3`로 삽화 생성 (프롬프트 엔지니어링 기반)
  - `ElevenLabs`로 감정/톤 반영된 음성 합성

---

## 🔍 AI 내부 구조 흐름

```plaintext
[아이 음성 입력]
       ↓
[부기: 음성 인식 + 단계별 이야기 수집 + 음성 클로닝 요청]
       ↓
[꼬기: 줄거리 기반 상세 스토리/삽화/음성 생성]
       ↓
[앱으로 동화책 전달 및 사용자 피드백 수집]
```

---

## 🔄 WebSocket API 사용법

### WebSocket 연결
```javascript
const ws = new WebSocket("wss://your-server.com/ws/audio?token=valid_token&child_name=민준&age=5&interests=공룡,우주,동물");

// 메시지 수신
ws.onmessage = (event) => {
  const response = JSON.parse(event.data);
  
  // 텍스트 처리
  console.log("AI 응답:", response.text);
  
  // 오디오 처리 (base64 디코딩 후 재생)
  if (response.audio) {
    const audio = new Audio("data:audio/mp3;base64," + response.audio);
    audio.play();
  }
};

// 오디오 데이터 전송 (예: MediaRecorder 사용)
navigator.mediaDevices.getUserMedia({ audio: true })
  .then(stream => {
    const mediaRecorder = new MediaRecorder(stream);
    mediaRecorder.ondataavailable = (event) => {
      if (event.data.size > 0 && ws.readyState === WebSocket.OPEN) {
        ws.send(event.data);
      }
    };
    // 100ms마다 오디오 데이터 전송
    mediaRecorder.start(100);
  });
```

### 응답 형식
```json
{
  "type": "ai_response",
  "text": "안녕 병찬아! 오늘은 어떤 이야기를 만들고 싶니?",
  "audio": "base64_encoded_audio_data...",
  "status": "ok",
  "user_text": "안녕 반가워"
}
``

### 에러 응답 형식
```json
{
  "type": "error",
  "error_message": "오디오 처리 중 오류가 발생했습니다",
  "error_code": "whisper_error",
  "status": "error"
}
```
