# Design System — Little Friends Learning Loft

## Brand Identity

Warm, playful, community-rooted, nature-connected. NOT corporate, sterile, or over-designed. The brand should feel like walking into a cozy, vibrant classroom where mud-stained kids are happily building things with their hands.

## Color Palette

### Core Colors

| Token | Hex | CSS Variable | Usage |
|---|---|---|---|
| Primary Yellow | `#F5C518` | `--color-primary` | Brand identity, CTA backgrounds, accents |
| Secondary Yellow | `#FFF3C4` | `--color-primary-light` | Card backgrounds, highlights |
| Background | `#FFFDF7` | `--color-background` | Page backgrounds (warm off-white) |
| Text Primary | `#2D2926` | `--color-text` | Main body text (warm charcoal) |
| Text Secondary | `#6B6560` | `--color-text-muted` | Supporting text |
| Accent Green | `#4A7C59` | `--color-green` | Nature, success states |
| Accent Blue | `#5B8DB8` | `--color-blue` | Links, interactive, focus indicators |
| Accent Coral | `#E8775D` | `--color-coral` | Alerts, important CTAs |
| Border | `#E8E4DF` | `--color-border` | Dividers, card borders |
| Card White | `#FFFFFF` | `--color-card` | Content cards on cream background |

### Tailwind Config

```typescript
// tailwind.config.ts
const config = {
  theme: {
    extend: {
      colors: {
        primary: {
          DEFAULT: '#F5C518',
          light: '#FFF3C4',
          hover: '#E8B30F',
        },
        background: '#FFFDF7',
        foreground: '#2D2926',
        muted: {
          DEFAULT: '#6B6560',
          foreground: '#6B6560',
        },
        green: '#4A7C59',
        blue: '#5B8DB8',
        coral: '#E8775D',
        border: '#E8E4DF',
        card: '#FFFFFF',
      },
    },
  },
}
```

### Color Accessibility Rules

- Text on yellow backgrounds: ALWAYS use `#2D2926` — must meet 4.5:1 contrast ratio
- Yellow (`#F5C518`) on white: NEVER use for text. Only for backgrounds, borders, decorative elements, icons paired with text labels
- All interactive elements: visible focus indicator — `2px solid #5B8DB8` with 2px offset

### Gradient Accents

```css
/* Hero section overlay */
background: linear-gradient(to bottom, transparent, rgba(245, 197, 24, 0.1));

/* CTA button hover */
background: linear-gradient(135deg, #F5C518, #E8B30F);

/* Section dividers (breathing space) */
background: linear-gradient(to right, rgba(245, 197, 24, 0.05), transparent, rgba(245, 197, 24, 0.05));
```

### Glassmorphism (Sparingly)

```css
/* Sticky navbar */
backdrop-filter: blur(12px);
background: rgba(255, 253, 247, 0.8);

/* Photo overlay testimonial cards */
backdrop-filter: blur(8px);
background: rgba(255, 255, 255, 0.85);

/* DO NOT use on: forms, portal UI, admin dashboard, mobile navigation */
```

## Typography

### Font Families

| Usage | Font | Weight(s) | Fallback |
|---|---|---|---|
| Headings (H1–H3) | Fredoka One | 400 | system-ui, sans-serif |
| All other text | Nunito | 400, 500, 600, 700 | system-ui, sans-serif |
| Admin data tables | JetBrains Mono | 400 | monospace |

### Font Loading Strategy

```html
<!-- Critical path (preloaded in <head>) -->
<link rel="preload" href="/fonts/nunito-400-latin.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/fonts/nunito-700-latin.woff2" as="font" type="font/woff2" crossorigin>

<!-- Secondary (async) -->
<link rel="preload" href="/fonts/fredoka-one-400-latin.woff2" as="font" type="font/woff2" crossorigin>
```

All fonts: Latin subset only (~60% smaller), `font-display: swap`.

### Type Scale (Mobile-First)

| Element | Size | Line Height | Letter Spacing |
|---|---|---|---|
| Hero Heading | `clamp(2rem, 5vw, 3.5rem)` | 1.2 | -0.02em |
| Page H1 | `clamp(1.75rem, 4vw, 2.75rem)` | 1.2 | -0.02em |
| Section H2 | `clamp(1.5rem, 3.5vw, 2.25rem)` | 1.2 | -0.02em |
| Subsection H3 | `clamp(1.25rem, 3vw, 1.75rem)` | 1.3 | -0.01em |
| Body | `1rem` (16px min) | 1.6 | 0.01em |
| Small/Caption | `0.875rem` | 1.4 | 0.01em |

### Typography Refinements

```css
font-feature-settings: "liga" 1, "kern" 1;
max-width: 65ch; /* on text blocks */
```

## Spacing

Use Tailwind's default spacing scale. Key usage:

| Context | Spacing |
|---|---|
| Section padding (vertical) | `py-16 md:py-24` |
| Card padding | `p-6 md:p-8` |
| Between sections | `space-y-16 md:space-y-24` |
| Between cards in grid | `gap-6 md:gap-8` |
| Between form fields | `space-y-4` |
| Between text paragraphs | `space-y-4` |

## Border Radius

Everything gets rounded corners for the "playful" feel:

| Element | Radius |
|---|---|
| Cards, buttons, images | `rounded-2xl` (16px) |
| Input fields | `rounded-xl` (12px) |
| Badges, pills | `rounded-full` |
| Small elements | `rounded-lg` (8px) |

No sharp rectangles anywhere on the public site.

## Shadows

```css
/* Card default */
shadow-sm: 0 1px 2px rgba(45, 41, 38, 0.05);

/* Card hover */
shadow-md: 0 4px 6px rgba(45, 41, 38, 0.07);

/* Elevated (modals, dropdowns) */
shadow-lg: 0 10px 15px rgba(45, 41, 38, 0.1);
```

## Motion Design System

### Animation Library: Framer Motion

### Global Motion Tokens

```css
:root {
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);    /* Playful bounce */
  --ease-smooth: cubic-bezier(0.4, 0, 0.2, 1);          /* Standard */
  --ease-gentle: cubic-bezier(0.25, 0.1, 0.25, 1);      /* Soft entrance */
  --duration-micro: 150ms;
  --duration-short: 250ms;
  --duration-medium: 400ms;
  --duration-long: 600ms;
}
```

### Animation Specs

| Element | Trigger | Animation | Duration | Easing |
|---|---|---|---|---|
| Hero heading | Page load | Fade up 20px + scale 0.95→1 | 600ms, 200ms delay | gentle |
| Hero subtitle | Page load | Fade up, staggered 150ms after heading | 400ms | gentle |
| Hero CTA | Page load | Fade in + scale, staggered 300ms | 300ms | spring |
| Section headings | Scroll (20% visible) | Fade up 30px | 400ms | gentle |
| Feature cards | Scroll | Staggered fade-up, 100ms between | 400ms each | smooth |
| Testimonial quotes | Scroll | Fade in + yellow accent line grows down | 500ms | smooth |
| Gallery images | Scroll | Fade in + scale 0.97→1 | 350ms | gentle |
| Step cards | Scroll | Sequential left-to-right, line draws | 400ms/step | smooth |
| CTA buttons | Hover | Scale 1.02 + shadow + Y -2px | 150ms | smooth |
| Nav links | Hover | Underline grows from center | 200ms | smooth |
| Cards | Hover | Shadow + Y -4px | 200ms | smooth |
| Form fields | Focus | Yellow border + glow | 150ms | smooth |
| Success states | Action | Checkmark SVG path draw-in | 500ms | spring |
| Page transitions | Route change | Crossfade + vertical shift | 300ms | smooth |
| Confetti | Last form / payment | Particle animation (portal only) | 1000ms | spring |
| Mobile sticky bar | Scroll direction | Slide up/down | 200ms | smooth |

### Scroll-Triggered Rules

- Intersection Observer via Framer Motion `whileInView`
- Animate at 20% visible (`amount: 0.2`)
- Never re-animate on scroll up: `once: true`
- Max 3 simultaneous animations in viewport
- Stagger: 80–120ms between siblings

### Performance Rules

```
ALWAYS animate:  transform, opacity (GPU-composited)
NEVER animate:   width, height, top, left, margin, padding
ALWAYS use:      will-change (sparingly, only while animating)
ALWAYS use:      Intersection Observer (never scroll event listeners)
ALWAYS test:     4x CPU throttle in Chrome DevTools
```

### Reduced Motion

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

When reduced motion is active: replace all animations with simple 200ms opacity fades. No transforms, no staggering, no parallax.

## Component Patterns

### Feature Card
- Hand-drawn SVG icon (brand yellow or accent green, 20-30% opacity)
- Heading (Fredoka One)
- Description (Nunito, 2-3 sentences)
- Rounded corners (16px), warm shadow, hover elevation

### Testimonial Card
- Quote text (Nunito italic or regular)
- Attribution: parent first name + last initial, child's age range
- Yellow accent line on left side (4px wide, animates on scroll)
- Optional parent photo

### Step Card (Admissions Stepper)
- Numbered circle (brand yellow bg, charcoal text)
- Title (Fredoka One)
- Description
- Animated connecting line to next step (SVG, draws on scroll)

### CTA Banner
- Yellow background with subtle gradient
- Bold heading (Fredoka One)
- Action button (charcoal bg, white text)
- Full-width, rounded corners on inner container

### FAQ Accordion
- Question (Nunito 600 weight)
- Smooth expand animation (Framer Motion `AnimatePresence`)
- FAQ schema markup on each Q&A pair
- Plus/minus icon toggle

### Status Badge
- Colored pill: Enrolled (green), Waitlist (yellow), Pending (blue), Expired (coral)
- `rounded-full`, `text-sm`, `font-medium`

### Mobile Sticky CTA Bar
- 3 equal sections: Call, Book Tour, Chat
- Fixed bottom, z-50
- Slide animation on scroll direction change
- Only on public pages, appears after scrolling past hero

## Hand-Drawn Illustrations

A small SVG set in a loose, hand-drawn style:
- Leaf, sun, small hand, building blocks, watering can, paintbrush
- Used as subtle decorative elements beside headings, as dividers, in empty states
- Colored in brand yellow or accent green at 20–30% opacity
- Style: teacher's quick sketch, NOT clip art

## Photo Treatment

- All photos: slightly warm color grade, rounded corners (16px)
- Hero/full-bleed: subtle parallax at 0.3 ratio
- Gallery: masonry layout, lightbox on click, lazy loading
- Gallery hover: scale 1.03 + shadow elevation
- All photos auto-strip EXIF/GPS metadata on upload
