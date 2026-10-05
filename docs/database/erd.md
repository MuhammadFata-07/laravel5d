# Database Design: Entity Relationship Diagram

Sistem E-Learning Adaptif dengan Tracking Progress Belajar.

Gambar: [`erd.png`](./erd.png)

## 1. Entity Relationship Diagram

```mermaid
erDiagram
    USERS ||--o| PROFILES : "has one"
    USERS ||--o{ COURSES : "teaches"
    CATEGORIES ||--o{ COURSES : "groups"
    COURSES ||--o{ MODULES : "contains"
    USERS ||--o{ ENROLLMENTS : "enrolls"
    COURSES ||--o{ ENROLLMENTS : "has students"
    ENROLLMENTS ||--o| CERTIFICATES : "issues"
    USERS ||--o{ MODULE_USER : "completes"
    MODULES ||--o{ MODULE_USER : "completed by"
    MODULES ||--o{ QUIZZES : "has"
    MODULES |o--o{ QUIZZES : "remedial for"
    QUIZZES ||--o{ QUESTIONS : "has"
    USERS ||--o{ QUIZ_ATTEMPTS : "takes"
    QUIZZES ||--o{ QUIZ_ATTEMPTS : "attempted in"
    COURSES ||--o{ DISCUSSIONS : "has"
    USERS ||--o{ DISCUSSIONS : "posts"
    DISCUSSIONS |o--o{ DISCUSSIONS : "replies to"
    USERS ||--o{ BADGE_USER : "earns"
    BADGES ||--o{ BADGE_USER : "awarded via"

    USERS {
        bigint id PK
        string name
        string email UK
        string password
        string role "student / instructor"
    }
    PROFILES {
        bigint id PK
        bigint user_id FK "unique"
        text bio
        string avatar
        int total_points
    }
    CATEGORIES {
        bigint id PK
        string name
        string slug UK
    }
    COURSES {
        bigint id PK
        bigint instructor_id FK "users.id"
        bigint category_id FK
        string title
        text description
        boolean is_published
    }
    MODULES {
        bigint id PK
        bigint course_id FK
        string title
        string content_type "text / video / pdf"
        string content_url
        int sort_order
    }
    ENROLLMENTS {
        bigint id PK
        bigint user_id FK
        bigint course_id FK
        tinyint progress_percent
        string status "active / completed"
        timestamp enrolled_at
        timestamp completed_at "nullable"
    }
    MODULE_USER {
        bigint user_id FK
        bigint module_id FK
        timestamp completed_at
    }
    QUIZZES {
        bigint id PK
        bigint module_id FK
        bigint remedial_module_id FK "nullable"
        string title
        int passing_score
    }
    QUESTIONS {
        bigint id PK
        bigint quiz_id FK
        text question_text
        json options
        string correct_answer
        int weight
    }
    QUIZ_ATTEMPTS {
        bigint id PK
        bigint user_id FK
        bigint quiz_id FK
        int score
        boolean is_passed
        timestamp taken_at
    }
    DISCUSSIONS {
        bigint id PK
        bigint course_id FK
        bigint user_id FK
        bigint parent_id FK "nullable"
        text body
    }
    CERTIFICATES {
        bigint id PK
        bigint enrollment_id FK "unique"
        string certificate_code UK
        timestamp issued_at
    }
    BADGES {
        bigint id PK
        string name
        text description
        int points_required
    }
    BADGE_USER {
        bigint user_id FK
        bigint badge_id FK
        timestamp earned_at
    }
```

## 2. Ringkasan Relasi

| Jenis | Relasi | Eloquent |
| --- | --- | --- |
| One-to-One | `User` ↔ `Profile` | `hasOne` / `belongsTo` |
| One-to-One | `Enrollment` ↔ `Certificate` | `hasOne` / `belongsTo` |
| One-to-Many | `User` (pengajar) → `Course` | `hasMany` / `belongsTo` |
| One-to-Many | `Category` → `Course` | `hasMany` / `belongsTo` |
| One-to-Many | `Course` → `Module` | `hasMany` / `belongsTo` |
| One-to-Many | `Module` → `Quiz` | `hasMany` / `belongsTo` |
| One-to-Many | `Quiz` → `Question` | `hasMany` / `belongsTo` |
| One-to-Many | `User` / `Quiz` → `QuizAttempt` | `hasMany` / `belongsTo` |
| One-to-Many | `Course` / `User` → `Discussion` | `hasMany` / `belongsTo` |
| Self-referencing | `Discussion` → `Discussion` (balasan lewat `parent_id`) | `hasMany` / `belongsTo` |
| Many-to-Many + pivot data | `User` ↔ `Course` (pivot `enrollments`: `progress_percent`, `status`) | `belongsToMany` + `withPivot` |
| Many-to-Many + pivot data | `User` ↔ `Module` (pivot `module_user`: `completed_at`) | `belongsToMany` + `withPivot` |
| Many-to-Many + pivot data | `User` ↔ `Badge` (pivot `badge_user`: `earned_at`) | `belongsToMany` + `withPivot` |
| Has-Many-Through | `User` (pengajar) → `Module` melalui `Course` | `hasManyThrough` |

## 3. Catatan Desain

- **Adaptive learning (rule-based):** jika `quiz_attempts.score` < `quizzes.passing_score`, sistem merekomendasikan modul di `quizzes.remedial_module_id`.
- **Progress belajar:** `module_user` mencatat modul yang selesai; `enrollments.progress_percent` dihitung dari jumlah modul selesai dibanding total modul kursus.
- **Gamifikasi:** `profiles.total_points` bertambah dari aktivitas; `badges.points_required` menentukan kapan badge diberikan lewat `badge_user`.
- **Sertifikat:** satu `certificate` dibuat saat `enrollments.status` menjadi `completed`; PDF dibuat saat implementasi.
- **Unique constraint:** `users.email`, `profiles.user_id`, `categories.slug`, `certificates.enrollment_id`, `certificates.certificate_code`, dan pasangan (`enrollments.user_id`, `enrollments.course_id`).
- **Pilihan jawaban soal** disimpan sebagai kolom JSON `questions.options` agar jumlah tabel tetap ringkas.
