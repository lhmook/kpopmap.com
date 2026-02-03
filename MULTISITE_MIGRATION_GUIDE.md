# WordPress 멀티사이트 → Single Site 통합 가이드

> 일반적인 WordPress 멀티사이트를 Single Site로 통합하고, Next.js로 점진적 마이그레이션하는 방법론

## 개요

여러 서브도메인으로 운영되는 WordPress 멀티사이트를 하나의 Single Site로 통합하여, 이후 Next.js로 섹션별 점진적 전환이 가능한 구조로 만드는 가이드입니다.

### 목표

- 여러 서브도메인 → 단일 도메인의 서브디렉토리로 통합
- 각 섹션을 독립적인 Custom Post Type으로 구분
- SEO 유지 (301 리다이렉트)
- Next.js 점진적 전환 준비

## 아키텍처 변경

### Before (멀티사이트)
```
https://section1.example.com/category/article-slug/
https://section2.example.com/category/article-slug/
https://section3.example.com/category/article-slug/
```

### After (Single Site)
```
https://www.example.com/section1/article-slug/
https://www.example.com/section2/article-slug/
https://www.example.com/section3/article-slug/
```

---

## 핵심 설계 결정

### 1. 섹션 구분 전략: Custom Post Type

**접근 방법:** 각 섹션을 독립적인 post_type으로 구분

**Post Type 매핑 예시:**
- Main: `post` (기존 유지)
- Section 1: `section1` (새 CPT)
- Section 2: `section2` (새 CPT)
- Section 3: `section3` (새 CPT)

**장점:**
- WordPress 표준 방식
- Admin에서 명확한 구분
- REST API 자동 지원: `/wp-json/wp/v2/section1`
- Next.js migration 용이

### 2. Taxonomy 구조: 각 post_type별 독립

**전략:**
- Main: `category`, `post_tag` (기존 유지)
- Section 1: `section1_category`, `section1_tag`
- Section 2: `section2_category`, `section2_tag`

**이유:**
- 각 섹션의 taxonomy 명확히 분리
- Admin 혼동 방지

### 3. Post ID 생성: AUTO_INCREMENT

**전략:**
- ID offset 사용하지 않음
- INSERT 시 AUTO_INCREMENT로 자동 생성
- 깔끔한 데이터베이스 구조

### 4. URL 구조: Post Type 기반

**전략:**
```
- Main: /article-slug/
- Section 1: /section1/article-slug/
- Section 2: /section2/article-slug/
```

---

## 마이그레이션 단계

### Phase 1: 준비 (1-2일)

**백업:**
```bash
mysqldump -u user -p database_name > backup_$(date +%Y%m%d).sql
tar -czf wp-content_backup.tar.gz /path/to/wp-content/
```

**현황 분석:**
```sql
-- 각 blog의 콘텐츠 수 확인
SELECT 'Blog 1' as site, COUNT(*) FROM wp_posts WHERE post_status='publish'
UNION ALL
SELECT 'Blog 2', COUNT(*) FROM wp_2_posts WHERE post_status='publish';
```

### Phase 2: 코드 준비 (2-3일)

**새로운 CPT 등록:**

```php
<?php
// wp-content/mu-plugins/merged-cpt.php

function register_merged_cpts() {
    // Section 1 CPT
    register_post_type('section1', [
        'labels' => ['name' => 'Section 1', 'singular_name' => 'Section 1 Article'],
        'public' => true,
        'show_in_rest' => true,
        'rewrite' => ['slug' => 'section1', 'with_front' => false],
        'supports' => ['title', 'editor', 'thumbnail', 'excerpt', 'comments', 'author'],
        'has_archive' => true,
        'taxonomies' => ['section1_category', 'section1_tag'],
    ]);

    // Section 2 CPT
    register_post_type('section2', [
        'labels' => ['name' => 'Section 2', 'singular_name' => 'Section 2 Article'],
        'public' => true,
        'show_in_rest' => true,
        'rewrite' => ['slug' => 'section2', 'with_front' => false],
        'supports' => ['title', 'editor', 'thumbnail', 'excerpt', 'comments', 'author'],
        'has_archive' => true,
        'taxonomies' => ['section2_category', 'section2_tag'],
    ]);
}
add_action('init', 'register_merged_cpts', 0);

function register_merged_taxonomies() {
    // Section 1 Taxonomies
    register_taxonomy('section1_category', 'section1', [
        'labels' => ['name' => 'Section 1 Categories'],
        'hierarchical' => true,
        'show_in_rest' => true,
        'rewrite' => ['slug' => 'section1/category'],
    ]);

    register_taxonomy('section1_tag', 'section1', [
        'labels' => ['name' => 'Section 1 Tags'],
        'hierarchical' => false,
        'show_in_rest' => true,
    ]);
}
add_action('init', 'register_merged_taxonomies', 0);
```

### Phase 3: DB Migration (2-3일)

**핵심 로직:**

```sql
START TRANSACTION;

-- 1. 백업 테이블 생성
CREATE TABLE wp_posts_backup AS SELECT * FROM wp_posts;
CREATE TABLE wp_postmeta_backup AS SELECT * FROM wp_postmeta;

-- 2. 임시 매핑 테이블
CREATE TEMPORARY TABLE post_id_mapping (
    old_post_id BIGINT,
    new_post_id BIGINT,
    source_blog INT
);

-- 3. Blog 2 Posts 복사 (AUTO_INCREMENT, post_type 변경)
INSERT INTO wp_posts (
    post_author, post_date, post_content, post_title,
    post_status, post_name, guid, post_type
)
SELECT
    post_author,
    post_date,
    post_content,
    post_title,
    post_status,
    post_name,
    REPLACE(guid, 'section1.example.com', 'www.example.com/section1'),
    'section1' -- post_type 변경
FROM wp_2_posts
WHERE post_type = 'post' AND post_status = 'publish';

-- 4. Post ID 매핑 저장
INSERT INTO post_id_mapping (old_post_id, new_post_id, source_blog)
SELECT
    p2.ID,
    (SELECT MAX(ID) FROM wp_posts
     WHERE post_title = p2.post_title AND post_date = p2.post_date),
    2
FROM wp_2_posts p2
WHERE p2.post_type = 'post';

-- 5. Postmeta 복사
INSERT INTO wp_postmeta (post_id, meta_key, meta_value)
SELECT pim.new_post_id, pm.meta_key, pm.meta_value
FROM wp_2_postmeta pm
JOIN post_id_mapping pim ON pm.post_id = pim.old_post_id;

-- 6. Comments 복사
INSERT INTO wp_comments (comment_post_ID, comment_author, comment_content, comment_date)
SELECT pim.new_post_id, c.comment_author, c.comment_content, c.comment_date
FROM wp_2_comments c
JOIN post_id_mapping pim ON c.comment_post_ID = pim.old_post_id;

COMMIT;
```

**Taxonomy 복사:**

```sql
-- Blog 2의 category → section1_category로 복사
INSERT INTO wp_terms (name, slug)
SELECT name, CONCAT('section1-', slug)
FROM wp_2_terms t
WHERE taxonomy = 'category';

INSERT INTO wp_term_taxonomy (term_id, taxonomy)
SELECT
    (SELECT term_id FROM wp_terms WHERE slug = CONCAT('section1-', t.slug)),
    'section1_category'
FROM wp_2_terms t
WHERE taxonomy = 'category';
```

### Phase 4: 코드 수정 (3-5일)

**멀티사이트 함수 수정:**

```php
// 기존 멀티사이트 함수
function custom_set_blog($site) {
    // switch_to_blog() 호출
}

// 변경 후 (no-op)
function custom_set_blog($site) {
    // Single site에서는 아무 작업도 하지 않음
    return;
}

// 또는 완전 제거하고 모든 호출부 삭제
```

**Blog ID 판단 함수 수정:**

```php
// 기존: URL로 판단
function get_current_section($url = null) {
    // URL 파싱으로 site 결정
}

// 변경 후: post_type으로 판단
function get_current_section() {
    if (is_singular()) {
        $post_type = get_post_type();
        $type_map = [
            'post' => 'main',
            'section1' => 'section1',
            'section2' => 'section2',
        ];
        return $type_map[$post_type] ?? 'main';
    }

    if (is_post_type_archive()) {
        $post_type = get_query_var('post_type');
        return $post_type ?? 'main';
    }

    return 'main';
}
```

**Query Filter 수정:**

```php
// 기존
function custom_query_filter($query) {
    if (get_current_section() == 'section1') {
        // section1 로직
    }
}

// 변경 후
function custom_query_filter($query) {
    if (is_admin() || !$query->is_main_query()) return;

    $post_type = $query->get('post_type');

    if ($post_type == 'section1') {
        // Section 1 전용 로직
    }
}
```

### Phase 5: Nginx 설정

**301 리다이렉트:**

```nginx
# 서브도메인 → 서브디렉토리 리다이렉트
server {
    listen 443 ssl http2;
    server_name section1.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    return 301 https://www.example.com/section1$request_uri;
}

# 메인 서버
server {
    listen 443 ssl http2;
    server_name www.example.com;

    root /path/to/wordpress;

    location ~ ^/(section1|section2|section3)/ {
        try_files $uri $uri/ /index.php?$args;
    }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }
}
```

### Phase 6: 프로덕션 배포

**배포 체크리스트:**

1. Maintenance Mode 활성화
2. 최종 백업
3. DB Migration 실행
4. wp-config.php 수정 (MULTISITE 비활성화)
5. 코드 배포
6. Cache & Rewrite 초기화
7. Nginx 설정 적용
8. Smoke Test
9. Maintenance Mode 해제

---

## 검증 방법

### 데이터 무결성

```sql
-- Post 수 검증 (post_type별)
SELECT post_type, COUNT(*) as post_count
FROM wp_posts
WHERE post_status='publish'
GROUP BY post_type;

-- Taxonomy 확인
SELECT taxonomy, COUNT(*) as term_count
FROM wp_term_taxonomy
WHERE taxonomy LIKE '%_category'
GROUP BY taxonomy;
```

### 기능 테스트

- [ ] 각 섹션 페이지 접근
- [ ] 개별 포스트 접근
- [ ] 301 리다이렉트 확인
- [ ] Admin 기능 정상 작동
- [ ] REST API 엔드포인트 테스트

---

## 롤백 계획

**트리거 조건:**
- 500 에러율 > 5% (5분간)
- 응답 시간 > 3초 (평균)
- 치명적 기능 장애

**롤백 절차:**
```bash
# DB 복원
mysql -u user -p database_name < backup.sql

# 또는 부분 롤백
DELETE FROM wp_posts WHERE post_type IN ('section1', 'section2');
DELETE FROM wp_term_taxonomy WHERE taxonomy LIKE 'section%';

# wp-config.php 원복
cp wp-config.php.backup wp-config.php

# 코드 원복
git reset --hard PREVIOUS_COMMIT

# Nginx 원복
sudo systemctl reload nginx

# Cache clear
wp cache flush
wp rewrite flush
```

---

## Next.js Migration 준비

### WordPress REST API 활용

```javascript
// Next.js에서 호출
// Section 1 posts
const response = await fetch('https://www.example.com/wp-json/wp/v2/section1?per_page=10');

// Section 1 categories
const categories = await fetch('https://www.example.com/wp-json/wp/v2/section1_category');

// Single post
const post = await fetch('https://www.example.com/wp-json/wp/v2/section1/123');
```

### Nginx 점진적 전환

```nginx
# Section 1만 Next.js로
location /section1/ {
    proxy_pass http://localhost:3000;
}

# WordPress API는 유지
location ~ ^/wp-(json|admin|content|includes)/ {
    try_files $uri $uri/ /index.php?$args;
}

# 나머지는 WordPress
location / {
    try_files $uri $uri/ /index.php?$args;
}
```

---

## 예상 타임라인

| Phase | 작업 | 기간 |
|-------|------|------|
| 1 | 준비 및 백업 | 1-2일 |
| 2 | 코드 준비 | 2-3일 |
| 3 | DB Migration 스크립트 | 2-3일 |
| 4 | 코드 수정 | 3-5일 |
| 5 | Nginx 설정 | 배포 시 |
| 6 | 프로덕션 배포 | 2-3시간 |
| **Total** | | **10-16일** |

---

## 리스크 및 완화

### SEO 리스크
- **완화:** Nginx 301 리다이렉트, Google Search Console 업데이트

### 데이터 손실
- **완화:** 다중 백업, Transaction 사용, 검증 스크립트

### 다운타임
- **완화:** Staging 테스트, 자동화 스크립트, 롤백 준비

### 성능 저하
- **완화:** DB 인덱스 추가, Object cache 강화

---

## 핵심 교훈

1. **Post Type 기반 구분이 Taxonomy보다 유리**
   - WordPress 표준 방식
   - REST API 자동 지원
   - Admin 관리 명확

2. **AUTO_INCREMENT가 ID Offset보다 깔끔**
   - 자연스러운 ID 흐름
   - 롤백 시 post_type으로 구분 가능

3. **독립 Taxonomy로 명확한 분리**
   - 각 섹션의 카테고리 혼동 방지
   - 섹션별 독립 관리

4. **Nginx 301 리다이렉트로 SEO 유지**
   - WordPress 부하 없음
   - 빠른 리다이렉트

---

## 참고 자료

- [WordPress Multisite to Single Site](https://wordpress.org/support/article/multisite-network-administration/)
- [WordPress REST API](https://developer.wordpress.org/rest-api/)
- [Custom Post Types](https://developer.wordpress.org/plugins/post-types/)
- [Next.js WordPress Integration](https://nextjs.org/docs/basic-features/data-fetching)
