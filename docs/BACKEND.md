# BACKEND.md — Backend Architecture

> Complete backend specification for the LFLL Next.js application.
> Cross-references: `ARCHITECTURE.md` (project structure), `DATA_MODEL.md` (schema), `SECURITY.md` (auth/RLS), `API_INTEGRATIONS.md` (external services).

---

## Supabase Client Setup

Three Supabase client types serve different contexts. Never use the wrong client — it's a security boundary.

### Browser Client (Client Components)

```typescript
// lib/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr'
import type { Database } from '@/types/database.types'

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

Used in: Client Components, hooks, Realtime subscriptions.
RLS enforced: Yes — operates with the logged-in user's JWT. The anon key provides no special access.

### Server Client (RSC + Route Handlers + Server Actions)

```typescript
// lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
import type { Database } from '@/types/database.types'

export async function createServerSupabaseClient() {
  const cookieStore = await cookies()
  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll() },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) => {
            cookieStore.set(name, value, options)
          })
        },
      },
    }
  )
}
```

Used in: Server Components, Route Handlers, Server Actions.
RLS enforced: Yes — reads session from cookies.

### Admin Client (Service Role — Server Only)

```typescript
// lib/supabase/admin.ts
import { createClient } from '@supabase/supabase-js'
import type { Database } from '@/types/database.types'

export const supabaseAdmin = createClient<Database>(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
)
```

Used in: Cron jobs, webhooks, bulk operations, admin reports.
RLS enforced: **NO** — bypasses all RLS. NEVER expose to client. Only use when the operation explicitly requires bypassing row-level security (e.g., generating cross-family reports, processing Stripe webhooks, running cron jobs).

---

## Database Query Layer

All database queries are organized by domain in `/lib/supabase/queries/`. Each module exports typed functions that wrap Supabase queries. Route Handlers and Server Actions call these — never construct raw queries in route files.

```
lib/supabase/queries/
├── leads.ts          # Lead CRUD, pipeline queries, lead stats
├── families.ts       # Family CRUD, enrollment status, directory
├── children.ts       # Child profiles, enrollment, medical records
├── attendance.ts     # Check in/out, daily rosters, reports
├── invoices.ts       # Invoice generation, payment tracking
├── messages.ts       # Thread management, unread counts
├── events.ts         # Event CRUD, RSVPs, capacity checks
├── blog.ts           # Blog CRUD, slug generation, publishing
├── photos.ts         # Photo metadata, curation queue, approval
├── signups.ts        # Before/after care, clubs, field trips, conferences
├── compliance.ts     # Immunizations, training, OCFS tracking
├── documents.ts      # Form tracking, file management
├── email.ts          # Campaigns, sequences, audience management
├── reports.ts        # Aggregated data for admin/board reports
└── settings.ts       # School settings key-value store
```

### Query Function Pattern

```typescript
// lib/supabase/queries/families.ts
import type { Database } from '@/types/database.types'
import type { SupabaseClient } from '@supabase/supabase-js'

type Family = Database['public']['Tables']['families']['Row']
type FamilyInsert = Database['public']['Tables']['families']['Insert']
type FamilyUpdate = Database['public']['Tables']['families']['Update']

export async function getFamilyById(
  supabase: SupabaseClient<Database>,
  familyId: string
): Promise<Family | null> {
  const { data, error } = await supabase
    .from('families')
    .select(`
      *,
      children (*),
      family_members (*, users (*))
    `)
    .eq('id', familyId)
    .is('deleted_at', null)
    .single()

  if (error) throw new DatabaseError('Failed to fetch family', error)
  return data
}

export async function getEnrollmentPipeline(
  supabase: SupabaseClient<Database>
): Promise<Record<string, Family[]>> {
  const { data, error } = await supabase
    .from('families')
    .select('*, children (*), leads (*)')
    .is('deleted_at', null)
    .order('updated_at', { ascending: false })

  if (error) throw new DatabaseError('Failed to fetch pipeline', error)

  // Group by enrollment_status for Kanban display
  return groupBy(data, 'enrollment_status')
}
```

### Error Handling in Query Layer

```typescript
// lib/supabase/errors.ts
export class DatabaseError extends Error {
  constructor(
    message: string,
    public readonly supabaseError: { message: string; code: string; details?: string }
  ) {
    super(`${message}: ${supabaseError.message}`)
    this.name = 'DatabaseError'
  }
}
```

---

## API Route Handlers

All mutations go through Next.js Route Handlers in `/src/app/api/`. Every route handler follows the same pattern:

1. Parse and validate request body with Zod
2. Check authentication (where required)
3. Check authorization (role-based)
4. Call query layer function
5. Trigger side effects (email, calendar, audit log)
6. Return typed JSON response

### Shared Utilities

```typescript
// lib/api/helpers.ts
import { NextResponse } from 'next/server'
import { ZodError, type ZodSchema } from 'zod'

export function unauthorized() {
  return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
}

export function forbidden() {
  return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
}

export function badRequest(message: string, details?: unknown) {
  return NextResponse.json({ error: message, details }, { status: 400 })
}

export function success<T>(data: T) {
  return NextResponse.json({ success: true, data })
}

export async function parseBody<T>(request: Request, schema: ZodSchema<T>): Promise<T> {
  const body = await request.json()
  return schema.parse(body)
}

export async function getAuthUser(request: Request) {
  const supabase = await createServerSupabaseClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return null

  const { data: profile } = await supabase
    .from('users')
    .select('*')
    .eq('id', user.id)
    .single()

  return profile
}

export function requireRole(user: User | null, ...roles: string[]) {
  if (!user || !roles.includes(user.role)) {
    throw new AuthorizationError('Insufficient permissions')
  }
  return user
}
```

### Audit Logging Helper

```typescript
// lib/api/audit.ts
export async function auditLog(
  userId: string | null,
  action: string,
  tableName: string,
  recordId?: string,
  oldValues?: Record<string, unknown>,
  newValues?: Record<string, unknown>,
  ipAddress?: string
) {
  await supabaseAdmin.from('audit_log').insert({
    user_id: userId,
    action,
    table_name: tableName,
    record_id: recordId,
    old_values: oldValues ?? null,
    new_values: newValues ?? null,
    ip_address: ipAddress ?? null,
  })
}
```

---

### Public API Routes (No Auth Required)

#### `POST /api/leads`

Create new enrollment inquiry lead. Primary conversion endpoint.

```typescript
const leadSchema = z.object({
  parentName: z.string().min(2).max(100),
  email: z.string().email(),
  phone: z.string().optional(),
  childName: z.string().optional(),
  childDob: z.string().date().optional(),
  enrollmentYear: z.string().optional(),
  programInterest: z.enum(['primary', 'kindergarten', 'summer_camp', 'enrichment', 'unsure']).optional(),
  source: z.enum(['website', 'referral', 'event', 'social_media', 'press', 'google', 'ai_search', 'other']).optional(),
  referralSource: z.string().optional(),
  siblingInfo: z.string().optional(),
  questions: z.string().optional(),
})
```

**Actions:**
1. Insert `leads` row (status: `inquiry`)
2. Create `email_sequence_progress` row (trigger: `new_lead` sequence)
3. Send acknowledgment email via Resend (`InquiryConfirmation` template)
4. Log to `audit_log`
5. If Realtime connected, broadcast new lead to admin dashboard

**Rate limit:** 5 per IP per hour.
**Spam protection:** Honeypot field + timing check (reject submissions < 2 seconds).

#### `POST /api/tours`

Book a campus tour. Calendar-first flow — user picks slot, then provides details.

```typescript
const tourSchema = z.object({
  leadId: z.string().uuid().optional(),
  tourType: z.enum(['individual', 'group']),
  scheduledDatetime: z.string().datetime(),
  parentName: z.string().min(2),
  parentEmail: z.string().email(),
  parentPhone: z.string().optional(),
  childAge: z.number().min(2).max(7).optional(),
  howHeardAbout: z.string().optional(),
})
```

**Actions:**
1. Verify slot availability against Google Calendar (reject if conflict)
2. Create `tour_bookings` row
3. Create or update `leads` row (status: `tour_scheduled`)
4. Create Google Calendar event on Rebecca's calendar with parent details in description
5. Send confirmation email with: directions to JCC, parking info, what to expect, what to bring
6. Create `email_sequence_progress` row (trigger: `post_tour` sequence, delayed start after tour date)

**Validation:** Double-booking prevention — check Google Calendar AND `tour_bookings` table.

#### `POST /api/contact`

General contact form. NOT for enrollment — heading clearly says "Get in Touch."

```typescript
const contactSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  message: z.string().min(10).max(2000),
})
```

**Actions:**
1. Send email to Rebecca via Resend with form contents
2. Send auto-reply to user acknowledging receipt
3. Log to `audit_log`

**Rate limit:** 3 per IP per hour. Honeypot field.

#### `POST /api/newsletter`

Newsletter signup from website footer or community page.

```typescript
const newsletterSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  relation: z.enum(['prospective_parent', 'current_parent', 'community_member', 'educator', 'other']).optional(),
})
```

**Actions:**
1. Check duplicate email in Resend audience
2. Add to `community` audience in Resend with metadata
3. Send welcome email
4. Return success (graceful handling if already subscribed)

#### `POST /api/chatbot`

AI chatbot message. See `AI_INTEGRATION.md` for full chatbot spec.

```typescript
const chatbotSchema = z.object({
  message: z.string().min(1).max(1000),
  conversationHistory: z.array(z.object({
    role: z.enum(['user', 'assistant']),
    content: z.string(),
  })).max(20),
  sessionId: z.string().optional(),
})
```

**Actions:**
1. Build system prompt with FAQ knowledge base + school info
2. Send to Claude Haiku API with conversation history
3. Apply safety rails (filter PII, block inappropriate content)
4. Return AI response with optional suggested actions

**Rate limit:** 20 messages per session per hour.
**Max conversation length:** 20 turns, then suggest "talk to a human."

#### `POST /api/careers/apply`

Job application submission. Multi-step form with file uploads.

**Body:** Multipart form data — validated server-side.
**Actions:**
1. Parse multipart form data
2. Validate file types (PDF, DOC, DOCX only) and sizes (max 5MB each)
3. Upload resume + cover letter to Supabase Storage (`documents` bucket, private)
4. Insert application record
5. Send acknowledgment email to applicant
6. Send notification to Rebecca with application summary
7. Log to `audit_log`

#### `POST /api/camp/register`

Summer camp registration with payment. Multi-step: child info → week selection → medical → payment.

```typescript
const campSchema = z.object({
  parentInfo: z.object({
    name: z.string(),
    email: z.string().email(),
    phone: z.string(),
  }),
  childInfo: z.object({
    firstName: z.string(),
    lastName: z.string(),
    dateOfBirth: z.string().date(),
    allergies: z.string().optional(),
    medicalNotes: z.string().optional(),
  }),
  weekSelections: z.array(z.string()).min(1),
  emergencyContacts: z.array(z.object({
    name: z.string(),
    phone: z.string(),
    relationship: z.string(),
  })).min(1),
  photoRelease: z.boolean(),
  swimPermission: z.boolean().optional(),
})
```

**Actions:**
1. Validate week availability (check capacity)
2. Calculate total cost (per-week pricing × weeks selected)
3. Create Stripe Checkout Session with line items
4. Store pending registration with `checkout_session_id`
5. Return Stripe Checkout URL
6. On `checkout.session.completed` webhook → finalize registration, send confirmation

#### `POST /api/events/rsvp`

Public event RSVP. Some events are free, some require payment.

```typescript
const rsvpSchema = z.object({
  eventId: z.string().uuid(),
  name: z.string(),
  email: z.string().email(),
  numAttendees: z.number().min(1).max(10),
})
```

**Actions:**
1. Check capacity (`current_rsvps + numAttendees ≤ max_capacity`)
2. Create `event_rsvps` row
3. Update `events.current_rsvps` count
4. Add email to events audience
5. If `requires_payment`: create Stripe Checkout Session → return URL
6. If free: send confirmation email immediately

#### `POST /api/donate`

Donation processing. One-time or recurring.

```typescript
const donateSchema = z.object({
  amount: z.number().min(1),
  fund: z.enum(['general', 'scholarship', 'garden', 'art', 'playground']).optional(),
  recurring: z.boolean().default(false),
  donorName: z.string(),
  donorEmail: z.string().email(),
  anonymous: z.boolean().default(false),
  message: z.string().optional(),
})
```

**Actions:**
1. If one-time: create Stripe Payment Intent → return client secret for Elements
2. If recurring: create Stripe Subscription with monthly price
3. Send tax-deductible receipt email (include EIN from `school_settings`)
4. Log donation for admin reporting

---

### Authenticated API Routes (Portal)

All portal routes use Supabase session from cookies. RLS enforces data access at the database level.

#### `GET /api/portal/dashboard`

Aggregate dashboard data for the logged-in parent.

**Auth:** `parent`
**Returns:** Combined response with today's snapshot, action items, upcoming events, recent messages, and quick action availability.

```typescript
// Returns:
{
  todaySnapshot: {
    children: Array<{ name, classroom, checkedIn, checkedInAt, schedule }>,
    upcomingEvents: Event[],
    afterSchoolToday: Signup[],
  },
  actionItems: Array<{
    type: 'form' | 'payment' | 'immunization' | 'permission_slip',
    title: string,
    dueDate: string | null,
    urgency: 'high' | 'medium' | 'low',
    deepLink: string,
  }>,
  recentMessages: Message[],
  announcements: Message[],
}
```

#### `GET /api/attendance` and `POST /api/attendance`

Get attendance roster for a classroom. Check in or check out a child.

**Auth:** `staff`, `admin`
**GET query:** `?classroomId=<uuid>&date=YYYY-MM-DD`
**POST body:**
```typescript
const attendanceSchema = z.object({
  childId: z.string().uuid(),
  classroomId: z.string().uuid(),
  action: z.enum(['check_in', 'check_out']),
  beforeCare: z.boolean().optional(),
  afterCare: z.boolean().optional(),
})
```

**POST actions:**
1. Insert or update `attendance` row (UPSERT on `child_id + date`)
2. If `beforeCare` or `afterCare`, create corresponding `signups` row for billing
3. Broadcast via Supabase Realtime (other staff in same classroom see instant update)
4. Log to `audit_log`
**Validation:** Cannot check out before check in. Cannot check in to wrong classroom.

#### `GET /api/messages` and `POST /api/messages`

Message threads and sending.

**Auth:** `parent`, `staff`, `admin`
**RLS:** Parents see only their threads. Staff see their classroom parent threads + admin. Admin sees all.

**POST body:**
```typescript
const messageSchema = z.object({
  recipientId: z.string().uuid(),
  threadId: z.string().uuid().optional(),
  subject: z.string().optional(),
  body: z.string().min(1).max(5000),
  messageType: z.enum(['direct', 'announcement', 'emergency']).default('direct'),
  attachments: z.array(z.string().url()).optional(),
})
```

**POST actions:**
1. Insert `messages` row
2. If new thread, generate `thread_id`
3. Look up recipient's notification preferences
4. Send notification: email, push, or SMS based on prefs (respect quiet hours unless emergency)
5. Broadcast via Supabase Realtime
**Validation:** Parents can only message their child's teachers + admin. Staff can only message parents in their classroom + admin. Admin can message anyone.

#### `POST /api/signups`

Create a signup for before/after care, field trips, clubs, volunteers, conferences.

```typescript
const signupSchema = z.object({
  childId: z.string().uuid().optional(),
  signupType: z.enum([
    'before_care', 'after_care', 'field_trip', 'club',
    'volunteer', 'mystery_reader', 'birthday_party',
    'hot_lunch_helper', 'veggie_week', 'play_doh', 'conference'
  ]),
  signupDate: z.string().date(),
  details: z.record(z.unknown()).optional(),
})
```

**Auth:** `parent`
**Actions:**
1. Check capacity (before/after care max 14 per day)
2. Insert `signups` row
3. If payment required (field trip, club): create invoice → send payment link
4. Send confirmation email
**Validation:** Cannot double-sign-up for same type + date. Capacity enforcement.

#### `GET /api/invoices`

Get invoices for a family with payment history.

**Auth:** `parent` (own family only), `admin` (any family)
**Query:** `?status=sent,overdue&year=2025`
**Returns:** Invoices with line items, payment history, outstanding balance.

#### `POST /api/photos` and `PATCH /api/photos/[id]`

Upload and curate photos.

**POST auth:** `staff`, `admin`
**POST body:** Multipart form with image file(s).
**POST actions:**
1. Validate file type (JPEG, PNG, HEIC) and size (max 5MB)
2. Process through Sharp pipeline (see `API_INTEGRATIONS.md`)
3. Upload all variants to Supabase Storage (`photos` bucket)
4. Insert `photos` row (`is_approved: false` — enters curation queue)
5. Tag with uploader's classroom ID automatically

**PATCH auth:** `admin`
**PATCH body:** `{ isApproved?, tags?, caption?, isPublic? }`
**Validation:** If `isPublic: true`, verify every tagged child has `photo_release = true` on their record. Block public sharing otherwise.

#### `POST /api/incident-reports`

Submit a digital incident report.

```typescript
const incidentSchema = z.object({
  childId: z.string().uuid(),
  classroomId: z.string().uuid(),
  incidentDate: z.string().date(),
  incidentTime: z.string().optional(),
  description: z.string().min(20),
  witnesses: z.string().optional(),
  actionTaken: z.string(),
  parentNotified: z.boolean().default(false),
  parentNotifiedBy: z.string().optional(),
})
```

**Auth:** `staff`, `admin`
**Actions:**
1. Insert `incident_reports` row
2. Send notification to admin
3. If `parentNotified: false`, create action item for Rebecca to contact parent
4. Attach to child's permanent record
5. Log to `audit_log`

---

### Admin-Only API Routes

#### `POST /api/admin/invoices/generate`

Bulk generate monthly invoices for all enrolled families.

**Auth:** `admin`
```typescript
const bulkInvoiceSchema = z.object({
  month: z.number().min(1).max(12),
  year: z.number(),
  includeAddOns: z.boolean().default(true),
  autoSend: z.boolean().default(false),
})
```

**Actions (background job):**
1. Create job record with status `pending`, return job ID immediately
2. For each enrolled family: calculate base tuition + recurring add-ons - work-trade credit
3. Create Stripe Invoice with line items → insert `invoices` row
4. If `autoSend`: send invoice via Stripe + notification email
5. Update job status: `completed` with summary
6. Broadcast completion via Realtime

#### `POST /api/admin/email/send`

Send mass email to a segment.

**Auth:** `admin`
```typescript
const emailSendSchema = z.object({
  subject: z.string().min(5).max(200),
  bodyHtml: z.string(),
  recipientSegment: z.enum([
    'all_families', 'classroom_sunflower', 'classroom_marigold', 'classroom_dandelion',
    'prospective', 'alumni', 'community', 'donors', 'staff', 'custom'
  ]),
  customRecipients: z.array(z.string().email()).optional(),
  scheduledAt: z.string().datetime().optional(),
})
```

**Actions:**
1. Create `email_campaigns` row
2. If immediate: resolve segment to email list → send via Resend Batch API
3. If scheduled: store for cron to pick up
4. Track opens + clicks via Resend webhooks

#### `POST /api/admin/email/sequence`

Create or update an automated email sequence.

**Auth:** `admin`
```typescript
const sequenceSchema = z.object({
  name: z.string(),
  triggerEvent: z.enum(['new_lead', 'post_tour', 'enrollment', 'immunization_due', 'birthday']),
  steps: z.array(z.object({
    delayDays: z.number().min(0),
    subject: z.string(),
    bodyHtml: z.string(),
  })),
  isActive: z.boolean().default(true),
})
```

#### `GET /api/admin/reports/[type]`

Generate admin or board reports.

**Auth:** `admin`, `board` (read-only)
**Types:**
- `enrollment` — pipeline funnel, conversion rates, capacity by classroom, waitlist count, year-over-year
- `financial` — revenue by month, outstanding balances, payment method breakdown, Stripe fees, work-trade value
- `attendance` — daily/weekly/monthly rates by classroom, before/after care usage, trends
- `compliance` — immunization status across all children, staff training hours by OCFS topic, certification expiry dates, incomplete forms
- `board` — executive summary combining all above, supports PDF export

**Board members:** Can only access `board` and `enrollment` report types. Financial details are aggregated (no family-level data).

#### `POST /api/admin/families/[familyId]/status`

Update family enrollment status (Kanban column move).

**Auth:** `admin`
```typescript
const statusSchema = z.object({
  newStatus: z.enum([
    'inquiry', 'touring', 'applied', 'accepted', 'enrolled',
    'waitlisted', 'withdrawn', 'graduated'
  ]),
  notes: z.string().optional(),
})
```

**Actions:**
1. Update `families.enrollment_status`
2. Trigger status-specific side effects:
   - `accepted` → Send acceptance email, create onboarding checklist
   - `enrolled` → Create user accounts for parents, assign classroom, start onboarding sequence
   - `waitlisted` → Send waitlist notification with position
   - `withdrawn` → Flag photos for review, send exit survey
3. Log old → new status in `audit_log`

---

### Webhook Routes

#### `POST /api/webhooks/stripe`

Stripe webhook handler. All payment lifecycle events.

```typescript
export async function POST(request: Request) {
  const body = await request.text()
  const signature = request.headers.get('stripe-signature')!

  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(body, signature, process.env.STRIPE_WEBHOOK_SECRET!)
  } catch (err) {
    return new Response('Webhook signature verification failed', { status: 400 })
  }

  switch (event.type) {
    case 'payment_intent.succeeded':
      // Update payments table → update invoice status to 'paid' → send receipt email
      break
    case 'invoice.paid':
      // Update subscription status → log
      break
    case 'invoice.payment_failed':
      // Send failure notification to family + admin → create action item
      break
    case 'customer.subscription.updated':
      // Update autopay status on family record
      break
    case 'charge.refunded':
      // Update payment + invoice status → send refund confirmation
      break
    case 'checkout.session.completed':
      // Finalize camp/event registration → send confirmation
      break
  }

  await auditLog(null, `stripe_webhook_${event.type}`, 'payments', event.id)
  return new Response('OK', { status: 200 })
}
```

**Critical:** Always verify webhook signature before processing. Use raw body (not JSON-parsed) for signature verification.

---

### Cron Jobs (Vercel Cron)

All cron routes are protected with a `CRON_SECRET` header check:

```typescript
export async function GET(request: Request) {
  const authHeader = request.headers.get('authorization')
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response('Unauthorized', { status: 401 })
  }
  // ... cron logic using supabaseAdmin (service role)
}
```

| Cron Job | Schedule | Description |
|---|---|---|
| `/api/cron/email-sequences` | Every 15 min | Check `email_sequence_progress` for steps due. Send next email, increment step, calculate next send time. |
| `/api/cron/daily-digest` | 6:00 AM ET daily | Compile overnight messages + upcoming events + action items → single digest email for opted-in parents. |
| `/api/cron/immunization-reminders` | Monday weekly | Find upcoming (30 days) and overdue immunizations → send reminders to parents + admin. |
| `/api/cron/payment-reminders` | 9:00 AM ET daily | Send reminders at 7, 3, and 1 day before due. Mark invoices overdue past due date. |
| `/api/cron/certification-expiry` | Monday weekly | Check staff CPR/First Aid/training expiry within 30-60 days → alert admin + staff. |

---

## Server Actions

For simple mutations that benefit from progressive enhancement:

```typescript
// lib/actions/attendance.ts
'use server'

import { createServerSupabaseClient } from '@/lib/supabase/server'
import { revalidatePath } from 'next/cache'

export async function toggleAttendance(childId: string, action: 'check_in' | 'check_out') {
  const supabase = await createServerSupabaseClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('Unauthorized')

  await markAttendance(supabase, childId, action, user.id)
  revalidatePath('/staff/classroom')
}
```

**Use Server Actions for:** Attendance toggle, message read receipt, form status update, photo approval toggle, signup cancellation, notification preference update, quick reply send.

**Use Route Handlers for:** Complex operations with multiple side effects, file uploads, webhook handling, operations needing detailed error responses, anything called from external services.

---

## Middleware

### Auth Middleware (`/src/middleware.ts`)

```typescript
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request })
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll() },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value))
          response = NextResponse.next({ request })
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options)
          )
        },
      },
    }
  )

  // Refresh session — MUST be called on every middleware invocation
  const { data: { user } } = await supabase.auth.getUser()
  const path = request.nextUrl.pathname

  // Route protection
  if (path.startsWith('/portal') && !user) {
    return NextResponse.redirect(new URL('/login?redirect=' + encodeURIComponent(path), request.url))
  }
  if (path.startsWith('/admin')) {
    if (!user) return NextResponse.redirect(new URL('/login?redirect=' + encodeURIComponent(path), request.url))
    const { data: profile } = await supabase
      .from('users').select('role').eq('id', user.id).single()
    if (profile?.role !== 'admin') return NextResponse.redirect(new URL('/portal/dashboard', request.url))
  }
  if (path.startsWith('/staff')) {
    if (!user) return NextResponse.redirect(new URL('/login?redirect=' + encodeURIComponent(path), request.url))
    const { data: profile } = await supabase
      .from('users').select('role').eq('id', user.id).single()
    if (!['staff', 'enrichment', 'admin'].includes(profile?.role ?? '')) {
      return NextResponse.redirect(new URL('/portal/dashboard', request.url))
    }
  }

  return response
}

export const config = {
  matcher: ['/portal/:path*', '/admin/:path*', '/staff/:path*'],
}
```

### Rate Limiting

Public form routes use `@upstash/ratelimit` with Vercel KV:

```typescript
// lib/api/rate-limit.ts
import { Ratelimit } from '@upstash/ratelimit'
import { kv } from '@vercel/kv'

export const publicFormLimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.slidingWindow(5, '1 h'),
  analytics: true,
  prefix: 'ratelimit:form',
})

export const chatbotLimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.slidingWindow(20, '1 h'),
  analytics: true,
  prefix: 'ratelimit:chatbot',
})
```

---

## Error Handling Pattern

Every API route follows this structure:

```typescript
export async function POST(request: Request) {
  try {
    // 1. Rate limit (public routes)
    const ip = request.headers.get('x-forwarded-for') ?? 'unknown'
    const rateLimitResult = await checkRateLimit(ip, publicFormLimit)
    if (rateLimitResult) return rateLimitResult

    // 2. Parse and validate
    const body = await parseBody(request, schema)

    // 3. Auth + role check (authenticated routes)
    const user = await getAuthUser(request)
    requireRole(user, 'admin', 'staff')

    // 4. Business logic via query layer
    const result = await queryFunction(supabase, body)

    // 5. Side effects
    await sendConfirmationEmail(result)

    // 6. Audit log (sensitive actions)
    await auditLog(user.id, 'action_name', 'table_name', result.id)

    // 7. Return success
    return success(result)

  } catch (error) {
    if (error instanceof ZodError) {
      return badRequest('Validation failed', error.errors)
    }
    if (error instanceof DatabaseError) {
      console.error('Database error:', error)
      return NextResponse.json({ error: 'Database operation failed' }, { status: 500 })
    }
    if (error instanceof AuthorizationError) {
      return forbidden()
    }
    console.error('Unhandled API error:', error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

---

## Background Jobs Architecture

For operations that take > 5 seconds (bulk invoice generation, board report PDF, mass email):

1. API route creates a `background_jobs` record with status `pending`
2. Returns job ID immediately to the client
3. Processing continues (Vercel serverless, same or separate invocation)
4. Client listens for status updates via Supabase Realtime or polls every 2 seconds
5. Status flow: `pending` → `processing` → `completed` | `failed`

---

## Zod Schema Library

Shared schemas live in `/lib/validations/`:

```
lib/validations/
├── lead.ts           # Lead form schema
├── tour.ts           # Tour booking schema
├── contact.ts        # Contact form schema
├── enrollment.ts     # Application form schema
├── attendance.ts     # Check in/out schema
├── message.ts        # Message schemas
├── invoice.ts        # Invoice generation schema
├── photo.ts          # Photo upload/update schema
├── signup.ts         # Signup schemas
├── incident.ts       # Incident report schema
├── camp.ts           # Summer camp registration schema
├── event.ts          # Event CRUD + RSVP schemas
├── email.ts          # Email campaign/sequence schemas
└── shared.ts         # Common patterns (phone, UUID, date, etc.)
```

Common shared patterns:

```typescript
// lib/validations/shared.ts
export const phoneSchema = z.string().regex(/^\+?[\d\s\-().]{10,}$/).optional()
export const uuidSchema = z.string().uuid()
export const dateSchema = z.string().regex(/^\d{4}-\d{2}-\d{2}$/)
export const emailSchema = z.string().email().toLowerCase().trim()
export const paginationSchema = z.object({
  page: z.coerce.number().min(1).default(1),
  limit: z.coerce.number().min(1).max(100).default(20),
})
```

---

## Environment Variables

See `DEPLOYMENT.md` for the complete list. Backend-critical variables:

| Variable | Used In | Notes |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | All clients | Public |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | All clients | Public — RLS enforced |
| `SUPABASE_SERVICE_ROLE_KEY` | Admin client only | **NEVER expose to client** |
| `STRIPE_SECRET_KEY` | API routes only | Server-side only |
| `STRIPE_WEBHOOK_SECRET` | Webhook route | Verify signatures |
| `RESEND_API_KEY` | Email functions | Server-side only |
| `TWILIO_ACCOUNT_SID` | SMS functions | Server-side only |
| `TWILIO_AUTH_TOKEN` | SMS functions | Server-side only |
| `GOOGLE_CALENDAR_CREDENTIALS` | Calendar functions | JSON service account key |
| `ANTHROPIC_API_KEY` | Chatbot route | Server-side only |
| `CRON_SECRET` | Cron routes | Vercel cron authorization |
