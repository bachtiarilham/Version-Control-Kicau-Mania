# ERD PostgreSQL Kicau Mania

Diagram ini mengikuti tabel dan foreign key dari dump PostgreSQL untuk `cloud-app`. Nama seperti `competition_rounds` pada diagram mewakili `competition.rounds`. [Dump sumber](../../db-dump/full_db.dump)

## Tabel dan relasi

```mermaid
erDiagram
    asset_bird_owners {
        uuid id PK
        uuid bird_id FK
        uuid user_id FK
        boolean is_primary_contact
        timestamptz created_at
    }
    asset_birds {
        uuid id PK
        varchar ring_number UK
        varchar bird_type
        varchar name
        timestamptz created_at
    }
    asset_devices {
        uuid id PK
        varchar type
        varchar ip_address
        text rtsp_url
        uuid lapak_id FK "opsional"
        timestamptz created_at
    }
    asset_lapaks {
        uuid id PK
        integer number UK
        text description
        timestamptz created_at
    }
    competition_events {
        uuid id PK
        varchar title
        date date
        text location
        varchar status
        timestamptz created_at
    }
    competition_round_participants {
        uuid id PK
        uuid round_id FK
        uuid lapak_id FK
        uuid bird_id FK
        uuid registered_by_user_id FK
        varchar registered_as_role
        uuid declared_owner_user_id FK "opsional"
        varchar declared_owner_phone
        varchar owner_consent_status
        boolean legal_declaration_agreed
        varchar target_recipient_phone
        varchar status
        timestamptz created_at
    }
    competition_rounds {
        uuid id PK
        uuid event_id FK
        integer round_number
        varchar bird_type
        timestamptz start_time
        timestamptz end_time
        varchar status
        timestamptz created_at
    }
    iam_roles {
        uuid id PK
        varchar name UK
        timestamptz created_at
    }
    iam_user_auth_providers {
        uuid id PK
        uuid user_id FK
        varchar provider
        varchar provider_user_id
        timestamptz created_at
    }
    iam_user_sessions {
        uuid id PK
        uuid user_id FK
        varchar refresh_token_hash
        text user_agent
        varchar ip_address
        timestamptz expires_at
        timestamptz revoked_at
        timestamptz created_at
    }
    iam_users {
        uuid id PK
        uuid role_id FK
        varchar name
        varchar phone UK
        varchar email UK
        varchar password_hash
        varchar nik_ktp UK
        boolean is_verified
        boolean is_active
        timestamptz created_at
        timestamptz updated_at
    }
    judging_round_scores {
        uuid id PK
        uuid round_participant_id FK
        uuid judge_user_id FK
        uuid criteria_id FK
        numeric score_value
        timestamptz created_at
    }
    judging_score_criteria {
        uuid id PK
        varchar criteria_name UK
        timestamptz created_at
    }
    media_round_recordings {
        uuid id PK
        uuid round_participant_id FK, UK
        text video_url
        text audio_url
        numeric max_peak_dbfs
        numeric avg_rms_dbfs
        integer clip_incident_count
        integer singing_duration_seconds
        boolean is_sent
        timestamptz sent_at
        varchar sent_to_phone
        timestamptz created_at
    }
    member_plans {
        uuid id PK
        varchar name
        integer quota_birds
        integer max_events
        numeric price
        timestamptz created_at
    }
    member_user_subscriptions {
        uuid id PK
        uuid user_id FK
        uuid plan_id FK
        timestamptz start_date
        timestamptz end_date
        varchar status
        timestamptz created_at
    }
    asset_birds ||..o{ asset_bird_owners : "bird_id"
    iam_users ||..o{ asset_bird_owners : "user_id"
    asset_lapaks |o..o{ asset_devices : "lapak_id"
    asset_birds ||..o{ competition_round_participants : "bird_id"
    iam_users |o..o{ competition_round_participants : "declared_owner_user_id"
    asset_lapaks ||..o{ competition_round_participants : "lapak_id"
    iam_users ||..o{ competition_round_participants : "registered_by_user_id"
    competition_rounds ||..o{ competition_round_participants : "round_id"
    competition_events ||..o{ competition_rounds : "event_id"
    iam_users ||..o{ iam_user_auth_providers : "user_id"
    iam_users ||..o{ iam_user_sessions : "user_id"
    iam_roles ||..o{ iam_users : "role_id"
    judging_score_criteria ||..o{ judging_round_scores : "criteria_id"
    iam_users ||..o{ judging_round_scores : "judge_user_id"
    competition_round_participants ||..o{ judging_round_scores : "round_participant_id"
    competition_round_participants ||..o| media_round_recordings : "round_participant_id"
    member_plans ||..o{ member_user_subscriptions : "plan_id"
    iam_users ||..o{ member_user_subscriptions : "user_id"
```

Mulai dari `competition.events → competition.rounds → competition.round_participants`. Dari peserta ronde, ikuti relasi ke burung, lapak, skor, dan rekaman. Setiap peserta dapat memiliki banyak baris skor dan maksimal satu baris rekaman. Lapak pada perangkat dan pemilik yang dideklarasikan pada pendaftaran bersifat opsional.

## Keunikan gabungan

| Tabel | Kolom yang harus unik sebagai pasangan atau kombinasi |
| --- | --- |
| `asset.bird_owners` | `bird_id`, `user_id` |
| `competition.round_participants` | `round_id`, `lapak_id` |
| `competition.rounds` | `event_id`, `round_number` |
| `iam.user_auth_providers` | `provider`, `provider_user_id` |
| `judging.round_scores` | `round_participant_id`, `judge_user_id`, `criteria_id` |

## Aturan penghapusan

| Tabel anak | Foreign key | Aksi saat induk dihapus |
| --- | --- | --- |
| `asset.bird_owners` | `bird_id` → `asset.birds` | `CASCADE` |
| `asset.bird_owners` | `user_id` → `iam.users` | `CASCADE` |
| `asset.devices` | `lapak_id` → `asset.lapaks` | `SET NULL` |
| `competition.round_participants` | `bird_id` → `asset.birds` | `RESTRICT` |
| `competition.round_participants` | `declared_owner_user_id` → `iam.users` | `SET NULL` |
| `competition.round_participants` | `lapak_id` → `asset.lapaks` | `RESTRICT` |
| `competition.round_participants` | `registered_by_user_id` → `iam.users` | `RESTRICT` |
| `competition.round_participants` | `round_id` → `competition.rounds` | `CASCADE` |
| `competition.rounds` | `event_id` → `competition.events` | `CASCADE` |
| `iam.user_auth_providers` | `user_id` → `iam.users` | `CASCADE` |
| `iam.user_sessions` | `user_id` → `iam.users` | `CASCADE` |
| `iam.users` | `role_id` → `iam.roles` | `RESTRICT` |
| `judging.round_scores` | `criteria_id` → `judging.score_criteria` | `RESTRICT` |
| `judging.round_scores` | `judge_user_id` → `iam.users` | `RESTRICT` |
| `judging.round_scores` | `round_participant_id` → `competition.round_participants` | `CASCADE` |
| `media.round_recordings` | `round_participant_id` → `competition.round_participants` | `CASCADE` |
| `member.user_subscriptions` | `plan_id` → `member.plans` | `RESTRICT` |
| `member.user_subscriptions` | `user_id` → `iam.users` | `CASCADE` |

## Jalur baca aplikasi

`cloud_read` berisi 16 view daftar/detail dan satu materialized view rekap administratif. View membaca tabel pada ERD. Rekap menggabungkan peserta ronde, ronde, dan skor, dengan satu baris per peserta ronde. [Panduan SQL](README.md)

```mermaid
flowchart LR
    Tables["Tabel PostgreSQL"] --> Views["cloud_read.v_*"]
    Tables --> Summary["cloud_read.mv_round_score_summary"]
    Views --> CRUD["Daftar dan detail"]
    Summary --> Admin["Rekap administratif"]
```

Cakupan diagram adalah PostgreSQL untuk fitur bisnis cloud. SQLite arena dan journal pertukaran cloud dijelaskan di dokumentasi masing-masing. [Offline-first arena](../../local-app/OFFLINE-FIRST.md) · [Cloud exchange](../README.md)
