<script lang="ts">
	import { onMount } from 'svelte';
	import TrackSelector from '$lib/components/TrackSelector.svelte';
	import TalkCard from '$lib/components/TalkCard.svelte';
	import BreakCard from '$lib/components/BreakCard.svelte';
	import SponsorToast from '$lib/components/SponsorToast.svelte';
	import scheduleData from '$lib/data/schedule.json';
	import adsData from '$lib/data/ads.json';

	const API_URL = import.meta.env.DEV
		? '/api/copia/events/rust-nation-uk-2026'
		: 'https://manage.copiaevents.com/api/public/events/rust-nation-uk-2026';

	// API Types
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

	// App Types
	interface Track {
		id: string | number;
		name: string;
	}

	interface SponsorAd {
		id: string | number;
		name: string;
		tier: string;
		logo: string;
		message: string;
		fullMessage?: string;
		image?: string;
		url: string;
	}

	interface Talk {
		id: string | number;
		track: string | number;
		time: string;
		duration: number;
		title: string;
		speaker: string;
		speakerPhoto: string;
		synopsis: string;
		tag: 'keynote' | 'talk' | 'tutorial';
		social: {
			twitter?: string;
			github?: string;
			website?: string;
		};
	}

	interface BreakItem {
		time: string;
		duration: number;
		type: string;
		label: string;
	}

	interface ScheduleItem {
		type: 'talk' | 'break';
		time: string;
		data: Talk | BreakItem;
	}

	// State
	let selectedTrack = $state<string | number>('');
	let tracks = $state<Track[]>([]);
	let talks = $state<Talk[]>([]);
	let breaks = $state<BreakItem[]>([]);
	let searchQuery = $state('');
	let selectedTags = $state<string[]>([]);
	let isLoading = $state(true);
	let currentTime = $state(new Date());
	let sponsorAds = $state<SponsorAd[]>([]);

	const allTags = ['keynote', 'talk', 'tutorial'];

	// Update current time every minute for "happening now" feature
	onMount(() => {
		const interval = setInterval(() => {
			currentTime = new Date();
		}, 60000);
		return () => clearInterval(interval);
	});

	// Check if a talk is happening now
	function isTalkNow(talk: Talk): boolean {
		const now = currentTime;
		const [hours, minutes] = talk.time.split(':').map(Number);

		const talkStart = new Date(now);
		talkStart.setHours(hours, minutes, 0, 0);

		const talkEnd = new Date(talkStart);
		talkEnd.setMinutes(talkEnd.getMinutes() + talk.duration);

		return now >= talkStart && now < talkEnd;
	}

	// Helper to calculate duration in minutes from time strings
	function calculateDuration(startTime: string, endTime: string): number {
		const [startH, startM] = startTime.split(':').map(Number);
		const [endH, endM] = endTime.split(':').map(Number);
		return (endH * 60 + endM) - (startH * 60 + startM);
	}

	// Helper to format time from "HH:MM:SS" to "HH:MM"
	function formatTime(time: string): string {
		return time.slice(0, 5);
	}

	// Map slot_type to tag
	function mapSlotTypeToTag(slotType: string): 'keynote' | 'talk' | 'tutorial' {
		if (slotType === 'keynote') return 'keynote';
		if (slotType === 'tutorial') return 'tutorial';
		return 'talk';
	}

	// Transform API slots into talks and breaks
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
				// Each session in the slot becomes a talk
				for (let i = 0; i < slot.sessions.length; i++) {
					const session = slot.sessions[i];
					let trackId: string;

					if (slot.track_id) {
						// Slot has explicit track assignment
						trackId = slot.track_id;
					} else if (slot.sessions.length > 1 && mainTracks.length > 0) {
						// Multiple sessions without track_id = parallel sessions
						// Distribute across main tracks by index
						trackId = mainTracks[i % mainTracks.length]?.id || mainTracks[0].id;
					} else if (mainTracks.length > 0) {
						// Single session (like keynote) - assign to first track
						trackId = mainTracks[0].id;
					} else {
						trackId = apiTracks[0]?.id || '';
					}

					// Get talk info from speaker object (talk_title/talk_abstract) or fallback to session/slot
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

	function getSchedule(trackId: string | number, query: string, tags: string[]): ScheduleItem[] {
		const hasFilter = query.trim() !== '' || tags.length > 0;
		let filteredTalks = [...talks];

		// Only filter by track if NOT searching/filtering
		if (!hasFilter && trackId) {
			filteredTalks = filteredTalks.filter((talk) => talk.track === trackId);
		}

		// Apply search filter
		if (query.trim()) {
			const q = query.toLowerCase();
			filteredTalks = filteredTalks.filter((talk) =>
				talk.title.toLowerCase().includes(q) ||
				talk.speaker.toLowerCase().includes(q)
			);
		}

		// Apply tag filter
		if (tags.length > 0) {
			filteredTalks = filteredTalks.filter((talk) => tags.includes(talk.tag));
		}

		const talkItems = filteredTalks.map((talk) => ({ type: 'talk' as const, time: talk.time, data: talk }));

		// Only show breaks if no search/filter active
		const breakItems = hasFilter ? [] : breaks.map((b) => ({
			type: 'break' as const,
			time: b.time,
			data: b
		}));

		const combined = [...talkItems, ...breakItems];
		combined.sort((a, b) => a.time.localeCompare(b.time));

		return combined;
	}

	// Get track name by ID
	function getTrackName(trackId: string | number): string {
		const track = tracks.find(t => t.id === trackId);
		return track?.name || '';
	}

	// Check if filtering is active
	let isFiltering = $derived(searchQuery.trim() !== '' || selectedTags.length > 0);

	let schedule = $derived(getSchedule(selectedTrack, searchQuery, selectedTags));

	function handleTrackSelect(trackId: string | number) {
		selectedTrack = trackId;
	}

	function toggleTag(tag: string) {
		if (selectedTags.includes(tag)) {
			selectedTags = selectedTags.filter((t) => t !== tag);
		} else {
			selectedTags = [...selectedTags, tag];
		}
	}

	function clearFilters() {
		searchQuery = '';
		selectedTags = [];
	}

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
				tracks = scheduleData.tracks;
				selectedTrack = tracks[0]?.id || 1;
				talks = scheduleData.talks as Talk[];
				breaks = scheduleData.breaks as BreakItem[];
			}
		} catch (error) {
			console.error('Failed to fetch data from API:', error);
			tracks = scheduleData.tracks;
			selectedTrack = tracks[0]?.id || 1;
			talks = scheduleData.talks as Talk[];
			breaks = scheduleData.breaks as BreakItem[];
		}

		isLoading = false;
	});
</script>

<svelte:head>
	<title>Schedule - Rust Nation UK</title>
</svelte:head>

<div class="schedule-page">
	{#if tracks.length > 0}
		<TrackSelector
			{tracks}
			{selectedTrack}
			onSelect={handleTrackSelect}
		/>
	{/if}

	<!-- Search and Filter Bar -->
	<div class="filter-bar">
		<div class="search-wrapper">
			<svg class="search-icon" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true">
				<circle cx="11" cy="11" r="8"/>
				<path d="m21 21-4.35-4.35"/>
			</svg>
			<input
				type="search"
				placeholder="Search talks or speakers..."
				bind:value={searchQuery}
				class="search-input"
				aria-label="Search talks or speakers"
			/>
		</div>
		<div class="tag-filters">
			{#each allTags as tag}
				<button
					class="tag-filter"
					class:active={selectedTags.includes(tag)}
					onclick={() => toggleTag(tag)}
					aria-pressed={selectedTags.includes(tag)}
				>
					{tag}
				</button>
			{/each}
		</div>
	</div>

	{#if searchQuery || selectedTags.length > 0}
		<div class="active-filters">
			<span class="filter-label">Filtering by:</span>
			{#if searchQuery}
				<span class="filter-chip">"{searchQuery}"</span>
			{/if}
			{#each selectedTags as tag}
				<span class="filter-chip">{tag}</span>
			{/each}
			<button class="clear-filters" onclick={clearFilters}>Clear all</button>
		</div>
	{/if}

	<SponsorToast ads={sponsorAds.length > 0 ? sponsorAds : adsData.ads} duration={8000} delayBetween={15000} />

	<div
		class="schedule-content"
		role="tabpanel"
		id="track-panel-{selectedTrack}"
		aria-label="{tracks.find(t => t.id === selectedTrack)?.name} schedule"
	>
		{#if isLoading}
			<!-- Loading State -->
			<div class="loading-state">
				<div class="loading-spinner"></div>
				<p>Loading schedule...</p>
			</div>
		{:else if schedule.length === 0}
			<!-- Empty State -->
			<div class="empty-state">
				<svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" aria-hidden="true">
					<circle cx="12" cy="12" r="10"/>
					<path d="M8 15s1.5 2 4 2 4-2 4-2"/>
					<line x1="9" y1="9" x2="9.01" y2="9"/>
					<line x1="15" y1="9" x2="15.01" y2="9"/>
				</svg>
				<h3>No talks found</h3>
				<p>Try adjusting your search or filters</p>
				<button class="clear-filters-btn" onclick={clearFilters}>Clear filters</button>
			</div>
		{:else}
			<div class="schedule-list">
				{#each schedule as item (item.type === 'talk' ? (item.data as Talk).id : item.time)}
					{#if item.type === 'talk'}
						<TalkCard
							talk={item.data as Talk}
							trackName={isFiltering ? getTrackName((item.data as Talk).track) : undefined}
							isNow={isTalkNow(item.data as Talk)}
						/>
					{:else}
						<BreakCard breakItem={item.data as BreakItem} />
					{/if}
				{/each}
			</div>
		{/if}
	</div>
</div>

<style>
	.schedule-page {
		flex: 1;
		display: flex;
		flex-direction: column;
		background: var(--color-gray-100);
	}

	/* Filter Bar */
	.filter-bar {
		display: flex;
		flex-wrap: wrap;
		align-items: center;
		gap: var(--space-sm);
		padding: var(--space-sm) var(--space-md);
		background: var(--color-white);
		border-bottom: 1px solid var(--color-gray-200);
	}

	.search-wrapper {
		position: relative;
		flex: 1;
		min-width: 150px;
	}

	.search-icon {
		position: absolute;
		left: var(--space-sm);
		top: 50%;
		transform: translateY(-50%);
		color: var(--color-gray-600);
		pointer-events: none;
	}

	.search-input {
		width: 100%;
		padding: var(--space-xs) var(--space-sm) var(--space-xs) 36px;
		border: 1px solid var(--color-gray-300);
		border-radius: var(--radius-md);
		font-size: var(--text-sm);
		background: var(--color-gray-100);
		color: var(--color-text);
		transition: all var(--transition-fast);
	}

	.search-input:focus {
		outline: none;
		border-color: var(--color-primary);
		background: var(--color-white);
		box-shadow: 0 0 0 3px rgba(120, 10, 233, 0.1);
	}

	.tag-filters {
		display: flex;
		gap: var(--space-xs);
	}

	.tag-filter {
		padding: var(--space-xs) var(--space-sm);
		border: 1px solid var(--color-gray-300);
		border-radius: var(--radius-full);
		font-size: var(--text-xs);
		font-weight: 500;
		text-transform: capitalize;
		color: var(--color-text-muted);
		background: var(--color-surface);
		transition: all var(--transition-fast);
	}

	.tag-filter:hover {
		border-color: var(--color-primary);
		color: var(--color-primary);
	}

	.tag-filter.active {
		background: var(--color-primary);
		border-color: var(--color-primary);
		color: var(--color-text-light);
	}

	/* Active Filters */
	.active-filters {
		display: flex;
		flex-wrap: wrap;
		align-items: center;
		gap: var(--space-sm);
		padding: var(--space-xs) var(--space-md);
		background: var(--color-primary);
		color: var(--color-text-light);
		font-size: var(--text-sm);
	}

	.filter-label {
		opacity: 1;
	}

	.filter-chip {
		padding: 2px var(--space-sm);
		background: rgba(255, 255, 255, 0.25);
		border-radius: var(--radius-sm);
		font-weight: 600;
		text-transform: capitalize;
	}

	.clear-filters {
		margin-left: auto;
		padding: 2px var(--space-sm);
		background: rgba(255, 255, 255, 0.25);
		border-radius: var(--radius-sm);
		font-size: var(--text-xs);
		font-weight: 600;
		transition: background var(--transition-fast);
	}

	.clear-filters:hover {
		background: rgba(255, 255, 255, 0.35);
	}

	.schedule-content {
		flex: 1;
		overflow-y: auto;
	}

	/* Loading State */
	.loading-state {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		padding: var(--space-2xl);
		color: var(--color-gray-600);
	}

	.loading-spinner {
		width: 40px;
		height: 40px;
		border: 3px solid var(--color-gray-200);
		border-top-color: var(--color-primary);
		border-radius: 50%;
		animation: spin 0.8s linear infinite;
		margin-bottom: var(--space-md);
	}

	@keyframes spin {
		to {
			transform: rotate(360deg);
		}
	}

	/* Empty State */
	.empty-state {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		padding: var(--space-2xl);
		text-align: center;
		color: var(--color-gray-600);
	}

	.empty-state svg {
		margin-bottom: var(--space-md);
		opacity: 0.5;
	}

	.empty-state h3 {
		font-size: var(--text-lg);
		font-weight: 600;
		color: var(--color-text);
		margin-bottom: var(--space-xs);
	}

	.empty-state p {
		font-size: var(--text-sm);
		margin-bottom: var(--space-md);
	}

	.clear-filters-btn {
		padding: var(--space-sm) var(--space-lg);
		background: var(--color-primary);
		color: var(--color-text-light);
		border-radius: var(--radius-md);
		font-weight: 500;
		transition: background var(--transition-fast);
	}

	.clear-filters-btn:hover {
		background: var(--color-primary-dark);
	}

	.schedule-list {
		display: flex;
		flex-direction: column;
		gap: var(--space-md);
		padding: var(--space-md);
		max-width: 800px;
		margin: 0 auto;
		width: 100%;
	}

	@media (min-width: 768px) {
		.filter-bar {
			padding: var(--space-sm) var(--space-lg);
		}

		.schedule-list {
			padding: var(--space-xl);
			gap: var(--space-lg);
		}
	}
</style>
