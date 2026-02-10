<script lang="ts">
	import { getToastStore, getModalStore } from '@skeletonlabs/skeleton';
	import type { ModalSettings } from '@skeletonlabs/skeleton';

	import type { Node } from '$lib/common/types';
	import { setNodeTags } from '$lib/common/api';
	import { toastError, toastWarning } from '$lib/common/funcs';
	import DOMPurify from 'dompurify';
	import CardListEntry from '../CardListEntry.svelte';

	import { App } from '$lib/States.svelte';

	type NodeTagsProps = {
		node: Node;
	};

	let { node = $bindable() }: NodeTagsProps = $props();

	let tags = $state(node.tags.map((tag) => tag.replace('tag:', '')));
	let pendingAction = $state(false);

	// Sync tags when node changes externally (but not during pending user actions)
	let prevNodeId = $state(node.id);
	let prevNodeTags = $state(node.tags);
	$effect(() => {
		if (node.id !== prevNodeId || (node.tags !== prevNodeTags && !pendingAction)) {
			tags = node.tags.map((tag) => tag.replace('tag:', ''));
			prevNodeId = node.id;
			prevNodeTags = node.tags;
		}
	});

	const availableTags = $derived(App.aclTags.value.filter((t) => !tags.includes(t)));

	let disabled = $state(false);
	let selectedTag = $state('');

	const ToastStore = getToastStore();
	const ModalStore = getModalStore();

	function resetTags() {
		tags = node.tags.map((tag) => tag.replace('tag:', ''));
		pendingAction = false;
	}

	async function applyTags() {
		disabled = true;
		try {
			const hadUser = !!node.user;
			const n = await setNodeTags(node, tags);
			App.updateValue(App.nodes, n);
			pendingAction = false;
			if (hadUser && !n.user) {
				toastWarning(
					`Tags applied. User ownership removed (tags and users are mutually exclusive).`,
					ToastStore,
				);
			}
		} catch (e) {
			toastError('Invalid Tags: ' + e, ToastStore);
			resetTags();
		} finally {
			disabled = false;
		}
	}

	async function saveTags() {
		pendingAction = true;

		// Prevent removing all tags from a tagged node (must have at least one tag for identity)
		if (!node.user && tags.length === 0) {
			toastError(
				'Tagged nodes must have at least one tag. To remove all tags, assign a user instead.',
				ToastStore,
			);
			resetTags();
			return;
		}

		// Warn user if they're about to remove user ownership by applying tags
		if (node.user && tags.length > 0 && node.tags.length === 0) {
			const safeName = DOMPurify.sanitize(node.user.name);
			const modal: ModalSettings = {
				type: 'confirm',
				title: 'Remove User Ownership?',
				body: `<p class="mb-2">Applying tags to this node will <strong>remove user ownership</strong>.</p>
					   <p class="mb-2">Current owner: <strong>${safeName}</strong></p>
					   <p>Tags and user accounts are mutually exclusive. This node will become a tag-based identity.</p>
					   <p class="mt-2 text-sm text-surface-600 dark:text-surface-400"><strong>Note:</strong> Once tagged, the node must always have at least one tag.</p>`,
				response: (confirmed: boolean) => {
					if (confirmed) {
						applyTags();
					} else {
						resetTags();
						disabled = false;
					}
				},
			};
			ModalStore.trigger(modal);
		} else {
			await applyTags();
		}
	}

	function addTag(tag: string) {
		if (tag && !tags.includes(tag)) {
			tags = [...tags, tag];
			selectedTag = '';
			saveTags();
		}
	}

	function removeTag(tag: string) {
		tags = tags.filter((t) => t !== tag);
		saveTags();
	}
</script>

<CardListEntry top title="Tags:">
	<div class="flex flex-wrap gap-1 items-center">
		{#each tags as tag}
			<span class="badge variant-filled-success flex items-center gap-1">
				{tag}
				<button {disabled} class="ml-1 hover:text-error-500" onclick={() => removeTag(tag)}>
					&times;
				</button>
			</span>
		{/each}
		{#if availableTags.length > 0}
			<select
				{disabled}
				class="select text-sm rounded-md w-auto min-w-[120px]"
				bind:value={selectedTag}
				onchange={() => {
					if (selectedTag) addTag(selectedTag);
				}}
			>
				<option value="">Add tag...</option>
				{#each availableTags as tag}
					<option value={tag}>{tag}</option>
				{/each}
			</select>
		{/if}
	</div>
</CardListEntry>
