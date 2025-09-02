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
	import ProfileAvatar from '$lib/components/ProfileAvatar.svelte';
	import { Input, Label, Select } from 'flowbite-svelte';
	import type { Event } from 'nostr-tools';
	import { page } from '$app/state';
	import { replaceState } from '$app/navigation';
	import { toasts } from 'svelte-toasts';

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

	async function copyNpub(npub: string) {
		await navigator.clipboard.writeText(npub);
		toasts.success({
			title: 'Copied NPUB!',
			description: 'NPUB copied to clipboard'
		});
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

	let expandedItems = new Set();

	function toggleExpanded(id) {
		if (expandedItems.has(id)) {
			expandedItems.delete(id);
		} else {
			expandedItems.add(id);
		}
		expandedItems = expandedItems;
	}
</script>

<ZapModal bind:this={ZapModalComponent} />

<div class="flex justify-center">
	<div class="w-[90%] space-y-12">
		<div class="text-center">
			<h1 class="text-2xl font-bold">Ratings table (Currently using replaceable events)</h1>
		</div>

		<div class="grid w-full grid-cols-12 gap-4">
			<div class="col-span-12 md:col-span-6 2xl:col-span-2">
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
	
			<div class="col-span-12 md:col-span-6 2xl:col-span-2">
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
	
			<div class="col-span-12 md:col-span-6 2xl:col-span-2">
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
	
			<div class="col-span-12 md:col-span-6 2xl:col-span-2">
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
	
			<div class="col-span-12 md:col-span-6 2xl:col-span-2">
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
	
			<div class="col-span-12 md:col-span-6 2xl:col-span-2">
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
	
		<table class="min-w-full">
		<thead>
			<tr>
				<th
					class="border-b border-gray-200 bg-slate-700 px-6 py-3 text-left text-xs font-medium uppercase leading-4 tracking-wider text-gray-200"
				>
					Rater
				</th>
				<th
					class="border-b border-gray-200 bg-slate-700 px-6 py-3 text-left text-xs font-medium uppercase leading-4 tracking-wider text-gray-200"
				>
					Rated
				</th>
				<th
					class="border-b border-gray-200 bg-slate-700 px-6 py-3 text-left text-xs font-medium uppercase leading-4 tracking-wider text-gray-200"
				>
					Rating
				</th>
				<th
					class="border-b border-gray-200 bg-slate-700 px-6 py-3 text-left text-xs font-medium uppercase leading-4 tracking-wider text-gray-200"
				>
					Has <br /> Business
				</th>
				<th class="border-b border-gray-200 bg-slate-700 px-6 py-3"></th>
			</tr>
		</thead>
	
		<tbody>
			<tr class="border-x border-y-0">
				<td class="whitespace-no-wrap border-b border-gray-200 px-6 py-4">
					<div class="flex items-center">
						<div class="h-10 w-10 flex-shrink-0">
							<a
								href="https://njump.me/npub1v3ps5nhexd9fdur4gz82xgc3jmhqwduqhrhy7lwtmm727m086u5sqnuvcz"
								target="_blank"
								class="h-full w-full"
							>
								<ProfileAvatar source="https://avatars.githubusercontent.com/u/25031483" />
							</a>
						</div>
	
						<div class="ml-4">
							<div class="text-sm font-medium leading-5 text-gray-200">TheMhv</div>
	
							<div class="group relative text-sm leading-5 text-gray-500">
								<button
									on:click={() =>
										copyNpub('npub1v3ps5nhexd9fdur4gz82xgc3jmhqwduqhrhy7lwtmm727m086u5sqnuvcz')}
								>
									{`${'npub1v3ps5nhexd9fdur4gz82xgc3jmhqwduqhrhy7lwtmm727m086u5sqnuvcz'.slice(0, 5)}...${'npub1v3ps5nhexd9fdur4gz82xgc3jmhqwduqhrhy7lwtmm727m086u5sqnuvcz'.slice(-5)}`}
								</button>
	
								<span
									class="absolute left-0 top-full z-10 hidden whitespace-nowrap rounded-md border border-white bg-gray-800 p-2 text-sm text-white group-hover:block"
								>
									npub1v3ps5nhexd9fdur4gz82xgc3jmhqwduqhrhy7lwtmm727m086u5sqnuvcz
								</span>
							</div>
						</div>
					</div>
				</td>
	
				<td class="whitespace-no-wrap border-b border-gray-200 px-6 py-4">
					<div class="flex items-center">
						<div class="h-10 w-10 flex-shrink-0">
							<a
								href="https://njump.me/npub1erkyl33ttzjpxznpra9f3r3xhaafnrpufhsctur26hyydfy8vlasm9u8qc"
								target="_blank"
								class="h-full w-full"
							>
								<ProfileAvatar
									source="https://blossom.primal.net/24242d039c11a10b61d3f5cbf8e4e771a5d7342c92a65e4440d267cbb5b1be9a.png"
								/>
							</a>
						</div>
	
						<div class="ml-4">
							<div class="text-sm font-medium leading-5 text-gray-200">Daniel Smith</div>
	
							<div class="group relative text-sm leading-5 text-gray-500">
								<button
									on:click={() =>
										copyNpub('npub1erkyl33ttzjpxznpra9f3r3xhaafnrpufhsctur26hyydfy8vlasm9u8qc')}
								>
									{`${'npub1erkyl33ttzjpxznpra9f3r3xhaafnrpufhsctur26hyydfy8vlasm9u8qc'.slice(0, 5)}...${'npub1erkyl33ttzjpxznpra9f3r3xhaafnrpufhsctur26hyydfy8vlasm9u8qc'.slice(-5)}`}
								</button>
	
								<span
									class="absolute left-0 top-full z-10 hidden whitespace-nowrap rounded-md border border-white bg-gray-800 p-2 text-sm text-white group-hover:block"
								>
									npub1erkyl33ttzjpxznpra9f3r3xhaafnrpufhsctur26hyydfy8vlasm9u8qc
								</span>
							</div>
						</div>
					</div>
				</td>
	
				<td class="whitespace-no-wrap border-b border-gray-200 px-6 py-4">
					<span
						class="inline-flex text-nowrap rounded-full bg-green-100 px-2 text-xs font-semibold leading-5 text-green-800"
						>✅ Positive</span
					>
				</td>
	
				<td class="whitespace-no-wrap border-b border-gray-200 px-6 py-4">
					<span
						class="inline-flex rounded-full bg-green-100 px-2 text-xs font-semibold leading-5 text-green-800"
						>✅ Yes</span
					>
				</td>
	
				<td
					class="whitespace-no-wrap border-b border-gray-200 px-6 py-4 text-right text-sm font-medium leading-5"
				>
					<a href="#" class="text-indigo-600 hover:text-indigo-900">Show More</a>
				</td>
			</tr>

			<tr class="border-x border-y-0">
				<td class="whitespace-no-wrap border-b border-gray-200 px-6 py-4">
					<div class="flex items-center">
						<div class="h-10 w-10 flex-shrink-0">
							<a
								href="https://njump.me/npub1v3ps5nhexd9fdur4gz82xgc3jmhqwduqhrhy7lwtmm727m086u5sqnuvcz"
								target="_blank"
								class="h-full w-full"
							>
								<ProfileAvatar source="https://avatars.githubusercontent.com/u/25031483" />
							</a>
						</div>
	
						<div class="ml-4">
							<div class="text-sm font-medium leading-5 text-gray-200">TheMhv</div>
	
							<div class="group relative text-sm leading-5 text-gray-500">
								<button
									on:click={() =>
										copyNpub('npub1v3ps5nhexd9fdur4gz82xgc3jmhqwduqhrhy7lwtmm727m086u5sqnuvcz')}
								>
									{`${'npub1v3ps5nhexd9fdur4gz82xgc3jmhqwduqhrhy7lwtmm727m086u5sqnuvcz'.slice(0, 5)}...${'npub1v3ps5nhexd9fdur4gz82xgc3jmhqwduqhrhy7lwtmm727m086u5sqnuvcz'.slice(-5)}`}
								</button>
	
								<span
									class="absolute left-0 top-full z-10 hidden whitespace-nowrap rounded-md border border-white bg-gray-800 p-2 text-sm text-white group-hover:block"
								>
									npub1v3ps5nhexd9fdur4gz82xgc3jmhqwduqhrhy7lwtmm727m086u5sqnuvcz
								</span>
							</div>
						</div>
					</div>
				</td>
	
				<td class="whitespace-no-wrap border-b border-gray-200 px-6 py-4">
					<div class="flex items-center">
						<div class="h-10 w-10 flex-shrink-0">
							<a
								href="https://njump.me/npub1erkyl33ttzjpxznpra9f3r3xhaafnrpufhsctur26hyydfy8vlasm9u8qc"
								target="_blank"
								class="h-full w-full"
							>
								<ProfileAvatar
									source="https://blossom.primal.net/24242d039c11a10b61d3f5cbf8e4e771a5d7342c92a65e4440d267cbb5b1be9a.png"
								/>
							</a>
						</div>
	
						<div class="ml-4">
							<div class="text-sm font-medium leading-5 text-gray-200">Daniel Smith</div>
	
							<div class="group relative text-sm leading-5 text-gray-500">
								<button
									on:click={() =>
										copyNpub('npub1erkyl33ttzjpxznpra9f3r3xhaafnrpufhsctur26hyydfy8vlasm9u8qc')}
								>
									{`${'npub1erkyl33ttzjpxznpra9f3r3xhaafnrpufhsctur26hyydfy8vlasm9u8qc'.slice(0, 5)}...${'npub1erkyl33ttzjpxznpra9f3r3xhaafnrpufhsctur26hyydfy8vlasm9u8qc'.slice(-5)}`}
								</button>
	
								<span
									class="absolute left-0 top-full z-10 hidden whitespace-nowrap rounded-md border border-white bg-gray-800 p-2 text-sm text-white group-hover:block"
								>
									npub1erkyl33ttzjpxznpra9f3r3xhaafnrpufhsctur26hyydfy8vlasm9u8qc
								</span>
							</div>
						</div>
					</div>
				</td>
	
				<td class="whitespace-no-wrap border-b border-gray-200 px-6 py-4">
					<span
						class="inline-flex text-nowrap rounded-full bg-green-100 px-2 text-xs font-semibold leading-5 text-green-800"
						>✅ Positive</span
					>
				</td>
	
				<td class="whitespace-no-wrap border-b border-gray-200 px-6 py-4">
					<span
						class="inline-flex rounded-full bg-green-100 px-2 text-xs font-semibold leading-5 text-green-800"
						>✅ Yes</span
					>
				</td>
	
				<td
					class="whitespace-no-wrap border-b border-gray-200 px-6 py-4 text-right text-sm font-medium leading-5"
				>
					<a href="#" class="text-indigo-600 hover:text-indigo-900">Show More</a>
				</td>
			</tr>
		</tbody>
	</table>
	</div>
</div>
