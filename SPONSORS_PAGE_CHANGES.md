# Sponsors Page Layout & Formatting Changes

This document describes the layout and formatting changes made to the sponsors page. Apply these changes to match the updated design.

---

## 1. Tier Ordering System

Sponsors are grouped by tier and displayed in a specific order. Update the `knownTierOrder` array to control the display order:

```typescript
const knownTierOrder = ['platinum', 'track', 'recruitment sponsor', 'video sponsor', 'delegate lounge sponsor', 'gold', 'silver', 'community partner'];
```

The sorting logic should:
- Show known tiers first in the specified order
- Sort any unknown tiers alphabetically after known tiers

```typescript
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
```

---

## 2. Flowing Desktop Layout

On desktop, tier sections flow horizontally (left to right) instead of stacking vertically. This allows single sponsors from different tiers to sit alongside each other.

### Template Structure

```svelte
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
```

### CSS for Flowing Layout

```css
.sponsors-content {
    padding: var(--space-lg);
    max-width: 1400px;
    margin: 0 auto;
}

.tier-section {
    margin-bottom: var(--space-xl);
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

/* Tablet: 2 columns */
@media (min-width: 640px) {
    .sponsors-grid > :global(*) {
        flex: 1 1 calc(50% - var(--space-md));
        max-width: calc(50% - var(--space-md) / 2);
    }
}

/* Desktop: Flowing layout with max 3 columns */
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
```

---

## 3. Tier Title Styling

Tier titles appear as small badges above each group:

```css
.tier-title {
    font-size: var(--text-xs);
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    margin-bottom: var(--space-md);
    padding: var(--space-xs) var(--space-sm);
    border-radius: var(--radius-sm);
    display: inline-block;
}
```

---

## 4. Collapsible Sponsor Bios

Bios are hidden by default. Users click "Read bio" to expand.

### SponsorCard Component Changes

Add state for expanded:

```typescript
let expanded = $state(false);

function toggleBio() {
    expanded = !expanded;
}
```

Replace the bio paragraph with a toggle button and conditional bio:

```svelte
{#if sponsor.bio}
    <button
        class="bio-toggle"
        onclick={toggleBio}
        aria-expanded={expanded}
        aria-controls="bio-{sponsor.id}"
    >
        {expanded ? 'Hide bio' : 'Read bio'}
        <svg
            class="chevron"
            class:rotated={expanded}
            width="16"
            height="16"
            viewBox="0 0 16 16"
            fill="none"
            aria-hidden="true"
        >
            <path d="M4 6L8 10L12 6" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
    </button>

    {#if expanded}
        <p id="bio-{sponsor.id}" class="sponsor-bio">{sponsor.bio}</p>
    {/if}
{/if}
```

### CSS for Bio Toggle

```css
.bio-toggle {
    display: flex;
    align-items: center;
    gap: var(--space-xs);
    padding: var(--space-xs) 0;
    font-size: var(--text-sm);
    font-weight: 500;
    margin-bottom: var(--space-sm);
}

.bio-toggle:hover,
.bio-toggle:focus-visible {
    text-decoration: underline;
}

.chevron {
    transition: transform var(--transition-fast);
}

.chevron.rotated {
    transform: rotate(180deg);
}

.sponsor-bio {
    font-size: var(--text-sm);
    line-height: 1.6;
    margin-bottom: var(--space-md);
    padding-top: var(--space-sm);
    border-top: 1px solid var(--color-gray-200);
}
```

---

## 5. Remove Tier Badge from Cards

Since the tier is shown in the section header, remove the tier badge from individual sponsor cards.

Remove from template:
```svelte
<!-- Remove this line -->
<span class="sponsor-tier">{sponsor.tier}</span>
```

Remove the tier-specific CSS classes and styling (`.sponsor-tier`, `.sponsor-card--platinum`, etc.)

Change the article class from:
```svelte
<article class="sponsor-card sponsor-card--{sponsor.tier}">
```
To:
```svelte
<article class="sponsor-card">
```

---

## Summary of Layout Behavior

- **Mobile**: Sponsors stack vertically, one per row
- **Tablet (640px+)**: 2 sponsors per row within each tier
- **Desktop (1024px+)**:
  - Tier sections flow horizontally (left to right)
  - Max 3 sponsors per row within a tier
  - Single sponsors from different tiers can sit alongside each other
  - Content is centered
