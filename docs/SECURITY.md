# Security & Privacy — Little Friends Learning Loft

## Data Classification

| Level | Examples | Access |
|---|---|---|
| **Level 1 — Public** | Website content, blog posts, public events | Anyone |
| **Level 2 — Private** | Parent contact info, family details, payment info | Authenticated users with role access |
| **Level 3 — Sensitive** | Child records, medical info, immunizations, incident reports, CPSE/disability services, attendance | Strict role-based, RLS-enforced, audit logged |

## Authentication (Supabase Auth)

### Methods
- **Email/password** — primary method
- **Magic link** — for easier parent portal onboarding
- **Google SSO** — optional
- **Password reset** — standard flow via Resend
- **2FA** — required for admin accounts

### Session Management
- JWT-based sessions via Supabase Auth
- **Admin dashboard**: auto-logout after 30 minutes of inactivity
- **Parent portal**: auto-logout after 60 minutes of inactivity
- **Staff portal**: auto-logout after 45 minutes of inactivity
- Force re-authentication for: changing password, viewing billing details, downloading child records

### Brute-Force Protection
- Account lockout after 5 failed login attempts (15-minute lockout)
- Rate limiting on all auth endpoints: 5 attempts per IP per 15 minutes
- CAPTCHA on login after 3 failed attempts

## Role-Based Access Control (RBAC)

```
admin:       Full access to everything. All actions logged.
board:       Read-only: financial reports, enrollment data, compliance.
staff:       Classroom management, attendance, photos, incidents, own training.
             CANNOT see billing, financial data, other classrooms.
enrichment:  Own schedule, session attendance only.
parent:      Own family data, children, payments, messages, signups.
             CANNOT see other families' data.
public:      Unauthenticated website access only.
```

### Permission Matrix

| Resource | admin | board | staff | enrichment | parent |
|---|---|---|---|---|---|
| All families | ✅ R/W | ✅ R | ❌ | ❌ | ❌ |
| Own family | ✅ R/W | ❌ | ❌ | ❌ | ✅ R/W |
| All children | ✅ R/W | ✅ R | Classroom only | ❌ | ❌ |
| Own children | ✅ R/W | ❌ | ❌ | ❌ | ✅ R |
| All invoices | ✅ R/W | ✅ R | ❌ | ❌ | ❌ |
| Own invoices | ✅ R/W | ❌ | ❌ | ❌ | ✅ R |
| All messages | ✅ R/W | ❌ | Own classroom | ❌ | ❌ |
| Own messages | ✅ R/W | ❌ | ✅ R/W | ❌ | ✅ R/W |
| Attendance | ✅ R/W | ❌ | ✅ R/W (classroom) | Own sessions | ❌ |
| Compliance | ✅ R/W | ✅ R | Own training | ❌ | ❌ |
| Billing center | ✅ R/W | ✅ R | ❌ | ❌ | ❌ |
| CMS content | ✅ R/W | ❌ | ❌ | ❌ | ❌ |
| Audit log | ✅ R | ❌ | ❌ | ❌ | ❌ |

## Row-Level Security (Supabase RLS)

### Critical RLS Policies

```sql
-- Parents can only see their own family
CREATE POLICY "parents_own_family" ON families
  FOR ALL USING (
    auth.uid() IN (
      SELECT user_id FROM family_members WHERE family_id = families.id
    )
    OR auth.jwt()->>'role' = 'admin'
  );

-- Parents can only see their own children
CREATE POLICY "parents_own_children" ON children
  FOR SELECT USING (
    family_id IN (
      SELECT family_id FROM family_members WHERE user_id = auth.uid()
    )
    OR auth.jwt()->>'role' IN ('admin', 'board')
    OR (auth.jwt()->>'role' = 'staff' AND classroom_id IN (
      SELECT classroom_id FROM staff_profiles WHERE user_id = auth.uid()
    ))
  );

-- Parents can only see their own invoices
CREATE POLICY "parents_own_invoices" ON invoices
  FOR SELECT USING (
    family_id IN (
      SELECT family_id FROM family_members WHERE user_id = auth.uid()
    )
    OR auth.jwt()->>'role' IN ('admin', 'board')
  );

-- Messages: only sender or recipient
CREATE POLICY "message_participants" ON messages
  FOR SELECT USING (
    sender_id = auth.uid() OR recipient_id = auth.uid()
    OR auth.jwt()->>'role' = 'admin'
  );

-- Staff see only their classroom's attendance
CREATE POLICY "staff_classroom_attendance" ON attendance
  FOR ALL USING (
    auth.jwt()->>'role' = 'admin'
    OR classroom_id IN (
      SELECT classroom_id FROM staff_profiles WHERE user_id = auth.uid()
    )
  );
```

### RLS Testing Requirements

**CRITICAL: One family must NEVER be able to see another family's data.**

Test scenarios that MUST pass:
1. Parent A cannot see Parent B's children
2. Parent A cannot see Parent B's invoices
3. Parent A cannot see Parent B's messages
4. Parent A cannot see Parent B's forms/documents
5. Staff in Classroom 1 cannot see Classroom 2's attendance
6. Staff cannot see any billing/financial data
7. Enrichment teachers can only see their session data
8. Board members get read-only access (no writes)
9. All admin actions are logged in audit_log

## Photo Privacy Pipeline

```
Upload → Sharp strips ALL EXIF/GPS metadata → Compress → Generate variants
→ Store in Supabase Storage → Insert photo record (is_approved: false)
→ Admin reviews in curation queue → Checks child photo_release flags
→ If public use: ALL tagged children must have photo_release = true
→ Portal-only photos: can include all enrolled children
```

### Photo Rules
- All uploaded photos auto-strip EXIF/GPS metadata (no exceptions)
- Photos tagged with children's first names only (never last names)
- Public photos (blog, website) require `photo_release = true` on every tagged child
- When a family withdraws, their child's photos flagged for review
- Staff see guidelines reminder on upload screen
- Maximum upload size: 5MB (compressed via Sharp)

## COPPA Considerations

The site is directed at parents, not children, but best practices apply:
- Never expose child data to other parents
- Minimal data collection on public forms (no child data until inquiry stage)
- Clear privacy policy explaining child data collection and usage
- No third-party tracking that profiles children
- Photo release explicitly consented per child

## Audit Trail

All significant actions logged to `audit_log`:

| Action Category | Examples |
|---|---|
| Authentication | Login, logout, failed login, password change |
| Data modification | Create/update/delete on sensitive tables |
| Financial | Invoices created, payments processed, refunds |
| Document management | Forms submitted, approved, expired |
| Admin actions | Role changes, settings updates, bulk operations |
| Communication | Mass emails sent, emergency broadcasts |
| Content | Blog published, pages edited, photos approved |

Each entry captures: user_id, action, table_name, record_id, old_values, new_values, ip_address, timestamp.

## Data Retention

| Data Type | Retention Period | Method |
|---|---|---|
| Student records | 5 years (NYS requirement) | Soft delete (`deleted_at`) |
| Financial records | 7 years | Soft delete, no hard deletion |
| Audit log | Indefinite | Append-only |
| Lead data | 3 years after last contact | Soft delete |
| Withdrawn family photos | Flagged for review | Admin decision |
| Active family data | Duration of enrollment + retention period | Active |

## Backup Strategy

- **Database**: Daily automated backups (Supabase Point-in-Time Recovery)
- **Storage**: Supabase Storage with redundancy
- **Critical data export**: Weekly CSV export of enrollment, financial data (automated)
- **Disaster recovery**: Supabase provides automated backup/restore

## Cookie Consent & Privacy Compliance

- GDPR and CCPA compliant (good practice even for NY-based)
- Cookie consent banner for analytics cookies (Google Analytics 4, Vercel Analytics)
- Essential cookies (session, auth) don't require consent
- Privacy policy at `/privacy` linked in footer
- Terms of use at `/terms` linked in footer

## Security Headers

```typescript
// next.config.ts headers
{
  'X-Frame-Options': 'DENY',
  'X-Content-Type-Options': 'nosniff',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Permissions-Policy': 'camera=(), microphone=(), geolocation=()',
  'Content-Security-Policy': "default-src 'self'; ...",
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
}
```

## API Security

- All API routes validate Zod schemas on input
- Authenticated endpoints verify Supabase session
- Role-based middleware checks before data access
- Stripe webhooks verified by signature
- Cron endpoints verified by `CRON_SECRET` header
- Rate limiting on all public endpoints
- No sensitive data in URL parameters
- CORS configured for production domain only
