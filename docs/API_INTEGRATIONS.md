# API_INTEGRATIONS.md — External Service Integrations

> Complete specification for all third-party service integrations.
> Cross-references: `BACKEND.md` (route handlers), `DEPLOYMENT.md` (env variables, service setup), `DATA_MODEL.md` (related tables).

---

## Integration Overview

| Service | Purpose | Phase | Priority | Monthly Cost |
|---|---|---|---|---|
| Supabase | Database, auth, storage, realtime | 1 | Critical | $25 (Pro) |
| Stripe | All payment processing | 4 | Critical | ~2.9% + 30¢/txn |
| Resend | Transactional + marketing email | 1 | Critical | Free tier (3k/mo) |
| Google Calendar API | Tour scheduling, school calendar | 2 | Critical | Free |
| Google Maps API | Embedded map, directions | 1 | Standard | Free tier |
| Sharp | Server-side image processing | 1 | Critical | Free (npm) |
| Twilio | SMS, emergency broadcasts, business phone | 3 | Standard | ~$5-10/mo |
| Anthropic (Claude) | AI chatbot | 6 | Enhancement | ~$5-15/mo |
| Vercel KV | Rate limiting | 1 | Standard | Included in Pro |

---

## Stripe Payment API (Critical — Phase 4)

### Purpose

All payment processing. Replaces Brightwheel billing, Venmo, and manual check tracking. Rebecca's requirement: digital payments only — the app handles credit card and ACH. Cash/check handled offline by accountant.

### Stripe Account Configuration

**Products to create in Stripe Dashboard:**

| Product | Pricing Model | Notes |
|---|---|---|
| Monthly Tuition | Subscription, fixed monthly | Amount from `school_settings.tuition_monthly` |
| Before Care (Daily) | One-time, per occurrence | Charged when parent signs up or staff marks attendance |
| After Care (Daily) | One-time, per occurrence | Same as above |
| Before Care (Weekly) | One-time, per week | Discounted weekly rate |
| After Care (Weekly) | One-time, per week | Discounted weekly rate |
| Field Trip | One-time, variable | Per-trip pricing |
| After-School Club | One-time, per session/semester | Variable by club |
| Summer Camp Week | One-time, per week | Per-week pricing, set in `camp_weeks` table |
| Event Ticket | One-time, variable | Per-event pricing |
| Donation — General | One-time or recurring | Custom amount |
| Donation — Scholarship | One-time or recurring | Custom amount, specific fund |

**Stripe Configuration:**

```typescript
// lib/stripe/client.ts
import Stripe from 'stripe'

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-12-18.acacia',
  typescript: true,
})
```

### Customer Management

Every enrolled family gets a Stripe Customer linked via `families.stripe_customer_id`:

```typescript
// lib/stripe/customers.ts
export async function createStripeCustomer(family: Family, primaryContact: User) {
  const customer = await stripe.customers.create({
    email: primaryContact.email,
    name: family.family_name,
    phone: primaryContact.phone ?? undefined,
    metadata: {
      family_id: family.id,
      primary_contact_id: primaryContact.id,
    },
  })
  // Update family record with stripe_customer_id
  await supabaseAdmin.from('families')
    .update({ stripe_customer_id: customer.id })
    .eq('id', family.id)
  return customer
}

export async function getOrCreateCustomer(family: Family, primaryContact: User) {
  if (family.stripe_customer_id) {
    return stripe.customers.retrieve(family.stripe_customer_id)
  }
  return createStripeCustomer(family, primaryContact)
}
```

### Invoice Generation

```typescript
// lib/stripe/invoicing.ts
export async function generateMonthlyInvoice(
  family: Family,
  month: number,
  year: number,
  includeAddOns: boolean
) {
  const customer = await getOrCreateCustomer(family, primaryContact)

  // Calculate line items
  const lineItems: Stripe.InvoiceItemCreateParams[] = []

  // 1. Base tuition
  const tuitionRate = await getSchoolSetting('tuition_monthly')
  lineItems.push({
    customer: customer.id,
    amount: Math.round(parseFloat(tuitionRate) * 100),
    currency: 'usd',
    description: `Monthly Tuition — ${monthName(month)} ${year}`,
  })

  // 2. Add-ons (before/after care signups for the month)
  if (includeAddOns) {
    const addOns = await getMonthlyAddOns(family.id, month, year)
    for (const addOn of addOns) {
      lineItems.push({
        customer: customer.id,
        amount: Math.round(addOn.amount * 100),
        currency: 'usd',
        description: addOn.description,
      })
    }
  }

  // 3. Subtract work-trade credit
  if (family.work_trade && family.work_trade_monthly_credit > 0) {
    lineItems.push({
      customer: customer.id,
      amount: -Math.round(family.work_trade_monthly_credit * 100),
      currency: 'usd',
      description: 'Work-Trade Credit',
    })
  }

  // Create invoice items, then invoice
  for (const item of lineItems) {
    await stripe.invoiceItems.create(item)
  }

  const invoice = await stripe.invoices.create({
    customer: customer.id,
    collection_method: 'send_invoice',
    days_until_due: 15,
    metadata: { family_id: family.id, month: String(month), year: String(year) },
  })

  // Store in database
  await supabaseAdmin.from('invoices').insert({
    family_id: family.id,
    type: 'tuition',
    description: `${monthName(month)} ${year} Tuition`,
    line_items_json: lineItems,
    subtotal: calculateSubtotal(lineItems),
    work_trade_credit: family.work_trade_monthly_credit ?? 0,
    total: calculateTotal(lineItems),
    due_date: new Date(year, month - 1, 15).toISOString(),
    status: 'draft',
    stripe_invoice_id: invoice.id,
    billing_period_start: new Date(year, month - 1, 1).toISOString(),
    billing_period_end: new Date(year, month, 0).toISOString(),
  })

  return invoice
}
```

### Autopay (Subscriptions)

```typescript
// lib/stripe/subscriptions.ts
export async function enableAutopay(familyId: string, paymentMethodId: string) {
  const family = await getFamilyById(supabaseAdmin, familyId)
  const customer = await getOrCreateCustomer(family, primaryContact)

  // Attach payment method to customer
  await stripe.paymentMethods.attach(paymentMethodId, { customer: customer.id })
  await stripe.customers.update(customer.id, {
    invoice_settings: { default_payment_method: paymentMethodId },
  })

  // Create subscription for monthly tuition
  const subscription = await stripe.subscriptions.create({
    customer: customer.id,
    items: [{ price: process.env.STRIPE_TUITION_PRICE_ID }],
    payment_behavior: 'default_incomplete',
    metadata: { family_id: familyId },
  })

  // Store autopay settings
  await supabaseAdmin.from('autopay_settings').upsert({
    family_id: familyId,
    stripe_subscription_id: subscription.id,
    stripe_payment_method_id: paymentMethodId,
    is_active: true,
  })

  return subscription
}
```

### Checkout Sessions (Camp, Events, Donations)

```typescript
// lib/stripe/checkout.ts
export async function createCampCheckoutSession(registration: CampRegistration) {
  const lineItems = registration.week_ids.map(weekId => ({
    price_data: {
      currency: 'usd',
      product_data: { name: `Summer Camp — Week ${weekId}` },
      unit_amount: Math.round(weekPrice * 100),
    },
    quantity: 1,
  }))

  const session = await stripe.checkout.sessions.create({
    mode: 'payment',
    line_items: lineItems,
    success_url: `${process.env.NEXT_PUBLIC_SITE_URL}/summer-camp/confirmation?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.NEXT_PUBLIC_SITE_URL}/summer-camp/register`,
    customer_email: registration.parent_email,
    metadata: { registration_id: registration.id, type: 'camp' },
  })

  return session
}
```

### Webhook Events

See `BACKEND.md` for the complete webhook handler. Events handled:

| Event | Action |
|---|---|
| `payment_intent.succeeded` | Update `payments` + `invoices` status → send receipt email |
| `invoice.paid` | Update subscription status → log |
| `invoice.payment_failed` | Notify family + admin → create action item |
| `customer.subscription.updated` | Update `autopay_settings` |
| `charge.refunded` | Update `payments` + `invoices` → send refund confirmation |
| `checkout.session.completed` | Finalize camp/event registration → send confirmation |

### Fee Tracking

Credit card processing: ~2.9% + 30¢ per transaction. Tracked in `payments.processing_fee` for accountant/tax purposes. Admin billing dashboard shows: total collected, total fees, net revenue per month. Year-end fee summary exportable as CSV.

---

## Resend — Email Service (Critical — Phase 1)

### Purpose

All transactional and marketing emails. Replaces manual email, Brightwheel notifications, and Canva newsletter.

### Configuration

```typescript
// lib/email/resend.ts
import { Resend } from 'resend'

const resend = new Resend(process.env.RESEND_API_KEY)

export async function sendTransactional(
  to: string | string[],
  subject: string,
  react: React.ReactElement,  // React Email component
  options?: { replyTo?: string; tags?: { name: string; value: string }[] }
) {
  return resend.emails.send({
    from: 'Little Friends Learning Loft <hello@littlefriendslearningloft.com>',
    to,
    subject,
    react,
    reply_to: options?.replyTo ?? 'rebecca@littlefriendslearningloft.com',
    tags: options?.tags,
  })
}

export async function sendBatch(emails: BatchEmail[]) {
  return resend.batch.send(
    emails.map(e => ({
      from: 'Little Friends Learning Loft <hello@littlefriendslearningloft.com>',
      to: e.to,
      subject: e.subject,
      react: e.react,
    }))
  )
}
```

### Email Templates (React Email)

All templates in `/lib/email/templates/`, built with React Email for server-side rendering. Each template accepts typed props and produces responsive HTML.

**Transactional (Immediate):**

| Template | Trigger | Props |
|---|---|---|
| `InquiryConfirmation` | New lead form submission | `{ parentName, childName, programInterest }` |
| `TourConfirmation` | Tour booked | `{ parentName, tourDate, tourType, directions, parkingInfo }` |
| `TourReminder` | Day before tour (cron) | `{ parentName, tourDate, tourType }` |
| `ContactAutoReply` | Contact form submission | `{ name }` |
| `WelcomeToPortal` | Parent account created | `{ parentName, loginUrl, setupSteps }` |
| `PaymentReceipt` | Payment succeeded | `{ familyName, amount, invoiceDetails, receiptUrl }` |
| `PaymentFailed` | Payment failed | `{ familyName, amount, updatePaymentUrl }` |
| `PasswordReset` | Password reset requested | `{ resetUrl }` |
| `IncidentNotification` | Incident report filed | `{ parentName, childName, date, description, portalLink }` |
| `FormReminder` | Incomplete forms | `{ parentName, formName, dueDate, formLink }` |
| `DonationReceipt` | Donation processed | `{ donorName, amount, fund, ein, taxInfo }` |

**Automated Sequences:**

| Sequence | Trigger | Steps |
|---|---|---|
| Lead Nurture | New inquiry | 5 emails over 30 days: Welcome → Philosophy → Day-in-Life → Tour invite → Gentle follow-up |
| Post-Tour | Tour completed | 3 emails over 10 days: Thank you + FAQs → Application invite → Last chance |
| Family Onboarding | Enrolled | 5 emails over 14 days: Welcome → Portal setup → Paperwork guide → Orientation prep → First-day checklist |
| Immunization Due | 30 days before due | 2 emails: 30-day reminder → 7-day urgent |
| Payment Due | Invoice created | 3 emails: 7 days → 3 days → 1 day |

**Marketing:**

| Template | Frequency | Notes |
|---|---|---|
| `Newsletter` | Monthly | Drag-and-drop sections in admin builder, see `ADMIN_DASHBOARD.md` |
| `EventAnnouncement` | As needed | Event details + RSVP CTA |
| `BlogNotify` | On publish | Post excerpt + read more link |
| `ReEnrollment` | Annual (spring) | Re-enrollment form link + deadline |
| `FundraisingAppeal` | As needed | Donation CTA with impact story |

### Audience Segments (Resend Audiences)

| Segment ID | Description | Auto-added |
|---|---|---|
| `current_families` | All enrolled families | On enrollment |
| `prospective` | Leads not yet enrolled | On inquiry form |
| `alumni` | Graduated/withdrawn families | On status change |
| `community` | Newsletter signups | On newsletter form |
| `donors` | Past donors | On donation |
| `staff` | All staff + enrichment | On account creation |

### Deliverability Configuration

Domain: `littlefriendslearningloft.com`
Sender: `hello@littlefriendslearningloft.com`
Reply-to: `rebecca@littlefriendslearningloft.com`

DNS records (configured in domain provider):

| Type | Name | Value |
|---|---|---|
| TXT | `@` | `v=spf1 include:amazonses.com ~all` |
| CNAME | `resend._domainkey` | Resend-provided DKIM value |
| TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:dmarc@littlefriendslearningloft.com` |

Every marketing email includes an unsubscribe link (CAN-SPAM compliance). Open + click tracking enabled for campaign analytics.

---

## Google Calendar API (Critical — Phase 2)

### Purpose

Central scheduling system. Replaces Calendly for tours and consultations. Provides calendar overlays in admin dashboard.

### Integration Type

2-way sync with Rebecca's Google Workspace calendars via OAuth2 service account.

### Calendar IDs to Configure

| Calendar | Purpose | Read/Write |
|---|---|---|
| Rebecca's main calendar | Her personal availability, meetings | Read (availability check) |
| LFLL School Calendar | School-wide events, holidays, closures | Read + Write |
| Tour Availability | Dedicated calendar for tour slots | Read + Write |
| JCC Building Calendar | Gym, social hall, library availability | Read only |
| Enrichment: Art | Art teacher sessions | Read + Write |
| Enrichment: Yoga/Dance | Yoga/dance sessions | Read + Write |
| Enrichment: Music | Music sessions | Read + Write |

### Implementation

```typescript
// lib/google/calendar.ts
import { google } from 'googleapis'

const auth = new google.auth.GoogleAuth({
  credentials: JSON.parse(process.env.GOOGLE_CALENDAR_CREDENTIALS!),
  scopes: ['https://www.googleapis.com/auth/calendar', 'https://www.googleapis.com/auth/calendar.events'],
})

const calendar = google.calendar({ version: 'v3', auth })

// Get available tour slots for a date range
export async function getAvailableSlots(
  startDate: string,
  endDate: string
): Promise<TimeSlot[]> {
  // 1. Get tour calendar events (existing bookings)
  const tourEvents = await calendar.events.list({
    calendarId: process.env.GOOGLE_CALENDAR_TOUR_ID!,
    timeMin: startDate,
    timeMax: endDate,
    singleEvents: true,
  })

  // 2. Get Rebecca's busy times
  const freeBusy = await calendar.freebusy.query({
    requestBody: {
      timeMin: startDate,
      timeMax: endDate,
      items: [
        { id: process.env.GOOGLE_CALENDAR_REBECCA_ID! },
        { id: process.env.GOOGLE_CALENDAR_JCC_ID! },
      ],
    },
  })

  // 3. Generate available slots from predefined windows minus conflicts
  return calculateAvailableSlots(tourEvents.data.items, freeBusy.data.calendars)
}

// Create a tour booking event
export async function createTourEvent(booking: TourBooking): Promise<string> {
  const event = await calendar.events.insert({
    calendarId: process.env.GOOGLE_CALENDAR_TOUR_ID!,
    requestBody: {
      summary: `Tour: ${booking.parentName}`,
      description: [
        `Parent: ${booking.parentName}`,
        `Email: ${booking.parentEmail}`,
        `Phone: ${booking.parentPhone ?? 'N/A'}`,
        `Child Age: ${booking.childAge ?? 'N/A'}`,
        `Type: ${booking.tourType}`,
        `Source: ${booking.howHeardAbout ?? 'N/A'}`,
      ].join('\n'),
      start: { dateTime: booking.scheduledDatetime, timeZone: 'America/New_York' },
      end: {
        dateTime: addMinutes(booking.scheduledDatetime, booking.durationMinutes ?? 60),
        timeZone: 'America/New_York',
      },
      reminders: { useDefault: false, overrides: [
        { method: 'email', minutes: 1440 },  // 24 hours
        { method: 'popup', minutes: 60 },
      ]},
    },
  })
  return event.data.id!
}

// Check for conflicts across multiple calendars
export async function checkConflicts(
  datetime: string,
  durationMinutes: number
): Promise<Conflict[]> {
  const endTime = addMinutes(datetime, durationMinutes)
  const freeBusy = await calendar.freebusy.query({
    requestBody: {
      timeMin: datetime,
      timeMax: endTime,
      items: [
        { id: process.env.GOOGLE_CALENDAR_REBECCA_ID! },
        { id: process.env.GOOGLE_CALENDAR_JCC_ID! },
        { id: process.env.GOOGLE_CALENDAR_TOUR_ID! },
      ],
    },
  })
  return parseConflicts(freeBusy.data.calendars)
}

// Create school event on school calendar
export async function createSchoolEvent(event: SchoolEvent): Promise<string> {
  const calEvent = await calendar.events.insert({
    calendarId: process.env.GOOGLE_CALENDAR_SCHOOL_ID!,
    requestBody: {
      summary: event.title,
      description: event.description,
      location: event.location,
      start: event.allDay
        ? { date: event.startDate }
        : { dateTime: event.startDatetime, timeZone: 'America/New_York' },
      end: event.allDay
        ? { date: event.endDate }
        : { dateTime: event.endDatetime, timeZone: 'America/New_York' },
    },
  })
  return calEvent.data.id!
}

// Update an existing event
export async function updateCalendarEvent(
  calendarId: string,
  eventId: string,
  updates: Partial<CalendarEventInput>
): Promise<void> {
  await calendar.events.patch({
    calendarId,
    eventId,
    requestBody: updates,
  })
}

// Delete/cancel an event
export async function deleteCalendarEvent(calendarId: string, eventId: string): Promise<void> {
  await calendar.events.delete({ calendarId, eventId })
}
```

### Tour Slot Configuration

Default tour windows (configurable in admin settings):
- Tuesday and Thursday: 9:30 AM, 10:30 AM
- One slot per time (individual tours)
- Group tours: first Saturday of month, 10:00 AM (when scheduled)
- Duration: 60 minutes (individual), 90 minutes (group)
- Blocked during: school holidays, summer break, enrichment sessions

### Enrichment Scheduling

Track per-teacher: sessions attended, cancellations, total hours, rate per session. Admin calendar overlay shows all enrichment across classrooms color-coded by teacher. Handle cancellations (reschedule or mark as cancelled, adjust hours).

---

## Twilio — SMS (Phase 3)

### Purpose

Emergency broadcasts, appointment reminders, and business phone number for school communications. Replaces Rebecca using her personal phone for school texts.

### Configuration

```typescript
// lib/sms/twilio.ts
import twilio from 'twilio'

const client = twilio(
  process.env.TWILIO_ACCOUNT_SID!,
  process.env.TWILIO_AUTH_TOKEN!
)

const SCHOOL_PHONE = process.env.TWILIO_PHONE_NUMBER!

export async function sendSMS(to: string, body: string): Promise<MessageResult> {
  const message = await client.messages.create({
    body,
    from: SCHOOL_PHONE,
    to: formatPhoneNumber(to),
  })
  return { sid: message.sid, status: message.status }
}

export async function sendBulkSMS(
  recipients: string[],
  body: string
): Promise<BatchResult> {
  const results = await Promise.allSettled(
    recipients.map(phone => sendSMS(phone, body))
  )
  return {
    total: recipients.length,
    sent: results.filter(r => r.status === 'fulfilled').length,
    failed: results.filter(r => r.status === 'rejected').length,
  }
}

export async function emergencyBroadcast(body: string): Promise<BroadcastResult> {
  // Get ALL family phone numbers — bypasses quiet hours
  const { data: families } = await supabaseAdmin
    .from('family_members')
    .select('phone')
    .not('phone', 'is', null)
    .in('family_id',
      supabaseAdmin.from('families').select('id').eq('enrollment_status', 'enrolled')
    )

  const phones = [...new Set(families.map(f => f.phone).filter(Boolean))]
  return sendBulkSMS(phones, `🚨 LFLL ALERT: ${body}`)
}
```

### Use Cases

| Use | Priority | Trigger | Quiet Hours |
|---|---|---|---|
| Emergency broadcast | Immediate | Admin sends | **OVERRIDE** — always sends |
| Snow day / closure | Immediate | Admin triggers | Override |
| Tour reminder | Day before | Cron job | Respect |
| Payment overdue | After grace period | Cron job | Respect |
| Pickup reminder | 30 min before close | Auto | Respect |

### Quiet Hours

Default: 9:00 PM – 7:00 AM (configurable per user in `notification_preferences`). NO SMS during quiet hours EXCEPT emergency broadcasts. Quiet hours check:

```typescript
export function isQuietHours(user: NotificationPreferences): boolean {
  const now = new Date()
  const currentTime = format(now, 'HH:mm', { timeZone: 'America/New_York' })
  return currentTime >= user.quiet_hours_start || currentTime < user.quiet_hours_end
}
```

### Business Phone Number

Twilio number replaces Rebecca's personal phone for:
- Inbound calls → voicemail → transcription → email to Rebecca
- Outbound SMS from the website/app system
- 2-way texting with parents (via admin interface)
- Caller ID shows school name

---

## Google Maps API (Phase 1)

### Purpose

Embedded map on contact page and directions integration.

### Implementation

```typescript
// components/public/SchoolMap.tsx
// Uses @vis.gl/react-google-maps or simple iframe embed

// Iframe embed (simplest, no API key needed for basic embed):
<iframe
  src="https://www.google.com/maps/embed?pb=!1m18!..."
  width="100%"
  height="400"
  style={{ border: 0 }}
  allowFullScreen
  loading="lazy"
  referrerPolicy="no-referrer-when-downgrade"
  title="Little Friends Learning Loft at Newburgh JCC"
/>

// Directions link (opens in Google Maps app on mobile):
const directionsUrl = `https://www.google.com/maps/dir/?api=1&destination=290+North+St+Newburgh+NY+12550`
```

### Static Maps (for emails)

```
https://maps.googleapis.com/maps/api/staticmap?
  center=290+North+St+Newburgh+NY+12550
  &zoom=15&size=600x300
  &markers=color:0xF5C518|290+North+St+Newburgh+NY+12550
  &key=GOOGLE_MAPS_API_KEY
```

Used in tour confirmation emails to show school location.

---

## Sharp — Image Processing (Critical — Phase 1)

### Purpose

Server-side image processing for all photo uploads. Strips EXIF/GPS metadata (CRITICAL for a school), generates responsive variants, converts to modern formats.

### Processing Pipeline

```
Upload (max 5MB, JPEG/PNG/HEIC) →
  1. Strip ALL EXIF/GPS metadata (Sharp .rotate() auto-applies orientation first)
  2. Resize: generate 400w, 800w, 1200w, 1600w variants (maintain aspect ratio)
  3. Format conversion:
     - WebP (primary — best quality/size, ~85% quality)
     - AVIF (progressive enhancement — ~70% quality, smallest files)
     - JPEG (fallback — ~85% quality, for older browsers)
  4. Thumbnail: 400px wide, 80% quality, JPEG (for curation queue)
  5. Store originals (EXIF-stripped) in Supabase Storage for high-quality prints
  6. Upload all variants to Supabase Storage
  7. Serve optimized via Vercel CDN / Supabase CDN
```

### Implementation

```typescript
// lib/images/pipeline.ts
import sharp from 'sharp'

const WIDTHS = [400, 800, 1200, 1600]
const FORMATS: Array<{ format: 'webp' | 'avif' | 'jpeg'; quality: number }> = [
  { format: 'webp', quality: 85 },
  { format: 'avif', quality: 70 },
  { format: 'jpeg', quality: 85 },
]

export async function processImage(buffer: Buffer): Promise<ProcessedImage> {
  // Step 1: Strip EXIF, auto-rotate, get metadata
  const stripped = sharp(buffer).rotate()  // .rotate() auto-applies EXIF orientation then strips
  const metadata = await stripped.metadata()

  // Step 2: Generate responsive variants
  const variants: ImageVariant[] = []

  for (const width of WIDTHS) {
    // Skip widths larger than original
    if (metadata.width && width > metadata.width) continue

    for (const { format, quality } of FORMATS) {
      const processed = await stripped
        .clone()
        .resize(width, null, { withoutEnlargement: true })
        .toFormat(format, { quality })
        .toBuffer()

      variants.push({
        width,
        format,
        buffer: processed,
        size: processed.length,
        key: `${width}w.${format}`,
      })
    }
  }

  // Step 3: Generate thumbnail
  const thumbnail = await stripped
    .clone()
    .resize(400, null, { withoutEnlargement: true })
    .jpeg({ quality: 80 })
    .toBuffer()

  // Step 4: Generate EXIF-stripped original (full resolution)
  const original = await stripped
    .clone()
    .jpeg({ quality: 95 })
    .toBuffer()

  return {
    variants,
    thumbnail,
    original,
    metadata: {
      width: metadata.width!,
      height: metadata.height!,
      format: metadata.format!,
    },
  }
}

// Validate upload before processing
export function validateImageUpload(file: File): ValidationResult {
  const MAX_SIZE = 5 * 1024 * 1024  // 5MB
  const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/heic', 'image/heif']

  if (file.size > MAX_SIZE) return { valid: false, error: 'File too large (max 5MB)' }
  if (!ALLOWED_TYPES.includes(file.type)) return { valid: false, error: 'Unsupported format' }
  return { valid: true }
}
```

### Storage Structure in Supabase

```
photos/
├── {photo_id}/
│   ├── original.jpg          # EXIF-stripped full resolution
│   ├── thumbnail.jpg         # 400px, curation queue
│   ├── 400w.webp             # Responsive variants
│   ├── 400w.avif
│   ├── 400w.jpg
│   ├── 800w.webp
│   ├── 800w.avif
│   ├── 800w.jpg
│   ├── 1200w.webp
│   ├── 1200w.avif
│   ├── 1200w.jpg
│   ├── 1600w.webp
│   ├── 1600w.avif
│   └── 1600w.jpg
```

### Next.js Image Integration

```tsx
// Use next/image with responsive srcSet pointing to Supabase Storage URLs
<Image
  src={photo.file_url}
  alt={photo.caption ?? 'School photo'}
  width={photo.width}
  height={photo.height}
  sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
  placeholder="blur"
  blurDataURL={photo.thumbnail_url}  // low-quality thumbnail as placeholder
  loading="lazy"
/>

// Hero image: preloaded in <head>
<Image src={heroUrl} priority={true} ... />
```

---

## Anthropic Claude API — AI Chatbot (Phase 6)

### Purpose

Warm, knowledgeable chatbot for the public website. Answers common questions, guides parents toward touring. See `AI_INTEGRATION.md` for full behavioral spec.

### Configuration

```typescript
// lib/ai/chatbot.ts
import Anthropic from '@anthropic-ai/sdk'

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY!,
})

export async function getChatbotResponse(
  userMessage: string,
  conversationHistory: Message[],
  faqContext: FAQ[]
): Promise<ChatbotResponse> {
  const systemPrompt = buildSystemPrompt(faqContext)

  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-5-20251001',
    max_tokens: 500,
    system: systemPrompt,
    messages: [
      ...conversationHistory.map(m => ({
        role: m.role as 'user' | 'assistant',
        content: m.content,
      })),
      { role: 'user', content: userMessage },
    ],
  })

  const text = response.content
    .filter(block => block.type === 'text')
    .map(block => block.text)
    .join('')

  return {
    message: text,
    suggestedActions: extractSuggestedActions(text),
  }
}
```

### System Prompt Structure

The system prompt is built dynamically from:
1. Base personality and behavioral rules (from `AI_INTEGRATION.md`)
2. Current FAQ content (from `faqs` table where `is_chatbot_knowledge = true`)
3. School settings (hours, address, programs, current enrollment status)
4. Current events (next 5 upcoming public events)

### Cost Control

- Model: Claude Haiku (cheapest, fast enough for chat)
- Max tokens per response: 500
- Max conversation length: 20 turns
- Rate limit: 20 messages per session per hour
- Estimated cost: $5-15/month at typical traffic

---

## Vercel KV — Rate Limiting (Phase 1)

### Purpose

Rate limiting for public form endpoints and chatbot using `@upstash/ratelimit`.

### Implementation

See `BACKEND.md` for rate limiting code. Limits:

| Endpoint | Limit | Window |
|---|---|---|
| `/api/leads` | 5 per IP | 1 hour |
| `/api/contact` | 3 per IP | 1 hour |
| `/api/tours` | 3 per IP | 1 hour |
| `/api/newsletter` | 3 per IP | 1 hour |
| `/api/chatbot` | 20 per session | 1 hour |
| `/api/careers/apply` | 2 per IP | 1 hour |

---

## Error Handling & Fallbacks

### Per-Service Fallback Strategy

| Service | Failure Mode | Fallback |
|---|---|---|
| Stripe | Payment fails | Show error, suggest retry, log for admin follow-up |
| Resend | Email fails | Queue for retry (3 attempts, exponential backoff), log failure |
| Google Calendar | API unavailable | Show manual booking form (email Rebecca directly), log |
| Twilio | SMS fails | Fall back to email notification, log failure |
| Claude API | Chatbot fails | Show "Our chatbot is resting" message + contact form link |
| Sharp | Processing fails | Reject upload with friendly error, log for debugging |

### Retry Pattern

```typescript
// lib/utils/retry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: { maxRetries: number; baseDelayMs: number; service: string }
): Promise<T> {
  for (let attempt = 0; attempt <= options.maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      if (attempt === options.maxRetries) {
        console.error(`${options.service} failed after ${options.maxRetries} retries:`, error)
        throw error
      }
      const delay = options.baseDelayMs * Math.pow(2, attempt)
      await new Promise(resolve => setTimeout(resolve, delay))
    }
  }
  throw new Error('Unreachable')
}

// Usage:
await withRetry(
  () => sendTransactional(to, subject, template),
  { maxRetries: 3, baseDelayMs: 1000, service: 'Resend' }
)
```

---

## Future Integrations (Phase 7)

### Instagram API

- Auto-post from CMS photo curation (approved + public photos)
- Feed widget on website showing latest posts
- Analytics pull for engagement tracking
- Requires Instagram Business Account + Facebook Graph API

### Google Business Profile API

- Post school updates directly from admin
- Review monitoring: alert Rebecca on new reviews
- Auto-reply templates for common review themes
- Keep business info (hours, photos, posts) in sync

### Social Scheduling (Buffer/Later)

- Cross-platform posting from curated photos
- Schedule posts from admin content management
- Integration from blog publish → social share

---

## Testing Strategy

### Service Mocking (Development)

| Service | Test Mode |
|---|---|
| Stripe | Use Stripe test mode + test cards (`4242424242424242`) |
| Resend | Use Resend test domain or check sent emails in dashboard |
| Google Calendar | Use a test calendar, not Rebecca's real calendar |
| Twilio | Use Twilio test credentials (messages go to logs, not real phones) |
| Claude API | Use real API with minimal test prompts (Haiku is cheap) |

### Integration Test Checklist

- [ ] Lead form → Resend email received → email sequence started
- [ ] Tour booking → Google Calendar event created → confirmation email sent
- [ ] Stripe payment → webhook received → invoice updated → receipt sent
- [ ] Emergency SMS → all enrolled family phones receive message
- [ ] Photo upload → Sharp processing → all variants in Storage → curation queue
- [ ] Chatbot → Claude API response → rate limit enforced
