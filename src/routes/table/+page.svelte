<script lang="ts">
	import { ReviewEvent } from '$lib';
	import {
		getProfileMetadata,
		nostrAuth,
		parseProfileFromJsonString,
		relayList,
		relayPool,
		type ProfileType,
		type Rating
	} from '$lib/nostr';
	import { npubEncode } from 'nostr-tools/nip19';
	import { onMount } from 'svelte';
	import ZapModal from '$lib/components/ZapModal.svelte';
	import { Input, Label, Select } from 'flowbite-svelte';
	import type { Event } from 'nostr-tools';
	import { page } from '$app/state';
	import { replaceState } from '$app/navigation';
	import { toasts } from 'svelte-toasts';
	import Table from '$lib/components/Table/Table.svelte';

	let ZapModalComponent: ZapModal;

	let ratings: Rating[] = [];

	// Necessary to ensure that page is loaded before try to set any search param
	// It ensures that replaceState would not be called before router initialization
	let pageInitialized: boolean = false;

	let filterRating: string = 'all';
	let filterBusiness: string = 'all';
	let filterFrom: string = getRaterParam();
	let filterTo: string = getRatedParam();

	$: filteredRatings = ratings.filter((rating) => {
		let ratingMatch = true;
		if (filterRating === 'positive') {
			ratingMatch = rating.score === true;
		} else if (filterRating === 'negative') {
			ratingMatch = rating.score === false;
		}

		let businessMatch = true;
		if (filterBusiness === 'yes') {
			businessMatch = rating.businessAlreadyDone === true;
		} else if (filterBusiness === 'no') {
			businessMatch = rating.businessAlreadyDone === false;
		}

		let fromMatch = true;
		if (filterFrom.trim() !== '') {
			fromMatch = rating.from.npub.toLowerCase().includes(filterFrom.toLowerCase());
		}

		let toMatch = true;
		if (filterTo.trim() !== '') {
			toMatch = rating.to.npub.toLowerCase().includes(filterTo.toLowerCase());
		}

		return ratingMatch && businessMatch && fromMatch && toMatch;
	});

	$: ratings.sort((a, b) => b.date - a.date);

	$: setRaterParam(filterFrom);
	$: setRatedParam(filterTo);

	const events: Event[] = [];

	onMount(() => {
		pageInitialized = true;

		relayPool.subscribeMany(
			relayList,
			[
				{
					kinds: [ReviewEvent],
					'#l': ['pls-wot-rating']
				}
			],
			{
				onevent(e) {
					try {
						events.push(e);

						const c = JSON.parse(e.content);

						const from: ProfileType = {
							npub: npubEncode(c.from),
							pubkey: c.from
						};

						const to: ProfileType = {
							npub: npubEncode(c.to),
							pubkey: c.to
						};

						const newRating: Rating = {
							eventId: e.id,
							from: from,
							to: to,
							date: e.created_at * 1000,
							score: c.score,
							businessAlreadyDone: c.businessAlreadyDone,
							description: c.description
						};

						ratings = ratings.filter((r) => !(r.eventId === newRating.eventId));

						ratings = [...ratings, newRating];

						Promise.all([getProfileMetadata(c.from), getProfileMetadata(c.to)])
							.then(([fromEvent, toEvent]) => {
								Object.assign(from, parseProfileFromJsonString(fromEvent?.content || '{}', from));

								Object.assign(to, parseProfileFromJsonString(toEvent?.content || '{}', to));
							})
							.catch((error) => {
								console.error('Error when processing the profile metadata:', error);
							})
							.finally(() => {
								const ratingIndex = ratings.findIndex((r) => r.eventId === newRating.eventId);
								ratings[ratingIndex] = newRating;
							});
					} catch (error) {
						console.error('Error processing the event:', error);
					}
				}
			}
		);
	});

	const download = (filename: string, text: any) => {
		var element = document.createElement('a');
		element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(text));
		element.setAttribute('download', filename);
		element.style.display = 'none';
		document.body.appendChild(element);
		element.click();
		document.body.removeChild(element);
	};

	function getRaterParam(): string {
		return page.url.searchParams.get('rater') || '';
	}

	function setRaterParam(rater: string) {
		// Ensure that page has initialized
		if (!pageInitialized) return;

		if (rater) {
			page.url.searchParams.set('rater', rater);
		} else {
			page.url.searchParams.delete('rater');
		}
		replaceState(page.url, page.state);
	}

	function getRatedParam(): string {
		return page.url.searchParams.get('rated') || '';
	}

	function setRatedParam(rated: string) {
		// Ensure that page has initialized
		if (!pageInitialized) return;

		if (rated) {
			page.url.searchParams.set('rated', rated);
		} else {
			page.url.searchParams.delete('rated');
		}
		replaceState(page.url, page.state);
	}

	async function copyLinkToClipboard() {
		await navigator.clipboard.writeText(page.url.toString());
		toasts.success({
			title: 'Copied!',
			description: 'Link copied to clipboard!'
		});
	}

	const handleDownload = (myRatings: boolean = false) => {
		if (myRatings) {
			if (!$nostrAuth?.pubkey) {
				toasts.error({
					title: 'Not logged in',
					description: 'You must be logged to perform this action'
				});
				return;
			}

			let myRatingsEventId = ratings
				.filter((r) => r.from.pubkey === $nostrAuth?.pubkey || r.to.pubkey === $nostrAuth?.pubkey)
				.map((r) => {
					return r.eventId;
				});

			let myRatingsEvents = events.filter((e) => myRatingsEventId.includes(e.id));

			return download('ratings.json', JSON.stringify(myRatingsEvents, null, '\t'));
		}

		let filteredRatingsEventId = filteredRatings.map((r) => {
			return r.eventId;
		});

		let filteredRatingsEvents = events.filter((e) => filteredRatingsEventId.includes(e.id));

		return download('ratings.json', JSON.stringify(filteredRatingsEvents, null, '\t'));
	};
</script>

<ZapModal bind:this={ZapModalComponent} />

<div class="w-full md:px-[5%] space-y-8">
	<h1 class="text-2xl font-bold text-center">Ratings table (Currently using replaceable events)</h1>

	<div class="grid grid-cols-12 justify-center gap-4 px-[5%] md:px-0">
		<div class="col-span-12 sm:col-span-6 xl:col-span-2">
			<Label for="filterRating" class="font-semibold">Filter by Rating:</Label>
			<Select
				id="filterRating"
				bind:value={filterRating}
				items={[
					{ value: 'all', name: 'All' },
					{ value: 'positive', name: '✅ Positive' },
					{ value: 'negative', name: '❌ Negative' }
				]}
				class="rounded border border-gray-300 bg-white px-2 py-1 text-black
				       transition-colors focus:outline-none focus:ring-2 focus:ring-blue-500"
			/>
		</div>

		<div class="col-span-12 sm:col-span-6 xl:col-span-2">
			<Label for="filterBusiness" class="font-semibold">Filter by Had Business:</Label>
			<Select
				id="filterBusiness"
				bind:value={filterBusiness}
				items={[
					{ value: 'all', name: 'All' },
					{ value: 'yes', name: '✅ Yes' },
					{ value: 'no', name: '❌ No' }
				]}
				class="rounded border border-gray-300 bg-white px-2 py-1 text-black
				       transition-colors focus:outline-none focus:ring-2 focus:ring-blue-500"
			/>
		</div>

		<div class="col-span-12 sm:col-span-6 xl:col-span-2">
			<Label for="filterFrom" class="font-semibold">Filter by Who Rated:</Label>
			<Input
				id="filterFrom"
				bind:value={filterFrom}
				placeholder="Enter Rater Key"
				autocomplete="off"
				class="rounded border border-gray-300 bg-white px-2 py-1 text-black
				       transition-colors focus:outline-none focus:ring-2 focus:ring-blue-500"
			/>
		</div>

		<div class="col-span-12 sm:col-span-6 xl:col-span-2">
			<Label for="filterTo" class="font-semibold">Filter by Who Was Rated:</Label>
			<Input
				id="filterTo"
				bind:value={filterTo}
				placeholder="Enter Rated Key"
				autocomplete="off"
				class="rounded border border-gray-300 bg-white px-2 py-1 text-black
				       transition-colors focus:outline-none focus:ring-2 focus:ring-blue-500"
			/>
		</div>

		<div class="col-span-6 xl:col-span-2">
			<label for="downloadReviews" class="font-semibold">Download reviews:</label>

			<div class="flex grid-cols-2 gap-2">
				<button
					type="button"
					class="rounded border border-gray-600 bg-gray-700 px-2 py-1 text-white transition-colors hover:bg-orange-600 focus:border-primary-500 focus:outline-none focus:ring-2 focus:ring-primary-500"
					on:click={() => handleDownload()}
				>
					From filters
				</button>

				{#if $nostrAuth?.pubkey}
					<button
						type="button"
						class="rounded border border-gray-600 bg-gray-700 px-2 py-1 text-white transition-colors hover:bg-orange-600 focus:border-primary-500 focus:outline-none focus:ring-2 focus:ring-primary-500"
						on:click={() => handleDownload(true)}
					>
						All My Reviews
					</button>
				{/if}
			</div>
		</div>

		<div class="col-span-6 xl:col-span-2">
			<label for="getFilterLinks" class="font-semibold">Get filters link:</label>

			<div class="grid-cols flex gap-2">
				<button
					type="button"
					class="rounded border border-gray-600 bg-gray-700 px-2 py-1 text-white transition-colors hover:bg-orange-600 focus:border-primary-500 focus:outline-none focus:ring-2 focus:ring-primary-500"
					on:click={() => copyLinkToClipboard()}
				>
					Copy to clipboard
				</button>
			</div>
		</div>
	</div>

	<Table ratings={filteredRatings} zapModal={ZapModalComponent} />
</div>
