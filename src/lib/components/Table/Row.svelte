<script lang="ts">
	import { toasts } from 'svelte-toasts';
	import ProfileAvatar from '../ProfileAvatar.svelte';
	import type { Rating } from '$lib/nostr';
	import ZapModal from '../ZapModal.svelte';

    export let rating: Rating;
    export let zapModal: ZapModal;
	export let expanded: boolean = false;

	async function copyNpub(npub: string) {
		await navigator.clipboard.writeText(npub);
		toasts.success({
			title: 'Copied NPUB!',
			description: 'NPUB copied to clipboard'
		});
	}

	let textarea: HTMLTextAreaElement;
	function toggleExpanded() {
		expanded = !expanded;
		if (!textarea) return;
		textarea.style.height = expanded ? textarea.scrollHeight + 'px' : 'auto';
	}
</script>

{#if rating}
	<tr class="border-x border-y-0 border-gray-200 bg-slate-800">
		<td class="whitespace-no-wrap px-6 py-4">
			<div class="flex items-center">
				<div class="h-10 w-10 flex-shrink-0">
					<a href="https://njump.me/{rating.from.npub}" target="_blank" class="h-full w-full">
						<ProfileAvatar source={rating.from.picture} />
					</a>
				</div>

				<div class="ml-4">
					<div class="text-sm font-medium leading-5 text-gray-200">
						{rating.from.display_name || rating.from.name || ''}
					</div>

					<div class="group relative text-sm leading-5 text-gray-500">
						<button on:click={() => copyNpub(rating.from.npub)}>
							{`${rating.from.npub.slice(0, 5)}...${rating.from.npub.slice(-5)}`}
						</button>

						<span
							class="absolute left-0 top-full z-10 hidden whitespace-nowrap rounded-md border border-white bg-gray-800 p-2 text-sm text-white group-hover:block"
						>
							{rating.from.npub}
						</span>
					</div>
				</div>
			</div>
		</td>

		<td class="whitespace-no-wrap px-6 py-4">
			<div class="flex items-center">
				<div class="h-10 w-10 flex-shrink-0">
					<a href="https://njump.me/{rating.to.npub}" target="_blank" class="h-full w-full">
						<ProfileAvatar source={rating.to.picture} />
					</a>
				</div>

				<div class="ml-4">
					<div class="text-sm font-medium leading-5 text-gray-200">
						{rating.to.display_name || rating.to.name || ''}
					</div>

					<div class="group relative text-sm leading-5 text-gray-500">
						<button on:click={() => copyNpub(rating.to.npub)}>
							{`${rating.to.npub.slice(0, 5)}...${rating.to.npub.slice(-5)}`}
						</button>

						<span
							class="absolute left-0 top-full z-10 hidden whitespace-nowrap rounded-md border border-white bg-gray-800 p-2 text-sm text-white group-hover:block"
						>
							{rating.to.npub}
						</span>
					</div>
				</div>
			</div>
		</td>

		<td class="whitespace-no-wrap px-6 py-4">
			{#if rating.score}
				<span
					class="inline-flex text-nowrap rounded-full bg-green-100 px-2 text-xs font-semibold leading-5 text-green-800"
					>✅ Positive</span
				>
			{:else}
				<span
					class="inline-flex text-nowrap rounded-full bg-red-100 px-2 text-xs font-semibold leading-5 text-red-800"
					>❌ Negative</span
				>
			{/if}
		</td>

		<td class="whitespace-no-wrap px-6 py-4">
			{#if rating.businessAlreadyDone}
				<span
					class="inline-flex text-nowrap rounded-full bg-green-100 px-2 text-xs font-semibold leading-5 text-green-800"
					>✅ Yes</span
				>
			{:else}
				<span
					class="inline-flex text-nowrap rounded-full bg-red-100 px-2 text-xs font-semibold leading-5 text-red-800"
					>❌ No</span
				>
			{/if}
		</td>

		<td class="whitespace-no-wrap px-6 py-4">
			{new Date(rating.date).toLocaleDateString(undefined, {
				day: '2-digit',
				month: '2-digit',
				year: 'numeric',
				hour: 'numeric',
				minute: 'numeric',
				second: 'numeric',
				hour12: false
			})}
		</td>

		<td class="whitespace-no-wrap px-6 py-4">
			{#if rating.from.lud16}
							<div class="p-2">
								<button
									type="button"
									class="rounded-lg p-2.5 text-sm text-orange-500 transition-colors hover:bg-orange-600 hover:text-white focus:ring-2 focus:ring-orange-300"
									on:click={() => zapModal.openModal(rating.from.npub, rating.eventId)}
								>
									Send Zap
								</button>
							</div>
						{/if}
		</td>
	</tr>

	<tr class="border-x border-b border-gray-200 bg-slate-700">
		{#if rating.description}
			<td colspan="6" class="whitespace-no-wrap max-w-fit px-6 py-4">
				<textarea
					bind:this={textarea}
					bind:value={rating.description}
					class="w-full resize-none border-0"
					rows="3"
					readonly
					disabled
				></textarea>

				{#if rating.description.length > 250}
					<div class="pt-4 text-center">
						<button class="text-sm text-orange-500 hover:text-white" on:click={toggleExpanded}>
							{expanded ? 'Collapse' : 'Expand'}
						</button>
					</div>
				{/if}
			</td>
		{/if}
	</tr>
{/if}
