# Chatwoot Custom Setup Guide

> **오픈소스 고객 지원 플랫폼 Chatwoot를 Docker로 5분 내 설치하는 가이드**

<div align="center">

![Chatwoot](https://img.shields.io/badge/Chatwoot-v4.5.3-blue)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)
![Rails](https://img.shields.io/badge/Rails-7.1.5-CC0000?logo=rubyonrails)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-316192?logo=postgresql)
![License](https://img.shields.io/badge/License-MIT-green)

</div>

## 📖 프로젝트 소개

이 저장소는 [Chatwoot](https://github.com/chatwoot/chatwoot) 오픈소스 고객 지원 플랫폼을 Docker Compose로 간편하게 설치하고 테스트할 수 있도록 구성한 가이드입니다.

### 주요 특징

- ✅ **간편한 설치**: Docker Compose 한 줄로 전체 스택 실행
- ✅ **올인원 구성**: PostgreSQL + Redis + Chatwoot 웹 서버 + Sidekiq
- ✅ **실시간 채팅**: WebSocket 기반 양방향 통신
- ✅ **테스트 페이지**: 즉시 테스트 가능한 위젯 샘플 페이지 포함
- ✅ **pgvector 지원**: AI 기능을 위한 벡터 검색 확장

---

## 🛠️ 기술 스택

| 기술 | 버전 | 용도 |
|------|------|------|
| **Ruby on Rails** | 7.1.5 | 백엔드 프레임워크 |
| **PostgreSQL** | 16 with pgvector | 데이터베이스 + AI 벡터 검색 |
| **Redis** | Alpine | 캐싱 & 세션 관리 |
| **Sidekiq** | 7.3.1 | 백그라운드 작업 처리 |
| **Docker** | Latest | 컨테이너화 |
| **WebSocket** | - | 실시간 통신 |

---

## 🚀 빠른 시작

### 사전 요구사항

- Docker Desktop (Windows/Mac) 또는 Docker Engine (Linux)
- Git
- 8GB+ RAM 권장

### 1. 저장소 클론

```bash
git clone https://github.com/당신의계정/chatwoot-portfolio.git
cd chatwoot-portfolio
```

### 2. 환경 변수 설정

```bash
# .env.example을 .env로 복사
cp .env.example .env

# .env 파일 편집 (비밀번호 변경 필수!)
# POSTGRES_PASSWORD=your_secure_password_here
# REDIS_PASSWORD=your_redis_password_here
```

### 3. Chatwoot 실행

```bash
docker-compose up -d
```

### 4. 관리자 계정 생성

```bash
docker exec -it chatwoot-web-1 bundle exec rails console

# Rails 콘솔에서 실행:
account = Account.create!(name: 'My Company')
user = User.create!(
  email: 'admin@example.com',
  password: 'SecurePassword123!',
  password_confirmation: 'SecurePassword123!',
  name: 'Admin User'
)
AccountUser.create!(account: account, user: user, role: :administrator)
```

### 5. 접속

- **관리자 대시보드**: http://localhost:3000
- **테스트 위젯**: `customer-test.html` 파일을 브라우저로 열기

---

## 📁 프로젝트 구조

```
chatwoot-portfolio/
├── docker-compose.yml      # Docker 구성 파일
├── .env.example            # 환경 변수 템플릿
├── customer-test.html      # 위젯 테스트 페이지
├── README.md               # 이 파일
└── .gitignore              # Git 제외 파일
```

---

## 🎯 주요 기능

### 1. 실시간 채팅 위젯

```html
<!-- 웹사이트에 삽입 -->
<script>
  (function(d,t) {
    var BASE_URL = "http://localhost:3000";
    var g = d.createElement(t), s = d.getElementsByTagName(t)[0];
    g.src = BASE_URL + "/packs/js/sdk.js";
    s.parentNode.insertBefore(g,s);
    g.onload = function(){
      window.chatwootSDK.run({
        websiteToken: 'YOUR_WEBSITE_TOKEN',
        baseUrl: BASE_URL
      });
    };
  })(document,"script");
</script>
```

### 2. 다중 채널 지원

- 웹 위젯
- 이메일
- Facebook Messenger
- WhatsApp
- Telegram
- Twitter DM
- LINE (커스텀 통합 가능)

### 3. 팀 협업 기능

- 대화 할당
- 팀 받은편지함
- 캐니드 응답 (빠른 답변)
- 내부 메모
- 라벨 및 태그

---

## 🔧 트러블슈팅

### 포트 충돌

```bash
# 3000번 포트가 사용 중인 경우
docker-compose down
# docker-compose.yml에서 "3000:3000"을 "8080:3000"으로 변경
docker-compose up -d
```

### 데이터베이스 초기화

```bash
docker-compose down
docker volume rm chatwoot-postgres-data
docker-compose up -d
```

### 로그 확인

```bash
# 전체 로그
docker-compose logs -f

# 특정 서비스 로그
docker-compose logs -f web
```

---

## 📊 성능 최적화

### 프로덕션 환경 권장 사항

1. **데이터베이스**
   - PostgreSQL 설정 튜닝
   - 정기 백업 설정
   - Connection pooling

2. **Redis**
   - 메모리 제한 설정
   - 영속성 옵션 선택

3. **웹 서버**
   - Puma worker/thread 최적화
   - CDN 사용 (정적 파일)
   - SSL/TLS 인증서 (Let's Encrypt)

---

## 🔐 보안 고려사항

### 필수 변경 사항

- [ ] `.env` 파일의 모든 비밀번호 변경
- [ ] `SECRET_KEY_BASE` 생성 (rails secret)
- [ ] 방화벽 규칙 설정
- [ ] HTTPS 설정
- [ ] Rate limiting 설정

### 데이터베이스 백업

```bash
# 백업 생성
docker exec chatwoot-postgres-1 pg_dump -U postgres chatwoot > backup.sql

# 백업 복원
docker exec -i chatwoot-postgres-1 psql -U postgres chatwoot < backup.sql
```

---

## 📚 학습 포인트

이 프로젝트를 통해 다음을 경험했습니다:

### 인프라 & DevOps
- Docker Compose를 활용한 마이크로서비스 아키텍처
- PostgreSQL pgvector 확장 활용
- Redis 캐싱 전략
- 환경 변수 관리

### 백엔드 개발
- Ruby on Rails 7 애플리케이션 구조
- Active Record 마이그레이션
- Sidekiq 백그라운드 작업
- WebSocket 실시간 통신

### 문제 해결
- 데이터베이스 스키마 불일치 해결
- Rate limiting (Rack::Attack) 설정
- CORS 설정
- 컬럼 누락 문제 디버깅

---

## 🌐 외부 접속 설정

### 도메인 연결

1. **DNS 설정**
   ```
   A 레코드: chat.yourdomain.com → 서버 IP
   ```

2. **Nginx 리버스 프록시**
   ```nginx
   server {
       listen 80;
       server_name chat.yourdomain.com;

       location / {
           proxy_pass http://localhost:3000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }
   }
   ```

3. **SSL 인증서**
   ```bash
   sudo certbot --nginx -d chat.yourdomain.com
   ```

---

## 📝 라이선스

이 가이드는 MIT 라이선스로 배포됩니다.

**Chatwoot 원본 프로젝트**:
- GitHub: [chatwoot/chatwoot](https://github.com/chatwoot/chatwoot)
- License: MIT
- Copyright: Chatwoot Inc.

---

## 🤝 기여

개선 사항이나 버그 리포트는 이슈로 등록해주세요!

---

## 📞 문의

프로젝트 관련 문의: [이메일 주소]

---

## 🔗 참고 자료

- [Chatwoot 공식 문서](https://www.chatwoot.com/docs)
- [Chatwoot API 문서](https://www.chatwoot.com/developers/api)
- [Docker Compose 문서](https://docs.docker.com/compose/)
- [PostgreSQL pgvector](https://github.com/pgvector/pgvector)

---

<div align="center">

**⭐ 도움이 되었다면 Star를 눌러주세요!**

Made with ❤️ for Customer Support Teams

</div>
