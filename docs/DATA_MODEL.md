# DATA_MODEL.md — Database Schema & Data Model

> Complete PostgreSQL schema for the LFLL application, hosted on Supabase.
> Cross-references: `SECURITY.md` (RLS policies), `BACKEND.md` (query patterns), `ARCHITECTURE.md` (data flow).

---

## Database: Supabase (PostgreSQL 15+)

All tables use UUIDs for primary keys (`gen_random_uuid()`). Timestamps are `TIMESTAMPTZ` in UTC. Soft deletes (`deleted_at`) used where NYS record retention requires multi-year storage. Auto-generated TypeScript types via `supabase gen types typescript`.

---

## Custom Types (PostgreSQL Enums)

```sql
-- Roles
CREATE TYPE user_role AS ENUM ('admin', 'staff', 'enrichment', 'parent', 'board');

-- Enrollment pipeline stages (matches Kanban columns in admin dashboard)
CREATE TYPE enrollment_status AS ENUM (
  'inquiry', 'touring', 'applied', 'accepted',
  'enrolled', 'waitlisted', 'withdrawn', 'graduated'
);

-- Lead pipeline stages
CREATE TYPE lead_status AS ENUM (
  'inquiry', 'tour_scheduled', 'tour_complete', 'application_received',
  'under_review', 'accepted', 'enrolled', 'waitlisted', 'declined', 'lost'
);

-- Invoice statuses
CREATE TYPE invoice_status AS ENUM ('draft', 'sent', 'paid', 'overdue', 'void', 'refunded');

-- Payment statuses
CREATE TYPE payment_status AS ENUM ('pending', 'succeeded', 'failed', 'refunded');

-- Message types
CREATE TYPE message_type AS ENUM ('direct', 'announcement', 'emergency', 'system');

-- Child enrollment status
CREATE TYPE child_status AS ENUM ('enrolled', 'waitlisted', 'graduated', 'withdrawn');

-- Document statuses
CREATE TYPE document_status AS ENUM ('not_started', 'in_progress', 'submitted', 'approved', 'expired');

-- Blog post statuses
CREATE TYPE post_status AS ENUM ('draft', 'scheduled', 'published', 'archived');

-- Background job statuses
CREATE TYPE job_status AS ENUM ('pending', 'processing', 'completed', 'failed');
```

---

## Core Tables

### Authentication & Users

```sql
-- User profiles linked to Supabase Auth
-- Supabase Auth handles auth.users automatically; this is the public profile layer
CREATE TABLE users (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT UNIQUE NOT NULL,
  role user_role NOT NULL DEFAULT 'parent',
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  phone TEXT,
  avatar_url TEXT,
  is_active BOOLEAN DEFAULT TRUE,
  onboarding_completed BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  last_login TIMESTAMPTZ
);

-- Trigger: auto-create users row on Supabase Auth signup
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.users (id, email, role, first_name, last_name)
  VALUES (
    NEW.id,
    NEW.email,
    COALESCE(NEW.raw_user_meta_data->>'role', 'parent'),
    COALESCE(NEW.raw_user_meta_data->>'first_name', ''),
    COALESCE(NEW.raw_user_meta_data->>'last_name', '')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

### Families & Children

```sql
CREATE TABLE families (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  primary_contact_user_id UUID REFERENCES users(id),
  family_name TEXT NOT NULL,
  address_line1 TEXT,
  address_line2 TEXT,
  city TEXT DEFAULT 'Newburgh',
  state TEXT DEFAULT 'NY',
  zip TEXT,
  enrollment_status enrollment_status DEFAULT 'inquiry',
  enrollment_year TEXT,                     -- e.g., '2025-2026'
  source TEXT,                              -- how they found LFLL
  referral_family_id UUID REFERENCES families(id),
  work_trade BOOLEAN DEFAULT FALSE,
  work_trade_monthly_credit NUMERIC(8,2) DEFAULT 0,  -- dollar amount credit per month
  stripe_customer_id TEXT,                  -- Stripe customer for billing
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ                    -- soft delete for 5-year retention
);

CREATE TABLE family_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id),        -- NULL if not yet registered in portal
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  relationship TEXT NOT NULL,               -- 'mother','father','guardian','grandparent','nanny','other'
  is_primary BOOLEAN DEFAULT FALSE,
  is_emergency_contact BOOLEAN DEFAULT FALSE,
  is_authorized_pickup BOOLEAN DEFAULT FALSE,
  phone TEXT,
  email TEXT,
  employer TEXT,
  work_phone TEXT
);

CREATE TABLE classrooms (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,                       -- e.g., 'Sunflower', 'Marigold', 'Dandelion'
  slug TEXT UNIQUE NOT NULL,                -- e.g., 'sunflower'
  lead_teacher_id UUID REFERENCES users(id),
  assistant_teacher_id UUID REFERENCES users(id),
  max_capacity INTEGER DEFAULT 12,
  age_range_min INTEGER DEFAULT 3,          -- years
  age_range_max INTEGER DEFAULT 6,
  description TEXT,
  school_year TEXT NOT NULL,                -- e.g., '2025-2026'
  is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE children (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  preferred_name TEXT,                      -- nickname
  date_of_birth DATE NOT NULL,
  gender TEXT,
  classroom_id UUID REFERENCES classrooms(id),
  enrollment_date DATE,
  expected_graduation DATE,
  actual_graduation_date DATE,
  allergies TEXT[],                          -- array for multiple allergies
  medical_notes TEXT,
  dietary_restrictions TEXT[],
  emergency_medical_info TEXT,              -- e.g., epi-pen instructions
  photo_release BOOLEAN DEFAULT FALSE,
  cpse_services BOOLEAN DEFAULT FALSE,      -- Committee on Preschool Special Education
  cpse_notes TEXT,
  status child_status DEFAULT 'enrolled',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ                    -- soft delete for 5-year retention
);
```

### Staff & Enrichment

```sql
CREATE TABLE staff_profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role_title TEXT NOT NULL,                 -- 'Lead Teacher', 'Assistant Teacher', 'Director'
  classroom_id UUID REFERENCES classrooms(id),
  hire_date DATE,
  employment_type TEXT CHECK (employment_type IN ('full_time', 'part_time', 'seasonal')),
  hourly_rate NUMERIC(8,2),
  emergency_contact_name TEXT,
  emergency_contact_phone TEXT,
  emergency_contact_relationship TEXT,
  cpr_expiry DATE,
  first_aid_expiry DATE,
  background_check_date DATE,
  background_check_clear BOOLEAN DEFAULT FALSE,
  fingerprint_date DATE,
  medical_statement_date DATE,             -- annual medical statement (OCFS requirement)
  tb_test_date DATE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

CREATE TABLE enrichment_teachers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  specialty TEXT NOT NULL,                  -- 'art', 'yoga_dance', 'music', 'skateboarding', 'spanish'
  bio TEXT,
  rate_per_session NUMERIC(8,2),
  contract_start DATE,
  contract_end DATE,
  status TEXT DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'pending')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE training_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  staff_id UUID NOT NULL REFERENCES staff_profiles(id) ON DELETE CASCADE,
  topic TEXT NOT NULL,                      -- OCFS topics: 'child_abuse_identification', 'safety_procedures',
                                            -- 'infection_control', 'nutrition', 'child_development',
                                            -- 'behavior_management', 'cultural_competency', etc.
  provider TEXT,
  date_completed DATE NOT NULL,
  hours NUMERIC(5,2) NOT NULL,
  certificate_url TEXT,                     -- Supabase Storage path
  expires_at DATE,                          -- NULL if no expiry
  verified_by UUID REFERENCES users(id),
  verified_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Enrollment Pipeline

```sql
CREATE TABLE leads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_name TEXT NOT NULL,
  parent_email TEXT NOT NULL,
  parent_phone TEXT,
  child_name TEXT,
  child_dob DATE,
  enrollment_year_interest TEXT,
  program_interest TEXT,                    -- 'primary', 'kindergarten', 'summer_camp', etc.
  source TEXT,                              -- 'website','referral','event','social_media','press','google','ai_search','other'
  referral_source TEXT,                     -- specific referrer name/family
  sibling_info TEXT,
  questions TEXT,
  status lead_status DEFAULT 'inquiry',
  assigned_to UUID REFERENCES users(id),    -- Rebecca by default
  family_id UUID REFERENCES families(id),   -- linked once converted
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tour_bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id UUID REFERENCES leads(id),
  tour_type TEXT CHECK (tour_type IN ('individual', 'group')),
  scheduled_datetime TIMESTAMPTZ NOT NULL,
  duration_minutes INTEGER DEFAULT 60,
  google_calendar_event_id TEXT,
  parent_name TEXT,
  parent_email TEXT,
  child_age INTEGER,
  how_heard_about TEXT,
  status TEXT DEFAULT 'scheduled'
    CHECK (status IN ('scheduled', 'completed', 'cancelled', 'no_show', 'rescheduled')),
  notes TEXT,
  follow_up_sent BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE consultation_bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID REFERENCES families(id),
  user_id UUID REFERENCES users(id),
  scheduled_datetime TIMESTAMPTZ NOT NULL,
  duration_minutes INTEGER DEFAULT 30,
  type TEXT CHECK (type IN ('enrolled_parent', 'external')),
  google_calendar_event_id TEXT,
  is_paid BOOLEAN DEFAULT FALSE,
  invoice_id UUID,
  topic TEXT,
  notes TEXT,
  status TEXT DEFAULT 'scheduled'
    CHECK (status IN ('scheduled', 'completed', 'cancelled', 'no_show'))
);

CREATE TABLE email_sequences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  trigger_event TEXT NOT NULL,              -- 'new_lead','post_tour','enrollment','immunization_due','birthday'
  steps_json JSONB NOT NULL,                -- [{delay_days, subject, body_html, template_id}]
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE email_sequence_progress (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id UUID REFERENCES leads(id) ON DELETE CASCADE,
  family_id UUID REFERENCES families(id),   -- for enrolled-family sequences
  sequence_id UUID NOT NULL REFERENCES email_sequences(id),
  current_step INTEGER DEFAULT 0,
  last_sent_at TIMESTAMPTZ,
  next_send_at TIMESTAMPTZ,
  status TEXT DEFAULT 'active'
    CHECK (status IN ('active', 'completed', 'paused', 'unsubscribed')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Daily Operations

```sql
CREATE TABLE attendance (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  child_id UUID NOT NULL REFERENCES children(id) ON DELETE CASCADE,
  classroom_id UUID NOT NULL REFERENCES classrooms(id),
  date DATE NOT NULL DEFAULT CURRENT_DATE,
  check_in_time TIMESTAMPTZ,
  check_out_time TIMESTAMPTZ,
  checked_in_by UUID REFERENCES users(id),
  checked_out_by UUID REFERENCES users(id),
  before_care BOOLEAN DEFAULT FALSE,
  after_care BOOLEAN DEFAULT FALSE,
  absent BOOLEAN DEFAULT FALSE,
  absence_reason TEXT,
  notes TEXT,
  UNIQUE(child_id, date)
);

CREATE TABLE daily_reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  classroom_id UUID NOT NULL REFERENCES classrooms(id),
  date DATE NOT NULL DEFAULT CURRENT_DATE,
  created_by UUID NOT NULL REFERENCES users(id),
  activities TEXT NOT NULL,                  -- what the children did today
  highlights TEXT,                           -- notable moments
  upcoming TEXT,                             -- what's coming up tomorrow/this week
  photo_ids UUID[],                          -- references to approved photos
  is_published BOOLEAN DEFAULT FALSE,        -- visible to parents when TRUE
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(classroom_id, date)
);

CREATE TABLE incident_reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  child_id UUID NOT NULL REFERENCES children(id),
  classroom_id UUID REFERENCES classrooms(id),
  reported_by UUID NOT NULL REFERENCES users(id),
  incident_date DATE NOT NULL,
  incident_time TIME,
  location TEXT,                             -- 'classroom', 'playground', 'gym', etc.
  incident_type TEXT,                        -- 'injury', 'behavioral', 'medical', 'property', 'other'
  description TEXT NOT NULL,
  witnesses TEXT,
  action_taken TEXT NOT NULL,
  follow_up_needed BOOLEAN DEFAULT FALSE,
  follow_up_notes TEXT,
  parent_notified BOOLEAN DEFAULT FALSE,
  parent_notified_at TIMESTAMPTZ,
  parent_notified_by TEXT,                   -- method: 'phone', 'in_person', 'portal', 'email'
  parent_signature_url TEXT,                 -- digital signature for acknowledgment
  document_url TEXT,                         -- PDF export path
  severity TEXT DEFAULT 'minor' CHECK (severity IN ('minor', 'moderate', 'serious')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE immunizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  child_id UUID NOT NULL REFERENCES children(id) ON DELETE CASCADE,
  vaccine_name TEXT NOT NULL,                -- 'DTaP', 'IPV', 'MMR', 'Varicella', 'Hep B', etc.
  dose_number INTEGER,                       -- 1, 2, 3, 4, 5
  date_administered DATE,
  next_due_date DATE,
  provider TEXT,
  document_url TEXT,                         -- uploaded record in Supabase Storage
  verified_by UUID REFERENCES users(id),
  verified_at TIMESTAMPTZ,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'verified', 'overdue', 'exempt')),
  exemption_type TEXT,                       -- 'medical', 'religious' (if exempt)
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  child_id UUID REFERENCES children(id) ON DELETE CASCADE,
  family_id UUID REFERENCES families(id) ON DELETE CASCADE,
  type TEXT NOT NULL
    CHECK (type IN (
      'enrollment_form', 'health_form', 'immunization_record', 'photo_release',
      'emergency_contact_form', 'authorization_pickup', 'ocfs_ldss_2221',
      'ocfs_ldss_4406', 'permission_slip', 'incident_report',
      'conference_report', 'progress_report', 'iep_document', 'other'
    )),
  name TEXT NOT NULL,
  description TEXT,
  file_url TEXT,                             -- Supabase Storage path
  status document_status DEFAULT 'not_started',
  due_date DATE,
  submitted_at TIMESTAMPTZ,
  submitted_by UUID REFERENCES users(id),
  approved_by UUID REFERENCES users(id),
  approved_at TIMESTAMPTZ,
  rejection_reason TEXT,
  expires_at TIMESTAMPTZ,
  version INTEGER DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);
```

### Billing & Payments

```sql
CREATE TABLE invoices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID NOT NULL REFERENCES families(id),
  type TEXT NOT NULL
    CHECK (type IN ('tuition', 'aftercare', 'field_trip', 'club', 'camp', 'event', 'donation', 'other')),
  description TEXT,
  line_items_json JSONB,                    -- [{description, quantity, unit_price, amount}]
  subtotal NUMERIC(10,2) NOT NULL,
  work_trade_credit NUMERIC(10,2) DEFAULT 0,
  tax NUMERIC(10,2) DEFAULT 0,
  total NUMERIC(10,2) NOT NULL,
  due_date DATE,
  status invoice_status DEFAULT 'draft',
  stripe_invoice_id TEXT,
  stripe_payment_intent_id TEXT,
  paid_at TIMESTAMPTZ,
  paid_amount NUMERIC(10,2),
  billing_period_start DATE,                -- for tuition: month start
  billing_period_end DATE,                  -- for tuition: month end
  notes TEXT,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id UUID NOT NULL REFERENCES invoices(id),
  family_id UUID NOT NULL REFERENCES families(id),
  amount NUMERIC(10,2) NOT NULL,
  processing_fee NUMERIC(10,2) DEFAULT 0,   -- Stripe fee (3.2%) tracked for accounting
  net_amount NUMERIC(10,2),                  -- amount - processing_fee
  method TEXT CHECK (method IN ('credit_card', 'ach', 'check', 'cash', 'other')),
  stripe_payment_id TEXT,
  stripe_charge_id TEXT,
  status payment_status DEFAULT 'pending',
  failure_reason TEXT,
  receipt_url TEXT,                           -- Stripe receipt URL
  refund_amount NUMERIC(10,2),
  refunded_at TIMESTAMPTZ,
  processed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE autopay_settings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID NOT NULL REFERENCES families(id) UNIQUE,
  stripe_subscription_id TEXT,
  stripe_payment_method_id TEXT,
  is_active BOOLEAN DEFAULT FALSE,
  payment_day INTEGER DEFAULT 1,             -- day of month to charge
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE donations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  donor_name TEXT NOT NULL,
  donor_email TEXT NOT NULL,
  amount NUMERIC(10,2) NOT NULL,
  fund TEXT DEFAULT 'general',               -- 'general', 'scholarship', 'garden', 'art', 'playground'
  is_recurring BOOLEAN DEFAULT FALSE,
  stripe_payment_id TEXT,
  stripe_subscription_id TEXT,               -- for recurring donations
  is_anonymous BOOLEAN DEFAULT FALSE,
  message TEXT,
  receipt_sent BOOLEAN DEFAULT FALSE,
  family_id UUID REFERENCES families(id),    -- if from current family
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Communication

```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sender_id UUID NOT NULL REFERENCES users(id),
  recipient_id UUID NOT NULL REFERENCES users(id),
  thread_id UUID DEFAULT gen_random_uuid(),
  subject TEXT,
  body TEXT NOT NULL,
  message_type message_type DEFAULT 'direct',
  is_read BOOLEAN DEFAULT FALSE,
  read_at TIMESTAMPTZ,
  attachments_json JSONB,                    -- [{name, url, type, size}]
  is_archived BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE quick_reply_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,                       -- e.g., 'Pickup Reminder', 'Field Trip Info'
  body TEXT NOT NULL,
  variables_json JSONB,                      -- ["{child_name}", "{parent_name}", "{date}", "{classroom}"]
  category TEXT,                             -- 'attendance', 'billing', 'general', 'events'
  created_by UUID REFERENCES users(id),
  is_shared BOOLEAN DEFAULT TRUE,            -- available to all staff
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE email_campaigns (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject TEXT NOT NULL,
  body_html TEXT NOT NULL,
  preview_text TEXT,
  recipient_segment TEXT,                    -- 'all_families','classroom_sunflower','prospective','alumni','community','donors','staff'
  recipient_count INTEGER DEFAULT 0,
  status post_status DEFAULT 'draft',
  scheduled_at TIMESTAMPTZ,
  sent_at TIMESTAMPTZ,
  resend_batch_id TEXT,
  open_count INTEGER DEFAULT 0,
  click_count INTEGER DEFAULT 0,
  unsubscribe_count INTEGER DEFAULT 0,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE notification_preferences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  category TEXT NOT NULL,                    -- 'urgent','daily_digest','billing','events','forms','messages','announcements'
  email_enabled BOOLEAN DEFAULT TRUE,
  push_enabled BOOLEAN DEFAULT TRUE,
  sms_enabled BOOLEAN DEFAULT FALSE,
  quiet_hours_start TIME DEFAULT '21:00',
  quiet_hours_end TIME DEFAULT '07:00',
  UNIQUE(user_id, category)
);
```

### Content Management

```sql
CREATE TABLE blog_posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  body_html TEXT NOT NULL,
  excerpt TEXT,
  featured_image_url TEXT,
  featured_image_alt TEXT,
  category TEXT,                             -- 'school_updates','montessori_education','community','press','seasonal','parenting'
  tags TEXT[],
  author_id UUID REFERENCES users(id),
  status post_status DEFAULT 'draft',
  published_at TIMESTAMPTZ,
  scheduled_at TIMESTAMPTZ,                  -- for scheduled publishing
  seo_title TEXT,
  seo_description TEXT,
  target_aeo_question TEXT,                  -- the question this post answers for AEO
  reading_time_minutes INTEGER,
  view_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE faqs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  question TEXT NOT NULL,
  answer_html TEXT NOT NULL,
  category TEXT,                             -- 'admissions','programs','logistics','policies','financial','summer_camp'
  sort_order INTEGER DEFAULT 0,
  is_published BOOLEAN DEFAULT TRUE,
  page_context TEXT[],                       -- ['admissions', 'programs', 'summer_camp'] — which pages show this FAQ
  is_chatbot_knowledge BOOLEAN DEFAULT TRUE, -- include in chatbot RAG context
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE pages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug TEXT UNIQUE NOT NULL,
  title TEXT NOT NULL,
  body_html TEXT,
  seo_title TEXT,
  seo_description TEXT,
  og_image_url TEXT,
  is_published BOOLEAN DEFAULT TRUE,
  updated_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE photos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  uploaded_by UUID NOT NULL REFERENCES users(id),
  original_filename TEXT,
  file_url TEXT NOT NULL,                    -- primary optimized version
  thumbnail_url TEXT,                        -- 400px thumbnail
  responsive_urls_json JSONB,                -- {400: url, 800: url, 1200: url, 1600: url}
  webp_url TEXT,
  avif_url TEXT,
  jpeg_fallback_url TEXT,
  width INTEGER,
  height INTEGER,
  file_size_bytes INTEGER,
  exif_stripped BOOLEAN DEFAULT TRUE,
  classroom_id UUID REFERENCES classrooms(id),
  event_id UUID REFERENCES events(id),
  child_ids UUID[],                          -- tagged children (first names only in UI)
  tags TEXT[],
  caption TEXT,
  taken_date DATE,
  is_approved BOOLEAN DEFAULT FALSE,
  approved_by UUID REFERENCES users(id),
  approved_at TIMESTAMPTZ,
  is_public BOOLEAN DEFAULT FALSE,           -- TRUE only if ALL tagged children have photo_release
  rejection_reason TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Events & Signups

```sql
CREATE TABLE events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  slug TEXT UNIQUE,
  description TEXT,
  event_type TEXT CHECK (event_type IN (
    'fundraiser', 'workshop', 'open_house', 'community',
    'field_trip', 'performance', 'celebration', 'meeting'
  )),
  location TEXT,
  location_address TEXT,
  start_datetime TIMESTAMPTZ NOT NULL,
  end_datetime TIMESTAMPTZ,
  all_day BOOLEAN DEFAULT FALSE,
  max_capacity INTEGER,
  current_rsvps INTEGER DEFAULT 0,
  ticket_price NUMERIC(10,2) DEFAULT 0,
  is_public BOOLEAN DEFAULT TRUE,            -- FALSE = portal-only events
  is_featured BOOLEAN DEFAULT FALSE,         -- show on homepage
  requires_payment BOOLEAN DEFAULT FALSE,
  requires_permission_slip BOOLEAN DEFAULT FALSE,
  image_url TEXT,
  google_calendar_event_id TEXT,
  target_audience TEXT,                       -- 'all','enrolled','prospective','community'
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE event_rsvps (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id),
  family_id UUID REFERENCES families(id),
  name TEXT NOT NULL,
  email TEXT NOT NULL,
  num_attendees INTEGER DEFAULT 1,
  payment_status TEXT DEFAULT 'not_required'
    CHECK (payment_status IN ('not_required', 'pending', 'paid', 'refunded')),
  stripe_payment_id TEXT,
  notes TEXT,
  rsvp_date TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(event_id, email)
);

CREATE TABLE signups (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID NOT NULL REFERENCES families(id),
  child_id UUID REFERENCES children(id),
  signup_type TEXT NOT NULL
    CHECK (signup_type IN (
      'before_care', 'after_care', 'field_trip', 'club',
      'volunteer', 'mystery_reader', 'birthday_party',
      'hot_lunch_helper', 'veggie_week', 'play_doh', 'conference'
    )),
  signup_date DATE NOT NULL,
  time_slot TEXT,                             -- e.g., '3:30 PM', 'Week of March 10'
  details_json JSONB,                        -- signup-type-specific data
  payment_required BOOLEAN DEFAULT FALSE,
  invoice_id UUID REFERENCES invoices(id),
  status TEXT DEFAULT 'confirmed'
    CHECK (status IN ('confirmed', 'cancelled', 'waitlisted', 'completed')),
  cancelled_at TIMESTAMPTZ,
  cancellation_reason TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(family_id, child_id, signup_type, signup_date)  -- prevent double-booking
);
```

### Summer Camp

```sql
CREATE TABLE camp_weeks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  year INTEGER NOT NULL,
  week_number INTEGER NOT NULL,              -- 1-8
  theme TEXT NOT NULL,                       -- e.g., 'Ocean Explorers', 'Garden Magic'
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  max_capacity INTEGER DEFAULT 24,
  current_enrollment INTEGER DEFAULT 0,
  price_per_week NUMERIC(10,2) NOT NULL,
  description TEXT,
  is_active BOOLEAN DEFAULT TRUE,
  UNIQUE(year, week_number)
);

CREATE TABLE camp_registrations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_name TEXT NOT NULL,
  parent_email TEXT NOT NULL,
  parent_phone TEXT NOT NULL,
  child_first_name TEXT NOT NULL,
  child_last_name TEXT NOT NULL,
  child_dob DATE NOT NULL,
  allergies TEXT,
  medical_notes TEXT,
  emergency_contacts_json JSONB NOT NULL,
  photo_release BOOLEAN DEFAULT FALSE,
  swim_permission BOOLEAN DEFAULT FALSE,
  week_ids UUID[] NOT NULL,                  -- array of camp_weeks IDs
  total_amount NUMERIC(10,2) NOT NULL,
  payment_status payment_status DEFAULT 'pending',
  stripe_checkout_session_id TEXT,
  stripe_payment_id TEXT,
  registration_status TEXT DEFAULT 'pending'
    CHECK (registration_status IN ('pending', 'confirmed', 'cancelled', 'waitlisted')),
  family_id UUID REFERENCES families(id),    -- linked if existing family
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Compliance & Operations

```sql
CREATE TABLE grants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  organization TEXT,
  amount NUMERIC(10,2),
  deadline DATE,
  category TEXT,                             -- 'education','facility','technology','arts','community'
  status TEXT DEFAULT 'discovered'
    CHECK (status IN ('discovered', 'researching', 'writing', 'submitted', 'awaiting', 'won', 'lost')),
  application_url TEXT,
  submission_date DATE,
  award_date DATE,
  award_amount NUMERIC(10,2),
  notes TEXT,
  documents_json JSONB,                      -- [{name, url, type}]
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE work_trade_entries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID NOT NULL REFERENCES families(id),
  task_description TEXT NOT NULL,
  date DATE NOT NULL,
  hours NUMERIC(5,2) NOT NULL,
  hourly_rate NUMERIC(8,2),                  -- rate used for credit calculation
  credit_amount NUMERIC(8,2),                -- hours × rate = credit applied to next invoice
  category TEXT,                             -- 'maintenance','classroom','administrative','events','garden'
  verified_by UUID REFERENCES users(id),
  verified_at TIMESTAMPTZ,
  applied_to_invoice_id UUID REFERENCES invoices(id),
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE inventory_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  category TEXT,                             -- 'classroom','art','cleaning','office','outdoor','kitchen'
  quantity INTEGER DEFAULT 0,
  unit TEXT DEFAULT 'each',                  -- 'each','box','pack','ream','bottle'
  reorder_threshold INTEGER,
  preferred_vendor TEXT,
  last_ordered DATE,
  last_order_cost NUMERIC(8,2),
  monthly_budget NUMERIC(8,2),
  notes TEXT,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE press_contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  organization TEXT,
  email TEXT,
  phone TEXT,
  beat TEXT,                                 -- 'education','local_news','family','business'
  relationship_notes TEXT,
  last_contacted DATE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE press_releases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  body_html TEXT,
  event_id UUID REFERENCES events(id),
  sent_to_json JSONB,                        -- [{contact_id, sent_at}]
  status TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'approved', 'sent')),
  approved_by UUID REFERENCES users(id),
  sent_at TIMESTAMPTZ,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE career_applications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  email TEXT NOT NULL,
  phone TEXT,
  position TEXT NOT NULL,
  experience TEXT,
  education TEXT,
  references_text TEXT,
  resume_url TEXT,                            -- Supabase Storage path
  cover_letter_url TEXT,
  status TEXT DEFAULT 'received'
    CHECK (status IN ('received', 'reviewing', 'interview_scheduled', 'offered', 'hired', 'rejected')),
  notes TEXT,
  reviewed_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### System Tables

```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  action TEXT NOT NULL,
  table_name TEXT,
  record_id UUID,
  old_values JSONB,
  new_values JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
-- Partitioned by month for performance (optional at scale)

CREATE TABLE background_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type TEXT NOT NULL,                         -- 'bulk_invoice','board_report','mass_email','data_export'
  status job_status DEFAULT 'pending',
  params JSONB,                               -- input parameters
  result JSONB,                               -- output/summary when completed
  error_message TEXT,                         -- if failed
  progress INTEGER DEFAULT 0,                 -- 0-100 percentage
  created_by UUID REFERENCES users(id),
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE school_settings (
  key TEXT PRIMARY KEY,
  value TEXT,
  description TEXT,
  updated_by UUID REFERENCES users(id),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
-- Seed data keys:
-- school_name, school_phone, school_email, school_address,
-- academic_year, enrollment_open, max_enrollment, tuition_monthly,
-- aftercare_daily_rate, aftercare_weekly_rate, ein_number,
-- tour_duration_minutes, before_care_start, after_care_end,
-- before_care_capacity, after_care_capacity
```

---

## Auto-Updating Timestamps

```sql
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to all tables with updated_at column:
CREATE TRIGGER set_updated_at BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON families FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON children FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON staff_profiles FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON enrichment_teachers FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON leads FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON email_sequences FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON documents FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON invoices FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON autopay_settings FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON quick_reply_templates FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON email_campaigns FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON blog_posts FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON faqs FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON pages FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON events FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON camp_registrations FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON grants FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON career_applications FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON school_settings FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## Soft Deletes

Tables with `deleted_at`: `families`, `children`, `staff_profiles`, `documents`.

All default queries filter: `WHERE deleted_at IS NULL`.
Admin can view archived records with explicit filter.
Supports NYS 5-year child record retention and 7-year financial retention requirements.

---

## Database Views

```sql
-- Active enrolled children with family and classroom info
CREATE VIEW v_enrolled_children AS
SELECT
  c.id, c.first_name, c.last_name, c.date_of_birth, c.classroom_id,
  c.allergies, c.medical_notes, c.photo_release, c.cpse_services,
  cl.name AS classroom_name, cl.lead_teacher_id,
  f.family_name, f.primary_contact_user_id
FROM children c
JOIN classrooms cl ON c.classroom_id = cl.id
JOIN families f ON c.family_id = f.id
WHERE c.status = 'enrolled' AND c.deleted_at IS NULL AND f.deleted_at IS NULL;

-- Today's attendance roster by classroom
CREATE VIEW v_todays_attendance AS
SELECT
  c.id AS child_id, c.first_name, c.last_name,
  c.classroom_id, cl.name AS classroom_name,
  a.check_in_time, a.check_out_time, a.before_care, a.after_care, a.absent,
  a.checked_in_by, a.checked_out_by
FROM children c
JOIN classrooms cl ON c.classroom_id = cl.id
LEFT JOIN attendance a ON c.id = a.child_id AND a.date = CURRENT_DATE
WHERE c.status = 'enrolled' AND c.deleted_at IS NULL;

-- Family billing summary
CREATE VIEW v_family_billing AS
SELECT
  f.id AS family_id, f.family_name, f.stripe_customer_id,
  f.work_trade, f.work_trade_monthly_credit,
  COUNT(DISTINCT i.id) FILTER (WHERE i.status = 'sent') AS pending_invoices,
  COALESCE(SUM(i.total) FILTER (WHERE i.status = 'sent'), 0) AS outstanding_balance,
  COALESCE(SUM(i.total) FILTER (WHERE i.status = 'overdue'), 0) AS overdue_amount,
  ap.is_active AS autopay_active
FROM families f
LEFT JOIN invoices i ON f.id = i.family_id AND i.status IN ('sent', 'overdue')
LEFT JOIN autopay_settings ap ON f.id = ap.family_id
WHERE f.deleted_at IS NULL AND f.enrollment_status = 'enrolled'
GROUP BY f.id, f.family_name, f.stripe_customer_id, f.work_trade, f.work_trade_monthly_credit, ap.is_active;

-- Compliance overview for admin dashboard
CREATE VIEW v_compliance_overview AS
SELECT
  sp.id AS staff_id, u.first_name, u.last_name, sp.role_title,
  sp.cpr_expiry, sp.first_aid_expiry, sp.background_check_date,
  sp.medical_statement_date, sp.tb_test_date,
  COALESCE(SUM(tr.hours), 0) AS total_training_hours,
  sp.cpr_expiry < CURRENT_DATE + INTERVAL '30 days' AS cpr_expiring_soon,
  sp.first_aid_expiry < CURRENT_DATE + INTERVAL '30 days' AS first_aid_expiring_soon
FROM staff_profiles sp
JOIN users u ON sp.user_id = u.id
LEFT JOIN training_records tr ON sp.id = tr.staff_id
  AND tr.date_completed >= DATE_TRUNC('year', CURRENT_DATE)
WHERE sp.deleted_at IS NULL
GROUP BY sp.id, u.first_name, u.last_name, sp.role_title,
  sp.cpr_expiry, sp.first_aid_expiry, sp.background_check_date,
  sp.medical_statement_date, sp.tb_test_date;
```

---

## Critical Indexes

```sql
-- Children lookups
CREATE INDEX idx_children_family ON children(family_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_children_classroom ON children(classroom_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_children_status ON children(status) WHERE deleted_at IS NULL;

-- Attendance (most-queried table)
CREATE INDEX idx_attendance_child_date ON attendance(child_id, date);
CREATE INDEX idx_attendance_classroom_date ON attendance(classroom_id, date);
CREATE INDEX idx_attendance_date ON attendance(date);

-- Billing
CREATE INDEX idx_invoices_family_status ON invoices(family_id, status);
CREATE INDEX idx_invoices_due_date ON invoices(due_date) WHERE status IN ('sent', 'overdue');
CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_family ON payments(family_id);

-- Lead pipeline
CREATE INDEX idx_leads_status ON leads(status);
CREATE INDEX idx_leads_created ON leads(created_at DESC);
CREATE INDEX idx_leads_email ON leads(parent_email);

-- Messages
CREATE INDEX idx_messages_recipient ON messages(recipient_id, is_read);
CREATE INDEX idx_messages_sender ON messages(sender_id);
CREATE INDEX idx_messages_thread ON messages(thread_id, created_at);

-- Signups
CREATE INDEX idx_signups_family_type ON signups(family_id, signup_type);
CREATE INDEX idx_signups_date ON signups(signup_date);
CREATE INDEX idx_signups_child_date ON signups(child_id, signup_date);

-- Photos
CREATE INDEX idx_photos_approved ON photos(is_approved, created_at DESC);
CREATE INDEX idx_photos_classroom ON photos(classroom_id, created_at DESC);

-- Blog
CREATE INDEX idx_blog_published ON blog_posts(status, published_at DESC) WHERE status = 'published';
CREATE INDEX idx_blog_slug ON blog_posts(slug) WHERE status = 'published';

-- Audit + system
CREATE INDEX idx_audit_table ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at DESC);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);

-- Notification preferences
CREATE INDEX idx_notif_prefs_user ON notification_preferences(user_id);

-- Email sequences
CREATE INDEX idx_email_seq_progress ON email_sequence_progress(status, next_send_at)
  WHERE status = 'active';

-- Immunizations
CREATE INDEX idx_immunizations_child ON immunizations(child_id);
CREATE INDEX idx_immunizations_due ON immunizations(next_due_date) WHERE status != 'exempt';

-- Training
CREATE INDEX idx_training_staff ON training_records(staff_id);
CREATE INDEX idx_training_expiry ON training_records(expires_at) WHERE expires_at IS NOT NULL;

-- Families
CREATE INDEX idx_families_status ON families(enrollment_status) WHERE deleted_at IS NULL;
CREATE INDEX idx_families_stripe ON families(stripe_customer_id) WHERE stripe_customer_id IS NOT NULL;

-- Camp
CREATE INDEX idx_camp_weeks_year ON camp_weeks(year, week_number);
CREATE INDEX idx_camp_reg_email ON camp_registrations(parent_email);

-- Events
CREATE INDEX idx_events_date ON events(start_datetime) WHERE start_datetime >= CURRENT_DATE;
CREATE INDEX idx_event_rsvps_event ON event_rsvps(event_id);

-- Background jobs
CREATE INDEX idx_jobs_status ON background_jobs(status, created_at DESC);
```

---

## Row-Level Security Summary

See `docs/SECURITY.md` for complete RLS policy definitions. Access rules:

| Role | Access Scope |
|---|---|
| `parent` | Own family, own children, own invoices, own messages, own signups only |
| `staff` | Children + families in assigned classroom, own profile + training, classroom messages |
| `enrichment` | Own session attendance, schedule, hour logging only |
| `admin` | All data — every action logged to audit_log |
| `board` | Read-only: aggregated financial reports, enrollment stats, compliance overview |

**CRITICAL: Test extensively for cross-family data leakage. One family must NEVER see another family's children, invoices, messages, or forms.**

---

## Supabase Storage Buckets

| Bucket | Access | Contents |
|---|---|---|
| `photos` | Authenticated (RLS) | Processed photo variants (WebP, AVIF, JPEG, thumbnails) |
| `documents` | Authenticated (RLS) | Enrollment forms, health records, immunization scans, incident PDFs |
| `public-assets` | Public | Website images, blog photos, event images, OG images |
| `exports` | Admin only | Generated reports, board packets, data exports |
| `resumes` | Admin only | Career application files |

---

## Seed Data (Development)

```sql
-- Development seed: creates test data for all roles
-- See supabase/seed.sql

-- 1 admin (Rebecca), 3 staff (lead teachers), 2 enrichment teachers
-- 3 classrooms (Sunflower, Marigold, Dandelion) with 8-12 children each
-- 5 enrolled families with 1-2 children each, complete records
-- 3 prospective leads at various pipeline stages
-- Sample invoices, attendance records, messages
-- Sample blog posts, FAQs, events
-- Email sequences (new_lead: 5 steps, post_tour: 3 steps, enrollment: 5 steps)
-- School settings with real LFLL values
```
