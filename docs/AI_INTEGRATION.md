# AI Integration — Little Friends Learning Loft

## Public Chatbot Specification

### Personality
Warm, knowledgeable, never pushy. Think: the helpful parent who already enrolled and is telling their friend about the school. Not a salesperson. Not overly enthusiastic. Naturally conversational.

### System Prompt

```
You are a friendly assistant for Little Friends Learning Loft, a Montessori preschool for ages 3–6 at the Newburgh Jewish Community Center (290 North St, Newburgh, NY 12550).

You are warm, helpful, and conversational — like a parent who already enrolled their child and is sharing their experience with a friend. You are NOT a salesperson. Never be pushy or use pressure tactics.

WHAT YOU KNOW:
- School serves children ages 3–6 in a Montessori multi-age environment
- 3 classrooms with a 2:12 teacher-to-student ratio
- Located inside the Newburgh JCC with access to: gym, art house, dance studio, library, social hall, 3 playgrounds, garden
- Founded by Rebecca Harrison (BFA from SVA, AMS Montessori certification)
- Enrichment programs: art, yoga/dance, music, skateboarding (seasonal)
- Before and after care available
- Summer camp program
- The school is secular — housed at the JCC but all families welcome regardless of background
- School hours, general schedule, admissions process steps
- FAQ answers loaded from the database

WHAT YOU MUST NEVER SHARE:
- Tuition amounts or any pricing information
- Staff personal information (names other than Rebecca, contact details)
- Any student or family information
- Specific enrollment availability numbers
- Medical or developmental advice

WHEN ASKED ABOUT TUITION/COST:
Respond: "Our tuition varies by program and schedule. I'd love to connect you with Rebecca for a personalized conversation — would you like to schedule a tour? That's the best way to learn about everything, including pricing."

WHEN YOU CAN'T ANSWER:
Respond: "That's a great question for Rebecca. Would you like me to help you reach her?" Then offer: schedule a tour, send an inquiry form, call the school, or email Rebecca.

IF SOMEONE SEEMS DISTRESSED OR UPSET:
Do not continue the conversation. Gently route them to Rebecca directly with her phone number and email.

Always identify yourself as an AI assistant when asked.
```

### Knowledge Base Sources

The chatbot's knowledge is loaded dynamically:
1. **FAQ table** — all published FAQs from the database
2. **Program descriptions** — from CMS pages table
3. **School settings** — address, phone, email, hours
4. **Current enrollment status** — from school_settings table
5. **Upcoming events** — next 5 public events

### Key Conversation Flows

| User Intent | Bot Response |
|---|---|
| "What ages do you accept?" | Direct answer: "ages 3–6" + link to programs |
| "How much does it cost?" | Deflect to tour: "Our tuition varies..." + tour scheduling |
| "Is there a waitlist?" | Check `school_settings.enrollment_status` → answer accordingly |
| "Can I visit?" | "Absolutely! Here are the next available tour times..." → calendar |
| "Do I have to be Jewish?" | "Not at all! Little Friends is a secular program..." |
| "What's the ratio?" | "2 teachers for every 12 children" |
| "What hours?" | School hours from settings |
| Can't answer | "Great question for Rebecca. Want me to help you reach her?" |

### UI Treatment

**Trigger Button:**
- Bottom-right floating button
- Friendly icon: speech bubble or sun (NOT a robot)
- Subtle pulse animation on first visit (once)
- Does not appear on portal/admin/staff pages

**Chat Panel:**
- Slide-in panel from bottom-right (not full-screen overlay)
- Max height: 70vh on desktop, 85vh on mobile
- Warm background matching brand
- Rounded corners, subtle shadow

**Chat Interface:**
- Greeting: "Hi there! I'm here to answer your questions about Little Friends. What would you like to know?"
- Typing indicator while waiting for response
- Quick-reply chips: "What ages?", "Tour times", "Programs", "Talk to Rebecca"
- "Talk to a human" option always visible at bottom
- Clear AI disclosure: small text "AI Assistant" label

### Technical Implementation

```typescript
// /api/chat/route.ts
import Anthropic from '@anthropic-ai/sdk'
import { createAdminClient } from '@/lib/supabase/admin'

export async function POST(request: NextRequest) {
  const { message, conversationHistory } = await request.json()
  
  // Load dynamic knowledge
  const supabase = createAdminClient()
  const [faqs, settings, events] = await Promise.all([
    supabase.from('faqs').select('question, answer_html').eq('is_published', true),
    supabase.from('school_settings').select('key, value'),
    supabase.from('events').select('title, start_datetime, location')
      .eq('is_public', true).gte('start_datetime', new Date().toISOString()).limit(5),
  ])
  
  const knowledgeContext = buildKnowledgeContext(faqs.data, settings.data, events.data)
  
  const anthropic = new Anthropic()
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 500,
    system: SYSTEM_PROMPT + '\n\nCURRENT KNOWLEDGE:\n' + knowledgeContext,
    messages: [
      ...conversationHistory,
      { role: 'user', content: message }
    ],
  })
  
  return NextResponse.json({ 
    response: response.content[0].text,
    suggestedActions: extractActions(response.content[0].text) 
  })
}
```

### Rate Limiting
- 20 messages per session per hour
- Session tracked via anonymous ID (localStorage or cookie)
- After limit: "I've been chatting a lot! For more detailed questions, you can reach Rebecca directly at [phone] or [email]."

### Analytics
Track chatbot interactions:
- Total conversations started
- Most common questions (cluster by intent)
- Conversion: chat → tour booking, chat → inquiry form
- Drop-off points
- Questions the bot couldn't answer (for FAQ improvement)

## Admin Content Assist

### Blog Draft Generation
- Admin provides: topic, target AEO question, outline (optional)
- Claude generates: 800–1,500 word draft in LFLL voice
- Admin edits, adds photos, publishes
- Model: `claude-sonnet-4-20250514`

### Social Media Captions
- Admin selects curated photo(s)
- Claude suggests 2-3 caption options for Instagram/Facebook
- Warm, authentic voice — not marketing-speak
- Include relevant hashtags

### Press Release Drafts
- Admin provides: event details, key quotes, distribution targets
- Claude generates: formatted press release draft
- Admin reviews and sends to press contacts

### Email Draft Assistance
- Help writing parent communications
- Tone checking: ensure messages sound warm, not corporate
- Variable insertion preview

### Grant Writing Support
- Load grant requirements and past applications
- Help draft responses to specific grant questions
- Maintain LFLL-specific data points for reuse

### Curriculum Brainstorming
- Generate activity ideas based on: theme, age group, season, available materials
- Montessori-aligned suggestions
- Cross-reference with JCC facilities available

### Implementation Notes
- All content assist features are admin-only
- Generated content is always a draft — never auto-published
- Use `claude-sonnet-4-20250514` for all content generation (balances quality/speed/cost)
- Cache FAQ knowledge base (refresh every hour or on FAQ update)
- Log all chatbot conversations for quality review (anonymized)
