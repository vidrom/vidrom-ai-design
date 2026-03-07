# Database Schema

## Infrastructure

- **Database**: PostgreSQL 16
- **Hosting**: Amazon RDS (managed)
- **Instance**: db.t3.micro (free tier eligible for 12 months)
- **Storage**: 20GB gp3
- **Backups**: Automated, 7-day retention
- **Region**: Same as application (via CDK)

---

## Tables

### buildings

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique identifier |
| name | VARCHAR(255) | NOT NULL | Building name |
| address | VARCHAR(500) | NOT NULL | Building address |
| door_code | VARCHAR(20) | NOT NULL | Access code for the building door |
| door_opening_time | INTEGER | NOT NULL, DEFAULT 5 | Duration in seconds the door stays unlocked |
| no_answer_timeout | INTEGER | NOT NULL, DEFAULT 30 | Seconds before an unanswered call auto-ends |
| language | VARCHAR(10) | NOT NULL, DEFAULT 'en' | Building language preference |
| volume | INTEGER | NOT NULL, DEFAULT 50, CHECK (volume BETWEEN 0 AND 100) | Volume level (0-100) |
| brightness | INTEGER | NOT NULL, DEFAULT 50, CHECK (brightness BETWEEN 0 AND 100) | Intercom screen brightness (0-100) |
| dark_mode | BOOLEAN | NOT NULL, DEFAULT false | Dark mode preference |
| sleep_mode | BOOLEAN | NOT NULL, DEFAULT false | Sleep mode preference |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record last update timestamp |

**Indexes:**
- `idx_buildings_name` on `name`

---

### apartments

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique identifier |
| building_id | UUID | NOT NULL, REFERENCES buildings(id) ON DELETE CASCADE | Parent building |
| number | VARCHAR(20) | NOT NULL | Apartment number |
| name | VARCHAR(255) | | Apartment display name |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record last update timestamp |

**Indexes:**
- `idx_apartments_building_id` on `building_id`
- `uq_apartments_building_number` UNIQUE on `(building_id, number)`

---

### users

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique identifier |
| email | VARCHAR(255) | NOT NULL, UNIQUE | User email |
| name | VARCHAR(255) | NOT NULL | User full name |
| role | VARCHAR(20) | NOT NULL, CHECK (role IN ('admin', 'manager', 'resident')) | User role |
| push_notification_token | VARCHAR(500) | | FCM/APNs push token (for residents) |
| authentication_method | VARCHAR(20) | CHECK (authentication_method IN ('google', 'apple')) | Auth provider |
| sleep_mode | BOOLEAN | NOT NULL, DEFAULT false | Suppress incoming call notifications |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record last update timestamp |

**Indexes:**
- `idx_users_email` on `email`
- `idx_users_role` on `role`

---

### intercoms

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique identifier |
| building_id | UUID | NOT NULL, REFERENCES buildings(id) ON DELETE CASCADE | Parent building |
| name | VARCHAR(255) | NOT NULL | Intercom display name |
| gate_id | VARCHAR(50) | | Physical gate identifier |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'disconnected', CHECK (status IN ('connected', 'disconnected')) | Connection status |
| is_door_open | BOOLEAN | NOT NULL, DEFAULT false | Runtime door state |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record last update timestamp |

**Indexes:**
- `idx_intercoms_building_id` on `building_id`
- `idx_intercoms_status` on `status`

---

### calls

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique identifier |
| building_id | UUID | NOT NULL, REFERENCES buildings(id) | Parent building |
| apartment_id | UUID | NOT NULL, REFERENCES apartments(id) | Target apartment |
| intercom_id | UUID | NOT NULL, REFERENCES intercoms(id) | Initiating intercom |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'calling', CHECK (status IN ('calling', 'accepted', 'ended', 'rejected', 'unanswered')) | Call status |
| started_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Call start timestamp |
| ended_at | TIMESTAMPTZ | | Call end timestamp |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record last update timestamp |

**Indexes:**
- `idx_calls_building_id` on `building_id`
- `idx_calls_apartment_id` on `apartment_id`
- `idx_calls_intercom_id` on `intercom_id`
- `idx_calls_status` on `status`
- `idx_calls_active` on `(apartment_id)` WHERE `status IN ('calling', 'accepted')` *(partial index for enforcing one active call per apartment)*

---

### notifications

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique identifier |
| building_id | UUID | NOT NULL, REFERENCES buildings(id) ON DELETE CASCADE | Scoped to building |
| text | TEXT | NOT NULL | Notification content |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record creation timestamp |

**Indexes:**
- `idx_notifications_building_id` on `building_id`
- `idx_notifications_created_at` on `created_at DESC`

---

### audit_logs

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique identifier |
| event_type | VARCHAR(30) | NOT NULL, CHECK (event_type IN ('call-initiated', 'call-accepted', 'call-rejected', 'call-ended', 'call-unanswered', 'door-open', 'access-code-success', 'access-code-failure', 'watch-camera-started')) | Event type |
| building_id | UUID | REFERENCES buildings(id) | Related building |
| apartment_id | UUID | REFERENCES apartments(id) | Related apartment (when relevant) |
| user_id | UUID | REFERENCES users(id) | Related user (when relevant) |
| intercom_id | UUID | REFERENCES intercoms(id) | Related intercom (when relevant) |
| call_id | UUID | REFERENCES calls(id) | Related call (for call events) |
| description | TEXT | | Free-text log message |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Event timestamp |

**Indexes:**
- `idx_audit_logs_event_type` on `event_type`
- `idx_audit_logs_building_id` on `building_id`
- `idx_audit_logs_created_at` on `created_at DESC`
- `idx_audit_logs_call_id` on `call_id` WHERE `call_id IS NOT NULL`

---

### global_settings

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| key | VARCHAR(100) | PRIMARY KEY | Setting name |
| value | VARCHAR(255) | NOT NULL | Setting value |
| description | TEXT | | Human-readable description |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Last update timestamp |

**Seed data:**

| key | value | description |
|-----|-------|-------------|
| max_call_duration | 60 | Maximum duration allowed for a call (seconds) |
| no_answer_timeout | 30 | Default duration before an unanswered call auto-ends (seconds) |
| call_polling_interval | 3 | How often to poll for call status (seconds) |
| intercom_polling_interval | 2 | How often to poll intercom status (seconds) |

---

## Junction Tables

### apartment_residents

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| apartment_id | UUID | NOT NULL, REFERENCES apartments(id) ON DELETE CASCADE | Apartment |
| user_id | UUID | NOT NULL, REFERENCES users(id) ON DELETE CASCADE | Resident user |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record creation timestamp |

**Constraints:**
- PRIMARY KEY on `(apartment_id, user_id)`

**Indexes:**
- `idx_apartment_residents_user_id` on `user_id`

---

### building_managers

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| building_id | UUID | NOT NULL, REFERENCES buildings(id) ON DELETE CASCADE | Building |
| user_id | UUID | NOT NULL, REFERENCES users(id) ON DELETE CASCADE | Manager user |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Record creation timestamp |

**Constraints:**
- PRIMARY KEY on `(building_id, user_id)`

**Indexes:**
- `idx_building_managers_user_id` on `user_id`

---

## Entity Relationship Summary

```
buildings 1──┬──* apartments
             ├──* intercoms
             ├──* notifications
             └──* building_managers ──* users (managers)

apartments *──* apartment_residents ──* users (residents)

calls *──1 buildings
calls *──1 apartments
calls *──1 intercoms

audit_logs *──? buildings
audit_logs *──? apartments
audit_logs *──? users
audit_logs *──? intercoms
audit_logs *──? calls
```

## Auto-update Trigger

All tables with `updated_at` should use a trigger to auto-set the value on UPDATE:

```sql
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to each table with updated_at:
CREATE TRIGGER trg_buildings_updated_at BEFORE UPDATE ON buildings FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_apartments_updated_at BEFORE UPDATE ON apartments FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_users_updated_at BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_intercoms_updated_at BEFORE UPDATE ON intercoms FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_calls_updated_at BEFORE UPDATE ON calls FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_global_settings_updated_at BEFORE UPDATE ON global_settings FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```
