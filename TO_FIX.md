# Functional Issues List

This document tracks all functional issues identified for the Rust Nation UK conference app.

---

## Outstanding Issues

### 1. No Offline Support
- App requires network connection to function
- Could implement Service Worker for caching
- Consider PWA manifest for install capability
- Cache static assets and JSON data for offline viewing

---

## Fixed Issues

### 7. No Keyboard Navigation for Track Selector
- **Status:** Fixed
- Arrow Left/Right to move between tabs (with wrap-around)
- Home/End to jump to first/last tab
- Proper `tabindex` management (only active tab in tab order)
- Focus moves with selection

### 5. No Error Boundaries
- **Status:** Fixed
- Added global `+error.svelte` page
- Shows friendly error message with error code
- Includes "Try again" and "Go home" buttons
- Handles both 404 and other error types

### 2. No Timezone Display
- **Status:** Fixed
- Added GMT timezone badge in the filter bar on schedule page

### 3. No Empty State Handling
- **Status:** Fixed
- Added friendly empty state UI when no talks match search/filter criteria
- Includes icon, message, and "Clear filters" button

### 4. No Loading States
- **Status:** Fixed
- Added loading spinner while schedule data loads
- Simulated loading delay for smooth UX

### 6. No Search/Filter for Talks
- **Status:** Fixed
- Added search input to filter by talk title or speaker name
- Added tag filter buttons (keynote, talk, tutorial)
- Active filters shown in purple bar with "Clear all" option

### 8. Double Sticky Headers Taking Too Much Space on Mobile
- **Status:** Fixed
- Removed sticky positioning from main header entirely
- Only track selector is now sticky (sticks to top of viewport)
- Gives more vertical space when scrolling through schedule

### 9. Color Contrast Issues
- **Status:** Fixed
- Improved scroll hint text opacity (0.7 → 0.9)
- Improved filter label opacity (0.8 → 0.9)

### 10. Sponsor Toast Not Announced to Screen Readers
- **Status:** Fixed
- Added `aria-live="polite"` region to sponsor toast
- Screen readers now announce sponsor name and message when toast appears
