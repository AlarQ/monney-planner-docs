# Portfolio Style Propagation Guide for Documentation Repositories

## Overview
This guide provides instructions for AI assistants to propagate the Ernest Bednarczyk portfolio design system to documentation repositories (GitHub Pages). The goal is to maintain visual consistency across all project documentation.

---

## 1. Color Palette

### Primary Colors
```css
Primary (Blue): #0ea5e9
Secondary (Orange): #f97316
```

### Background Colors
```css
Default Background: #0a1118
Paper/Card Background: #141b22
Navbar Background: rgba(20, 27, 34, 0.85)
```

### Text Colors
```css
Primary Text: #ffffff (white)
Link Color (default): #0ea5e9
Link Color (hover): #7dd3fc
Link Color (active): #38bdf8
Dark Mode Foreground: #ededed
Light Mode Foreground: #171717
```

### Additional Colors
```css
Service Card Orange: #c55a0d
Service Card Lime Green: #5f9610
```

### Transparency & Effects
```css
Border Color: rgba(255, 255, 255, 0.12)
Backdrop Blur: blur(16px)
Mobile Drawer Background: rgba(20, 27, 34, 0.95)
Mobile Backdrop: rgba(0, 0, 0, 0.5)
```

---

## 2. Navigation Bar

### Structure
- **Position**: Fixed at top
- **Layout**: Horizontal with logo on left, nav links on right
- **Responsive**: Desktop nav for ≥ medium screens, mobile hamburger menu for < medium

### Desktop Navigation Styling
```javascript
Container:
  - Position: fixed
  - Top: 24px (mobile: 16px)
  - Left: 50% with translateX(-50%) centering
  - Z-index: 1000
  - Max-width: 1200px (mobile: calc(100% - 32px))
  - Padding: 24px horizontal on desktop, 16px on mobile

Inner Box:
  - Background: rgba(20, 27, 34, 0.85)
  - Backdrop filter: blur(16px)
  - Border: 1px solid rgba(255, 255, 255, 0.12)
  - Box shadow: 0 8px 32px rgba(0, 0, 0, 0.5), 0 2px 8px rgba(0, 0, 0, 0.3)
  - Border radius: 12px (borderRadius: 3 in MUI = 12px)
  - Padding: 12px vertical, 24px horizontal (16px on mobile)
```

### Logo/Name Styling
```javascript
  - Color: #ffffff
  - Font weight: 700
  - Font size: 1.25rem (desktop), 1rem (mobile)
  - Text shadow: 0 2px 4px rgba(0, 0, 0, 0.4)
```

### Navigation Links
```javascript
Base Style:
  - Color: #0ea5e9 (active: #38bdf8)
  - Text transform: uppercase
  - Font weight: 700
  - Font size: 0.9375rem (desktop), 0.8125rem (mobile)
  - Letter spacing: 0.05em
  - Text shadow: 0 2px 4px rgba(0, 0, 0, 0.4)

Hover State:
  - Color: #7dd3fc
  - Transform: translateY(-1px)
  - Scale: 1.05

Active Indicator:
  - Underline: 2px height
  - Color: Primary color (#0ea5e9)
  - Bottom position: -4px
  - Transform: scaleX(1) when active, scaleX(0) when inactive
  - Transition: transform 0.3s ease

Spacing:
  - Gap between links: 24px (gap: 3 in MUI)
```

### Mobile Navigation
```javascript
Hamburger Button:
  - Positioned right side of navbar
  - Interactive with visual feedback

Drawer:
  - Width: 80%, max 320px
  - Position: Fixed right side, full height
  - Background: rgba(20, 27, 34, 0.95)
  - Backdrop filter: blur(12px)
  - Box shadow: -8px 0 32px rgba(0, 0, 0, 0.5)
  - Z-index: 1300
  - Padding top: 64px
  - Padding horizontal: 32px
  - Slide animation from right

Drawer Backdrop:
  - Background: rgba(0, 0, 0, 0.5)
  - Z-index: 1200
  - Covers full viewport
  - Click to close

Link Layout in Drawer:
  - Vertical stack
  - Gap: 32px
  - Min height per link: 48px (touch-friendly)
  - Staggered fade-in animation (0.1s delay per item)
```

---

## 3. Typography

### Font Families
```css
Primary: Geist Sans (or system fallback: Arial, Helvetica, sans-serif)
Monospace: Geist Mono
CSS Variable: var(--font-geist-sans)
```

### Font Smoothing
```css
-webkit-font-smoothing: antialiased
-moz-osx-font-smoothing: grayscale
```

---

## 4. Spacing & Layout

### Page Layout
```javascript
Content Padding Top:
  - Mobile (xs): 56px (to clear fixed navbar)
  - Desktop (sm+): 64px

Max Width Patterns:
  - Navbar: 1200px
  - Adjust for mobile: calc(100% - 32px) on xs, calc(100% - 48px) on sm
```

### Component Padding
```javascript
Navbar:
  - Vertical: 12px
  - Horizontal: 16px (mobile), 24px (desktop)

Container:
  - Horizontal: 16px (xs), 24px (sm)
```

---

## 5. Responsive Breakpoints (Material-UI)

```javascript
xs: 0px      // Mobile
sm: 600px    // Small tablet
md: 900px    // Tablet/Desktop threshold
lg: 1200px   // Desktop
xl: 1536px   // Large desktop
```

### Key Responsive Changes
- **< md (900px)**: Switch to mobile hamburger menu
- **xs**: Smaller font sizes, reduced padding, narrower navbar
- **sm+**: Full desktop nav, increased spacing

---

## 6. Visual Effects

### Shadows
```css
Navbar Shadow:
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.5), 0 2px 8px rgba(0, 0, 0, 0.3)

Mobile Drawer Shadow:
  box-shadow: -8px 0 32px rgba(0, 0, 0, 0.5)

Text Shadow (for headers/nav):
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.4)
```

### Backdrop Blur
```css
Desktop Navbar: blur(16px)
Mobile Drawer: blur(12px)
```

### Border Radius
```css
Navbar Container: 12px
```

---

## 7. Animations & Transitions

### Link Hover Animation
```javascript
- Scale: 1.05
- Opacity: 1
- Transform: translateY(-1px)
- Duration: 0.2s ease
```

### Active Underline
```javascript
- Transform: scaleX(0) → scaleX(1)
- Duration: 0.3s ease
```

### Mobile Drawer Animation
```javascript
Type: Spring animation
Stiffness: 300
Damping: 30

States:
  - Closed: x: 100%
  - Open: x: 0

Backdrop:
  - Closed: opacity: 0
  - Open: opacity: 1
```

### Link Stagger (Mobile Drawer)
```javascript
Initial: { opacity: 0, x: 20 }
Animate: { opacity: 1, x: 0 }
Delay: index * 0.1s (each link delayed by 100ms)
```

---

## 8. Global Styles

### CSS Reset
```css
* {
  box-sizing: border-box;
  padding: 0;
  margin: 0;
}

html, body {
  max-width: 100vw;
  overflow-x: hidden;
}
```

### Body Styles
```css
body {
  color: var(--foreground);
  background: var(--background);
  font-family: Arial, Helvetica, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
```

### Links
```css
a {
  color: inherit;
  text-decoration: none;
}
```

### Dark Mode (Default)
```css
:root {
  --background: #0a0a0a;
  --foreground: #ededed;
}
```

---

## 9. Implementation Instructions for AI

### Step 1: Analyze Current Documentation Structure
1. Identify the documentation framework (Jekyll, Docusaurus, MkDocs, etc.)
2. Locate the main layout/template files
3. Find the CSS/styling configuration

### Step 2: Apply Color Palette
1. Create or update CSS variables with the portfolio color palette
2. Update background colors to match dark theme (#0a1118, #141b22)
3. Set text colors to white (#ffffff) and link colors to blue (#0ea5e9)
4. Apply the exact RGBA values for transparent elements

### Step 3: Implement Navigation Bar
1. Create/modify the header component to match portfolio navbar structure
2. Apply the fixed positioning with proper z-index (1000)
3. Implement the glassmorphism effect (backdrop-filter, border, shadow)
4. Add the exact padding, margin, and border-radius values
5. Style the logo/site name with matching typography
6. Style navigation links with uppercase, font-weight 700, and blue colors
7. Implement hover states (scale 1.05, color #7dd3fc, translateY)
8. Add active state underline animation

### Step 4: Add Responsive Behavior
1. Set breakpoint at 900px (md) for mobile/desktop switch
2. Implement hamburger menu for < 900px with slide-in drawer
3. Apply mobile-specific padding and font sizes
4. Ensure touch-friendly targets (48px min height) on mobile
5. Add backdrop blur effect to both desktop and mobile nav
6. Implement drawer close on backdrop click and Escape key

### Step 5: Apply Typography
1. Use Geist Sans or fallback to system fonts (Arial, Helvetica)
2. Apply font smoothing properties
3. Set proper font weights (700 for nav, headers)
4. Use exact letter-spacing (0.05em for nav links)

### Step 6: Set Spacing & Layout
1. Add padding-top to main content (56px mobile, 64px desktop)
2. Ensure navbar stays fixed at top (24px from top on desktop, 16px mobile)
3. Center navbar with 50% left + translateX(-50%)
4. Set max-width constraints (1200px for navbar content)

### Step 7: Add Visual Effects
1. Apply text shadows to headers and nav elements
2. Add box shadows to navbar (layered shadows for depth)
3. Implement backdrop-filter with 16px blur on navbar
4. Set border with rgba(255, 255, 255, 0.12)

### Step 8: Implement Animations
1. Add Framer Motion or CSS transitions for link hovers
2. Implement underline animation for active nav links
3. Add spring animation for mobile drawer (if possible)
4. Create staggered fade-in for mobile menu items

### Step 9: Test & Verify
1. Test on mobile (< 600px), tablet (600-900px), desktop (> 900px)
2. Verify navbar stays fixed and visible on scroll
3. Check hover states and active indicators
4. Ensure mobile menu opens/closes smoothly
5. Validate backdrop click closes mobile menu
6. Test Escape key functionality for mobile menu
7. Verify all colors match exactly

### Step 10: Maintain Consistency
1. Apply the same color palette to all pages
2. Use consistent spacing patterns throughout
3. Match button/link styles to navigation link styling
4. Keep typography consistent across documentation

---

## 10. Framework-Specific Notes

### For Jekyll (GitHub Pages Default)
- Create `_includes/header.html` with navbar structure
- Add styles to `assets/css/style.scss`
- Use CSS for animations (no Framer Motion)
- Implement mobile menu with vanilla JavaScript

### For Docusaurus
- Customize `src/theme/Navbar/index.js`
- Add styles to `src/css/custom.css`
- Override MUI theme if using
- Leverage built-in responsive utilities

### For MkDocs
- Create custom theme or override `main.html`
- Add styles to `docs/stylesheets/extra.css`
- Use JavaScript for mobile menu interactions
- Override Material theme variables if using Material theme

### For Plain HTML/CSS
- Create reusable header component
- Use CSS variables for easy color management
- Implement mobile menu with JavaScript
- Consider using a minimal framework for animations

---

## 11. Key Files to Reference

In the portfolio repository, reference these files:
- **Colors & Theme**: `src/theme/theme.ts`
- **Global Styles**: `src/app/globals.css`
- **Navbar Structure**: `src/components/navigation/Navigation.tsx`
- **Desktop Nav**: `src/components/navigation/DesktopNav.tsx`
- **Mobile Nav**: `src/components/navigation/MobileNav.tsx`
- **Link Component**: `src/components/navigation/NavLink.tsx`
- **Layout**: `src/app/layout.tsx`

---

## 12. CSS Variable Template

```css
:root {
  /* Colors */
  --primary-color: #0ea5e9;
  --secondary-color: #f97316;
  --background-default: #0a1118;
  --background-paper: #141b22;
  --text-primary: #ffffff;
  --text-secondary: #ededed;
  --link-default: #0ea5e9;
  --link-hover: #7dd3fc;
  --link-active: #38bdf8;

  /* Navbar */
  --navbar-bg: rgba(20, 27, 34, 0.85);
  --navbar-border: rgba(255, 255, 255, 0.12);
  --navbar-blur: 16px;
  --navbar-shadow: 0 8px 32px rgba(0, 0, 0, 0.5), 0 2px 8px rgba(0, 0, 0, 0.3);
  --navbar-radius: 12px;
  --navbar-z-index: 1000;

  /* Mobile */
  --drawer-bg: rgba(20, 27, 34, 0.95);
  --drawer-blur: 12px;
  --backdrop-bg: rgba(0, 0, 0, 0.5);
  --drawer-z-index: 1300;
  --backdrop-z-index: 1200;

  /* Typography */
  --font-family: 'Geist Sans', Arial, Helvetica, sans-serif;
  --font-weight-bold: 700;
  --letter-spacing-nav: 0.05em;

  /* Shadows */
  --text-shadow: 0 2px 4px rgba(0, 0, 0, 0.4);
  --drawer-shadow: -8px 0 32px rgba(0, 0, 0, 0.5);

  /* Breakpoints */
  --breakpoint-xs: 0px;
  --breakpoint-sm: 600px;
  --breakpoint-md: 900px;
  --breakpoint-lg: 1200px;
}
```

---

## 13. Checklist for AI Implementation

- [ ] Color palette applied to all elements
- [ ] Navbar created with glassmorphism effect
- [ ] Logo/name styled with correct typography
- [ ] Navigation links styled (uppercase, blue, bold)
- [ ] Hover states implemented (scale, color change)
- [ ] Active link underline animation added
- [ ] Mobile breakpoint set at 900px
- [ ] Hamburger menu implemented for mobile
- [ ] Slide-in drawer created with backdrop
- [ ] Backdrop blur applied to navbar and drawer
- [ ] All shadows and borders match specifications
- [ ] Responsive padding and spacing applied
- [ ] Font smoothing enabled
- [ ] Animations smooth and performant
- [ ] Mobile menu closes on backdrop click
- [ ] Mobile menu closes on Escape key
- [ ] Touch targets are minimum 48px on mobile
- [ ] Tested on multiple screen sizes
- [ ] All colors match exactly (no approximations)
- [ ] Documentation remains readable and functional

---

## 14. Example: Converting to Plain HTML/CSS

```html
<!-- HTML Structure -->
<header class="portfolio-navbar">
  <div class="navbar-container">
    <div class="navbar-inner">
      <div class="navbar-logo">Ernest Bednarczyk</div>

      <!-- Desktop Nav -->
      <nav class="desktop-nav">
        <a href="/" class="nav-link active">Home</a>
        <a href="/guide" class="nav-link">Guide</a>
        <a href="/api" class="nav-link">API</a>
      </nav>

      <!-- Mobile Menu Button -->
      <button class="hamburger-btn" aria-label="Toggle menu">
        <span></span>
        <span></span>
        <span></span>
      </button>
    </div>
  </div>
</header>

<!-- Mobile Drawer -->
<div class="mobile-drawer-backdrop"></div>
<div class="mobile-drawer">
  <nav class="mobile-nav">
    <a href="/" class="nav-link">Home</a>
    <a href="/guide" class="nav-link">Guide</a>
    <a href="/api" class="nav-link">API</a>
  </nav>
</div>
```

```css
/* CSS Implementation */
.portfolio-navbar {
  position: fixed;
  top: 24px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1000;
  width: 100%;
  max-width: 1200px;
  padding: 0 24px;
}

.navbar-inner {
  background-color: rgba(20, 27, 34, 0.85);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  border: 1px solid rgba(255, 255, 255, 0.12);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.5), 0 2px 8px rgba(0, 0, 0, 0.3);
  border-radius: 12px;
  padding: 12px 24px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.navbar-logo {
  color: #ffffff;
  font-weight: 700;
  font-size: 1.25rem;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.4);
}

.nav-link {
  color: #0ea5e9;
  text-transform: uppercase;
  font-weight: 700;
  font-size: 0.9375rem;
  letter-spacing: 0.05em;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.4);
  text-decoration: none;
  position: relative;
  transition: color 0.2s ease, transform 0.2s ease;
}

.nav-link:hover {
  color: #7dd3fc;
  transform: translateY(-1px) scale(1.05);
}

.nav-link.active::after {
  content: '';
  position: absolute;
  bottom: -4px;
  left: 0;
  right: 0;
  height: 2px;
  background-color: #0ea5e9;
  transform: scaleX(1);
  transition: transform 0.3s ease;
}

.desktop-nav {
  display: flex;
  gap: 24px;
  align-items: center;
}

.hamburger-btn {
  display: none;
}

/* Mobile Styles */
@media (max-width: 899px) {
  .portfolio-navbar {
    top: 16px;
    padding: 0 16px;
  }

  .navbar-inner {
    padding: 12px 16px;
  }

  .navbar-logo {
    font-size: 1rem;
  }

  .desktop-nav {
    display: none;
  }

  .hamburger-btn {
    display: block;
    background: none;
    border: none;
    cursor: pointer;
  }

  .mobile-drawer-backdrop {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.5);
    z-index: 1200;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.3s;
  }

  .mobile-drawer-backdrop.open {
    opacity: 1;
    pointer-events: auto;
  }

  .mobile-drawer {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    width: 80%;
    max-width: 320px;
    background: rgba(20, 27, 34, 0.95);
    backdrop-filter: blur(12px);
    box-shadow: -8px 0 32px rgba(0, 0, 0, 0.5);
    z-index: 1300;
    transform: translateX(100%);
    transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .mobile-drawer.open {
    transform: translateX(0);
  }

  .mobile-nav {
    display: flex;
    flex-direction: column;
    gap: 32px;
    padding: 64px 32px;
  }

  .mobile-nav .nav-link {
    font-size: 0.8125rem;
    min-height: 48px;
    display: flex;
    align-items: center;
  }
}
```

---

## Notes
- All measurements in MUI sx prop use spacing units (1 unit = 8px)
- Border radius in MUI: borderRadius: 3 = 12px
- When in doubt, reference the source files in the portfolio repo
- Maintain exact color values - do not approximate
- Test on real devices for mobile responsiveness
- Glassmorphism (backdrop-filter) may not work in older browsers - provide fallback
