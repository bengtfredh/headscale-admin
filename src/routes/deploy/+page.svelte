<script lang="ts">
	import {
		copyToClipboard,
		isValidCIDR,
		toastError,
		toastSuccess,
	} from '$lib/common/funcs';
	import DeployCheck from './DeployCheck.svelte';
	import Page from '$lib/page/Page.svelte';
	import PageHeader from '$lib/page/PageHeader.svelte';
	import type { Deployment } from '$lib/common/types';
	import { InputChip, getToastStore } from '@skeletonlabs/skeleton';
	import { page } from '$app/state';
	import { slide } from 'svelte/transition';
	import { createPreAuthKey } from '$lib/common/api';
	import { debug } from '$lib/common/debug';

	import { App } from '$lib/States.svelte';

	const ToastStore = getToastStore();

	let deployment: Deployment = $state(App.deploymentDefaults.value);
	let creatingKey = $state(false);
	let createKeyEphemeral = $state(false);
	let createKeyReusable = $state(false);
	let createKeyAclTags: string[] = $state([]);

	let userOwnedTags = $derived.by(() => {
		const user = App.users.value.find((u) => u.id === deployment.preAuthKeyUser);
		if (!user) return [];
		const tagOwners = App.aclTagOwners.value;
		return Object.entries(tagOwners)
			.filter(([, owners]) => owners.includes(user.name))
			.map(([tag]) => tag.replace(/^tag:/, ''));
	});

	let command = $derived.by(() => {
		const d = deployment;
		const cmd = ['tailscale up --login-server=' + (App.apiUrl.value || page.url.origin)];

		// general
		d.shieldsUp && cmd.push('--shields-up');
		d.generateQR && cmd.push('--qr');
		d.reset && cmd.push('--reset');
		d.operator && d.operatorValue != '' && cmd.push('--operator=' + d.operatorValue);
		d.forceReauth && cmd.push('--force-reauth');
		d.sshServer && cmd.push('--ssh');
		d.usePreAuthKey && d.preAuthKey !== '' && cmd.push('--auth-key=' + d.preAuthKey);
		d.unattended && cmd.push('--unattended');

		// advertise
		d.advertiseExitNode && cmd.push('--advertise-exit-node');
		d.advertiseExitNodeLocalAccess && cmd.push('--exit-node-allow-lan-access');
		d.advertiseRoutes &&
			d.advertiseRoutesValues.length > 0 &&
			cmd.push('--advertise-routes=' + d.advertiseRoutesValues.join(','));
		d.advertiseTags &&
			d.advertiseTagsValues.length > 0 &&
			cmd.push(
				'--advertise-tags=' +
					d.advertiseTagsValues.map((s) => (s.startsWith('tag:') ? s : 'tag:' + s)).join(','),
			);

		// accept
		d.acceptDns ? cmd.push('--accept-dns') : cmd.push('--accept-dns=false');
		d.acceptRoutes && cmd.push('--accept-routes');
		d.acceptExitNode && d.acceptExitNodeValue && cmd.push('--exit-node=' + d.acceptExitNodeValue);
		return cmd.join(' ');
	});
</script>

<Page>
	<PageHeader title="Deploy" buttonText={''} show={true}>
		{#snippet button()}
			<button
				class="bg-gray-400/30 dark:bg-gray-800/70 border border-dashed border-slate-200 border-1 pr-0 pl-4 rounded-lg justify-start text-left w-[90%]"
				onclick={() => copyToClipboard(command, ToastStore, 'Copied Command to Clipboard!')}
				><code class="text-black dark:text-white text-sm block py-4 w-full">{command}</code>
			</button>
		{/snippet}
	</PageHeader>

	<div class="grid grid-cols-12">
		<p class="text-xl col-span-12">General:</p>
		<DeployCheck
			bind:checked={deployment.shieldsUp}
			name="Shields Up"
			help="Block incoming connections"
		/>
		<DeployCheck
			bind:checked={deployment.generateQR}
			name="Generate QR Code"
			help="Create a scannable QR code to import into TailScale client"
		/>
		<DeployCheck
			bind:checked={deployment.reset}
			name="Reset"
			help="Reset unspecified settings to default values"
		/>
		<DeployCheck
			bind:checked={deployment.operator}
			name="Operator"
			help="(Unix Only) Run as a different user"
		>
			<input type="text" class="input text-sm rounded-md" bind:value={deployment.operatorValue} />
		</DeployCheck>
		<DeployCheck
			bind:checked={deployment.forceReauth}
			name="Force Reauthentication"
			help="Force user to re-authenticate to Headscale server"
		/>
		<DeployCheck
			bind:checked={deployment.sshServer}
			name="SSH Server"
			help="Run a local SSH server accessible by administrators"
		/>
		<DeployCheck
			bind:checked={deployment.usePreAuthKey}
			name="PreAuth Key"
			help="A generated key to automatically authenticate the node for a given user"
		>
			<div class="flex flex-col gap-2">
				<input
					type="text"
					class="input text-sm rounded-md font-mono"
					placeholder="Paste a key or create one below"
					bind:value={deployment.preAuthKey}
				/>
				<div class="flex flex-row flex-wrap gap-2 items-center">
					<select
						bind:value={deployment.preAuthKeyUser}
						class="input rounded-md flex-1"
						onchange={() => {
							createKeyAclTags = [];
						}}
					>
						<option value="">Select user</option>
						{#each App.users.value as user}
							<option value={user.id}>{user.name}</option>
						{/each}
					</select>
					<button
						type="button"
						disabled={!deployment.preAuthKeyUser || creatingKey}
						class="btn btn-sm rounded-md variant-filled-success px-3"
						onclick={async () => {
							creatingKey = true;
							try {
								const user = App.users.value.find((u) => u.id === deployment.preAuthKeyUser);
								if (!user) return;
								const pak = await createPreAuthKey(
									user,
									createKeyEphemeral,
									createKeyReusable,
									new Date(Date.now() + 60 * 60 * 1000),
									createKeyAclTags,
								);
								deployment.preAuthKey = pak.key;
								App.preAuthKeys.value.push(pak);
							} catch (e) {
								debug(e);
								toastError('Failed to create PreAuth Key', ToastStore);
							} finally {
								creatingKey = false;
							}
						}}
					>
						{creatingKey ? 'Creating...' : 'Create'}
					</button>
				</div>
				<div class="flex flex-row gap-3 items-center text-sm">
					<label class="flex items-center space-x-1">
						<input class="checkbox" type="checkbox" bind:checked={createKeyEphemeral} />
						<span>Ephemeral</span>
					</label>
					<label class="flex items-center space-x-1">
						<input class="checkbox" type="checkbox" bind:checked={createKeyReusable} />
						<span>Reusable</span>
					</label>
				</div>
				{#if createKeyAclTags.length > 0}
					<div class="flex flex-wrap gap-1">
						{#each createKeyAclTags as tag}
							<span class="badge variant-filled-success flex items-center gap-1">
								{tag}
								<button
									class="ml-1 hover:text-error-500"
									onclick={() => {
										createKeyAclTags = createKeyAclTags.filter((t) => t !== tag);
									}}
								>
									&times;
								</button>
							</span>
						{/each}
					</div>
				{/if}
				{#if userOwnedTags.filter((t) => !createKeyAclTags.includes(t)).length > 0}
					<select
						class="select text-sm rounded-md"
						onchange={(e) => {
							const val = e.currentTarget.value;
							if (val && !createKeyAclTags.includes(val)) {
								createKeyAclTags = [...createKeyAclTags, val];
							}
							e.currentTarget.value = '';
						}}
					>
						<option value="">ACL Tags (registers as tagged device)</option>
						{#each userOwnedTags.filter((t) => !createKeyAclTags.includes(t)) as tag}
							<option value={tag}>{tag}</option>
						{/each}
					</select>
				{/if}
			</div>
		</DeployCheck>
		<DeployCheck
			bind:checked={deployment.unattended}
			name="Unattended"
			help="Run the tailscale client in unattended mode (on startup)"
		/>
		<DeployCheck
			bind:checked={deployment.advertiseExitNodeLocalAccess}
			name="Allow LAN Access"
			help="Allow local network access while connected to the TailNet and using an exit node"
		/>

		<p class="text-xl col-span-12 py-4">Advertise:</p>
		<DeployCheck
			bind:checked={deployment.advertiseExitNode}
			name="Advertise Exit Node"
			help="Allow other nodes on the TailNet to use this node as a gateway"
		/>
		<DeployCheck
			bind:checked={deployment.advertiseTags}
			name="Advertise Tags"
			help="Give tagged permissions to this device. You must be listed in TagOwners to apply tags."
		>
			<div class="flex flex-col gap-2">
				{#if deployment.advertiseTagsValues.length > 0}
					<div class="flex flex-wrap gap-1">
						{#each deployment.advertiseTagsValues as tag}
							<span class="badge variant-filled-primary flex items-center gap-1">
								{tag}
								<button
									class="ml-1 hover:text-error-500"
									onclick={() => {
										deployment.advertiseTagsValues = deployment.advertiseTagsValues.filter((t) => t !== tag);
									}}
								>
									&times;
								</button>
							</span>
						{/each}
					</div>
				{/if}
				{#if App.aclTags.value.filter((t) => !deployment.advertiseTagsValues.includes(t)).length > 0}
					<select
						class="select text-sm rounded-md"
						onchange={(e) => {
							const val = e.currentTarget.value;
							if (val && !deployment.advertiseTagsValues.includes(val)) {
								deployment.advertiseTagsValues = [...deployment.advertiseTagsValues, val];
							}
							e.currentTarget.value = '';
						}}
					>
						<option value="">Select a tag</option>
						{#each App.aclTags.value.filter((t) => !deployment.advertiseTagsValues.includes(t)) as tag}
							<option value={tag}>{tag}</option>
						{/each}
					</select>
				{/if}
			</div>
		</DeployCheck>
		<DeployCheck
			bind:checked={deployment.advertiseRoutes}
			name="Advertise Routes"
			help="List of subnets which are reachable via this node"
		>
			<InputChip
				name="advertiseRoutesValues"
				bind:value={deployment.advertiseRoutesValues}
				validation={isValidCIDR}
				on:invalid={() => {
					toastError('Invalid CIDR Format', ToastStore);
				}}
			/>
		</DeployCheck>

		<p class="text-xl col-span-12 py-4">Accept:</p>
		<DeployCheck
			bind:checked={deployment.acceptDns}
			name="Accept DNS"
			help="Accept the HeadScale-provided DNS settings"
		/>
		<DeployCheck
			bind:checked={deployment.acceptRoutes}
			name="Accept Routes"
			help="Accept other nodes' advertised subnets"
		/>
		<DeployCheck
			bind:checked={deployment.acceptExitNode}
			name="Exit Node"
			help="Use this node as a gateway (target node must advertise exit node)"
		>
			<label class="label">
				<select class="select" bind:value={deployment.acceptExitNodeValue}>
					{#each App.nodes.value as node}
						<option value={node.ipAddresses.filter((s) => /^\d+\.\d+\.\d+\.\d+$/.test(s))[0]}
							>{node.givenName} ({node.name})</option
						>
					{/each}
				</select>
			</label>
		</DeployCheck>
	</div>
	<button
		class="btn rounded-md variant-filled-secondary mt-4"
		onclick={() => {
			App.saveDeploymentDefaults(deployment);
			toastSuccess('Saved Deployment Defaults', ToastStore);
		}}
	>
		Save Defaults
	</button>
</Page>
