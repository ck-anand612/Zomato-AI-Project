---
name: Gourmet Intelligence
colors:
  surface: '#f7f9ff'
  surface-dim: '#c9dcf3'
  surface-bright: '#f7f9ff'
  surface-container-lowest: '#ffffff'
  surface-container-low: '#edf4ff'
  surface-container: '#e3efff'
  surface-container-high: '#d9eaff'
  surface-container-highest: '#d1e4fb'
  on-surface: '#091d2e'
  on-surface-variant: '#594139'
  inverse-surface: '#203243'
  inverse-on-surface: '#e8f2ff'
  outline: '#8d7168'
  outline-variant: '#e1bfb5'
  surface-tint: '#ab3500'
  primary: '#ab3500'
  on-primary: '#ffffff'
  primary-container: '#ff6b35'
  on-primary-container: '#5f1900'
  inverse-primary: '#ffb59d'
  secondary: '#b6181d'
  on-secondary: '#ffffff'
  secondary-container: '#da3532'
  on-secondary-container: '#fffbff'
  tertiary: '#bb162c'
  on-tertiary: '#ffffff'
  tertiary-container: '#ff676a'
  on-tertiary-container: '#6a0012'
  error: '#ba1a1a'
  on-error: '#ffffff'
  error-container: '#ffdad6'
  on-error-container: '#93000a'
  primary-fixed: '#ffdbd0'
  primary-fixed-dim: '#ffb59d'
  on-primary-fixed: '#390c00'
  on-primary-fixed-variant: '#832600'
  secondary-fixed: '#ffdad6'
  secondary-fixed-dim: '#ffb4ac'
  on-secondary-fixed: '#410003'
  on-secondary-fixed-variant: '#93000e'
  tertiary-fixed: '#ffdad8'
  tertiary-fixed-dim: '#ffb3b1'
  on-tertiary-fixed: '#410007'
  on-tertiary-fixed-variant: '#92001c'
  background: '#f7f9ff'
  on-background: '#091d2e'
  surface-variant: '#d1e4fb'
  background-off-white: '#F8F9FA'
  surface-white: '#FFFFFF'
  zomato-red: '#E23744'
  ink-dark: '#1C1C1C'
typography:
  display-lg:
    fontFamily: Lexend
    fontSize: 48px
    fontWeight: '700'
    lineHeight: 56px
    letterSpacing: -0.02em
  headline-lg:
    fontFamily: Lexend
    fontSize: 32px
    fontWeight: '600'
    lineHeight: 40px
  headline-lg-mobile:
    fontFamily: Lexend
    fontSize: 28px
    fontWeight: '600'
    lineHeight: 36px
  headline-md:
    fontFamily: Lexend
    fontSize: 24px
    fontWeight: '600'
    lineHeight: 32px
  body-lg:
    fontFamily: Inter
    fontSize: 18px
    fontWeight: '400'
    lineHeight: 28px
  body-md:
    fontFamily: Inter
    fontSize: 16px
    fontWeight: '400'
    lineHeight: 24px
  label-md:
    fontFamily: Inter
    fontSize: 14px
    fontWeight: '600'
    lineHeight: 20px
    letterSpacing: 0.01em
  label-sm:
    fontFamily: Inter
    fontSize: 12px
    fontWeight: '500'
    lineHeight: 16px
    letterSpacing: 0.04em
rounded:
  sm: 0.25rem
  DEFAULT: 0.5rem
  md: 0.75rem
  lg: 1rem
  xl: 1.5rem
  full: 9999px
spacing:
  base-unit: 4px
  container-max-width: 1280px
  gutter-desktop: 24px
  margin-desktop: 40px
  gutter-mobile: 16px
  margin-mobile: 20px
---

## Brand & Style
The design system centers on a "Modern Appétit" aesthetic—blending the utility of a high-tech AI recommender with the warm, sensory appeal of the culinary world. It targets urban food enthusiasts who seek curated, reliable dining experiences without the fatigue of endless scrolling.

The visual style is **Corporate / Modern** with a **Tactile** edge. It prioritizes high-quality food photography and clear information hierarchy to build trust. Interfaces are clean and airy, using generous whitespace to allow food imagery and AI-driven insights to breathe, while subtle micro-interactions provide a sense of personalized responsiveness.

## Colors
The palette is designed to be "appetizing." The **Primary Orange** (#FF6B35) serves as the main action color, stimulating appetite and suggesting energy. The **Warm Red** and **Zomato Red** are used sparingly for critical brand touchpoints and status indicators (like "Hot Right Now" tags).

**Dark Text** (#2C3E50) ensures high legibility and a grounded, professional feel. The background is strictly **Off-White** (#F8F9FA) to reduce glare and differentiate surface layers from the pure white (#FFFFFF) used for cards and interactive containers.

## Typography
The system uses a dual-font approach. **Lexend** is the display typeface, chosen for its excellent readability and modern, friendly character—perfect for headlines and restaurant names. **Inter** handles all body copy and functional UI labels, providing a systematic and neutral foundation that stays out of the way of the content.

Headlines should use tight letter-spacing to feel impactful. Body text maintains a generous line height to ensure descriptions of dishes and reviews are easy to scan.

## Layout & Spacing
The layout follows a **Fluid Grid** model based on a 12-column structure for desktop and a 4-column structure for mobile. 

A strict 8px/4px spatial rhythm is enforced across all components. Vertical rhythm is prioritized to create a "feed" feel that encourages discovery. 
- **Desktop:** Use a 1280px max-width container with 40px outer margins.
- **Mobile:** Elements should span the full width minus the 20px side margins. 
- **AI Sections:** Special "AI Insight" sections should use increased internal padding (32px) to distinguish them from standard search results.

## Elevation & Depth
Depth is conveyed through **Tonal Layers** supplemented by **Ambient Shadows**. 

1. **Level 0 (Background):** Off-White (#F8F9FA).
2. **Level 1 (Cards/Surfaces):** Pure White (#FFFFFF) with a very soft, diffused shadow (0px 4px 20px rgba(44, 62, 80, 0.08)).
3. **Level 2 (Hover/Active states):** Increased shadow spread and a slight 2% scale-up effect to simulate physical lift.

Avoid heavy borders; use light 1px strokes in a pale neutral (hex: #E9ECEF) only when cards sit on an identical white background.

## Shapes
The shape language is consistently **Rounded**, evoking a sense of friendliness and comfort. 
- **Standard Components:** Buttons, input fields, and small cards use 0.5rem (8px).
- **Restaurant Cards:** Larger containers use `rounded-lg` (16px) to create a soft, modern look.
- **Search Bars & Pills:** Use a full "Pill" radius (9999px) to signify fluid, dynamic interaction.

## Components

### Buttons & CTAs
Primary buttons utilize the Primary Orange (#FF6B35) with white text. Apply a `transform: scale(0.98)` on press and `scale(1.02)` on hover for a tactile, "squishy" feel. Secondary buttons should use a ghost style with a 1px stroke of the primary color.

### Restaurant Cards
Cards are the core of the experience. They feature a high-aspect-ratio image (16:9) at the top, followed by 16px of padding for the restaurant title (Lexend) and metadata. AI-recommendation badges should be overlaid on the top-right of the image using a semi-transparent glassmorphic blur.

### Input Fields
Inputs use the Off-White background with a subtle inner shadow to feel "recessed." On focus, the border transitions to Primary Orange with a soft glow.

### AI Recommender Chips
Use small, pill-shaped chips for filters (e.g., "Spicy," "Romantic," "Value for Money"). When selected, these chips fill with the Primary Orange; when inactive, they use a light grey background with Dark Text.

### Selection States
Checkboxes and radio buttons use the Primary Orange for the "checked" state. Ensure the hit area is a minimum of 44x44px for mobile accessibility.