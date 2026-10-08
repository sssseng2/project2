# PeoplPickle 데이터베이스 설계서

## 📋 목차
1. [데이터베이스 개요](#1-데이터베이스-개요)
2. [엔티티 리스트](#2-엔티티-리스트)
3. [테이블 상세 설계](#3-테이블-상세-설계)
4. [테이블 관계도 (ERD)](#4-엔티티-관계도-erd)
5. [데이터 타입 및 제약조건](#5-데이터-타입-및-제약조건)
6. [인덱싱 전략](#6-인덱싱-전략)
7. [보안 정책](#7-보안-정책)

---

## 1. 데이터베이스 개요

### 1.1 기본 정보
- **DBMS**: MySQL 8.0+
- **Character Set**: UTF8MB4 (한글 지원)
- **Collation**: utf8mb4_unicode_ci
- **Engine**: InnoDB (트랜잭션, 외래키 지원)
- **데이터베이스명**: peoplepickle

### 1.2 설계 원칙
- 정규화: 3차 정규화 (3NF) 적용
- 확장성: 마이크로서비스 고려한 느슨한 결합
- 성능: 자주 조회되는 데이터는 역정규화 고려
- 보안: 민감한 데이터는 암호화, 비밀번호는 bcrypt 해싱

---

## 2. 엔티티 리스트

### Core Entities (핵심)
1. **Users** - 사용자 기본 정보
2. **UserProfiles** - 사용자 포트폴리오 (기술스택, 관심분야 등)
3. **Projects** - 프로젝트 정보
4. **Applications** - 팀원 지원 정보
5. **TeamMembers** - 승인된 팀원 정보

### Collaboration Entities (협업)
6. **ChatMessages** - 실시간 채팅 메시지
7. **FileShares** - 파일 공유 정보
8. **Schedules** - 일정관리 (WBS/R&R/TODO)
9. **ProjectNotices** - 중요 정보 박스

### Community Entities (커뮤니티)
10. **Posts** - 게시판 게시물
11. **Comments** - 게시판 댓글

### System Entities (시스템)
12. **Notifications** - 알림
13. **AdminVerifications** - 관리자 회원가입 인증
14. **Tags** - 태그 (프로젝트, 게시물 등)
15. **Logs** - 시스템 로그 (감시용)

---

## 3. 테이블 상세 설계

### 3.1 Users (사용자)

```sql
CREATE TABLE Users (
    user_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '사용자 ID',
    username VARCHAR(50) UNIQUE NOT NULL COMMENT '아이디',
    email VARCHAR(100) UNIQUE NOT NULL COMMENT '이메일',
    password_hash VARCHAR(255) NOT NULL COMMENT 'bcrypt 해싱된 비밀번호',
    
    -- 기본 정보
    name VARCHAR(100) NOT NULL COMMENT '이름',
    phone VARCHAR(20) COMMENT '휴대폰번호',
    university VARCHAR(100) NOT NULL COMMENT '대학교명',
    major VARCHAR(100) COMMENT '전공',
    student_id VARCHAR(50) COMMENT '학번',
    
    -- 프로필 이미지
    profile_image_url VARCHAR(500) COMMENT '프로필 사진 URL',
    
    -- 상태 관리
    account_status ENUM('PENDING', 'ACTIVE', 'INACTIVE', 'DELETED') DEFAULT 'PENDING' COMMENT 'PENDING:승인대기, ACTIVE:활성, INACTIVE:휴면, DELETED:삭제',
    is_admin BOOLEAN DEFAULT FALSE COMMENT '관리자 여부',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '가입 날짜',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '마지막 수정 날짜',
    deleted_at TIMESTAMP NULL COMMENT '삭제 날짜',
    last_login_at TIMESTAMP NULL COMMENT '마지막 로그인 시간',
    
    -- 인덱스
    INDEX idx_email (email),
    INDEX idx_username (username),
    INDEX idx_university (university),
    INDEX idx_account_status (account_status),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**칼럼 설명:**
- `user_id`: PK, 자동 증가
- `password_hash`: 절대 평문 저장 금지, bcrypt 해싱 필수
- `account_status`: 회원가입 승인 상태 관리
- `is_admin`: 관리자 권한 플래그
- 타임스탬프: 생성/수정/삭제/로그인 시간 추적

---

### 3.2 UserProfiles (사용자 포트폴리오)

```sql
CREATE TABLE UserProfiles (
    profile_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '포트폴리오 ID',
    user_id BIGINT NOT NULL UNIQUE COMMENT '사용자 ID',
    
    -- AI 매칭용 정보
    tech_stack JSON COMMENT '기술스택 (JSON Array: ["Spring", "React", "MySQL"])',
    interest_fields JSON COMMENT '관심분야 (JSON Array: ["AI", "웹", "모바일"])',
    desired_roles JSON COMMENT '희망 역할 (JSON Array: ["Backend", "DevOps"])',
    personality_type VARCHAR(50) COMMENT '성격 유형 (MBTI 등)',
    
    -- 활동 정보
    available_times JSON COMMENT '활동가능시간 (JSON: {"mon_fri": "20:00-24:00", "sat_sun": "10:00-22:00"})',
    preferred_project_types JSON COMMENT '선호하는 프로젝트 유형',
    
    -- 포트폴리오 텍스트
    bio TEXT COMMENT '자기소개',
    portfolio_link VARCHAR(500) COMMENT '포트폴리오 링크 (깃허브, 개인 블로그 등)',
    
    -- 통계
    completed_projects INT DEFAULT 0 COMMENT '완료한 프로젝트 수',
    average_rating DECIMAL(3, 2) DEFAULT 0 COMMENT '평균 평점',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- 외래키
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    
    -- 인덱스
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**칼럼 설명:**
- JSON 컬럼: 배열/객체 형태의 데이터 저장 (기술스택, 활동가능시간 등)
- `tech_stack`, `interest_fields`: AI 매칭 점수 계산에 사용
- `available_times`: 시간대별 활동 가능 여부

---

### 3.3 Projects (프로젝트)

```sql
CREATE TABLE Projects (
    project_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '프로젝트 ID',
    creator_id BIGINT NOT NULL COMMENT '프로젝트 생성자 ID',
    
    -- 기본 정보
    title VARCHAR(200) NOT NULL COMMENT '프로젝트명',
    description TEXT COMMENT '프로젝트 설명',
    objective TEXT COMMENT '프로젝트 목표',
    
    -- 모집 정보
    required_roles JSON COMMENT '필요 역할 (JSON Array)',
    required_tech_stack JSON COMMENT '필요 기술스택 (JSON Array)',
    required_member_count INT NOT NULL COMMENT '모집 인원',
    current_member_count INT DEFAULT 1 COMMENT '현재 팀원 수',
    
    -- 조건
    preferred_traits JSON COMMENT '선호하는 성격/특성',
    preferred_keywords JSON COMMENT '선호 키워드',
    
    -- 진행 정보
    progress_method VARCHAR(100) COMMENT '진행 방식 (온라인/오프라인/하이브리드)',
    start_date DATE COMMENT '프로젝트 시작일',
    end_date DATE COMMENT '프로젝트 종료일',
    recruitment_deadline DATE COMMENT '팀원 모집 마감일',
    
    -- 상태
    status ENUM('RECRUITING', 'RECRUITING_COMPLETED', 'IN_PROGRESS', 'COMPLETED') DEFAULT 'RECRUITING' COMMENT '프로젝트 상태',
    
    -- 통계
    view_count INT DEFAULT 0 COMMENT '조회수',
    application_count INT DEFAULT 0 COMMENT '지원자 수',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,
    
    -- 외래키
    FOREIGN KEY (creator_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    
    -- 인덱스
    INDEX idx_creator_id (creator_id),
    INDEX idx_status (status),
    INDEX idx_recruitment_deadline (recruitment_deadline),
    INDEX idx_created_at (created_at),
    FULLTEXT INDEX ftx_title_description (title, description)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.4 Applications (팀원 지원)

```sql
CREATE TABLE Applications (
    application_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '지원 ID',
    project_id BIGINT NOT NULL COMMENT '프로젝트 ID',
    user_id BIGINT NOT NULL COMMENT '지원자 ID',
    
    -- 지원 정보
    application_message TEXT COMMENT '지원 동기/의견',
    
    -- 상태
    status ENUM('PENDING', 'APPROVED', 'REJECTED') DEFAULT 'PENDING' COMMENT '지원 상태',
    reviewed_at TIMESTAMP NULL COMMENT '검토 완료 시간',
    review_message VARCHAR(500) COMMENT '검토 사유 (거부 시)',
    
    -- AI 매칭 점수
    ai_match_score DECIMAL(5, 2) COMMENT 'AI 매칭 점수 (0-100)',
    ai_match_breakdown JSON COMMENT 'AI 점수 상세 ({tech: 40, role: 30, time: 15, ...})',
    is_ai_recommended BOOLEAN DEFAULT FALSE COMMENT 'AI 추천 여부',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- 외래키
    FOREIGN KEY (project_id) REFERENCES Projects(project_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    
    -- 제약조건
    UNIQUE KEY uk_project_user (project_id, user_id),
    
    -- 인덱스
    INDEX idx_project_id (project_id),
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**특징:**
- `UNIQUE KEY (project_id, user_id)`: 중복 지원 방지
- `ai_match_score`: 0-100 점수
- `ai_match_breakdown`: JSON으로 점수 상세 정보 저장

---

### 3.5 TeamMembers (승인된 팀원)

```sql
CREATE TABLE TeamMembers (
    team_member_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '팀원 ID',
    project_id BIGINT NOT NULL COMMENT '프로젝트 ID',
    user_id BIGINT NOT NULL COMMENT '사용자 ID',
    
    -- 역할 정보
    assigned_role VARCHAR(100) NOT NULL COMMENT '프로젝트 내 담당 역할',
    responsibility TEXT COMMENT '담당 책임 범위',
    
    -- 팀 내 역할
    is_team_leader BOOLEAN DEFAULT FALSE COMMENT '팀장 여부',
    
    -- 상태
    member_status ENUM('ACTIVE', 'INACTIVE', 'WITHDRAWN') DEFAULT 'ACTIVE' COMMENT '팀원 상태',
    
    -- 타임스탬프
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '팀 참여 시간',
    withdrawal_at TIMESTAMP NULL COMMENT '탈퇴 시간',
    
    -- 외래키
    FOREIGN KEY (project_id) REFERENCES Projects(project_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    
    -- 제약조건
    UNIQUE KEY uk_project_user (project_id, user_id),
    
    -- 인덱스
    INDEX idx_project_id (project_id),
    INDEX idx_user_id (user_id),
    INDEX idx_is_team_leader (is_team_leader)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.6 ChatMessages (실시간 채팅)

```sql
CREATE TABLE ChatMessages (
    message_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '메시지 ID',
    project_id BIGINT NOT NULL COMMENT '프로젝트 ID',
    user_id BIGINT NOT NULL COMMENT '발신자 ID',
    
    -- 메시지 내용
    content TEXT NOT NULL COMMENT '메시지 내용',
    message_type ENUM('TEXT', 'FILE', 'NOTICE', 'SYSTEM') DEFAULT 'TEXT' COMMENT '메시지 타입',
    
    -- 파일 링크 (선택사항)
    file_share_id BIGINT COMMENT '첨부 파일 ID',
    
    -- 읽음 상태 (JSON으로 저장: {user_id: read_at_timestamp})
    read_status JSON COMMENT '팀원별 읽음 상태',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NULL COMMENT '수정 시간',
    deleted_at TIMESTAMP NULL COMMENT '삭제 시간',
    
    -- 외래키
    FOREIGN KEY (project_id) REFERENCES Projects(project_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (file_share_id) REFERENCES FileShares(file_share_id) ON DELETE SET NULL,
    
    -- 인덱스
    INDEX idx_project_id (project_id),
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.7 FileShares (파일 공유)

```sql
CREATE TABLE FileShares (
    file_share_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '파일 공유 ID',
    project_id BIGINT NOT NULL COMMENT '프로젝트 ID',
    uploader_id BIGINT NOT NULL COMMENT '업로더 ID',
    
    -- 파일 정보
    file_name VARCHAR(255) NOT NULL COMMENT '파일명',
    file_path VARCHAR(500) NOT NULL COMMENT '파일 저장 경로 (S3 또는 로컬)',
    file_size BIGINT NOT NULL COMMENT '파일 크기 (바이트)',
    file_type VARCHAR(50) COMMENT '파일 타입 (jpg, pdf, zip 등)',
    
    -- 분류
    category VARCHAR(50) COMMENT '파일 카테고리 (디자인, 기획, 코드 등)',
    tags JSON COMMENT '태그 (JSON Array)',
    
    -- 설명
    description TEXT COMMENT '파일 설명',
    
    -- 버전 관리
    version INT DEFAULT 1 COMMENT '버전',
    is_latest BOOLEAN DEFAULT TRUE COMMENT '최신 버전 여부',
    previous_file_id BIGINT COMMENT '이전 버전 파일 ID',
    
    -- 통계
    download_count INT DEFAULT 0 COMMENT '다운로드 수',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,
    
    -- 외래키
    FOREIGN KEY (project_id) REFERENCES Projects(project_id) ON DELETE CASCADE,
    FOREIGN KEY (uploader_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (previous_file_id) REFERENCES FileShares(file_share_id) ON DELETE SET NULL,
    
    -- 인덱스
    INDEX idx_project_id (project_id),
    INDEX idx_uploader_id (uploader_id),
    INDEX idx_category (category),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.8 Schedules (일정 관리 - WBS/R&R/TODO)

```sql
CREATE TABLE Schedules (
    schedule_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '일정 ID',
    project_id BIGINT NOT NULL COMMENT '프로젝트 ID',
    creator_id BIGINT NOT NULL COMMENT '작성자 ID',
    
    -- 기본 정보
    title VARCHAR(200) NOT NULL COMMENT '일정/작업명',
    description TEXT COMMENT '상세 설명',
    schedule_type ENUM('WBS', 'TODO', 'MILESTONE', 'REVIEW') DEFAULT 'TODO' COMMENT '일정 타입',
    
    -- 작업 정보
    assigned_to BIGINT COMMENT '담당자 ID',
    start_date DATE NOT NULL COMMENT '시작일',
    due_date DATE NOT NULL COMMENT '마감일',
    
    -- 상태
    status ENUM('NOT_STARTED', 'IN_PROGRESS', 'COMPLETED', 'DELAYED', 'CANCELLED') DEFAULT 'NOT_STARTED' COMMENT '진행 상태',
    progress_percentage INT DEFAULT 0 COMMENT '진행률 (0-100)',
    priority ENUM('LOW', 'MEDIUM', 'HIGH', 'URGENT') DEFAULT 'MEDIUM' COMMENT '우선순위',
    
    -- 계층 구조 (WBS)
    parent_schedule_id BIGINT COMMENT '상위 일정 ID',
    depth INT DEFAULT 0 COMMENT '계층 깊이',
    
    -- 역할 정보 (R&R)
    responsibility JSON COMMENT '담당 책임 범위 (JSON)',
    
    -- 알림
    is_notified BOOLEAN DEFAULT FALSE COMMENT '알림 발송 여부',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    completed_at TIMESTAMP NULL COMMENT '완료 시간',
    
    -- 외래키
    FOREIGN KEY (project_id) REFERENCES Projects(project_id) ON DELETE CASCADE,
    FOREIGN KEY (creator_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (assigned_to) REFERENCES Users(user_id) ON DELETE SET NULL,
    FOREIGN KEY (parent_schedule_id) REFERENCES Schedules(schedule_id) ON DELETE CASCADE,
    
    -- 인덱스
    INDEX idx_project_id (project_id),
    INDEX idx_assigned_to (assigned_to),
    INDEX idx_due_date (due_date),
    INDEX idx_status (status),
    INDEX idx_priority (priority)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.9 ProjectNotices (중요 정보 박스)

```sql
CREATE TABLE ProjectNotices (
    notice_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '공지사항 ID',
    project_id BIGINT NOT NULL COMMENT '프로젝트 ID',
    creator_id BIGINT NOT NULL COMMENT '작성자 ID (팀장)',
    
    -- 공지사항 정보
    title VARCHAR(200) NOT NULL COMMENT '공지사항 제목',
    content TEXT NOT NULL COMMENT '공지사항 내용',
    
    -- 표시 정보
    notice_type ENUM('URGENT', 'IMPORTANT', 'GENERAL') DEFAULT 'GENERAL' COMMENT '공지 타입',
    color_code VARCHAR(7) COMMENT '배경 색상 (예: #DC2626)',
    icon VARCHAR(50) COMMENT '아이콘 (예: warning, info, pin)',
    
    -- 기한
    expiration_date TIMESTAMP NULL COMMENT '자동 삭제 기한',
    
    -- 읽음 상태
    read_status JSON COMMENT '팀원별 읽음 상태 ({user_id: read_at_timestamp})',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,
    
    -- 외래키
    FOREIGN KEY (project_id) REFERENCES Projects(project_id) ON DELETE CASCADE,
    FOREIGN KEY (creator_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    
    -- 인덱스
    INDEX idx_project_id (project_id),
    INDEX idx_created_at (created_at),
    INDEX idx_expiration_date (expiration_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.10 Posts (게시판 게시물)

```sql
CREATE TABLE Posts (
    post_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '게시물 ID',
    author_id BIGINT NOT NULL COMMENT '작성자 ID',
    
    -- 게시물 정보
    title VARCHAR(200) NOT NULL COMMENT '제목',
    content LONGTEXT NOT NULL COMMENT '내용',
    category VARCHAR(50) NOT NULL COMMENT '카테고리 (질문/팁/공유/버그/요청 등)',
    tags JSON COMMENT '태그 (JSON Array)',
    
    -- 통계
    view_count INT DEFAULT 0 COMMENT '조회수',
    like_count INT DEFAULT 0 COMMENT '좋아요 수',
    comment_count INT DEFAULT 0 COMMENT '댓글 수',
    
    -- 채택된 답변
    accepted_comment_id BIGINT COMMENT '채택된 댓글 ID',
    is_solved BOOLEAN DEFAULT FALSE COMMENT '해결됨 여부',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,
    
    -- 외래키
    FOREIGN KEY (author_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (accepted_comment_id) REFERENCES Comments(comment_id) ON DELETE SET NULL,
    
    -- 인덱스
    INDEX idx_author_id (author_id),
    INDEX idx_category (category),
    INDEX idx_created_at (created_at),
    FULLTEXT INDEX ftx_title_content (title, content)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.11 Comments (게시판 댓글)

```sql
CREATE TABLE Comments (
    comment_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '댓글 ID',
    post_id BIGINT NOT NULL COMMENT '게시물 ID',
    author_id BIGINT NOT NULL COMMENT '작성자 ID',
    
    -- 댓글 정보
    content TEXT NOT NULL COMMENT '댓글 내용',
    
    -- 계층 구조 (대댓글)
    parent_comment_id BIGINT COMMENT '상위 댓글 ID (대댓글 시)',
    depth INT DEFAULT 0 COMMENT '깊이 (1: 답글, 2: 대답글)',
    
    -- 통계
    like_count INT DEFAULT 0 COMMENT '좋아요 수',
    
    -- 타임스탐프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,
    
    -- 외래키
    FOREIGN KEY (post_id) REFERENCES Posts(post_id) ON DELETE CASCADE,
    FOREIGN KEY (author_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (parent_comment_id) REFERENCES Comments(comment_id) ON DELETE CASCADE,
    
    -- 인덱스
    INDEX idx_post_id (post_id),
    INDEX idx_author_id (author_id),
    INDEX idx_parent_comment_id (parent_comment_id),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.12 Notifications (알림)

```sql
CREATE TABLE Notifications (
    notification_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '알림 ID',
    user_id BIGINT NOT NULL COMMENT '알림 수신자 ID',
    
    -- 알림 정보
    notification_type VARCHAR(50) NOT NULL COMMENT '알림 타입 (APPLICATION, APPROVAL, CHAT, DEADLINE 등)',
    title VARCHAR(200) NOT NULL COMMENT '알림 제목',
    message VARCHAR(500) COMMENT '알림 메시지',
    
    -- 관련 정보
    related_project_id BIGINT COMMENT '관련 프로젝트 ID',
    related_user_id BIGINT COMMENT '관련 사용자 ID',
    related_application_id BIGINT COMMENT '관련 지원 ID',
    action_url VARCHAR(500) COMMENT '알림 클릭 시 이동할 URL',
    
    -- 상태
    is_read BOOLEAN DEFAULT FALSE COMMENT '읽음 여부',
    read_at TIMESTAMP NULL COMMENT '읽음 시간',
    
    -- 타임스탠프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 외래키
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (related_project_id) REFERENCES Projects(project_id) ON DELETE SET NULL,
    FOREIGN KEY (related_user_id) REFERENCES Users(user_id) ON DELETE SET NULL,
    FOREIGN KEY (related_application_id) REFERENCES Applications(application_id) ON DELETE SET NULL,
    
    -- 인덱스
    INDEX idx_user_id (user_id),
    INDEX idx_is_read (is_read),
    INDEX idx_created_at (created_at),
    INDEX idx_notification_type (notification_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.13 AdminVerifications (관리자 회원가입 인증)

```sql
CREATE TABLE AdminVerifications (
    verification_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '인증 ID',
    user_id BIGINT NOT NULL COMMENT '사용자 ID',
    
    -- 인증 정보
    university_verification_image_url VARCHAR(500) NOT NULL COMMENT '대학교 인증 사진 URL',
    verification_status ENUM('PENDING', 'VERIFIED', 'REJECTED') DEFAULT 'PENDING' COMMENT '인증 상태',
    
    -- 검토 정보
    reviewed_by BIGINT COMMENT '검토자 관리자 ID',
    rejection_reason VARCHAR(500) COMMENT '거부 사유',
    reviewed_at TIMESTAMP NULL COMMENT '검토 시간',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- 외래키
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (reviewed_by) REFERENCES Users(user_id) ON DELETE SET NULL,
    
    -- 인덱스
    INDEX idx_user_id (user_id),
    INDEX idx_verification_status (verification_status),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.14 Tags (태그)

```sql
CREATE TABLE Tags (
    tag_id INT PRIMARY KEY AUTO_INCREMENT COMMENT '태그 ID',
    
    -- 태그 정보
    tag_name VARCHAR(50) UNIQUE NOT NULL COMMENT '태그명',
    tag_type VARCHAR(50) COMMENT '태그 타입 (기술스택, 관심분야, 역할 등)',
    
    -- 통계
    usage_count INT DEFAULT 0 COMMENT '사용 횟수',
    
    -- 타임스탰프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 인덱스
    INDEX idx_tag_name (tag_name),
    INDEX idx_tag_type (tag_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.15 Logs (시스템 로그)

```sql
CREATE TABLE Logs (
    log_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '로그 ID',
    
    -- 로그 정보
    log_level VARCHAR(20) COMMENT '로그 레벨 (INFO, WARN, ERROR)',
    log_message LONGTEXT COMMENT '로그 메시지',
    
    -- 사용자 정보
    user_id BIGINT COMMENT '관련 사용자 ID',
    action VARCHAR(100) COMMENT '액션 (회원가입, 로그인, 지원 등)',
    
    -- 상세 정보
    related_table VARCHAR(100) COMMENT '관련 테이블',
    related_id BIGINT COMMENT '관련 레코드 ID',
    
    -- 타임스탬프
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 인덱스
    INDEX idx_user_id (user_id),
    INDEX idx_action (action),
    INDEX idx_created_at (created_at),
    INDEX idx_log_level (log_level)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 4. 엔티티 관계도 (ERD)

### 4.1 관계도 (Mermaid)

```mermaid
erDiagram
    USERS ||--o{ USER_PROFILES : has
    USERS ||--o{ PROJECTS : creates
    USERS ||--o{ APPLICATIONS : submits
    USERS ||--o{ TEAM_MEMBERS : joins
    USERS ||--o{ CHAT_MESSAGES : sends
    USERS ||--o{ FILE_SHARES : uploads
    USERS ||--o{ SCHEDULES : creates
    USERS ||--o{ PROJECT_NOTICES : creates
    USERS ||--o{ POSTS : writes
    USERS ||--o{ COMMENTS : writes
    USERS ||--o{ NOTIFICATIONS : receives
    USERS ||--o{ ADMIN_VERIFICATIONS : registers
    
    PROJECTS ||--o{ APPLICATIONS : receives
    PROJECTS ||--o{ TEAM_MEMBERS : has
    PROJECTS ||--o{ CHAT_MESSAGES : contains
    PROJECTS ||--o{ FILE_SHARES : uses
    PROJECTS ||--o{ SCHEDULES : tracks
    PROJECTS ||--o{ PROJECT_NOTICES : displays
    
    APPLICATIONS ||--|| USERS : from
    APPLICATIONS ||--|| PROJECTS : to
    
    TEAM_MEMBERS ||--|| USERS : contains
    TEAM_MEMBERS ||--|| PROJECTS : belongs_to
    
    CHAT_MESSAGES ||--|| USERS : from
    CHAT_MESSAGES ||--|| PROJECTS : in
    CHAT_MESSAGES ||--o{ FILE_SHARES : links
    
    FILE_SHARES ||--|| USERS : uploaded_by
    FILE_SHARES ||--|| PROJECTS : belongs_to
    
    SCHEDULES ||--|| USERS : created_by
    SCHEDULES ||--|| USERS : assigned_to
    SCHEDULES ||--|| PROJECTS : belongs_to
    SCHEDULES ||--o{ SCHEDULES : has_subtasks
    
    PROJECT_NOTICES ||--|| USERS : created_by
    PROJECT_NOTICES ||--|| PROJECTS : belongs_to
    
    POSTS ||--|| USERS : written_by
    POSTS ||--o{ COMMENTS : has
    
    COMMENTS ||--|| USERS : written_by
    COMMENTS ||--|| POSTS : on
    COMMENTS ||--o{ COMMENTS : replies_to
    
    NOTIFICATIONS ||--|| USERS : sent_to
    NOTIFICATIONS ||--o{ PROJECTS : related_to
    NOTIFICATIONS ||--o{ APPLICATIONS : related_to
```

### 4.2 핵심 관계 설명

| 관계 | From | To | 설명 | 카디널리티 |
|------|------|----|----|-----------|
| 회원가입 | Users | AdminVerifications | 사용자가 회원가입 시 인증 생성 | 1:1 |
| 포트폴리오 | Users | UserProfiles | 각 사용자는 하나의 포트폴리오 | 1:1 |
| 프로젝트 생성 | Users | Projects | 한 사용자가 여러 프로젝트 생성 가능 | 1:N |
| 지원 | Users | Applications | 한 사용자가 여러 프로젝트에 지원 가능 | 1:N |
| 팀원 | Users | TeamMembers | 한 사용자가 여러 프로젝트의 팀원 가능 | 1:N |
| 프로젝트-지원 | Projects | Applications | 한 프로젝트는 여러 지원 받을 수 있음 | 1:N |
| 프로젝트-팀원 | Projects | TeamMembers | 한 프로젝트는 여러 팀원 보유 | 1:N |
| 채팅 | Projects | ChatMessages | 한 프로젝트마다 채팅방 생성 | 1:N |
| 파일 공유 | Projects | FileShares | 한 프로젝트에 여러 파일 공유 | 1:N |
| 일정 관리 | Projects | Schedules | 한 프로젝트에 여러 일정/작업 | 1:N |
| 게시판 | Posts | Comments | 한 게시물에 여러 댓글 | 1:N |

---

## 5. 데이터 타입 및 제약조건

### 5.1 기본 데이터 타입

| 데이터 타입 | 사용 예시 | 크기 | 설명 |
|----------|---------|------|------|
| BIGINT | user_id, project_id | 8 bytes | 매우 큰 정수값 (0 ~ 9,223,372,036,854,775,807) |
| INT | view_count | 4 bytes | 일반 정수 (0 ~ 2,147,483,647) |
| VARCHAR(n) | email, username | n bytes | 가변 문자열 |
| TEXT | description | 64KB | 긴 텍스트 |
| LONGTEXT | content | 4GB | 매우 긴 텍스트 |
| JSON | tech_stack | 변동 | JSON 문서 저장 및 쿼리 가능 |
| TIMESTAMP | created_at | 4 bytes | 날짜/시간 (자동 업데이트) |
| DATE | start_date | 3 bytes | 날짜만 (시간 없음) |
| DECIMAL(5,2) | ai_match_score | 4 bytes | 정확한 소수 (최대 999.99) |
| BOOLEAN | is_admin | 1 byte | TRUE/FALSE (0/1로 저장) |
| ENUM | status | 1-2 bytes | 정해진 값만 저장 (ACTIVE, INACTIVE 등) |

### 5.2 제약조건

| 제약조건 | 사용 예시 | 설명 |
|---------|---------|------|
| PRIMARY KEY | user_id | 각 행을 고유하게 식별 |
| UNIQUE | email, username | 중복 값 불가 (NULL은 여러 개 가능) |
| NOT NULL | title, created_at | 필수 입력 필드 |
| DEFAULT | account_status = 'PENDING' | 기본값 설정 |
| FOREIGN KEY | creator_id → Users.user_id | 참조 무결성 보장 (관계 설정) |
| CHECK | progress_percentage BETWEEN 0 AND 100 | 범위 제약 |
| UNIQUE KEY | (project_id, user_id) | 복합 키로 중복 지원 방지 |

### 5.3 ON DELETE 옵션

| 옵션 | 설명 | 사용 예시 |
|------|------|---------|
| CASCADE | 부모 삭제 시 자식도 자동 삭제 | creator_id → Users (프로젝트 생성자 삭제 시 프로젝트도 삭제) |
| SET NULL | 부모 삭제 시 자식의 외래키를 NULL로 설정 | reviewed_by → Users (검토자 삭제 시 검토자만 NULL) |
| RESTRICT | 부모 삭제 방지 (자식이 있으면 삭제 불가) | 드물게 사용 |

---

## 6. 인덱싱 전략

### 6.1 필수 인덱스

```sql
-- Users 테이블
CREATE INDEX idx_email ON Users(email);          -- 로그인 시 빠른 검색
CREATE INDEX idx_username ON Users(username);    -- 회원가입 시 중복 체크
CREATE INDEX idx_account_status ON Users(account_status); -- 승인 대기자 조회

-- Projects 테이블
CREATE INDEX idx_status ON Projects(status);     -- 진행중인 프로젝트 필터링
CREATE INDEX idx_recruitment_deadline ON Projects(recruitment_deadline); -- 마감임박 정렬
CREATE INDEX idx_created_at ON Projects(created_at); -- 최신순 정렬
CREATE FULLTEXT INDEX ftx_title_description ON Projects(title, description); -- 프로젝트 검색

-- Applications 테이블
CREATE INDEX idx_project_id ON Applications(project_id); -- 프로젝트별 지원자 조회
CREATE INDEX idx_user_id ON Applications(user_id);       -- 사용자의 지원 내역 조회
CREATE INDEX idx_status ON Applications(status);         -- 대기중인 지원 조회

-- Schedules 테이블
CREATE INDEX idx_due_date ON Schedules(due_date);        -- 마감 임박 일정 조회
CREATE INDEX idx_assigned_to ON Schedules(assigned_to);  -- 담당자별 일정 조회
CREATE INDEX idx_status ON Schedules(status);            -- 진행중인 작업 필터링

-- Notifications 테이블
CREATE INDEX idx_user_id ON Notifications(user_id);      -- 사용자의 알림 조회
CREATE INDEX idx_is_read ON Notifications(is_read);      -- 안 읽은 알림 조회
CREATE INDEX idx_created_at ON Notifications(created_at); -- 최신 알림부터 조회

-- ChatMessages 테이블
CREATE INDEX idx_project_id ON ChatMessages(project_id); -- 프로젝트 채팅방 메시지 조회
CREATE INDEX idx_created_at ON ChatMessages(created_at); -- 시간순 메시지 조회

-- Posts 테이블
CREATE FULLTEXT INDEX ftx_title_content ON Posts(title, content); -- 게시판 검색
```

### 6.2 복합 인덱스 (Composite Index)

```sql
-- 자주 함께 사용되는 칼럼들
CREATE INDEX idx_project_status ON Projects(status, recruitment_deadline);
CREATE INDEX idx_app_project_status ON Applications(project_id, status);
CREATE INDEX idx_schedule_project_due ON Schedules(project_id, due_date, status);
```

### 6.3 인덱싱 성능 팁

| 규칙 | 설명 | 예시 |
|------|------|------|
| **선택도 높음** | WHERE 절에서 개수를 많이 줄이는 칼럼 우선 인덱싱 | `status` (3-4개 값) 보다 `email` (거의 모두 고유) 우선 |
| **정렬이 필요하면** | ORDER BY 절에 사용되는 칼럼 인덱싱 | `created_at`, `deadline` |
| **조인이 많으면** | 외래키에 인덱스 생성 | `creator_id`, `user_id` |
| **LIKE 검색** | FULLTEXT INDEX 사용 | 게시판 검색, 프로젝트 검색 |
| **범위 검색** | 범위를 제한하는 칼럼 인덱싱 | `due_date BETWEEN ? AND ?` |

---

## 7. 보안 정책

### 7.1 비밀번호 보안

```
❌ 절대 평문 저장 금지
✅ bcrypt로 해싱 저장 (cost = 10)

예:
- 입력 비밀번호: "MyP@ssw0rd123"
- 저장: "$2b$10$abcdef...xyz (60자 해시)"
```

### 7.2 민감한 데이터 암호화

| 데이터 | 암호화 | 이유 |
|-------|--------|------|
| password_hash | bcrypt | 단방향 해싱 |
| email | 선택사항 | GDPR 준수 시 암호화 |
| phone | 선택사항 | 개인정보 보호 |
| university_verification_image_url | HTTPS 전송 + S3 버킷 암호화 | 신원 증명 정보 |
| api_keys (향후) | 대칭 암호화 (AES-256) | 서드파티 API 연동 시 |

### 7.3 접근 제어 (Authorization)

```
Role-Based Access Control (RBAC)

ADMIN:
  - 모든 사용자/프로젝트 조회/관리
  - 회원가입 승인/거부
  - 게시물/댓글 관리

USER:
  - 자신의 프로필만 수정
  - 생성한 프로젝트만 수정/삭제
  - 작성한 게시물/댓글만 수정/삭제

PROJECT_LEADER (팀장):
  - 팀원 승인/거부
  - 일정 관리
  - 중요 정보 박스 작성
  - 팀 채팅 관리
```

### 7.4 데이터 노출 방지

```sql
-- ❌ 절대 금지
SELECT password_hash FROM Users;

-- ✅ 권장
SELECT user_id, username, email, created_at FROM Users;
```

### 7.5 감시 및 로깅

```
모든 주요 작업 기록:
- 회원가입, 로그인, 로그아웃
- 프로젝트 생성/수정/삭제
- 팀원 승인/거부
- 게시물/댓글 삭제
- 관리자 조치
```

---

## 8. 성능 최적화 가이드

### 8.1 쿼리 최적화

```sql
-- ❌ 비효율적
SELECT * FROM Projects WHERE id IN (
  SELECT project_id FROM Applications WHERE user_id = 1
);

-- ✅ 효율적 (JOIN 사용)
SELECT DISTINCT p.* FROM Projects p
INNER JOIN Applications a ON p.project_id = a.project_id
WHERE a.user_id = 1;
```

### 8.2 연결 풀링

- **Max Connections**: 100-200 (환경에 따라)
- **Connection Timeout**: 30초
- **Idle Timeout**: 10분

### 8.3 캐싱 전략 (Redis)

```
캐시할 데이터:
- 사용자 포트폴리오 (key: user:profile:{user_id})
- 프로젝트 정보 (key: project:info:{project_id})
- 태그 목록 (key: tags:all)
- 사용자 알림 (key: notifications:{user_id})

TTL: 1시간 ~ 24시간
```

---

**최종 수정일**: 2026년 10월 7일  
**버전**: 1.0 (초판)  
**총 테이블 수**: 15개  
**총 칼럼 수**: 약 200+개
