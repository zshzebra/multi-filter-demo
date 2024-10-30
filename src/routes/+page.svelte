<script lang="ts">
	import { createFilterStore, type FilterDimension } from '@zshzebra/svelte-multi-filter';

	import * as Card from '$lib/components/ui/card';

	import { codeToHtml } from 'shiki';
	import { onMount } from 'svelte';
	import Button from '$lib/components/ui/button/button.svelte';
	import Label from '$lib/components/ui/label/label.svelte';
	import * as RadioGroup from '$lib/components/ui/radio-group';
	import { slide } from 'svelte/transition';

	interface ProductFilter {
		category: 'Shirt' | 'Pants' | 'Jacket';
		color: 'Red' | 'Blue' | 'Black';
	}

	interface Product {
		category: 'Shirt' | 'Pants' | 'Jacket';
		color: 'Red' | 'Blue' | 'Black';
		name: string;
		expand: boolean;
		code: string;
	}

	const products: Product[] = $state([
		{ category: 'Shirt', color: 'Red', name: 'T-Shirt', expand: false },
		{ category: 'Pants', color: 'Blue', name: 'Slim Fit Pants', expand: false },
		{ category: 'Shirt', color: 'Black', name: 'Long Sleeve Shirt', expand: false },
		{ category: 'Jacket', color: 'Red', name: 'Classic Jacket', expand: false },
		{ category: 'Jacket', color: 'Blue', name: 'Cargo Jacket', expand: false },
		{ category: 'Pants', color: 'Black', name: 'Cropped Jeans', expand: false }
	]);

	const config = {
		category: ['Shirt', 'Pants', 'Jacket'],
		color: ['Red', 'Blue', 'Black']
	} satisfies Record<string, unknown>;

	const filter = createFilterStore(products as unknown as Record<string, unknown>[], config);

	// State for dimensions and results
	let dimensions = $state<{
		category: FilterDimension<string>;
		color: FilterDimension<string>;
	}>();
	let results = $state<Product[]>([]);
	let availableCategoryOptions = $state<string[]>([]);
	let availableColorOptions = $state<string[]>([]);

	// Subscribe to store changes
	filter.subscribe((dims) => {
		dimensions = dims;
	});

	// Subscribe to filtered items
	filter.filteredItems.subscribe((items) => {
		results = items;
	});

	// Subscribe to available options
	filter.getAvailableOptions('category').subscribe((options) => {
		availableCategoryOptions = options;
	});

	filter.getAvailableOptions('color').subscribe((options) => {
		availableColorOptions = options;
	});

	onMount(async () => {
		products.forEach(async (product) => {
			product.code = await codeToHtml(
				JSON.stringify(
					{
						category: product.category,
						color: product.color,
						name: product.name
					},
					null,
					4
				),
				{
					lang: 'json',
					theme: 'catppuccin-mocha'
				}
			);
		});
	});
</script>

{#snippet result(item)}
	<Card.Root
		class="max-w-sm cursor-pointer"
		onclick={() => {
			let expand = item.expand;
			results.forEach((result) => {
				result.expand = false;
			});
			item.expand = !expand;
		}}
	>
		<Card.Content class="flex w-96 flex-col">
			<div class="flex flex-row justify-between">
				<h3>{item.name}</h3>
				<div
					class="h-4 w-4 rounded-full"
					class:bg-red-400={item.color === 'Red'}
					class:bg-blue-400={item.color === 'Blue'}
					class:bg-black={item.color === 'Black'}
				></div>
			</div>
			{#if item.expand}
				<div transition:slide class="mt-4">
					{@html item.code}
				</div>
			{/if}
		</Card.Content>
	</Card.Root>
{/snippet}

<div class="m-10 flex h-max w-full flex-row gap-8">
	<Card.Root class="h-max w-64">
		<Card.Content>
			<div class="filters">
				<!-- Category Filter -->
				<h3 class="mb-2 text-2xl font-bold">Category</h3>
				<RadioGroup.Root value={dimensions?.category?.selected}>
					{#each ['Any', ...config.category] as option}
						<div class="flex items-center space-x-2">
							<RadioGroup.Item
								value={option}
								id={option}
								onclick={() => filter.select('category', option)}
								disabled={!availableCategoryOptions.includes(option) && !(option === 'Any')}
							/>
							<Label for={option}>{option}</Label>
						</div>
					{/each}
				</RadioGroup.Root>

				<!-- Color Filter -->
				<h3 class="mb-2 text-2xl font-bold">Color</h3>
				<RadioGroup.Root value={dimensions?.color?.selected}>
					{#each ['Any', ...config.color] as option}
						<div class="flex items-center space-x-2">
							<RadioGroup.Item
								value={option}
								id="col-{option}"
								onclick={() => filter.select('color', option)}
								disabled={!availableColorOptions.includes(option) && !(option === 'Any')}
							/>
							<Label for="col-{option}">{option}</Label>
						</div>
					{/each}
				</RadioGroup.Root>
			</div>
			<br />
			<Button onclick={() => filter.reset()}>Reset Filters</Button>
		</Card.Content>
	</Card.Root>

	<div class="flex flex-col gap-4">
		{#each results as item}
			<div transition:slide>
				{@render result(item)}
			</div>
		{/each}
	</div>

	<div class="relative">
		<div class="flex flex-col gap-4">
			{#each products as item}
				{#if results.filter((result) => result.name === item.name).length == 0}
					<div transition:slide class="opacity-60">
						{@render result(item)}
					</div>
				{/if}
			{/each}
		</div>
	</div>
</div>

<style>
	:global(pre) {
		@apply w-full rounded-xl p-4;
	}
</style>
