<script lang="ts">
    import PaginationInline from './paginationInline.svelte';

    export let items = [];
    export let limit = 5;

    let offset = 0;

    $: total = items?.length;
    $: paginatedItems = items.slice(offset, offset + limit);
</script>

<slot {paginatedItems} />

<div class="u-flex u-margin-block-start-32 u-main-space-between">
    <p class="text">Total results: {total}</p>
    <PaginationInline {limit} bind:offset sum={total} hidePages />
</div>
