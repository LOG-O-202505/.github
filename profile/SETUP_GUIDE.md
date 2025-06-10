# LOG:O 프로젝트 통합 설정 가이드

LOG:O는 **AI 기반 멀티모달 여행 검색 엔진 및 자동 가계부 서비스**를 제공하는 통합 플랫폼입니다. 이 가이드는 Spring Boot 백엔드, ElasticSearch, Ollama AI 모델, Vue.js 프론트엔드를 포함한 전체 시스템의 설정 방법을 안내합니다.

## 🏗️ 시스템 아키텍처

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   LOG-O-vue     │    │ LOG-O-SpringAI  │    │  LOG-O-Ollama   │    │   LOG-O-ELK     │
│   (Frontend)    │    │   (Backend)     │    │   (AI Models)   │    │ (ElasticSearch) │
│                 │    │                 │    │                 │    │                 │
│ • Vue.js 3      │◄──►│ • Spring Boot   │◄──►│ • Llava (이미지) │◄──►│ • ElasticSearch │
│ • 카카오맵       │    │ • Spring AI     │    │ • Llama 3.2     │    │ • Kibana        │
│ • Google Vision │    │ • JWT 인증      │    │ • OCR 모델      │    │ • 벡터 검색     │
│ • D3.js 시각화  │    │ • OAuth2        │    │ • 벡터 생성     │    │ • 한국어 분석   │
│                 │    │ • MySQL/Redis   │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 📋 사전 준비사항

### 시스템 요구사항
- **운영체제**: Windows 10+, macOS 10.15+, Ubuntu 18.04+
- **메모리**: 최소 8GB RAM (16GB 권장)
- **저장공간**: 최소 30GB 여유 공간
- **네트워크**: 안정적인 인터넷 연결 (모델 다운로드 시)

### 필수 소프트웨어 설치
- **Java 21**: Spring Boot 3.4 실행용
- **MySQL 8.0**: 데이터베이스 서버
- **Redis 7.0**: 캐시 및 토큰 관리
- **Docker Desktop**: ElasticSearch 및 Kibana 실행용
- **Node.js 18+**: Vue.js 프론트엔드 실행용
- **Git**: 프로젝트 클론 및 Git LFS 사용

---

## 🚀 전체 설정 순서

### 1단계: Docker 및 ElasticSearch 설정 (LOG-O-ELK)
### 2단계: Ollama AI 모델 설치 및 설정 (LOG-O-Ollama)
### 3단계: Spring Boot 백엔드 설정 (LOG-O-SpringAI)
### 4단계: Vue.js 프론트엔드 설정 (LOG-O-vue)
### 5단계: API 키 발급 및 환경 변수 설정
### 6단계: 전체 시스템 테스트

---

## 📦 1단계: ElasticSearch 설정 (LOG-O-ELK)

### Docker Desktop 설치

#### Windows 환경
1. [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/) 다운로드 및 설치
2. PowerShell 또는 Command Prompt를 **관리자 권한**으로 실행

#### macOS 환경
1. [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/) 다운로드 및 설치
2. Terminal 실행

### ElasticSearch 및 Kibana 실행

```bash
# LOG-O-ELK 디렉토리로 이동
cd LOG-O-ELK

# Docker Compose 실행
docker-compose up -d

# 서비스 상태 확인
docker-compose ps
```

### 서비스 접속 확인
- **ElasticSearch**: http://localhost:9200
- **Kibana**: http://localhost:5601

### 인덱스 생성

1. Kibana DevTools 접속: http://localhost:5601 → 좌측 메뉴 "Dev Tools" 클릭

2. **travel_places 인덱스 생성**:
```json
PUT /travel_places 
{ 
  "settings": { 
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": { 
      "analyzer": { 
        "korean": {
          "type": "nori"
        } 
      } 
    } 
  }, 
  "mappings": { 
    "properties": { 
      "p_id": { "type": "long" },
      "p_name": {
        "type": "text", 
        "analyzer": "korean",
        "fields": { 
          "keyword": {
            "type": "keyword", 
            "ignore_above": 256 
          } 
        } 
      }, 
      "p_address": {
        "type": "text", 
        "analyzer": "korean",
        "fields": { 
          "keyword": { 
            "type": "keyword", 
            "ignore_above": 256 
          } 
        } 
      }, 
      "p_vector": {
        "type": "dense_vector", 
        "dims": 10,
        "index": true,
        "similarity": "cosine"
      }
    } 
  } 
}
```

3. **store_index 인덱스 생성**:
```json
PUT /store_index
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "my_nori_tokenizer": {
          "type": "nori_tokenizer",
          "decompound_mode": "mixed",
          "discard_punctuation": true
        }
      },
      "filter": {
        "my_pos_filter": {
          "type": "nori_part_of_speech",
          "stoptags": ["E", "J", "MAG", "MAJ", "MM", "SP", "XSN", "XSA", "XSV", "UNA"]
        }
      },
      "analyzer": {
        "my_nori_analyzer": {
          "type": "custom",
          "tokenizer": "my_nori_tokenizer",
          "filter": ["my_pos_filter"]
        }
      }
    }
  }
}
```

---

## 🤖 2단계: Ollama AI 모델 설정 (LOG-O-Ollama)

### Ollama 설치

#### macOS/Linux
```bash
curl -fsSL https://ollama.ai/install.sh | sh
```

#### Windows
1. [Ollama 공식 사이트](https://ollama.ai)에서 Windows 설치 파일 다운로드
2. 설치 파일 실행 후 설치 완료

### 설치 확인
```bash
ollama --version
```

### 필요한 모델 다운로드

#### 기본 모델 다운로드
```bash
# Llava 모델 (이미지 분석용)
ollama pull llava-phi3:latest
```

#### 한국어 모델 다운로드 (권장: Git LFS 사용)
```bash
# Git LFS 설치 및 설정
git lfs install

# 한국어 Llama 모델 다운로드 (약 2.02GB)
git clone https://huggingface.co/Bllossom/llama-3.2-Korean-Bllossom-3B-gguf-Q4_K_M

# GGUF 파일을 현재 디렉토리로 복사
cp llama-3.2-Korean-Bllossom-3B-gguf-Q4_K_M/llama-3.2-Korean-Bllossom-3B-gguf-Q4_K_M.gguf ./LOG-O-Ollama/

# 임시 디렉토리 삭제
rm -rf llama-3.2-Korean-Bllossom-3B-gguf-Q4_K_M
```

### 커스텀 모델 생성

LOG-O-Ollama 디렉토리로 이동한 후 다음 명령어들을 실행합니다:

```bash
cd LOG-O-Ollama

# 1. light_2 모델 생성 (이미지 분석)
ollama create light_2 -f light_3.Modelfile

# 2. OCR_basic 모델 생성 (영수증 분석)
ollama create OCR_basic -f ocr_3.Modelfile

# 3. ko_2 모델 생성 (벡터 생성)
ollama create ko_2 -f vector.Modelfile

# 4. ko_3 모델 생성 (번역 및 키워드 추출)
ollama create ko_3 -f vector3.Modelfile
```

### 모델 생성 확인
```bash
ollama list
```

**생성된 모델 목록:**
- `light_2`: 여행지 이미지 분석 및 JSON 형식 설명 생성
- `OCR_basic`: 영수증 OCR 데이터 분석
- `ko_2`: 여행지 설명을 10차원 벡터로 변환
- `ko_3`: 영문 설명 한국어 번역 및 키워드 추출

### 전체 모델 생성 자동화 스크립트

```bash
#!/bin/bash
# LOG-O-Ollama 디렉토리에서 실행

# Ollama 설치 확인
if ! command -v ollama &> /dev/null; then
    echo "❌ Ollama가 설치되어 있지 않습니다."
    echo "설치 명령어: curl -fsSL https://ollama.ai/install.sh | sh"
    exit 1
fi

echo "🚀 기본 모델 다운로드 중..."
ollama pull llava-phi3:latest

echo "🔧 커스텀 모델 생성 중..."
ollama create light_2 -f light_3.Modelfile
ollama create OCR_basic -f ocr_3.Modelfile
ollama create ko_2 -f vector.Modelfile
ollama create ko_3 -f vector3.Modelfile

echo "✅ 모든 모델 생성 완료!"
ollama list
```

---

## ⚙️ 3단계: Spring Boot 백엔드 설정 (LOG-O-SpringAI)

### SpringAI 프로젝트 Repo (상세한 설명은 해당 레포 README 확인)
[SpringAI Repository](https://github.com/LOG-O-202505/LOG-O-vue)

### Java 21 설치

#### Windows 환경
1. [Oracle JDK 21](https://www.oracle.com/java/technologies/downloads/#java21) 다운로드 및 설치
2. 또는 [OpenJDK 21](https://adoptium.net/) 설치 (권장)
3. 환경 변수 JAVA_HOME 설정

#### macOS 환경
```bash
# Homebrew로 OpenJDK 21 설치
brew install openjdk@21

# PATH 설정 (zshrc 또는 bash_profile)
echo 'export PATH="/opt/homebrew/opt/openjdk@21/bin:$PATH"' >> ~/.zshrc
echo 'export JAVA_HOME="/opt/homebrew/opt/openjdk@21"' >> ~/.zshrc
source ~/.zshrc
```

#### Ubuntu/Linux 환경
```bash
# Ubuntu 22.04+
sudo apt update
sudo apt install openjdk-21-jdk

# JAVA_HOME 설정
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> ~/.bashrc
source ~/.bashrc
```

### 설치 확인
```bash
java --version
javac --version
```

### MySQL 8.0 설치 및 설정

#### MySQL 설치
```bash
# macOS (Homebrew)
brew install mysql@8.0
brew services start mysql@8.0

# Ubuntu/Linux
sudo apt update
sudo apt install mysql-server-8.0
sudo systemctl start mysql
sudo systemctl enable mysql

# Windows: MySQL 공식 사이트에서 설치 파일 다운로드 혹은 Docker에 mysql container 실행
```

#### 데이터베이스 및 사용자 생성
```sql
-- MySQL 접속
mysql -u root -p

-- 데이터베이스 생성
CREATE DATABASE loggodb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 사용자 생성 및 권한 부여 (옵션)
CREATE USER 'loggo_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON loggodb.* TO 'loggo_user'@'localhost';
FLUSH PRIVILEGES;

-- 연결 테스트
USE loggodb;
SHOW TABLES;
```

### Redis 7.0 설치

#### Redis 설치
```bash
# macOS (Homebrew)
brew install redis
brew services start redis

# Ubuntu/Linux
sudo apt update
sudo apt install redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Windows: Redis 공식 사이트에서 설치 파일 다운로드
```

#### Redis 연결 테스트
```bash
redis-cli ping
# 응답: PONG
```

### MinIO 설치 및 설정

#### Docker Compose로 MinIO 실행
```bash
# LOG-O-SpringAI 디렉토리로 이동
cd LOG-O-SpringAI/src/main/resources

# Docker Compose 실행
docker-compose up -d minio

# MinIO 상태 확인
docker-compose ps
```

#### MinIO 접속 및 버킷 생성
1. **MinIO 콘솔 접속**: http://localhost:9001
2. **로그인**: 
   - 사용자명: `root`
   - 비밀번호: `root`
3. **버킷 생성**: `travel-storage` 이름으로 새 버킷 생성
4. **권한 설정**: Public 읽기 권한 설정

### 환경 변수 설정 예시

`LOG-O-SpringAI` 루트 디렉토리에 `.env` 파일 생성 혹은 `Intellij` 환경 변수 등록

```env
# 데이터베이스 설정
DB_URL=jdbc:mysql://localhost:3306/loggodb?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Seoul
DB_USER=root
DB_PASSWORD=your_mysql_password

# JWT 보안 설정
JWT_SECRET=your-very-long-and-secure-jwt-secret-key-at-least-512-bits-long

# AI API 키
OPENAI_API_KEY=your_openai_api_key
ANTHROPIC_API_KEY=your_anthropic_api_key
GPT_MODEL=gpt-4o
CLAUDE_MODEL=claude-3.7-sonnet-20240229

# OAuth2 설정
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
KAKAO_CLIENT_ID=your_kakao_client_id
KAKAO_CLIENT_SECRET=your_kakao_client_secret
NAVER_CLIENT_ID=your_naver_client_id
NAVER_CLIENT_SECRET=your_naver_client_secret

# MinIO 설정
MINIO_ACCESS_KEY=banchan
MINIO_SECRET_KEY=banchandev
MINIO_BUCKET_NAME=logo-bucket

# Notion API (선택사항)
NOTION_ID=your_notion_integration_id
NOTION_SECRET=your_notion_integration_secret

# 애플리케이션 도메인
APP_DOMAIN=localhost
```

### Spring Boot 애플리케이션 실행

#### Gradle로 실행
```bash
# LOG-O-SpringAI 디렉토리로 이동
cd LOG-O-SpringAI

# 의존성 설치 및 빌드
./gradlew clean build

# 애플리케이션 실행
./gradlew bootRun
```

#### JAR 파일로 실행
```bash
# JAR 파일 빌드
./gradlew bootJar

# JAR 실행
java -jar build/libs/logo-server-0.0.1-SNAPSHOT.jar
```

### 서비스 접속 확인
- **API 서버**: http://localhost:8080
- **Swagger UI**: http://localhost:8080/swagger-ui.html
- **헬스체크**: http://localhost:8080/health

### 실행 후 DB 데이터 삽입
- 실행 후 생성된 loggodb 데이터베이스의 areas 테이블에 `/resources/assets/areas-insert-sql-lowercase.sql` 데이터를 추가

### API 테스트
```bash
# 헬스체크 API 테스트
curl http://localhost:8080/health

# Swagger API 문서 확인
curl http://localhost:8080/v3/api-docs
```

---

## 🎨 4단계: Vue.js 프론트엔드 설정 (LOG-O-vue)

### Node.js 환경 확인
```bash
node --version  # 18.0.0 이상
npm --version   # 8.0.0 이상
```

### 프로젝트 의존성 설치
```bash
cd LOG-O-vue

# 의존성 설치
npm install
```

---

## 🔑 5단계: API 키 발급 및 환경 변수 설정

### 필요한 API 키 발급

#### 1. Google Maps API 키
1. [Google Cloud Console](https://console.cloud.google.com) 접속
2. 새 프로젝트 생성 또는 기존 프로젝트 선택
3. "API 및 서비스" → "라이브러리" → "Maps JavaScript API" 활성화
4. "사용자 인증 정보" → "API 키 만들기"

#### 2. 카카오맵 API 키
1. [카카오 Developers](https://developers.kakao.com) 접속
2. 애플리케이션 추가
3. "설정" → "일반" → "플랫폼" → "Web" 도메인 등록
4. JavaScript 키 및 REST API 키 복사

#### 3. Google Cloud Vision API 키
1. Google Cloud Console에서 "Vision API" 활성화
2. 서비스 계정 생성 또는 API 키 사용

### 환경 변수 설정

**LOG-O-vue/.env 파일 생성:**
```bash
# Ollama API 설정
VUE_APP_OLLAMA_API=http://localhost:11434/api

# ElasticSearch API 설정  
VUE_APP_ES_API=http://localhost:9200

# Ollama 모델명 설정
VUE_APP_MODEL_NAME=light_2

# Google Maps API 키
VUE_APP_GOOGLE_MAPS_API_KEY=your_google_maps_api_key_here

# 카카오맵 JavaScript API 키
VUE_APP_KAKAO_MAPS_API_KEY=your_kakao_maps_api_key_here

# Google Cloud Vision API 키
VUE_APP_GOOGLE_CLOUD_API_KEY=your_google_cloud_api_key_here

# 카카오맵 REST API 키
VUE_APP_KAKAO_MAPS_REST_KEY=your_kakao_maps_rest_key_here

# 백엔드 API 기본 URL
VUE_APP_BASE_URL=http://localhost:8080
```

---

## ✅ 6단계: 전체 시스템 테스트

### 시스템 가동 순서

#### 1. ElasticSearch 서비스 시작
```bash
cd LOG-O-ELK
docker-compose up -d

# 상태 확인
curl http://localhost:9200
```

#### 2. Ollama 서비스 시작 (백그라운드 실행)
```bash
# macOS/Linux
ollama serve &

# Windows (별도 터미널에서)
ollama serve
```

#### 3. Spring Boot 서비스 시작
```bash
cd LOG-O-SpringAI
./gradlew bootRun
```

#### 4. Vue.js 개발 서버 시작
```bash
cd LOG-O-vue
npm run serve
```

### 기능 테스트

#### ElasticSearch 테스트
```bash
# Kibana DevTools에서 실행
GET /travel_places/_search
{
  "query": {
    "match_all": {}
  }
}
```

#### Ollama 모델 테스트
```bash
# 이미지 분석 테스트
ollama run light_2 "이미지 설명을 해주세요"

# 벡터 생성 테스트
ollama run ko_2 "This is a beautiful mountain landscape"

# OCR 테스트
ollama run OCR_basic "맥도날드 강남점 2025-01-01 빅맥세트 9500원"
```

#### Vue.js 애플리케이션 테스트
- 브라우저에서 http://localhost:8080 접속
- 이미지 업로드 기능 테스트
- 지도 표시 확인
- 검색 기능 동작 확인

---

## 🔧 고급 설정 및 최적화

### Docker 리소스 최적화
```yaml
# docker-compose.yml 수정
services:
  elasticsearch:
    environment:
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"  # 메모리 할당량 조정
    mem_limit: 4g
    cpus: 2
```

### Ollama 성능 최적화
```bash
# 환경 변수 설정
export OLLAMA_NUM_PARALLEL=2        # 병렬 처리 수
export OLLAMA_MAX_LOADED_MODELS=4   # 메모리에 로드할 최대 모델 수
export OLLAMA_FLASH_ATTENTION=1     # Flash Attention 활성화
```

### Vue.js 프로덕션 빌드
```bash
cd LOG-O-vue

# 프로덕션 빌드
npm run build

# 정적 파일 서버로 실행 (선택사항)
npm install -g serve
serve -s dist
```

---

## 🚨 문제 해결 가이드

### 일반적인 문제 및 해결책

#### 1. Docker 관련 문제
```bash
# Docker Desktop이 실행되지 않는 경우
sudo systemctl start docker  # Linux
# 또는 Docker Desktop 재시작

# 포트 충돌 해결
docker-compose down
docker container prune
```

#### 2. Ollama 모델 로딩 실패
```bash
# 모델 재다운로드
ollama rm light_2
ollama create light_2 -f light_3.Modelfile

# 모델 파일 권한 확인
chmod 644 *.gguf
```

#### 3. Vue.js 빌드 오류
```bash
# node_modules 재설치
rm -rf node_modules package-lock.json
npm install

# 캐시 정리
npm run serve -- --reset-cache
```

#### 4. API 연결 오류
- 방화벽 설정 확인
- CORS 정책 확인
- API 키 유효성 검증

### 로그 확인 방법
```bash
# ElasticSearch 로그
docker-compose logs elasticsearch

# Ollama 로그
ollama logs

# Vue.js 로그
npm run serve (개발자 콘솔 확인)
```

---

## 📚 추가 리소스

### 공식 문서
- [ElasticSearch 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Ollama 공식 문서](https://ollama.ai/docs)
- [Vue.js 공식 문서](https://vuejs.org/guide/)

### API 문서
- [카카오맵 API](https://apis.map.kakao.com/web/)
- [Google Maps API](https://developers.google.com/maps/documentation)
- [Google Vision API](https://cloud.google.com/vision/docs)

---

## 👨‍💻 개발팀 정보

|                                                          **박병찬**                                                          |                                                          **한승수**                                                          |
| :--------------------------------------------------------------------------------------------------------------------------: | :--------------------------------------------------------------------------------------------------------------------------: |
| [<img src="https://avatars.githubusercontent.com/u/47100335?v=4" height=150 width=150> <br/> @qudcks8084](https://github.com/qudcks8084) | [<img src="https://github.com/user-attachments/assets/9948b5ea-272d-4ce4-9dd9-5a6d4cbd619f" height=150 width=150> <br/> @SEUNGSU-HAN](https://github.com/SEUNGSU-HAN) |
|                                                    **Frontend, ElasticStack, AI**                                                    |                                                    **Backend, Database, Infrastructure**                                                    |
|                                                     qudcks8084@gmail.com                                                     |                                                     h2sorginal@gmail.com                                                     |

---

## 📄 라이선스

이 프로젝트는 MIT 라이선스 하에 있습니다. 자세한 내용은 [LICENSE](LICENSE) 파일을 참조하세요.

---

**LOG:O** - AI와 함께하는 스마트한 여행 경험 🌍✨

*마지막 업데이트: 2025년 1월* 
