# HIIT App - SQLite Database Schema

## Overview
This document defines the SQLite database schema for the HIIT Training App.
The schema is designed to eliminate duplicates, ensure data integrity, and support efficient queries.

## Design Principles
1. **Normalized Structure**: Proper 3NF normalization to eliminate redundancy
2. **Referential Integrity**: Foreign keys with cascade rules
3. **Duplicate Prevention**: UNIQUE constraints on business keys
4. **Audit Trail**: Created/updated timestamps on all tables
5. **Type Safety**: CHECK constraints for enums
6. **Performance**: Strategic indexes on frequently queried columns

## Tables

### 1. `workouts`
Stores the exercise library (individual exercises/movements)

```sql
CREATE TABLE workouts (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL UNIQUE COLLATE NOCASE,  -- Case-insensitive unique constraint
  description TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_workouts_name ON workouts(name COLLATE NOCASE);
CREATE INDEX idx_workouts_created ON workouts(created_at DESC);
```

**Constraints:**
- `name` must be unique (case-insensitive) - prevents duplicate exercises
- `id` is UUID-style string for cross-platform compatibility

---

### 2. `timer_configs`
Stores timer configurations (both Circuit Training and Straight-Set HIIT)

```sql
CREATE TABLE timer_configs (
  id TEXT PRIMARY KEY,
  type TEXT NOT NULL CHECK(type IN ('Circuit Training', 'Straight-Set HIIT')),
  name TEXT NOT NULL UNIQUE COLLATE NOCASE,  -- Case-insensitive unique constraint
  num_exercises INTEGER NOT NULL CHECK(num_exercises > 0),

  -- Common fields
  work_duration INTEGER NOT NULL CHECK(work_duration > 0),
  warm_up_duration INTEGER NOT NULL CHECK(warm_up_duration >= 0),
  cooldown_duration INTEGER NOT NULL CHECK(cooldown_duration >= 0),
  pre_start_countdown INTEGER NOT NULL CHECK(pre_start_countdown >= 0),

  -- Circuit Training specific (NULL for Straight-Set HIIT)
  num_rounds INTEGER CHECK(num_rounds IS NULL OR num_rounds > 0),
  rest_between_exercises INTEGER CHECK(rest_between_exercises IS NULL OR rest_between_exercises >= 0),
  rest_between_rounds INTEGER CHECK(rest_between_rounds IS NULL OR rest_between_rounds >= 0),

  -- Straight-Set HIIT specific (NULL for Circuit Training)
  sets_per_exercise INTEGER CHECK(sets_per_exercise IS NULL OR sets_per_exercise > 0),
  rest_duration INTEGER CHECK(rest_duration IS NULL OR rest_duration >= 0),

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

  -- Validation: Circuit Training must have circuit-specific fields
  CHECK(
    (type = 'Circuit Training' AND num_rounds IS NOT NULL AND rest_between_exercises IS NOT NULL AND rest_between_rounds IS NOT NULL)
    OR
    (type = 'Straight-Set HIIT' AND sets_per_exercise IS NOT NULL AND rest_duration IS NOT NULL)
  )
);

CREATE INDEX idx_timer_configs_type ON timer_configs(type);
CREATE INDEX idx_timer_configs_created ON timer_configs(created_at DESC);
```

**Constraints:**
- `name` must be unique (case-insensitive) - prevents duplicate config names
- `type` must be one of the two valid types
- Conditional validation ensures type-specific fields are present
- All duration/count fields must be non-negative

---

### 3. `config_exercises`
Junction table linking timer configs to exercises (many-to-many with ordering)

```sql
CREATE TABLE config_exercises (
  id TEXT PRIMARY KEY,
  config_id TEXT NOT NULL,
  workout_id TEXT NOT NULL,
  position INTEGER NOT NULL CHECK(position >= 0),

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (config_id) REFERENCES timer_configs(id) ON DELETE CASCADE,
  FOREIGN KEY (workout_id) REFERENCES workouts(id) ON DELETE RESTRICT,

  UNIQUE(config_id, workout_id),  -- Prevent same exercise twice in one config
  UNIQUE(config_id, position)      -- Prevent position conflicts
);

CREATE INDEX idx_config_exercises_config ON config_exercises(config_id, position);
CREATE INDEX idx_config_exercises_workout ON config_exercises(workout_id);
```

**Constraints:**
- `CASCADE` on config deletion - remove all exercise assignments
- `RESTRICT` on workout deletion - prevent deleting workouts used in configs
- `position` ensures proper ordering of exercises
- Prevents duplicate exercises in same configuration

---

### 4. `workout_sessions`
Tracks completed/attempted workout sessions (history)

```sql
CREATE TABLE workout_sessions (
  id TEXT PRIMARY KEY,
  config_id TEXT,  -- Nullable: config might be deleted
  config_name TEXT NOT NULL,  -- Denormalized for history preservation
  config_type TEXT NOT NULL,

  start_time DATETIME NOT NULL,
  end_time DATETIME,
  completed BOOLEAN NOT NULL DEFAULT 0,
  total_duration INTEGER,  -- In seconds

  notes TEXT,  -- Optional user notes

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (config_id) REFERENCES timer_configs(id) ON DELETE SET NULL
);

CREATE INDEX idx_sessions_start_time ON workout_sessions(start_time DESC);
CREATE INDEX idx_sessions_config ON workout_sessions(config_id);
CREATE INDEX idx_sessions_completed ON workout_sessions(completed, start_time DESC);
```

**Constraints:**
- `SET NULL` on config deletion - preserve session history even if config deleted
- Denormalized `config_name` and `config_type` to maintain historical data
- Indexes optimized for "recent sessions" and "sessions by config" queries

---

## Database Initialization

```sql
-- Enable WAL mode for better concurrency
PRAGMA journal_mode = WAL;

-- Enable foreign key constraints
PRAGMA foreign_keys = ON;

-- Set synchronous mode for better performance
PRAGMA synchronous = NORMAL;
```

## Migration Strategy

### From AsyncStorage to SQLite

**Version 1.0.0 → 2.0.0**

1. Read existing AsyncStorage data
2. Insert into SQLite with validation
3. Handle duplicates gracefully (keep first, log others)
4. Preserve creation timestamps where available
5. Clear AsyncStorage after successful migration
6. Mark migration as complete in metadata

**Rollback Plan:**
- Keep AsyncStorage data until user confirms migration success
- Provide export/import functionality
- Version all schema changes

## Query Patterns

### Common Queries

```sql
-- Get all workouts alphabetically
SELECT * FROM workouts ORDER BY name COLLATE NOCASE;

-- Get all timer configs with exercise count
SELECT
  tc.*,
  COUNT(ce.id) as exercise_count
FROM timer_configs tc
LEFT JOIN config_exercises ce ON tc.id = ce.config_id
GROUP BY tc.id
ORDER BY tc.created_at DESC;

-- Get full timer config with exercises
SELECT
  tc.*,
  json_group_array(
    json_object('id', w.id, 'name', w.name, 'position', ce.position)
  ) as exercises
FROM timer_configs tc
LEFT JOIN config_exercises ce ON tc.id = ce.config_id
LEFT JOIN workouts w ON ce.workout_id = w.id
WHERE tc.id = ?
GROUP BY tc.id;

-- Get workout session history
SELECT * FROM workout_sessions
WHERE completed = 1
ORDER BY start_time DESC
LIMIT 20;

-- Get statistics
SELECT
  config_name,
  COUNT(*) as times_completed,
  AVG(total_duration) as avg_duration,
  MAX(start_time) as last_completed
FROM workout_sessions
WHERE completed = 1
GROUP BY config_name
ORDER BY times_completed DESC;
```

## Data Integrity Rules

1. **Workout Deletion**:
   - RESTRICTED if used in any timer config
   - User must remove from configs first

2. **Timer Config Deletion**:
   - CASCADE delete all exercise assignments
   - SET NULL in session history (preserve history)

3. **Duplicate Prevention**:
   - Enforced at database level via UNIQUE constraints
   - Application layer provides user-friendly error messages

4. **Type Safety**:
   - CHECK constraints ensure valid enum values
   - NOT NULL constraints prevent incomplete data

## Future Enhancements

Planned schema additions:

1. **User Profiles** (multi-user support)
2. **Exercise Categories/Tags** (filtering)
3. **Custom Exercise Fields** (sets, reps, weights)
4. **Workout Programs** (scheduled training plans)
5. **Performance Metrics** (heart rate, calories, etc.)

---

**Schema Version**: 1.0.0
**Last Updated**: 2025-12-08
**Author**: Senior Development Team
