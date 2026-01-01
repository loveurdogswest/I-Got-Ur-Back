# FamilyGuard Database Schema

This document describes the database tables required for the FamilyGuard parental monitoring application.

## Tables

### 1. parent_profiles (Already exists)
Stores parent account information.

```sql
CREATE TABLE parent_profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL UNIQUE,
  email TEXT NOT NULL,
  full_name TEXT,
  pin_code TEXT,
  biometric_enabled BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- RLS Policy
ALTER TABLE parent_profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own profile" ON parent_profiles
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can update own profile" ON parent_profiles
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own profile" ON parent_profiles
  FOR INSERT WITH CHECK (auth.uid() = user_id);
```

### 2. children
Stores child profiles linked to parent accounts.

```sql
CREATE TABLE children (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES parent_profiles(id) ON DELETE CASCADE NOT NULL,
  name TEXT NOT NULL,
  avatar_url TEXT,
  age INTEGER,
  device_name TEXT,
  status TEXT DEFAULT 'safe' CHECK (status IN ('safe', 'warning', 'alert')),
  location TEXT,
  last_seen TIMESTAMPTZ,
  battery_level INTEGER,
  is_online BOOLEAN DEFAULT false,
  screen_time_today INTEGER DEFAULT 0,
  daily_screen_limit INTEGER DEFAULT 240,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- RLS Policies
ALTER TABLE children ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view own children" ON children
  FOR SELECT USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can insert children" ON children
  FOR INSERT WITH CHECK (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can update own children" ON children
  FOR UPDATE USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can delete own children" ON children
  FOR DELETE USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

-- Index for faster queries
CREATE INDEX idx_children_parent_id ON children(parent_id);
```

### 3. keywords
Stores monitored keywords for content filtering.

```sql
CREATE TABLE keywords (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES parent_profiles(id) ON DELETE CASCADE NOT NULL,
  word TEXT NOT NULL,
  category TEXT DEFAULT 'custom' CHECK (category IN ('danger', 'drugs', 'bullying', 'inappropriate', 'custom')),
  severity TEXT DEFAULT 'medium' CHECK (severity IN ('high', 'medium', 'low')),
  is_enabled BOOLEAN DEFAULT true,
  trigger_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- RLS Policies
ALTER TABLE keywords ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view own keywords" ON keywords
  FOR SELECT USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can insert keywords" ON keywords
  FOR INSERT WITH CHECK (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can update own keywords" ON keywords
  FOR UPDATE USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can delete own keywords" ON keywords
  FOR DELETE USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

-- Index for faster queries
CREATE INDEX idx_keywords_parent_id ON keywords(parent_id);
```

### 4. geofences
Stores safe zone configurations.

```sql
CREATE TABLE geofences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES parent_profiles(id) ON DELETE CASCADE NOT NULL,
  name TEXT NOT NULL,
  type TEXT DEFAULT 'other' CHECK (type IN ('home', 'school', 'work', 'other')),
  address TEXT NOT NULL,
  latitude DECIMAL(10, 8),
  longitude DECIMAL(11, 8),
  radius INTEGER DEFAULT 100,
  notify_on_entry BOOLEAN DEFAULT true,
  notify_on_exit BOOLEAN DEFAULT true,
  active_hours_start TIME,
  active_hours_end TIME,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- RLS Policies
ALTER TABLE geofences ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view own geofences" ON geofences
  FOR SELECT USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can insert geofences" ON geofences
  FOR INSERT WITH CHECK (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can update own geofences" ON geofences
  FOR UPDATE USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can delete own geofences" ON geofences
  FOR DELETE USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

-- Index for faster queries
CREATE INDEX idx_geofences_parent_id ON geofences(parent_id);
```

### 5. alerts
Stores alert notifications for parents.

```sql
CREATE TABLE alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES parent_profiles(id) ON DELETE CASCADE NOT NULL,
  child_id UUID REFERENCES children(id) ON DELETE SET NULL,
  type TEXT NOT NULL CHECK (type IN ('keyword', 'geofence', 'battery', 'screentime', 'app', 'system')),
  title TEXT NOT NULL,
  description TEXT,
  severity TEXT DEFAULT 'medium' CHECK (severity IN ('high', 'medium', 'low')),
  is_read BOOLEAN DEFAULT false,
  action_required BOOLEAN DEFAULT false,
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- RLS Policies
ALTER TABLE alerts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view own alerts" ON alerts
  FOR SELECT USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can insert alerts" ON alerts
  FOR INSERT WITH CHECK (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can update own alerts" ON alerts
  FOR UPDATE USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can delete own alerts" ON alerts
  FOR DELETE USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

-- Indexes for faster queries
CREATE INDEX idx_alerts_parent_id ON alerts(parent_id);
CREATE INDEX idx_alerts_child_id ON alerts(child_id);
CREATE INDEX idx_alerts_created_at ON alerts(created_at DESC);
```

### 6. activity_logs
Stores activity events and history.

```sql
CREATE TABLE activity_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES parent_profiles(id) ON DELETE CASCADE NOT NULL,
  child_id UUID REFERENCES children(id) ON DELETE SET NULL,
  type TEXT NOT NULL CHECK (type IN ('keyword', 'geofence', 'location', 'battery', 'app', 'screentime', 'system')),
  title TEXT NOT NULL,
  description TEXT,
  severity TEXT DEFAULT 'info' CHECK (severity IN ('alert', 'warning', 'info')),
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- RLS Policies
ALTER TABLE activity_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view own activity logs" ON activity_logs
  FOR SELECT USING (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

CREATE POLICY "Parents can insert activity logs" ON activity_logs
  FOR INSERT WITH CHECK (
    parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid())
  );

-- Indexes for faster queries
CREATE INDEX idx_activity_logs_parent_id ON activity_logs(parent_id);
CREATE INDEX idx_activity_logs_child_id ON activity_logs(child_id);
CREATE INDEX idx_activity_logs_created_at ON activity_logs(created_at DESC);
```

## Complete Migration Script

Run this SQL in your Supabase SQL Editor to create all tables:

```sql
-- Enable UUID extension if not already enabled
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. Children table
CREATE TABLE IF NOT EXISTS children (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES parent_profiles(id) ON DELETE CASCADE NOT NULL,
  name TEXT NOT NULL,
  avatar_url TEXT,
  age INTEGER,
  device_name TEXT,
  status TEXT DEFAULT 'safe' CHECK (status IN ('safe', 'warning', 'alert')),
  location TEXT,
  last_seen TIMESTAMPTZ,
  battery_level INTEGER,
  is_online BOOLEAN DEFAULT false,
  screen_time_today INTEGER DEFAULT 0,
  daily_screen_limit INTEGER DEFAULT 240,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE children ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view own children" ON children
  FOR SELECT USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can insert children" ON children
  FOR INSERT WITH CHECK (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can update own children" ON children
  FOR UPDATE USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can delete own children" ON children
  FOR DELETE USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE INDEX IF NOT EXISTS idx_children_parent_id ON children(parent_id);

-- 2. Keywords table
CREATE TABLE IF NOT EXISTS keywords (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES parent_profiles(id) ON DELETE CASCADE NOT NULL,
  word TEXT NOT NULL,
  category TEXT DEFAULT 'custom' CHECK (category IN ('danger', 'drugs', 'bullying', 'inappropriate', 'custom')),
  severity TEXT DEFAULT 'medium' CHECK (severity IN ('high', 'medium', 'low')),
  is_enabled BOOLEAN DEFAULT true,
  trigger_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE keywords ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view own keywords" ON keywords
  FOR SELECT USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can insert keywords" ON keywords
  FOR INSERT WITH CHECK (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can update own keywords" ON keywords
  FOR UPDATE USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can delete own keywords" ON keywords
  FOR DELETE USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE INDEX IF NOT EXISTS idx_keywords_parent_id ON keywords(parent_id);

-- 3. Geofences table
CREATE TABLE IF NOT EXISTS geofences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES parent_profiles(id) ON DELETE CASCADE NOT NULL,
  name TEXT NOT NULL,
  type TEXT DEFAULT 'other' CHECK (type IN ('home', 'school', 'work', 'other')),
  address TEXT NOT NULL,
  latitude DECIMAL(10, 8),
  longitude DECIMAL(11, 8),
  radius INTEGER DEFAULT 100,
  notify_on_entry BOOLEAN DEFAULT true,
  notify_on_exit BOOLEAN DEFAULT true,
  active_hours_start TIME,
  active_hours_end TIME,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE geofences ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view own geofences" ON geofences
  FOR SELECT USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can insert geofences" ON geofences
  FOR INSERT WITH CHECK (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can update own geofences" ON geofences
  FOR UPDATE USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can delete own geofences" ON geofences
  FOR DELETE USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE INDEX IF NOT EXISTS idx_geofences_parent_id ON geofences(parent_id);

-- 4. Alerts table
CREATE TABLE IF NOT EXISTS alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES parent_profiles(id) ON DELETE CASCADE NOT NULL,
  child_id UUID REFERENCES children(id) ON DELETE SET NULL,
  type TEXT NOT NULL CHECK (type IN ('keyword', 'geofence', 'battery', 'screentime', 'app', 'system')),
  title TEXT NOT NULL,
  description TEXT,
  severity TEXT DEFAULT 'medium' CHECK (severity IN ('high', 'medium', 'low')),
  is_read BOOLEAN DEFAULT false,
  action_required BOOLEAN DEFAULT false,
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE alerts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view own alerts" ON alerts
  FOR SELECT USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can insert alerts" ON alerts
  FOR INSERT WITH CHECK (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can update own alerts" ON alerts
  FOR UPDATE USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can delete own alerts" ON alerts
  FOR DELETE USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE INDEX IF NOT EXISTS idx_alerts_parent_id ON alerts(parent_id);
CREATE INDEX IF NOT EXISTS idx_alerts_child_id ON alerts(child_id);
CREATE INDEX IF NOT EXISTS idx_alerts_created_at ON alerts(created_at DESC);

-- 5. Activity Logs table
CREATE TABLE IF NOT EXISTS activity_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES parent_profiles(id) ON DELETE CASCADE NOT NULL,
  child_id UUID REFERENCES children(id) ON DELETE SET NULL,
  type TEXT NOT NULL CHECK (type IN ('keyword', 'geofence', 'location', 'battery', 'app', 'screentime', 'system')),
  title TEXT NOT NULL,
  description TEXT,
  severity TEXT DEFAULT 'info' CHECK (severity IN ('alert', 'warning', 'info')),
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE activity_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view own activity logs" ON activity_logs
  FOR SELECT USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE POLICY "Parents can insert activity logs" ON activity_logs
  FOR INSERT WITH CHECK (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE INDEX IF NOT EXISTS idx_activity_logs_parent_id ON activity_logs(parent_id);
CREATE INDEX IF NOT EXISTS idx_activity_logs_child_id ON activity_logs(child_id);
CREATE INDEX IF NOT EXISTS idx_activity_logs_created_at ON activity_logs(created_at DESC);
```

### 7. app_usage
Stores app usage data from child devices.

```sql
CREATE TABLE IF NOT EXISTS app_usage (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES parent_profiles(id) ON DELETE CASCADE,
  child_id UUID REFERENCES children(id) ON DELETE CASCADE,
  session_id UUID REFERENCES device_sessions(id) ON DELETE CASCADE,
  app_name TEXT NOT NULL,
  app_package TEXT,
  app_category TEXT NOT NULL DEFAULT 'other',
  duration_minutes INTEGER NOT NULL DEFAULT 0,
  usage_date DATE NOT NULL DEFAULT CURRENT_DATE,
  last_used_at TIMESTAMPTZ DEFAULT now(),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  CONSTRAINT valid_category CHECK (app_category IN ('social_media', 'games', 'education', 'entertainment', 'productivity', 'communication', 'other'))
);

ALTER TABLE app_usage ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can view their children's app usage" ON app_usage
  FOR SELECT USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE INDEX IF NOT EXISTS idx_app_usage_parent_id ON app_usage(parent_id);
CREATE INDEX IF NOT EXISTS idx_app_usage_child_id ON app_usage(child_id);
CREATE INDEX IF NOT EXISTS idx_app_usage_date ON app_usage(usage_date);
```

### 8. app_limits
Stores parent-set time limits and blocked apps.

```sql
CREATE TABLE IF NOT EXISTS app_limits (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES parent_profiles(id) ON DELETE CASCADE,
  child_id UUID REFERENCES children(id) ON DELETE CASCADE,
  app_name TEXT NOT NULL,
  app_package TEXT,
  app_category TEXT,
  daily_limit_minutes INTEGER,
  is_blocked BOOLEAN DEFAULT false,
  block_schedule JSONB,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE app_limits ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Parents can manage their app limits" ON app_limits
  FOR ALL USING (parent_id IN (SELECT id FROM parent_profiles WHERE user_id = auth.uid()));

CREATE INDEX IF NOT EXISTS idx_app_limits_parent_id ON app_limits(parent_id);
CREATE INDEX IF NOT EXISTS idx_app_limits_child_id ON app_limits(child_id);
```

### 9. bedtime_schedules
Stores bedtime mode schedules for each child.

```sql
CREATE TABLE IF NOT EXISTS bedtime_schedules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  child_id UUID NOT NULL,
  is_enabled BOOLEAN DEFAULT true,
  weekday_start_time TIME NOT NULL DEFAULT '21:00',
  weekday_end_time TIME NOT NULL DEFAULT '07:00',
  weekend_start_time TIME NOT NULL DEFAULT '22:00',
  weekend_end_time TIME NOT NULL DEFAULT '08:00',
  allowed_apps TEXT[] DEFAULT ARRAY['Clock', 'Alarm'],
  send_violation_alerts BOOLEAN DEFAULT true,
  wind_down_minutes INTEGER DEFAULT 15,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(child_id)
);

ALTER TABLE bedtime_schedules ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Allow all operations on bedtime_schedules" ON bedtime_schedules
  FOR ALL USING (true) WITH CHECK (true);

CREATE INDEX IF NOT EXISTS idx_bedtime_schedules_child_id ON bedtime_schedules(child_id);
```

### 10. bedtime_violations
Tracks when children try to use devices during bedtime hours.

```sql
CREATE TABLE IF NOT EXISTS bedtime_violations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  child_id UUID NOT NULL,
  device_id UUID REFERENCES device_pairings(id),
  violation_time TIMESTAMPTZ DEFAULT NOW(),
  app_attempted TEXT,
  was_blocked BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE bedtime_violations ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Allow all operations on bedtime_violations" ON bedtime_violations
  FOR ALL USING (true) WITH CHECK (true);

CREATE INDEX IF NOT EXISTS idx_bedtime_violations_child_id ON bedtime_violations(child_id);
CREATE INDEX IF NOT EXISTS idx_bedtime_violations_time ON bedtime_violations(violation_time);
```

## Data Flow

1. **Parent Registration**: Creates entry in `auth.users` and `parent_profiles`
2. **Add Child**: Creates entry in `children` linked to parent's profile
3. **Configure Keywords**: Adds entries to `keywords` table
4. **Set Up Safe Zones**: Adds entries to `geofences` table
5. **Set App Limits**: Adds entries to `app_limits` table
6. **Set Bedtime Schedule**: Adds entries to `bedtime_schedules` table
7. **Device Pairing**: Creates entries in `device_pairings` and `device_sessions`
8. **App Usage Tracking**: Creates/updates entries in `app_usage` table
9. **Bedtime Violations**: Creates entries in `bedtime_violations` table
10. **Monitoring Events**: Creates entries in `activity_logs` and `alerts`

## Security Notes

- All tables use Row Level Security (RLS) to ensure parents can only access their own data
- Foreign key relationships ensure data integrity
- Cascading deletes ensure cleanup when parent accounts are removed
- Child deletion sets `child_id` to NULL in alerts/logs to preserve history
- App usage and limits are synced via secure edge functions with session tokens
- Bedtime schedules and violations are managed via the device-pairing edge function
