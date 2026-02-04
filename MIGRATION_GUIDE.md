# Migration Guide: API Integration & Light Mode Changes

This document describes all changes needed to migrate a conference website from static JSON data to API-driven content, force light mode, and improve toast pop-up UX.

## Overview of Changes

1. **API Integration** - Schedule, Sponsors, and Speakers pages now fetch from an external API
2. **CORS Proxy** - Vite proxy configured to bypass CORS restrictions
3. **Light Mode Only** - Removed all dark mode CSS and detection
4. **Improved Toast UX** - Made sponsor toast pop-ups more obviously clickable

---

## 1. Vite CORS Proxy Configuration

**File:** `vite.config.js` (or `vite.config.ts`)

Add a proxy to bypass CORS when fetching from the external API:

```javascript
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({
	plugins: [sveltekit()],
	server: {
		proxy: {
			'/api/copia': {
				target: 'https://manage.copiaevents.com',
				changeOrigin: true,
				rewrite: (path) => path.replace(/^\/api\/copia/, '/api/public')
			}
		}
	}
});
```

**Note:** Replace the API URL with the appropriate endpoint for your conference. The proxy rewrites `/api/copia/...` to `https://manage.copiaevents.com/api/public/...`.

---

## 2. Remove Dark Mode

### 2.1 Theme CSS

**File:** `src/lib/styles/theme.css`

Remove the entire `@media (prefers-color-scheme: dark)` block that overrides CSS variables for dark mode.

### 2.2 Header Component

**File:** `src/lib/components/Header.svelte`

Remove any `@media (prefers-color-scheme: dark)` blocks in the `<style>` section, particularly those affecting nav button colors (which can cause white-on-white text issues).

### 2.3 SponsorToast Component

**File:** `src/lib/components/SponsorToast.svelte`

- Remove `isDarkMode` state variable and any dark mode detection logic
- Remove any `matchMedia('(prefers-color-scheme: dark)')` calls
- Remove event listeners for color scheme changes

---

## 3. Sponsor Toast Pop-up Improvements

**File:** `src/lib/components/SponsorToast.svelte`

### 3.1 Add "Tap for details" Indicator

Add a visual hint that the toast is clickable. Inside the toast content area, add:

```svelte
<span class="tap-hint">
	Tap for details
	<svg class="tap-chevron" width="16" height="16" viewBox="0 0 16 16" fill="none" aria-hidden="true">
		<path d="M6 4L10 8L6 12" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
	</svg>
</span>
```

### 3.2 Add CSS for the Hint

```css
.tap-hint {
	display: flex;
	align-items: center;
	gap: 4px;
	margin-top: var(--space-xs);
	font-size: var(--text-xs);
	color: var(--color-primary);
	font-weight: 500;
}

.tap-chevron {
	animation: bounce-right 1s ease-in-out infinite;
}

@keyframes bounce-right {
	0%, 100% {
		transform: translateX(0);
	}
	50% {
		transform: translateX(3px);
	}
}
```

### 3.3 Add Hover/Active States to Toast

```css
.toast-content {
	cursor: pointer;
	transition: transform var(--transition-fast), box-shadow var(--transition-fast);
}

.toast-content:hover {
	transform: translateY(-2px);
	box-shadow: var(--shadow-lg);
}

.toast-content:active {
	transform: translateY(0);
}
```

---

## 4. Schedule Page API Integration

**File:** `src/routes/schedule/+page.svelte`

### 4.1 API URL Constant

```typescript
const API_URL = '/api/copia/events/YOUR-EVENT-SLUG';
```

### 4.2 API Type Definitions

Add these interfaces for the API response:

```typescript
interface ApiSponsor {
	id: string;
	name: string;
	sponsorship_type: string;
	logo_url: string;
	has_advert: boolean;
	advert?: {
		image_url: string;
		headline: string;
		body: string;
		link_url: string;
	};
}

interface ApiTrack {
	id: string;
	name: string;
	description: string | null;
	color: string;
	room_name: string | null;
	sort_order: number;
}

interface ApiSpeaker {
	id: string;
	full_name: string;
	job_title: string | null;
	headshot_url: string | null;
	organisation: string | null;
	talk_title: string | null;
	talk_abstract: string | null;
}

interface ApiSession {
	id: string;
	title: string | null;
	description: string | null;
	speaker: ApiSpeaker;
}

interface ApiSlot {
	id: string;
	track_id: string | null;
	date: string;
	start_time: string;
	end_time: string;
	title: string | null;
	slot_type: string;
	is_break: boolean;
	sessions: ApiSession[];
}
```

### 4.3 Data Transformation Function

```typescript
function transformApiData(slots: ApiSlot[], apiTracks: ApiTrack[]) {
	const transformedTalks: Talk[] = [];
	const transformedBreaks: BreakItem[] = [];

	// Get non-tutorial tracks for distributing parallel sessions
	const mainTracks = apiTracks.filter(t => !t.name.toLowerCase().includes('tutorial'));

	for (const slot of slots) {
		const time = formatTime(slot.start_time);
		const duration = calculateDuration(slot.start_time, slot.end_time);

		if (slot.is_break) {
			transformedBreaks.push({
				time,
				duration,
				type: slot.slot_type,
				label: slot.title || slot.slot_type.replace('_', ' ')
			});
		} else if (slot.sessions && slot.sessions.length > 0) {
			for (let i = 0; i < slot.sessions.length; i++) {
				const session = slot.sessions[i];
				let trackId: string;

				if (slot.track_id) {
					trackId = slot.track_id;
				} else if (slot.sessions.length > 1 && mainTracks.length > 0) {
					// Parallel sessions - distribute across tracks
					trackId = mainTracks[i % mainTracks.length]?.id || mainTracks[0].id;
				} else if (mainTracks.length > 0) {
					trackId = mainTracks[0].id;
				} else {
					trackId = apiTracks[0]?.id || '';
				}

				// Get talk info from speaker object (talk_title/talk_abstract)
				const talkTitle = session.speaker?.talk_title || session.title || slot.title || 'TBA';
				const talkAbstract = session.speaker?.talk_abstract || session.description || '';
				const speakerName = session.speaker?.full_name || 'TBA';

				transformedTalks.push({
					id: session.id,
					track: trackId,
					time,
					duration,
					title: talkTitle,
					speaker: speakerName,
					speakerPhoto: session.speaker?.headshot_url || `https://ui-avatars.com/api/?name=${encodeURIComponent(speakerName)}&background=780AE9&color=fff&size=200`,
					synopsis: talkAbstract,
					tag: mapSlotTypeToTag(slot.slot_type),
					social: {}
				});
			}
		}
	}

	return { talks: transformedTalks, breaks: transformedBreaks };
}
```

### 4.4 Helper Functions

```typescript
function calculateDuration(startTime: string, endTime: string): number {
	const [startH, startM] = startTime.split(':').map(Number);
	const [endH, endM] = endTime.split(':').map(Number);
	return (endH * 60 + endM) - (startH * 60 + startM);
}

function formatTime(time: string): string {
	return time.slice(0, 5); // "HH:MM:SS" -> "HH:MM"
}

function mapSlotTypeToTag(slotType: string): 'keynote' | 'talk' | 'tutorial' {
	if (slotType === 'keynote') return 'keynote';
	if (slotType === 'tutorial') return 'tutorial';
	return 'talk';
}
```

### 4.5 onMount Data Fetching

```typescript
onMount(async () => {
	try {
		const response = await fetch(API_URL);
		if (response.ok) {
			const data = await response.json();

			// Fetch tracks
			const apiTracks: ApiTrack[] = data.schedule?.tracks || [];
			if (apiTracks.length > 0) {
				tracks = apiTracks
					.sort((a, b) => a.sort_order - b.sort_order)
					.map((track) => ({
						id: track.id,
						name: track.name
					}));
				selectedTrack = tracks[0].id;

				// Fetch and transform schedule slots
				const apiSlots: ApiSlot[] = data.schedule?.slots || [];
				const transformed = transformApiData(apiSlots, apiTracks);
				talks = transformed.talks;
				breaks = transformed.breaks;
			} else {
				// Fallback to static data
				tracks = scheduleData.tracks;
				selectedTrack = tracks[0]?.id || 1;
				talks = scheduleData.talks as Talk[];
				breaks = scheduleData.breaks as BreakItem[];
			}

			// Fetch sponsor ads
			const sponsors: ApiSponsor[] = data.sponsors || [];
			sponsorAds = sponsors
				.filter((sponsor) => sponsor.has_advert && sponsor.advert)
				.map((sponsor) => ({
					id: sponsor.id,
					name: sponsor.name,
					tier: sponsor.sponsorship_type,
					logo: sponsor.logo_url,
					message: sponsor.advert!.headline,
					fullMessage: sponsor.advert!.body,
					image: sponsor.advert!.image_url,
					url: sponsor.advert!.link_url
				}));
		} else {
			// Fallback to static data
			// ... load from static JSON
		}
	} catch (error) {
		console.error('Failed to fetch data from API:', error);
		// Fallback to static data
	}

	isLoading = false;
});
```

---

## 5. Sponsors Page API Integration

**File:** `src/routes/sponsors/+page.svelte`

### 5.1 API URL and Types

```typescript
const API_URL = '/api/copia/events/YOUR-EVENT-SLUG';

interface ApiSponsor {
	id: string;
	name: string;
	sponsorship_type: string;
	logo_url: string;
	website_url: string;
	bio: string | null;
	sort_order: number;
}
```

### 5.2 Data Fetching

```typescript
onMount(async () => {
	try {
		const response = await fetch(API_URL);
		if (response.ok) {
			const data = await response.json();
			const apiSponsors: ApiSponsor[] = data.sponsors || [];

			sponsors = apiSponsors
				.sort((a, b) => a.sort_order - b.sort_order)
				.map((s) => ({
					id: s.id,
					name: s.name,
					tier: s.sponsorship_type.toLowerCase(),
					logo: s.logo_url,
					website: s.website_url,
					bio: s.bio || ''
				}));
		} else {
			sponsors = sponsorsData.sponsors as Sponsor[];
		}
	} catch (error) {
		console.error('Failed to fetch sponsors:', error);
		sponsors = sponsorsData.sponsors as Sponsor[];
	}
	isLoading = false;
});
```

---

## 6. Speakers Page API Integration

**File:** `src/routes/speakers/+page.svelte`

### 6.1 API URL and Types

```typescript
const API_URL = '/api/copia/events/YOUR-EVENT-SLUG';

interface ApiSpeaker {
	id: string;
	full_name: string;
	job_title: string | null;
	organisation: string | null;
	headshot_url: string | null;
	bio: string | null;
	linkedin_url: string | null;
	twitter_handle: string | null;
	website: string | null;
}
```

### 6.2 Data Fetching

```typescript
onMount(async () => {
	try {
		const response = await fetch(API_URL);
		if (response.ok) {
			const data = await response.json();
			const apiSpeakers: ApiSpeaker[] = data.speakers || [];

			if (apiSpeakers.length > 0) {
				speakers = apiSpeakers.map((s) => ({
					id: s.id,
					name: s.full_name,
					photo: s.headshot_url || undefined,
					title: s.job_title || undefined,
					company: s.organisation || undefined,
					bio: s.bio || undefined,
					social: {
						twitter: s.twitter_handle?.replace('@', '') || undefined,
						linkedin: s.linkedin_url || undefined,
						website: s.website || undefined
					}
				}));
			}
		}
	} catch (error) {
		console.error('Failed to fetch speakers:', error);
	}
	isLoading = false;
});
```

---

## Summary Checklist

- [ ] Update `vite.config.js` with CORS proxy (use correct API URL)
- [ ] Remove dark mode from `src/lib/styles/theme.css`
- [ ] Remove dark mode from `src/lib/components/Header.svelte`
- [ ] Update `src/lib/components/SponsorToast.svelte`:
  - [ ] Remove dark mode detection
  - [ ] Add "Tap for details" indicator
  - [ ] Add hover/active states
- [ ] Update `src/routes/schedule/+page.svelte` with full API integration
- [ ] Update `src/routes/sponsors/+page.svelte` with API integration
- [ ] Update `src/routes/speakers/+page.svelte` with API integration

**Important:** Replace `YOUR-EVENT-SLUG` with the actual event slug for the conference API endpoint.
