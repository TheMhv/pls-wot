<script lang="ts">
	import type { Rating } from '$lib/nostr';
	import ZapModal from '../ZapModal.svelte';
	import Head from './Head.svelte';
	import Row from './Row.svelte';

    export let ratings: Rating[];
    export let zapModal: ZapModal;
</script>

<table class="min-w-full">
	<Head />

	<tbody>
        {#each ratings as rating}
            <Row rating={rating} zapModal={zapModal}/>
        {/each}
	</tbody>
</table>
