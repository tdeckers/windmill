<script context="module" lang="ts">
	import { writable } from 'svelte/store'
	const activeDropdown = writable<{ id: string | null; close: (() => void) | null }>({
		id: null,
		close: null
	})
</script>

<script lang="ts">
	import { MoreVertical } from 'lucide-svelte'
	import type { Placement } from '@floating-ui/core'
	import type { Item } from '$lib/utils'
	import DropdownV2Inner from './DropdownV2Inner.svelte'
	import { pointerDownOutside } from '$lib/utils'
	import { createDropdownMenu, melt, createSync } from '@melt-ui/svelte'
	import ResolveOpen from '$lib/components/common/menu/ResolveOpen.svelte'
	import Button from '$lib/components/common/button/Button.svelte'
	import { twMerge } from 'tailwind-merge'
	import { triggerableByAI } from '$lib/actions/triggerableByAI'

	export let aiId: string | undefined = undefined
	export let aiDescription: string | undefined = undefined
	export let items: Item[] | (() => Item[]) | (() => Promise<Item[]>) = []
	export let disabled = false
	export let placement: Placement = 'bottom-end'
	export let usePointerDownOutside = false
	export let closeOnOtherDropdownOpen = true
	export let fixedHeight = true
	export let hidePopup = false
	export let open = false
	export let customWidth: number | undefined = undefined
	export let customMenu = false

	let buttonEl: HTMLButtonElement | undefined = undefined

	const {
		elements: { menu, item, trigger },
		states,
		ids: { menu: dropdownId }
	} = createDropdownMenu({
		positioning: {
			placement
		},
		loop: true,
		onOpenChange: ({ next }) => {
			if (closeOnOtherDropdownOpen) {
				if (next) {
					// Close previous dropdown if exists
					if ($activeDropdown.close && $activeDropdown.id !== $dropdownId) {
						$activeDropdown.close()
					}
					// Set this dropdown as active
					activeDropdown.set({ id: $dropdownId, close })
				} else if ($activeDropdown.id === $dropdownId) {
					activeDropdown.set({ id: null, close: null })
				}
			}
			return next
		}
	})

	const sync = createSync(states)
	$: sync.open(open, (v) => (open = Boolean(v)))

	export function close() {
		open = false
	}

	async function computeItems(): Promise<Item[]> {
		if (typeof items === 'function') {
			const result = await items()
			return Array.isArray(result) ? result.filter((item) => !item.hide) : []
		} else {
			return items.filter((item) => !item.hide)
		}
	}
	async function getMenuElements(): Promise<HTMLElement[]> {
		return Array.from(document.querySelectorAll('[data-menu]')) as HTMLElement[]
	}
</script>

<ResolveOpen {open} on:open on:close />

<button
	bind:this={buttonEl}
	use:triggerableByAI={{
		id: aiId,
		description: aiDescription,
		callback: () => buttonEl?.click()
	}}
	class={twMerge('w-full flex items-center justify-end', fixedHeight && 'h-8', $$props.class)}
	use:melt={$trigger}
	{disabled}
	on:click={(e) => e.stopPropagation()}
	use:pointerDownOutside={{
		capture: true,
		stopPropagation: false,
		exclude: getMenuElements,
		customEventName: 'pointerdown_menu'
	}}
	on:pointerdown_outside={() => {
		if (usePointerDownOutside) {
			close()
		}
	}}
	data-menu
>
	{#if $$slots.buttonReplacement}
		<slot name="buttonReplacement" />
	{:else}
		<Button
			nonCaptureEvent
			size="xs"
			color="light"
			startIcon={{ icon: MoreVertical }}
			btnClasses="bg-transparent"
		/>
	{/if}
</button>

{#if open && !hidePopup}
	<div use:melt={$menu} data-menu class="z-[6000] transition-all duration-100">
		{#if customMenu}
			<slot name="menu" />
		{:else}
			<div
				class="bg-surface border w-56 origin-top-right rounded-md shadow-md focus:outline-none overflow-y-auto py-1 max-h-[50vh]"
				style={customWidth ? `width: ${customWidth}px` : ''}
			>
				<DropdownV2Inner {aiId} items={computeItems} meltItem={item} />
			</div>
		{/if}
	</div>
{/if}
