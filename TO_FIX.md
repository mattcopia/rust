# Functional Issues List

This document tracks all functional issues identified for the Rust Nation UK conference app.

---

## Outstanding Issues

### 1. No Offline Support
- App requires network connection to function
- Could implement Service Worker for caching
- Consider PWA manifest for install capability
- Cache static assets and JSON data for offline viewing

### 5. No Error Boundaries
- No graceful handling if data fails to load
- API errors would show broken UI
- Need Svelte error boundary components
- Should show user-friendly error messages with retry options

### 7. No Keyboard Navigation for Track Selector
- Track tabs lack arrow key navigation
- Should support Left/Right arrow keys to move between tabs
- Enter/Space to select focused tab
- Home/End keys to jump to first/last tab
- Add proper `tabindex` and focus management

---

## Fixed Issues

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
- Reduced header padding on mobile (from 1rem to 0.5rem)
- Reduced logo height on mobile (from 32px to 24px)
- Reduced nav link size and padding
- Updated TrackSelector sticky position accordingly

### 9. Color Contrast Issues
- **Status:** Fixed
- Improved scroll hint text opacity (0.7 → 0.9)
- Improved filter label opacity (0.8 → 0.9)

### 10. Sponsor Toast Not Announced to Screen Readers
- **Status:** Fixed
- Added `aria-live="polite"` region to sponsor toast
- Screen readers now announce sponsor name and message when toast appears
