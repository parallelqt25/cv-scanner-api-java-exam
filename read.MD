CV Scanner API — Spring Boot (MariaDB) Take‑Home Assessment

Goal: Build a small, production‑ish Java 17+ / Spring Boot 3.x API that can scan CVs to extract basic profile data and detect skills based on keywords stored in MariaDB.

Use Java 17+, Spring Boot 3.x, Spring Web, Spring Validation, Spring Data JPA, MariaDB.

Use the candidate’s email as the primary key for CVs (DB‑level).

Publish a public GitHub repository with a clear, exact README.md (this file) and a Postman collection.


Table of Contents

Functional Requirements

Non‑Functional Requirements

Tech Constraints

Relational Data Model

Full MariaDB DDL

API Specification

Scanning Rules

Pagination, Sorting, Filtering

Error Model

Caching Requirements

Authentication Using JWT (Optional)

Batch Scan (Optional)

Basic UI (Optional)

Project Structure

Configuration & Setup

Build & Run

Dependencies (POM excerpt)

Postman Collection

Quick cURL Tests

Quality Bar & Rubric

DTO Signatures (Reference)

Minimal Service Outline (Pseudocode)

Submission Instructions

Functional Requirements
1) Keyword CRUD with activation/deactivation

Create, read, update, delete keywords.

Each keyword has:

keywordId (surrogate PK)

keywordPhrase (case‑insensitive unique; normalized)

keywordIsActive (boolean)

createdAt, updatedAt (timestamps)

Activation toggle via dedicated endpoint.

2) Keyword search

Search with pagination, sorting, filtering.

Text query q matches phrase (case‑insensitive).

3) Scan & Rescan

POST /scan accepts a single CV (PDF/DOCX/TXT) or raw text.
Returns:

extractedCandidateName

extractedCandidateEmail (primary key)

extractedPhoneNumber

matchedKeywords[] with per‑keyword occurrenceCount

matchScore (0..1)

POST /rescan?email=... reprocesses stored CV text using current active keywords.

Persistence

Store keywords and CVs in MariaDB (DDL below).

CVs are keyed by email (PK).

Each scan is recorded in cv_scans; per‑keyword matches in cv_scan_matches.

cvs.last_scan_id references the latest scan.

Non‑Functional Requirements

Precise, descriptive names (e.g., normalizedCvTextContent, keywordSearchQuery, keywordCacheTtlSeconds).

Java 17+, Spring Boot 3.x, Bean Validation on all inputs.

Clear layering: controller → service → repository with DTOs.

Useful logs, structured errors, robust 4xx/5xx handling.

Conventional Commits (or consistent) in a public repo.

Unit tests for pure logic (normalization, keyword matching).

Integration tests (e.g., Testcontainers MariaDB).

Tech Constraints

Build: Maven (or Gradle).

DB: MariaDB 10.6+ (InnoDB, utf8mb4).

JPA: Hibernate (Spring Data JPA).

File parsing: Apache Tika (PDF/DOC/DOCX/TXT).

Optional cache: Spring Cache + Caffeine.

JWT (optional): spring-boot-starter-oauth2-resource-server or jjwt.

Relational Data Model

Entities

keywords — skill phrases (case‑insensitive unique).

cvs — uploaded/normalized CV text, PK = email.

cv_scans — each scan result (summary and score).

cv_scan_matches — N:M between scans and keywords with occurrence_count.

Why this shape?

Normalized storage enables rescan against updated keyword sets.

cv_scans records history; cvs.last_scan_id points to latest.

Full MariaDB DDL

Put this file at src/main/resources/db/migration/V1__baseline.sql (Flyway), or run manually.

-- Requires MariaDB 10.6+ (InnoDB; utf8mb4)
SET NAMES utf8mb4;
SET sql_mode = 'STRICT_ALL_TABLES';

-- ============================================================================
-- TABLE: keywords  (case-insensitive uniqueness via GENERATED LOWER() column)
-- ============================================================================
CREATE TABLE IF NOT EXISTS keywords (
  keyword_id              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  keyword_phrase          VARCHAR(255)     NOT NULL,
  keyword_phrase_lower    VARCHAR(255)
    GENERATED ALWAYS AS (LOWER(TRIM(keyword_phrase))) PERSISTENT,
  keyword_is_active       TINYINT(1)       NOT NULL DEFAULT 1,
  created_at              DATETIME(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at              DATETIME(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
                                           ON UPDATE CURRENT_TIMESTAMP(6),

  CONSTRAINT pk_keywords PRIMARY KEY (keyword_id),
  CONSTRAINT chk_keyword_phrase_nonempty CHECK (CHAR_LENGTH(TRIM(keyword_phrase)) > 0),

  UNIQUE KEY uk_keywords_phrase_lower (keyword_phrase_lower),
  KEY idx_keywords_active_phrase (keyword_is_active, keyword_phrase_lower),
  KEY idx_keywords_updated_at (updated_at)
)
ENGINE=InnoDB
DEFAULT CHARSET=utf8mb4
COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: cvs  (Primary key is candidate email; stores normalized text)
-- ============================================================================
CREATE TABLE IF NOT EXISTS cvs (
  email                         VARCHAR(320)   NOT NULL,
  original_filename             VARCHAR(255)   NULL,
  uploaded_at                   DATETIME(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  last_scanned_at               DATETIME(6)    NULL,
  cv_text_content               MEDIUMTEXT     NOT NULL,

  extracted_candidate_name      VARCHAR(255)   NULL,
  extracted_phone_number        VARCHAR(64)    NULL,

  last_scan_id                  BIGINT UNSIGNED NULL,

  CONSTRAINT pk_cvs PRIMARY KEY (email),
  KEY idx_cvs_last_scanned_at (last_scanned_at)
)
ENGINE=InnoDB
DEFAULT CHARSET=utf8mb4
COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: cv_scans  (each scan result; enforces email consistency vs cvs.email)
-- ============================================================================
CREATE TABLE IF NOT EXISTS cv_scans (
  scan_id                        BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  cv_email                       VARCHAR(320)    NOT NULL,
  processed_at                   DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),

  extracted_candidate_email      VARCHAR(320)    NOT NULL,
  extracted_candidate_name       VARCHAR(255)    NULL,
  extracted_phone_number         VARCHAR(64)     NULL,

  matched_keyword_count          INT UNSIGNED    NOT NULL DEFAULT 0,
  total_active_keyword_count     INT UNSIGNED    NOT NULL DEFAULT 0,
  match_score                    DECIMAL(6,5)    NOT NULL DEFAULT 0.00000,

  CONSTRAINT pk_cv_scans PRIMARY KEY (scan_id),
  CONSTRAINT fk_cv_scans_cv FOREIGN KEY (cv_email) REFERENCES cvs(email)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT chk_emails_consistent CHECK (LOWER(extracted_candidate_email) = LOWER(cv_email)),

  KEY idx_cv_scans_email_processed (cv_email, processed_at DESC)
)
ENGINE=InnoDB
DEFAULT CHARSET=utf8mb4
COLLATE=utf8mb4_unicode_ci;

-- Link latest scan from cvs to cv_scans
ALTER TABLE cvs
  ADD CONSTRAINT fk_cvs_last_scan FOREIGN KEY (last_scan_id)
  REFERENCES cv_scans(scan_id)
  ON DELETE SET NULL ON UPDATE CASCADE;

-- ============================================================================
-- TABLE: cv_scan_matches  (per-keyword occurrence counts for a scan)
-- ============================================================================
CREATE TABLE IF NOT EXISTS cv_scan_matches (
  scan_match_id               BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  scan_id                     BIGINT UNSIGNED NOT NULL,
  keyword_id                  BIGINT UNSIGNED NOT NULL,
  occurrence_count            INT UNSIGNED    NOT NULL,

  CONSTRAINT pk_cv_scan_matches PRIMARY KEY (scan_match_id),
  CONSTRAINT fk_scan_matches_scan FOREIGN KEY (scan_id)
    REFERENCES cv_scans(scan_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_scan_matches_keyword FOREIGN KEY (keyword_id)
    REFERENCES keywords(keyword_id)
    ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT chk_occurrence_positive CHECK (occurrence_count > 0),

  UNIQUE KEY uk_scan_keyword (scan_id, keyword_id),
  KEY idx_scan_matches_keyword (keyword_id)
)
ENGINE=InnoDB
DEFAULT CHARSET=utf8mb4
COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- OPTIONAL: app_users for JWT demo (if implementing DB-backed auth)
-- ============================================================================
CREATE TABLE IF NOT EXISTS app_users (
  user_id                 BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  username                VARCHAR(100)    NOT NULL,
  username_lower          VARCHAR(100)
    GENERATED ALWAYS AS (LOWER(TRIM(username))) PERSISTENT,
  password_hash           VARCHAR(100)    NOT NULL, -- e.g., BCrypt 60 chars
  display_name            VARCHAR(150)    NULL,
  role                    VARCHAR(50)     NOT NULL DEFAULT 'ADMIN',
  is_active               TINYINT(1)      NOT NULL DEFAULT 1,
  created_at              DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at              DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
                                         ON UPDATE CURRENT_TIMESTAMP(6),

  CONSTRAINT pk_app_users PRIMARY KEY (user_id),
  UNIQUE KEY uk_app_users_username_lower (username_lower)
)
ENGINE=InnoDB
DEFAULT CHARSET=utf8mb4
COLLATE=utf8mb4_unicode_ci;

-- SAMPLE DATA (optional)
-- INSERT INTO keywords (keyword_phrase, keyword_is_active) VALUES
--   ('Java', 1), ('Spring Boot', 1), ('Hibernate', 1), ('Docker', 1), ('Kubernetes', 0);


Notes

Generated *_lower columns ensure true case‑insensitive uniqueness regardless of collation.

cvs.email is the PK and referenced everywhere; we additionally CHECK that extracted_candidate_email equals cv_email.

All tables use InnoDB, utf8mb4, STRICT SQL mode.

API Specification

Base URL: http://localhost:8080/api

Health
GET /health
→ 200 { "status": "ok", "service": "cv-scanner-api", "ts": "<iso>" }

Keywords

Create

POST /keywords
Content-Type: application/json

{
  "keywordPhrase": "Spring Boot",
  "keywordIsActive": true
}
→ 201 { keywordId, keywordPhrase, keywordIsActive, createdAt, updatedAt }


Get by ID

GET /keywords/{keywordId}
→ 200 {...} or 404


Update

PUT /keywords/{keywordId}
{
  "keywordPhrase": "hibernate",
  "keywordIsActive": false
}
→ 200 {...} or 404


Delete

DELETE /keywords/{keywordId}
→ 204 or 404


Toggle activation

PATCH /keywords/{keywordId}/activation
{ "keywordIsActive": true }
→ 200 {...} or 404


Search (pagination/sort/filter)

GET /keywords/search?q=script&keywordIsActive=true&page=1&pageSize=20&sortBy=createdAt&sortDirection=desc
→ 200 {
     "page": 1,
     "pageSize": 20,
     "totalItems": 1,
     "totalPages": 1,
     "items": [{ "keywordId": 1, "keywordPhrase": "javascript", "keywordIsActive": true, "createdAt": "...", "updatedAt": "..." }]
   }

Scan

Upload & scan a CV

POST /scan
Content-Type: multipart/form-data

file: <PDF|DOCX|TXT>
-- OR --
rawText: <string>

→ 200 ScanResult


Rescan existing CV by email

POST /rescan?email=jane.doe@example.com
→ 200 ScanResult
→ 404 if CV not found

Response Types
{
  "extractedCandidateName": "Jane Doe",
  "extractedCandidateEmail": "jane.doe@example.com",
  "extractedPhoneNumber": "+14155551234",
  "matchedKeywords": [
    { "keywordPhrase": "java", "occurrenceCount": 7 },
    { "keywordPhrase": "spring boot", "occurrenceCount": 3 }
  ],
  "matchScore": 0.4,
  "totalActiveKeywordCount": 5,
  "processedAtIso": "2025-01-01T12:34:56.789Z"
}

Scanning Rules

Text extraction

Use Apache Tika to extract from PDF/DOC/DOCX/TXT.

If both file and rawText are provided, prefer file.

Normalization

Lowercase; collapse whitespace; strip non‑alphanumeric except +, #, ., and hyphens inside tokens.

Implement String normalizeText(String raw).

Email & phone extraction

Email: robust regex (RFC‑ish).

Phone: permissive international regex; return best first.

Name extraction (best‑effort)

Look in the first ~5 non‑empty lines for 2–4 capitalized tokens not containing email/phone.

Fallback: derive from the email local‑part (jane.doe → Jane Doe).

Keyword matching

Load active keywords only (use cache).

Match whole words/phrases; count occurrenceCount for each (regex with \b where sensible; escape c++, node.js, etc.).

Include only keywords with occurrenceCount > 0.

Score

matchScore = matchedKeywordCount / totalActiveKeywordCount (0..1).

If no active keywords exist, matchScore = 0 and totalActiveKeywordCount = 0.

Persistence

Upsert cvs by email (PK).

Insert into cv_scans and cv_scan_matches; update cvs.last_scan_id & cvs.last_scanned_at.

Pagination, Sorting, Filtering

Pagination: page (1‑based), pageSize (default 20, max 100).

Sorting: sortBy ∈ keywordPhrase | createdAt | updatedAt | keywordIsActive; sortDirection ∈ asc|desc.

Filtering: keywordIsActive boolean; q applies to keywordPhrase contains (case‑insensitive).

Implement via Spring Data Pageable + custom Specification/Example.

Error Model

All errors return JSON:

{
  "errorCode": "VALIDATION_ERROR | NOT_FOUND | CONFLICT | UNAUTHORIZED | FORBIDDEN | INTERNAL_ERROR",
  "errorMessage": "Human-readable detail",
  "details": { "field": "keywordPhrase" }
}


400: validation

401/403: auth (if using JWT)

404: not found

409: uniqueness conflict (duplicate keyword)

415: unsupported media type

500: unexpected error

Caching Requirements

Use Spring Cache (Caffeine) for active keywords:

Cache name: activeKeywordsCache

TTL: 60 seconds (config: keywordCacheTtlSeconds).

Add X-Cache: HIT|MISS header on routes that use it.

Invalidate on keyword create/update/delete/toggle.

application.yml fragment

spring:
  cache:
    cache-names: activeKeywordsCache
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=60s
app:
  keywordCacheTtlSeconds: 60

Authentication Using JWT (Optional)

POST /auth/login with a demo user (env or app_users table) → JWT.

Protect mutating routes (POST/PUT/PATCH/DELETE, /scan, /rescan, /batch/scan).

Use Authorization: Bearer <token>.

Env: JWT_SECRET, optional DEMO_USERNAME, DEMO_PASSWORD (or seed a user in app_users with BCrypt).

Batch Scan (Optional)
POST /batch/scan
Content-Type: multipart/form-data
archive: <ZIP with .pdf/.doc/.docx/.txt>

→ 200 {
  "batchId": "2025-...uuid",
  "items": [
    { "filename": "jane.pdf", "status": "ok", "result": { ...ScanResult } },
    { "filename": "dup_jane.pdf", "status": "error", "errorMessage": "duplicate email..." }
  ]
}


Partial success allowed.

On duplicate emails within the same ZIP, keep the first and mark the rest as "error".

Basic UI (Optional)

Serve a tiny HTML page at /:

Upload control for /scan and display JSON result.

Keyword table (from /keywords/search) with activation toggles.

Project Structure
.
├─ src/main/java/com/example/cvscanner/
│  ├─ CvScannerApplication.java
│  ├─ api/
│  │  ├─ KeywordController.java
│  │  ├─ ScanController.java
│  │  └─ AuthController.java           // optional
│  ├─ service/
│  │  ├─ KeywordService.java
│  │  ├─ ScanService.java
│  │  └─ TextAnalysisService.java
│  ├─ repository/
│  │  ├─ KeywordRepository.java
│  │  ├─ CvRepository.java
│  │  ├─ CvScanRepository.java
│  │  └─ CvScanMatchRepository.java
│  ├─ domain/                          // JPA entities
│  │  ├─ Keyword.java
│  │  ├─ Cv.java
│  │  ├─ CvScan.java
│  │  └─ CvScanMatch.java
│  ├─ dto/
│  │  ├─ CreateKeywordRequest.java
│  │  ├─ KeywordResponse.java
│  │  ├─ KeywordSearchResponse.java
│  │  └─ ScanResult.java
│  ├─ config/
│  │  ├─ CacheConfig.java
│  │  └─ SecurityConfig.java           // optional JWT
│  ├─ util/
│  │  ├─ TextNormalizer.java
│  │  ├─ EmailPhoneExtractor.java
│  │  └─ NameHeuristicExtractor.java
│  └─ middleware/
│     └─ GlobalExceptionHandler.java
├─ src/main/resources/
│  ├─ application.yml
│  └─ db/migration/V1__baseline.sql    // DDL above (Flyway)
├─ postman/CV-Scanner.postman_collection.json
├─ pom.xml
└─ README.md

Configuration & Setup

MariaDB

# Create DB and user (adjust creds)
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS cv_scanner DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -p -e "CREATE USER IF NOT EXISTS 'cv_scanner_user'@'%' IDENTIFIED BY 'changeme';"
mysql -u root -p -e "GRANT ALL PRIVILEGES ON cv_scanner.* TO 'cv_scanner_user'@'%'; FLUSH PRIVILEGES;"


application.yml (example)

server:
  port: 8080

spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/cv_scanner?useUnicode=true&characterEncoding=utf8
    username: cv_scanner_user
    password: changeme
    driver-class-name: org.mariadb.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: validate           # Flyway manages schema
    properties:
      hibernate:
        format_sql: true
        jdbc:
          time_zone: UTC
    open-in-view: false
  flyway:
    enabled: true
    baseline-on-migrate: true

app:
  keywordCacheTtlSeconds: 60
  uploadMaxSizeMb: 10

Build & Run
# Build & run
./mvnw spring-boot:run
# or
./mvnw clean package && java -jar target/cv-scanner-*.jar


Server runs at http://localhost:8080.

Dependencies (POM excerpt)
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.mariadb.jdbc</groupId>
    <artifactId>mariadb-java-client</artifactId>
    <version>3.4.0</version>
  </dependency>
  <dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-core</artifactId>
    <version>2.9.0</version>
  </dependency>
  <dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-parsers-standard-package</artifactId>
    <version>2.9.0</version>
  </dependency>
  <!-- Optional cache -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
  </dependency>
  <dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
  </dependency>
  <!-- Optional JWT stack -->
  <!--
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
  </dependency>
  -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mariadb</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>

Postman Collection

Include postman/CV-Scanner.postman_collection.json with folders:

Health — GET /api/health

Keywords — create/get/update/toggle/delete/search

Scan — POST /api/scan (multipart or rawText), POST /api/rescan?email=...

Optional — POST /api/auth/login, POST /api/batch/scan

Example Postman tests

pm.test("Status is 200", () => pm.response.to.have.status(200));
pm.test("Has matchScore", () => pm.expect(pm.response.json()).to.have.property("matchScore"));

Quick cURL Tests
# Health
curl -s http://localhost:8080/api/health

# Create a keyword
curl -s -X POST http://localhost:8080/api/keywords \
  -H "Content-Type: application/json" \
  -d '{"keywordPhrase":"Spring Boot","keywordIsActive":true}'

# Search keywords
curl -s "http://localhost:8080/api/keywords/search?q=boot&page=1&pageSize=5"

# Scan with raw text
curl -s -X POST http://localhost:8080/api/scan \
  -F rawText='Jane Doe
jane.doe@example.com
+1 415 555 1234
Skills: Java, Spring Boot, Hibernate, Docker'

# Rescan
curl -s -X POST "http://localhost:8080/api/rescan?email=jane.doe@example.com"

Quality Bar & Rubric (100 pts)

Core functionality (CRUD, search, scan/rescan): 40

Code quality & structure (naming, layering, DTOs, validation): 20

Docs & run instructions (this README): 10

Tests (unit for text/regex; integration with MariaDB): 10

Git hygiene (public repo, meaningful commits): 10

Extras (any of: JWT, batch/scan, basic UI, caching): 10

DTO Signatures (Reference)
public record CreateKeywordRequest(
    @NotBlank @Size(max = 255) String keywordPhrase,
    Boolean keywordIsActive
) {}

public record KeywordResponse(
    Long keywordId,
    String keywordPhrase,
    boolean keywordIsActive,
    Instant createdAt,
    Instant updatedAt
) {}

public record KeywordSearchResponse(
    int page, int pageSize, long totalItems, int totalPages,
    List<KeywordResponse> items
) {}

public record ScanResult(
    String extractedCandidateName,
    @jakarta.validation.constraints.Email String extractedCandidateEmail,
    String extractedPhoneNumber,
    List<MatchedKeyword> matchedKeywords,
    double matchScore,
    int totalActiveKeywordCount,
    String processedAtIso
) {
  public record MatchedKeyword(String keywordPhrase, int occurrenceCount) {}
}

Minimal Service Outline (Pseudocode)
// KeywordService
List<Keyword> getActiveKeywordsCached(); // also set X-Cache: HIT|MISS
Keyword createKeyword(CreateKeywordRequest req);
Keyword updateKeyword(Long id, UpdateKeywordRequest req);
void toggleActivation(Long id, boolean isActive);
Page<Keyword> searchKeywords(KeywordSearchQuery query);

// ScanService
ScanResult scan(MultipartFile file, String rawText);
ScanResult rescanByEmail(String email);

// TextAnalysisService
String normalizeText(String raw);
Optional<String> extractEmail(String text);
Optional<String> extractPhone(String text);
Optional<String> extractName(String originalText, String fallbackFromEmail);
Map<String,Integer> countKeywordOccurrences(String normalizedText, List<String> activeKeywordPhrases);

Submission Instructions

Create a public GitHub repository (recommended name: cv-scanner-api-springboot).

Implement the solution following this spec.

Include:

This README.md,

A Postman collection in /postman,

src/main/resources/db/migration/V1__baseline.sql (DDL above),

Minimal sample files in /samples (one PDF or TXT is enough).

Verify locally:

./mvnw spring-boot:run boots the server,

GET /api/health returns 200.

Share the repo URL and (optional) a brief note on trade‑offs.
