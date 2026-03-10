# AIRI Self-Hosting Guide (N100 NAS + Portainer)

## 아키텍처

| 구성요소 | 위치 | 역할 |
|---------|------|------|
| **AIRI Stage Web** | N100 NAS (Docker) | nginx 정적 파일 서빙만 — 매우 가벼움 |
| **NPM (Nginx Proxy Manager)** | N100 NAS (Docker) | 리버스 프록시 + Let's Encrypt SSL 자동 관리 |
| **메인 PC 브라우저** | 로컬 | Live2D 렌더링 + TTS (Kokoro 82M Q8 WASM, CPU) + STT |
| **Gemini API** | 외부 | 브라우저에서 직접 호출 — 나스에 LLM 부하 없음 |

> **핵심 포인트**: AIRI stage-web은 완전한 SPA입니다.  
> Live2D / TTS / STT 모두 브라우저 클라이언트에서 실행되므로, N100은 정적 파일 서빙만 담당합니다.

---

## Portainer에서 배포하기

### 1. Stack 생성

1. Portainer → **Stacks** → **Add Stack**
2. Build method: **Repository**
3. Repository URL: `https://github.com/kimbermania/airi-test`
4. Compose path: `docker-compose.yml`
5. ✅ **Automatic updates** (선택 사항 — 브랜치 업데이트 시 자동 재배포)

### 2. Environment variables 입력

Stack 생성 화면의 **Environment variables** 섹션에서 아래 변수들을 입력합니다.  
`.env` 파일은 사용하지 않습니다 — Portainer UI에서 직접 주입합니다.

| Variable | Required | Default | Description |
|----------|:--------:|---------|-------------|
| `LLM_API_KEY` | ✅ | — | Gemini API 키 |
| `LLM_PROVIDER` | | `google` | LLM 제공자 |
| `LLM_MODEL` | | `gemini-2.0-flash` | 사용할 Gemini 모델 |
| `AIRI_WEB_PORT` | | `3000` | airi-web 컨테이너 노출 포트 |
| `NPM_HTTP_PORT` | | `80` | NPM HTTP 포트 |
| `NPM_HTTPS_PORT` | | `443` | NPM HTTPS 포트 |
| `NPM_ADMIN_PORT` | | `81` | NPM 관리자 UI 포트 |

> **참고**: `VITE_` 접두사 변수는 빌드 타임에 번들에 하드코딩됩니다.  
> `docker-compose.yml` 의 `build.args` 가 이를 처리하므로 별도 설정 불필요합니다.

### 3. Deploy the stack

**Deploy the stack** 버튼을 클릭합니다.  
최초 빌드는 N100에서 10–20분 정도 소요될 수 있습니다.

---

## NPM (Nginx Proxy Manager) 설정

Stack이 배포된 후 NPM Admin UI를 설정합니다.

### 1. Admin UI 접속

`http://나스IP:81` 접속  
초기 계정: `admin@example.com` / `changeme` (최초 접속 시 변경 요구됨)

### 2. Proxy Host 추가

**Hosts → Proxy Hosts → Add Proxy Host**

| 항목 | 값 |
|------|-----|
| Domain Names | `airi.yourdomain.com` (원하는 도메인) |
| Forward Hostname / IP | `airi-web` |
| Forward Port | `80` |
| ✅ WebSocket Support | 활성화 |

### 3. SSL 설정

**SSL 탭**

- ✅ Request a new SSL Certificate (Let's Encrypt)
- ✅ Force SSL

### 4. Advanced 설정 (WASM/ONNX CORS 헤더)

**Advanced 탭** → Custom Nginx Configuration에 아래 내용 추가:

```nginx
# Kokoro TTS WASM + ONNX 모델 로딩에 필요한 CORS 헤더
add_header Cross-Origin-Embedder-Policy "require-corp" always;
add_header Cross-Origin-Opener-Policy "same-origin" always;
```

---

## 메인 PC에서 접속

1. `https://airi.yourdomain.com` 접속
2. **Settings** 에서 LLM Provider를 `Google Gemini` 로 설정
3. **TTS** 에서 `Kokoro 82M Q8 WASM` 선택 (브라우저 CPU에서 실행)

---

## 로컬에서 직접 실행 (Portainer 없이)

```bash
# 1. 리포지토리 클론
git clone https://github.com/kimbermania/airi-test.git
cd airi-test

# 2. .env 파일 생성 후 API 키 입력
cp .env.example .env
# LLM_API_KEY= 에 실제 Gemini API 키 입력

# 3. 빌드 및 실행
docker compose up -d

# 4. NPM Admin UI 접속 (http://localhost:81)
```

---

## 주의사항

- **API 키 보안**: `.env` 파일은 `.gitignore` 에 의해 Git에 커밋되지 않습니다. Portainer 사용 시에는 UI에서 직접 입력하세요.
- **빌드 타임 주입**: `VITE_LLM_API_KEY` 등 Vite 환경변수는 빌드 시점에 번들에 포함됩니다. 이미지를 퍼블릭 레지스트리에 올리지 마세요.
- **메모리 제한**: `docker-compose.yml` 에 N100 환경에 맞는 메모리 제한이 설정되어 있습니다 (airi-web: 512M, npm: 256M).
- **WASM SharedArrayBuffer**: Kokoro TTS WASM 은 `Cross-Origin-Embedder-Policy: require-corp` 헤더가 필요합니다. NPM Advanced 설정에서 반드시 추가하세요.
