# Chatwoot 프로젝트 개발 일지

## 📋 프로젝트 개요

**프로젝트명**: Chatwoot 오픈소스 고객 지원 플랫폼 Docker 구축
**기간**: 2025년 11월
**목표**: Chatwoot v4.5.3을 Docker Compose로 구축하고 실시간 양방향 채팅 기능 구현
**저장소**: https://github.com/doublesilver/chatwoot-portfolio

---

## 🎯 프로젝트 목표

1. Docker Compose를 활용한 멀티 컨테이너 환경 구축
2. PostgreSQL + Redis + Rails 통합 환경 설정
3. 실시간 양방향 채팅 위젯 구현 및 테스트
4. 프로덕션 레벨의 데이터베이스 스키마 문제 해결
5. GitHub 포트폴리오 배포

---

## 🛠️ 기술 스택 및 선택 이유

### 1. **Docker & Docker Compose**
- **버전**: Docker Compose v3
- **선택 이유**:
  - 복잡한 멀티 서비스 환경을 코드로 관리 가능
  - 로컬 개발 환경과 프로덕션 환경의 일관성 보장
  - 서비스 간 네트워크 격리 및 의존성 관리 자동화
  - 팀원과의 환경 공유 용이
- **구성 서비스**:
  - PostgreSQL (데이터베이스)
  - Redis (캐싱 & 세션)
  - Chatwoot Web (Rails 애플리케이션)
  - Sidekiq (백그라운드 작업 처리)

### 2. **PostgreSQL 16 with pgvector**
- **선택 이유**:
  - Chatwoot의 공식 지원 데이터베이스
  - `pgvector` 확장: AI 기능을 위한 벡터 검색 지원
  - ACID 트랜잭션 보장으로 데이터 무결성 유지
  - JSON/JSONB 컬럼 지원으로 유연한 스키마 설계
- **핵심 기능**:
  - `CREATE EXTENSION vector` - 임베딩 벡터 저장 및 유사도 검색
  - 복잡한 관계형 데이터 모델 지원 (contacts, conversations, messages)

### 3. **Redis (Alpine)**
- **선택 이유**:
  - 초고속 인메모리 데이터 저장소
  - Sidekiq 작업 큐 관리
  - 세션 스토어 및 캐싱으로 응답 속도 향상
  - Rate limiting (Rack::Attack) 카운터 저장
- **사용 사례**:
  - WebSocket 연결 상태 관리
  - 캐시 무효화 전략 (FLUSHALL로 rate limit 해제)

### 4. **Ruby on Rails 7.1.5**
- **선택 이유**:
  - Chatwoot의 백엔드 프레임워크
  - Active Record ORM으로 데이터베이스 추상화
  - Convention over Configuration 철학으로 빠른 개발
  - 강력한 마이그레이션 시스템
- **주요 기능 활용**:
  - Rails Console을 통한 데이터 직접 조작
  - `rails runner`로 스크립트 실행
  - Active Record Migration으로 스키마 관리

### 5. **Sidekiq 7.3.1**
- **선택 이유**:
  - Redis 기반 고성능 백그라운드 작업 처리
  - 비동기 메시지 전송, 이메일 발송, 파일 처리
  - 멀티 스레드로 동시 작업 처리 효율성
  - 재시도 메커니즘으로 안정성 확보

### 6. **WebSocket (Action Cable)**
- **선택 이유**:
  - 실시간 양방향 통신 필수 (고객 ↔ 상담원)
  - HTTP 폴링 대비 낮은 레이턴시와 서버 부하
  - Rails의 Action Cable 프레임워크 통합
  - 메시지 즉시 전달 및 타이핑 인디케이터 구현

---

## 📝 작업 타임라인

### Phase 1: 환경 구축 (초기 설정)

#### 1.1 Docker Compose 구성
```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16  # pgvector 지원
  redis:
    image: redis:alpine
  web:
    image: chatwoot/chatwoot:latest
  sidekiq:
    image: chatwoot/chatwoot:latest
```

**문제**: 기본 `postgres:12` 이미지는 pgvector 미지원
**해결**: `pgvector/pgvector:pg16`로 변경하고 볼륨 재생성

#### 1.2 환경 변수 설정
- `.env` 파일 생성 (실제 비밀번호)
- `.env.example` 생성 (템플릿, GitHub용)
- DATABASE_URL, REDIS_URL 설정

#### 1.3 컨테이너 실행 및 검증
```bash
docker-compose -f docker-compose.simple.yaml up -d
docker logs chatwoot-web-1  # 로그 확인
```

---

### Phase 2: 데이터베이스 초기화

#### 2.1 관리자 계정 생성
**문제**: Web UI에서 계정 생성이 작동하지 않음
**해결**: Rails Console을 통한 직접 생성

```ruby
# C:\chatbot\chatwoot\create_admin.rb
account = Account.create!(name: 'Test Company')
user = User.create!(
  email: 'admin@test.com',
  password: 'Admin@123',
  password_confirmation: 'Admin@123',
  name: 'Admin'
)
AccountUser.create!(account: account, user: user, role: :administrator)
```

**실행**:
```bash
docker exec chatwoot-web-1 bundle exec rails runner /app/create_admin.rb
```

**학습 포인트**: Rails Console의 강력함과 Active Record 모델 직접 조작

#### 2.2 Inbox(수신함) 생성
**문제**: Web UI의 Inbox 생성 폼이 비활성화 상태
**해결**: 스크립트로 Channel + Inbox 생성

```ruby
# C:\chatbot\chatwoot\create_inbox_fixed.rb
channel = Channel::WebWidget.create!(
  account: account,
  website_url: 'http://localhost',
  widget_color: '#1f93ff',
  welcome_title: '안녕하세요!',
  welcome_tagline: '무엇을 도와드릴까요?',
  feature_flags: 3,
  reply_time: 'in_a_few_minutes',
  hmac_mandatory: false,
  pre_chat_form_enabled: false
)

inbox = Inbox.create!(
  account: account,
  name: '웹사이트 상담',
  channel: channel,
  greeting_enabled: true,
  greeting_message: '안녕하세요! 무엇을 도와드릴까요?',
  enable_auto_assignment: true
)

puts "✅ Website Token: #{channel.website_token}"
# 출력: vpkM8vfzgMMykZ1mbzrhLd2e
```

**학습 포인트**: Chatwoot의 Channel-Inbox 아키텍처 이해

---

### Phase 3: 실시간 채팅 위젯 구현

#### 3.1 테스트 페이지 작성
**파일**: `customer-test.html`

```html
<script>
(function(d,t) {
    var BASE_URL = "http://localhost:3000";
    var g = d.createElement(t), s = d.getElementsByTagName(t)[0];
    g.src = BASE_URL + "/packs/js/sdk.js";
    g.defer = true;
    g.async = true;
    s.parentNode.insertBefore(g,s);
    g.onload = function(){
        window.chatwootSDK.run({
            websiteToken: 'vpkM8vfzgMMykZ1mbzrhLd2e',
            baseUrl: BASE_URL
        });
    };
})(document,"script");
</script>
```

**기능**:
- 비동기 SDK 로딩으로 페이지 속도 저하 방지
- 우측 하단에 채팅 위젯 버튼 자동 표시
- 클릭 시 채팅창 열림

#### 3.2 양방향 통신 테스트
1. **고객 화면**: `customer-test.html` 브라우저로 열기
2. **상담원 화면**: `http://localhost:3000` 로그인
3. 메시지 주고받기 → ✅ **성공**

**결과**: WebSocket 기반 실시간 메시지 송수신 확인

---

### Phase 4: 데이터베이스 스키마 문제 해결

#### 4.1 문제: 누락된 컬럼들

Chatwoot의 코드와 실제 데이터베이스 스키마 불일치 발견.
**원인**: Migration 파일은 있지만 실제 DB에 적용되지 않은 상태

#### 4.2 해결 과정

##### 문제 1: `blocked` 컬럼 누락 (contacts 테이블)
**에러**:
```
NameError: undefined local variable or method 'blocked' for Contact
Location: app/models/contact.rb:160
```

**해결**:
```ruby
# add_blocked_final.rb
ActiveRecord::Migration.add_column :contacts, :blocked, :boolean, default: false
```

**실행 후 서버 재시작 필수** (Active Record 캐시 초기화)

##### 문제 2: `allowed_domains` 컬럼 누락 (channel_web_widgets 테이블)
**에러**:
```
NoMethodError: undefined method 'allowed_domains' for Channel::WebWidget
Location: app/controllers/widgets_controller.rb:80
```

**해결**:
```bash
docker exec chatwoot-web-1 sh -c "echo \"ActiveRecord::Migration.add_column :channel_web_widgets, :allowed_domains, :text, default: ''; puts '✅ allowed_domains added'\" | bundle exec rails runner -"
```

##### 문제 3: Captain 관련 테이블 누락
**누락 테이블**:
- `captain_assistants`
- `captain_inboxes`
- `captain_custom_tools`
- `captain_documents`
- `captain_scenarios`

**해결**: 각 테이블 생성 스크립트 작성 및 실행

```ruby
ActiveRecord::Migration.create_table :captain_assistants, force: true do |t|
  t.string :name, null: false
  t.bigint :account_id, null: false
  t.string :description
  t.timestamps
  t.jsonb :config, default: {}, null: false
  t.jsonb :response_guidelines, default: []
  t.jsonb :guardrails, default: []
end
ActiveRecord::Migration.add_index :captain_assistants, :account_id
```

**학습 포인트**: JSONB 컬럼을 활용한 유연한 설정 저장

##### 문제 4: Conversations 테이블 컬럼 누락
- `assignee_agent_bot_id` (봇 할당 기능)
- `cached_label_list` (라벨 캐싱)

**해결**: 각각 추가
```ruby
ActiveRecord::Migration.add_column :conversations, :assignee_agent_bot_id, :bigint
ActiveRecord::Migration.add_column :conversations, :cached_label_list, :text
```

#### 4.3 효율성 개선 시도

**초기 접근**: 에러 발생 → 로그 확인 → 컬럼 추가 → 재시작 (반복)
**문제점**: 너무 비효율적 ("하나씩 컬럼 추가하고 로그 보는게 너무 비효율적")

**개선 시도**: `rails db:migrate` 실행으로 한 번에 해결 시도
**결과**: Migration 에러 발생
```
StandardError: uninitialized constant ActsAsTaggableOn::Taggable::Cache
Migration: 20231211010807_add_cached_labels_list.rb
```

**최종 결론**: 수동 컬럼 추가로 진행 (migration 시스템 문제 우회)

---

### Phase 5: Rate Limiting 문제 해결

#### 5.1 문제 발생
**증상**: 위젯이 로드되지 않음
**로그**:
```
[Rack::Attack][Blocked] remote_ip: "172.18.0.1", path: "/widget"
HTTP 429 Too Many Requests
```

#### 5.2 원인 분석
- Rack::Attack 미들웨어가 짧은 시간 내 반복 요청을 차단
- 테스트 중 페이지 새로고침을 여러 번 하면서 limit 도달

#### 5.3 해결
```bash
docker exec chatwoot-redis-1 redis-cli FLUSHALL
```

**학습 포인트**:
- Rate limiting의 필요성과 작동 원리
- Redis를 활용한 카운터 관리
- 개발 환경에서는 느슨한 limit 필요

---

### Phase 6: GitHub 포트폴리오 준비

#### 6.1 보안 고려사항

**문제**: 실제 비밀번호가 포함된 `.env` 파일 노출 방지
**해결**:
1. 별도 디렉토리 생성: `C:\chatbot\chatwoot-portfolio`
2. `.gitignore` 작성:
```
.env          # 실제 비밀번호
*.log
postgres_data/
redis_data/
```

3. `.env.example` 생성 (플레이스홀더):
```
POSTGRES_PASSWORD=your_secure_password_here
REDIS_PASSWORD=your_redis_password_here
SECRET_KEY_BASE=replace_with_generated_secret_key
```

#### 6.2 문서화
**파일**: `README.md` (600+ 줄)

**포함 내용**:
- 프로젝트 소개 및 뱃지
- 기술 스택 표
- 빠른 시작 가이드
- 트러블슈팅 섹션
- 성능 최적화 팁
- 보안 고려사항
- 프로덕션 배포 가이드
- 학습 포인트

#### 6.3 Git 저장소 초기화
```bash
cd C:\chatbot\chatwoot-portfolio
git init
git add .
git commit -m "Initial commit: Chatwoot Docker setup guide"
git branch -M main
git remote add origin https://github.com/doublesilver/chatwoot-portfolio.git
git push -u origin main
```

**결과**: ✅ 공개 저장소 배포 완료

---

## 🔥 주요 문제 해결 사례

### Case 1: PostgreSQL Extension 누락
**문제**: pgvector extension 없음
**시도**: `CREATE EXTENSION vector` 실행 → 실패
**해결**: Docker 이미지 변경 `postgres:12` → `pgvector/pgvector:pg16`
**교훈**: 인프라 레벨 문제는 애플리케이션 레벨에서 해결 불가

### Case 2: Active Record 캐시 문제
**문제**: 컬럼 추가 후에도 `NoMethodError` 계속 발생
**원인**: Rails 서버가 구 스키마 정보를 캐싱
**해결**: 서버 재시작 (`docker-compose restart web`)
**교훈**: ORM 캐시 무효화 필요성

### Case 3: Migration 시스템 손상
**문제**: `rails db:migrate` 실패 (ActsAsTaggableOn 에러)
**분석**:
- 일부 migration만 실행된 상태
- 의존성 젬 버전 불일치 가능성
**해결**: Migration 우회, SQL 직접 실행
**교훈**: 레거시 프로젝트의 migration 히스토리 관리 중요성

### Case 4: WebSocket 연결 CORS
**문제**: 초기에 WebSocket 연결 실패 (추정)
**해결**: docker-compose에서 환경 변수 올바르게 설정
```yaml
environment:
  - FRONTEND_URL=http://localhost:3000
```
**교훈**: 환경 변수 하나가 전체 기능에 영향

---

## 📊 성과 및 결과

### 정량적 성과
- ✅ **4개 서비스** 통합 (PostgreSQL, Redis, Web, Sidekiq)
- ✅ **10+ 데이터베이스 컬럼/테이블** 수동 추가
- ✅ **실시간 양방향 채팅** 구현 및 검증
- ✅ **5개 파일** GitHub 포트폴리오 배포
- ✅ **600+ 줄** 프로페셔널 문서 작성

### 정성적 성과
- Docker Compose 멀티 컨테이너 아키텍처 경험
- Rails Console 및 Active Record 심화 활용
- 프로덕션 레벨 디버깅 경험 (로그 분석, 에러 추적)
- Git 워크플로우 및 보안 고려사항 적용
- 기술 문서 작성 능력 향상

---

## 💡 학습한 기술 및 개념

### 1. 인프라 & DevOps
- **Docker Compose**: 서비스 의존성 관리 (`depends_on`)
- **Volume 관리**: 데이터 영속성 보장
- **네트워크 격리**: 컨테이너 간 통신
- **환경 변수 주입**: 12 Factor App 원칙

### 2. 데이터베이스
- **PostgreSQL 고급 기능**:
  - pgvector extension (벡터 유사도 검색)
  - JSONB 컬럼 (유연한 스키마)
  - Index 전략 (복합 인덱스, unique 제약)
- **Redis 활용**:
  - 작업 큐 (Sidekiq)
  - Rate limiting 카운터
  - 캐시 무효화 (`FLUSHALL`)

### 3. Ruby on Rails
- **Active Record ORM**:
  - Migration DSL
  - 모델 관계 (has_many, belongs_to)
  - Callbacks 및 Validations
- **Rails Console**: 프로덕션 데이터 직접 조작
- **Rails Runner**: 스크립트 실행 환경

### 4. 웹 기술
- **WebSocket**: 실시간 양방향 통신
- **JavaScript SDK 통합**: 비동기 스크립트 로딩
- **CORS 설정**: 크로스 오리진 정책

### 5. 보안
- **환경 변수 관리**: 민감 정보 분리
- **Rate Limiting**: DDoS 방어
- **.gitignore**: 비밀번호 누출 방지
- **Docker Secrets**: 프로덕션 비밀 관리 (미래 개선점)

### 6. 문제 해결
- **로그 분석**: Docker logs를 활용한 에러 추적
- **Stack Trace 읽기**: Rails 에러 메시지 해석
- **Incremental 디버깅**: 한 번에 하나씩 문제 해결
- **우회 전략**: 막힌 길 우회 (migration 대신 SQL)

---

## 🔮 향후 개선 계획

### 단기 (현재 해결 필요)
1. **Admin UI 복구**
   - Settings 메뉴 404 에러 해결
   - Agents, Teams 페이지 활성화
2. **Migration 시스템 정상화**
   - `rails db:migrate:status` 점검
   - 누락된 migration 재실행
3. **에이전트 관리 UI**
   - 콘솔 없이 UI에서 에이전트 추가

### 중기 (기능 확장)
1. **자동 응답 설정**
   - 근무 시간 외 메시지
   - Canned Responses (빠른 답변)
2. **라벨 및 태그 시스템**
   - 대화 분류 및 필터링
3. **리포트 대시보드**
   - 응답 시간 분석
   - 고객 만족도 통계

### 장기 (프로덕션 배포)
1. **도메인 연결**
   - DNS 설정
   - Nginx 리버스 프록시
   - SSL/TLS 인증서 (Let's Encrypt)
2. **카카오톡 채널 통합**
   - KakaoTalk API 연동
   - 알림톡/친구톡 설정
3. **클라우드 배포**
   - AWS ECS 또는 DigitalOcean App Platform
   - RDS/ElastiCache 사용
   - S3 파일 업로드

---

## 🎓 핵심 교훈

### 1. "작동하는 것"이 최우선
- 완벽한 migration보다 작동하는 SQL이 낫다
- 이론적 베스트 프랙티스 < 실용적 해결책

### 2. 인프라가 애플리케이션을 지배한다
- Docker 이미지 선택의 중요성 (pgvector)
- 환경 변수 하나가 전체 기능 결정

### 3. 로그는 거짓말하지 않는다
- 에러 메시지를 끝까지 읽기
- Stack trace의 첫 번째 줄이 핵심

### 4. 캐시는 양날의 검
- 성능 향상 vs 디버깅 어려움
- 문제 발생 시 캐시 의심 (서버 재시작)

### 5. 보안은 시작부터
- .gitignore는 첫 커밋에
- 환경 변수 분리는 필수

---

## 📚 참고 자료

- [Chatwoot GitHub](https://github.com/chatwoot/chatwoot)
- [Docker Compose 문서](https://docs.docker.com/compose/)
- [PostgreSQL pgvector](https://github.com/pgvector/pgvector)
- [Rails Guides - Active Record](https://guides.rubyonrails.org/active_record_basics.html)
- [Sidekiq Wiki](https://github.com/sidekiq/sidekiq/wiki)

---

## 🏆 결론

이 프로젝트를 통해 **실제 프로덕션 환경에서 발생할 수 있는 문제들을 경험**하고 해결했습니다.

단순히 튜토리얼을 따라하는 것이 아닌, **오픈소스 프로젝트의 불완전함을 직접 마주하고 극복**하는 과정에서 진정한 개발 역량을 키울 수 있었습니다.

특히:
- ✅ **문제 정의**: 로그에서 핵심 에러 추출
- ✅ **원인 분석**: 코드와 DB 스키마 대조
- ✅ **해결 전략**: Migration 우회, 직접 SQL 실행
- ✅ **검증**: 서버 재시작 후 기능 확인
- ✅ **문서화**: 다른 개발자가 따라할 수 있도록 정리

이러한 경험은 **어떤 새로운 기술 스택을 만나도 빠르게 적응하고 문제를 해결할 수 있는 기반**이 되었습니다.

---

**작성일**: 2025년 11월 27일
**작성자**: doublesilver
**저장소**: https://github.com/doublesilver/chatwoot-portfolio
