<script lang="ts">
	import CardListEntry from '../CardListEntry.svelte';
	import type { Node } from '$lib/common/types';
	import OnlineUserIndicator from '$lib/parts/OnlineUserIndicator.svelte';
	import { openDrawer } from '$lib/common/funcs';
	import { getDrawerStore } from '@skeletonlabs/skeleton';

	type NodeOwnerProps = {
		node: Node;
	};
	let { node }: NodeOwnerProps = $props();

	const drawerStore = getDrawerStore();
</script>

<CardListEntry title="Owner:" top>
	<div class="flex flex-row items-center gap-3 justify-end">
		{#if node.user}
			<a
				href=" "
				onclick={() => {
					openDrawer(drawerStore, 'userDrawer-' + node.user.id, node.user);
				}}
			>
				{node.user.name}
			</a>
			<OnlineUserIndicator user={node.user} />
		{:else if node.tags.length > 0}
			<span class="badge variant-soft-secondary">
				Tagged: {node.tags.map((t) => t.replace('tag:', '')).join(', ')}
			</span>
		{:else}
			<span class="text-surface-500">No Owner</span>
		{/if}
	</div>
</CardListEntry>
