<script lang="ts">
	import { onMount } from 'svelte';
	import SponsorCard from '$lib/components/SponsorCard.svelte';
	import sponsorsData from '$lib/data/sponsors.json';

	const API_URL = import.meta.env.DEV
		? '/api/copia/events/rust-nation-uk-2026'
		: 'https://manage.copiaevents.com/api/public/events/rust-nation-uk-2026';

	interface Sponsor {
		id: string | number;
		name: string;
		tier: string;
		logo: string;
		website: string;
		bio: string;
	}

	interface ApiSponsor {
		id: string;
		name: string;
		sponsorship_type: string;
		logo_url: string;
		website_url: string;
		bio: string | null;
		sort_order: number;
	}

	let sponsors = $state<Sponsor[]>([]);
	let isLoading = $state(true);

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
				// Fallback to static data if API fails
				sponsors = sponsorsData.sponsors as Sponsor[];
			}
		} catch (error) {
			console.error('Failed to fetch sponsors:', error);
			// Fallback to static data on error
			sponsors = sponsorsData.sponsors as Sponsor[];
		}
		isLoading = false;
	});

	function groupByTier(sponsorList: Sponsor[]): Map<string, Sponsor[]> {
		const grouped = new Map<string, Sponsor[]>();

		// Group all sponsors by their tier
		for (const sponsor of sponsorList) {
			const tier = sponsor.tier.toLowerCase();
			if (!grouped.has(tier)) {
				grouped.set(tier, []);
			}
			grouped.get(tier)!.push(sponsor);
		}

		return grouped;
	}

	let sponsorsByTier = $derived(groupByTier(sponsors));

	// Order tiers: known tiers first in priority order, then any custom tiers alphabetically
	const knownTierOrder = ['platinum', 'track', 'recruitment sponsor', 'video sponsor', 'delegate lounge sponsor', 'gold', 'silver', 'community partner'];
	let tierOrder = $derived.by(() => {
		const tiers = [...sponsorsByTier.keys()];
		return tiers.sort((a, b) => {
			const aIndex = knownTierOrder.indexOf(a.toLowerCase());
			const bIndex = knownTierOrder.indexOf(b.toLowerCase());
			if (aIndex !== -1 && bIndex !== -1) return aIndex - bIndex;
			if (aIndex !== -1) return -1;
			if (bIndex !== -1) return 1;
			return a.localeCompare(b);
		});
	});
</script>

<svelte:head>
	<title>Sponsors - Rust Nation UK</title>
</svelte:head>

<div class="sponsors-page">
	<div class="sponsors-header">
		<h1 class="sponsors-title">Our Sponsors</h1>
		<p class="sponsors-subtitle">
			Rust Nation UK is made possible by these amazing companies
		</p>
	</div>

	<div class="sponsors-content">
		{#if isLoading}
			<div class="loading-state">
				<div class="loading-spinner"></div>
				<p>Loading sponsors...</p>
			</div>
		{:else if sponsors.length === 0}
			<div class="empty-state">
				<p>No sponsors available yet.</p>
			</div>
		{:else}
			{#each tierOrder as tier}
				{@const tierId = tier.replace(/\s+/g, '-')}
				{@const tierLabel = tier.split(' ').map(word => word.charAt(0).toUpperCase() + word.slice(1)).join(' ')}
				{#if sponsorsByTier.has(tier)}
					<section class="tier-section" aria-labelledby="tier-{tierId}">
						<h2 id="tier-{tierId}" class="tier-title">
							{tierLabel} Sponsors
						</h2>
						<div class="sponsors-grid">
							{#each sponsorsByTier.get(tier) || [] as sponsor (sponsor.id)}
								<SponsorCard {sponsor} />
							{/each}
						</div>
					</section>
				{/if}
			{/each}
		{/if}
	</div>
</div>

<style>
	.sponsors-page {
		flex: 1;
		background: var(--color-gray-100);
	}

	.sponsors-header {
		background:
			linear-gradient(135deg, transparent 60%, rgba(116, 8, 227, 0.5) 60%, rgba(116, 8, 227, 0.5) 70%, transparent 70%),
			linear-gradient(225deg, transparent 70%, rgba(132, 26, 242, 0.4) 70%, rgba(132, 26, 242, 0.4) 85%, transparent 85%),
			linear-gradient(180deg, #780AE9 0%, #6a08d4 100%);
		padding: var(--space-xl) var(--space-lg);
		text-align: center;
	}

	.sponsors-title {
		font-size: var(--text-2xl);
		font-weight: 700;
		color: var(--color-text-light);
		margin-bottom: var(--space-xs);
	}

	.sponsors-subtitle {
		font-size: var(--text-base);
		color: var(--color-text-light);
		opacity: 0.9;
	}

	.sponsors-content {
		padding: var(--space-lg);
		max-width: 1400px;
		margin: 0 auto;
	}

	.tier-section {
		margin-bottom: var(--space-xl);
	}

	.tier-title {
		font-size: var(--text-sm);
		font-weight: 600;
		text-transform: uppercase;
		letter-spacing: 0.05em;
		color: var(--color-primary);
		margin-bottom: var(--space-md);
	}

	.loading-state,
	.empty-state {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		padding: var(--space-2xl);
		color: var(--color-text-muted);
		text-align: center;
		width: 100%;
		flex: 1 1 100%;
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

	.sponsors-grid {
		display: flex;
		flex-wrap: wrap;
		gap: var(--space-md);
	}

	.sponsors-grid > :global(*) {
		flex: 1 1 100%;
		min-width: 0;
	}

	@media (min-width: 640px) {
		.sponsors-grid > :global(*) {
			flex: 1 1 calc(50% - var(--space-md));
			max-width: calc(50% - var(--space-md) / 2);
		}
	}

	@media (min-width: 768px) {
		.sponsors-header {
			padding: var(--space-2xl);
		}

		.sponsors-title {
			font-size: var(--text-3xl);
		}

		.sponsors-content {
			padding: var(--space-xl);
		}
	}

	@media (min-width: 1024px) {
		.sponsors-content {
			display: flex;
			flex-wrap: wrap;
			gap: var(--space-lg);
			align-items: flex-start;
			justify-content: center;
			max-width: 1200px;
		}

		.tier-section {
			flex: 0 0 auto;
			margin-bottom: 0;
		}

		.sponsors-grid {
			max-width: calc(320px * 3 + var(--space-md) * 2);
		}

		.sponsors-grid > :global(*) {
			flex: 0 0 280px;
			max-width: 320px;
		}
	}
</style>
