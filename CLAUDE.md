# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Cryo Tank Calculator** is a Progressive Web App (PWA) for cryogenic tank transfer calculations. It helps drivers and technicians calculate product movement (SCF, gallons, pounds) for cryogenic gases (LIN, LOX, LAR) during tank offloading operations.

The app is designed for field use on mobile devices with offline capability via service workers.

## Architecture

### Single-Page Application Structure

The app uses a **multi-pane swipe navigation** model implemented in `calculator.html`:
- **Pane 0 (Calculator)**: Main tank calculator with vertical/laydown tank support
- **Pane 1 (Split)**: Distributes total SCF across two customer tanks
- **Pane 2 (Reserve)**: Converts pounds to SCF and allocates across multiple tanks

Navigation between panes:
- Swipe gestures (left/right)
- Arrow keys (desktop)
- Buttons that call `window.goToCalc()`, `window.goToSplit()`, `window.goToReserve()`

### Entry Point Flow

1. **index.html** - Shows 3-second splash screen, then loads `calculator.html` in an iframe
2. **login.html** - Optional landing page (not currently in the main flow)
3. **calculator.html** - Main application with all three panes

### Service Worker (sw.js)

Implements offline-first caching strategy:
- **Navigate requests**: Network-first, fallback to cached `index.html`
- **Same-origin assets**: Stale-while-revalidate
- **Cross-origin**: Pass-through

Cache versions: `CACHE_STATIC` and `CACHE_RUNTIME`

### Gas Constants

All three panes share gas conversion constants defined in `window.gases`:
```javascript
{
  LIN: { scfPerGal: 93.1,  lbPerGal: 6.74, scfPerLb: 13.8  },  // Nitrogen
  LOX: { scfPerGal: 115.0, lbPerGal: 9.52, scfPerLb: 12.06 },  // Oxygen
  LAR: { scfPerGal: 87.3,  lbPerGal: 9.40, scfPerLb: 9.30  }   // Argon
}
```

## Key Features & Implementation Details

### Calculator Pane (Pane 0)

**Tank Types:**
- **Vertical**: Simple linear calculation (inches × SCF-per-inch)
- **Laydown (Horizontal)**: Geometry-based using circular segment area calculation (`horizFrac()`)

**Horizontal Measure Modal:**
- Uses device camera to measure tank diameter and length
- Implements reference-based measurement: user draws a line over a known length
- Calculates pixel-to-inch ratio for subsequent measurements
- Auto-populates `geoDiameter` and `geoLength` fields

**Timer + Flow:**
- Start/Stop timer for elapsed time tracking
- Flow input with unit toggle (gallons/min ↔ pounds/min)
- "Use timer as primary" checkbox: prioritizes timer×flow over inch-based calculation
- "Estimate diameter from timer" checkbox: reverse-calculates tank diameter from delivered SCF

**Laydown-specific:**
- Shows geometry fields (diameter, length) only when tank type = "laydown"
- Dead time adjustment: subtracts ~5% (10-30 seconds) from elapsed time for horizontal fills
- `horizFrac(d, h)`: calculates liquid fraction based on diameter `d` and height `h` using circular segment geometry

### Split Pane (Pane 1)

Distributes total SCF (from Calculator) across **two tanks** proportionally by inch change:
- Each tank has Start/End inches and Tank Capacity (gallons)
- Gas selector at top to convert SCF ↔ gallons
- Allocation formula: `SCF_tank = (inches_tank / total_inches) × total_SCF`
- Displays gallon equivalents under each tank's allocated SCF

### Reserve Pane (Pane 2)

Converts **pounds from COA** to SCF and allocates across **three tanks**:
- Input field labeled "Total Pounds to SCF" (max 6 digits)
- SCF capped at 999,999 for truck counter compatibility
- Real-time remaining SCF calculation as tank values change
- "Truck Remaining Inches" fallback: linear mapping of remaining SCF to inches

## Development Commands

### Testing the App

**Local Development:**
Open `index.html` or `calculator.html` directly in a browser. The app has no build process.

**Inline Tests:**
Navigate to `calculator.html?test=1` to run basic navigation tests (checks pane transitions via console.assert).

### PWA Installation

The app can be installed as a PWA on mobile devices:
- Manifest: `manifest.webmanifest.txt` (rename to `.webmanifest` when deploying)
- Icons: `/icons/` directory contains 192px, 512px, and maskable variants
- Service worker auto-registers on page load

### Deployment Headers

`_headers.txt` contains recommended security headers for static hosting (e.g., Netlify/Vercel):
- X-Frame-Options: DENY
- X-Content-Type-Options: nosniff
- Referrer-Policy: strict-origin-when-cross-origin

## File Structure

```
cryo-calculator/
├── index.html              # Splash screen + iframe wrapper
├── calculator.html         # Main app (all three panes + scripts)
├── login.html             # Optional landing page
├── sw.js                  # Service worker
├── manifest.webmanifest.txt
├── _headers.txt
└── icons/
    ├── icon-192.png
    ├── icon-512.png
    ├── maskable-512.png
    └── apple-touch-icon-180.png
```

**Note:** The repository contains three duplicate versions (`v3.4`, `v3.4 2`, `v3.4 3`) - treat `v3.4` as the canonical version.

## Code Organization Patterns

### Script Blocks in calculator.html

Each pane has its own IIFE-wrapped script block:
1. **SCRIPTS: MEASURE MODAL** (lines 491-686): Camera-based measurement tool
2. **SCRIPTS: CALCULATOR** (lines 688-981): Pane 0 logic
3. **SCRIPTS: SPLIT** (lines 984-1190): Pane 1 logic
4. **SCRIPTS: RESERVE** (lines 1193-1310): Pane 2 logic
5. **SCRIPTS: TESTS** (lines 1313-1338): Inline test suite
6. **SCRIPTS: NAV** (lines 1340-1393): Swipe/keyboard navigation

### Common UI Patterns

- **Compact mode**: Applied via `.compact` class when viewport height < threshold
- **Sticky action bars**: Bottom-pinned buttons on Split/Reserve panes
- **Flow unit toggle**: Custom switch component (`.switch .knob`) for gal/min ↔ lb/min
- **Modal pattern**: `.modal.open` class controls visibility

### Helper Functions

- `toNum(v)`: Safe parseFloat with 0 fallback
- `fmt0(n)`, `fmt1(n)`: Locale-aware number formatting
- `fmtTime(ms)`: Formats milliseconds as HH:MM:SS
- `horizFrac(d, h)`: Circular segment area fraction for laydown tanks
- `adjustedElapsedMin(rawMin)`: Subtracts dead time for horizontal fills

## Important Notes

- **No authentication**: Login page exists but is not enforced
- **Client-side only**: All calculations happen in browser JavaScript
- **Gas constants are hardcoded**: Modify `window.gases` to adjust conversion factors
- **Mobile-first**: Designed for touch input with safe-area-inset support for notched devices
- **Offline-capable**: Service worker caches all assets for offline use

## Cross-Pane Data Flow

1. Calculator → Split: `window.setAutoSCFTotal(scf)` auto-fills Total SCF on Split pane
2. Split gas selector updates gallon hints in real-time
3. Reserve page reads from `window.gases` for pound → SCF conversion

## Common Tasks

**Adding a new gas type:**
Update `window.gases` object in all three script blocks (Calculator, Split, Reserve) with `scfPerGal`, `lbPerGal`, and `scfPerLb` values.

**Modifying tank geometry:**
Edit `horizFrac(d, h)` function in Calculator script block. This uses circular segment area formula.

**Adjusting timer dead time:**
Modify `adjustedElapsedMin()` function - currently subtracts 5% (min 10s, max 30s) from elapsed time.

**Updating service worker cache:**
Increment version numbers in `CACHE_STATIC` and `CACHE_RUNTIME` constants in `sw.js`.
