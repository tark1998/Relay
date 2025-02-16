<script lang="ts">
	import LoggedIn from "./LoggedIn.svelte";
	import Relays from "./Relays.svelte";
	import ManageRelay from "./ManageRelay.svelte";
	import { type Relay } from "../Relay";
	import ModalSettingsNav from "./ModalSettingsNav.svelte";
	import type Live from "src/main";
	import type { SharedFolder } from "src/SharedFolder";
	import ManageSharedFolder from "./ManageSharedFolder.svelte";

	interface RelayEventDetail {
		relay: Relay;
	}

	interface View {
		currentRelay?: Relay;
		sharedFolder?: SharedFolder;
		component: typeof Relays | typeof ManageRelay | typeof ManageSharedFolder;
	}

	interface SharedFolderEventDetail {
		folder: SharedFolder;
		relay?: Relay;
	}
	interface ManageRelayEvent extends CustomEvent<RelayEventDetail> {}
	interface ManageSharedFolderEvent
		extends CustomEvent<SharedFolderEventDetail> {}
	interface GoBackEvent extends CustomEvent {
		clear?: boolean;
	}
	interface CreateRelayEvent extends CustomEvent {}
	interface CloseEvent extends CustomEvent {}
	interface JoinRelayEvent extends CustomEvent<RelayEventDetail> {}

	export let plugin: Live;
	const app = plugin.app;
	const relayManager = plugin.relayManager;
	const relayRoles = relayManager.relayRoles;
	const relays = relayManager.relays;
	const sharedFolders = plugin.sharedFolders;
	let sharedFolder: SharedFolder | undefined;

	let currentComponent:
		| typeof Relays
		| typeof ManageRelay
		| typeof ManageSharedFolder = Relays;

	let currentRelay: Relay | undefined;
	const history: View[] = [{ component: Relays }];

	export let close: () => void;

	function handleManageRelayEvent(event: ManageRelayEvent) {
		history.push({
			currentRelay,
			sharedFolder,
			component: currentComponent,
		});
		currentRelay = event.detail.relay;
		sharedFolder = undefined;
		currentComponent = ManageRelay;
	}
	function handleManageSharedFolderEvent(event: ManageSharedFolderEvent) {
		history.push({
			currentRelay,
			sharedFolder,
			component: currentComponent,
		});
		sharedFolder = event.detail.folder;
		currentRelay = event.detail.relay;
	}

	async function handleCreateRelayEvent(event: CreateRelayEvent) {
		history.push({
			currentRelay,
			sharedFolder,
			component: currentComponent,
		});
		currentRelay = await plugin.relayManager.createRelay("");
		currentComponent = ManageRelay;
	}

	function handleGoBack(event: GoBackEvent) {
		if (event.detail.clear) {
			history.length = 0;
			currentRelay = undefined;
			sharedFolder = undefined;
			currentComponent = Relays;
			return;
		}

		let view = history.pop();
		if (view) {
			while (view) {
				if (!view.currentRelay && !view.sharedFolder) {
					currentRelay = view.currentRelay;
					sharedFolder = view.sharedFolder;
					currentComponent = view.component;
				} else if (view.sharedFolder && sharedFolders.has(view.sharedFolder)) {
					currentRelay = view.currentRelay;
					sharedFolder = view.sharedFolder;
					currentComponent = view.component;
					break;
				} else if (
					view.currentRelay &&
					relayManager.relays.get(view.currentRelay.id)
				) {
					currentRelay = view.currentRelay;
					sharedFolder = view.sharedFolder;
					currentComponent = view.component;
					break;
				}
				view = history.pop();
			}
		} else {
			currentRelay = undefined;
			sharedFolder = undefined;
			currentComponent = Relays;
		}
	}

	function handleClose(event: CloseEvent) {
		history.length = 0;
		close();
	}

	function handleJoinRelay(event: JoinRelayEvent) {
		currentRelay = event.detail.relay;
		currentComponent = ManageRelay;
	}

	$: {
		if (currentRelay && !$relays.has(currentRelay.id)) {
			currentRelay = undefined;
			currentComponent = Relays;
		}
	}

	$: {
		if (sharedFolder && !sharedFolders.has(sharedFolder)) {
			sharedFolder = undefined;
			currentComponent = Relays;
		}
	}

	$: if (currentComponent || currentRelay || sharedFolder) {
		setTimeout(() => {
			const content = document.querySelector(".vertical-tab-content");
			if (content) {
				content.scrollTop = 0;
			}
		}, 0);
	}
</script>

{#if currentRelay || sharedFolder}
	<ModalSettingsNav on:goBack={handleGoBack}></ModalSettingsNav>
{:else if plugin.networkStatus.status}
	<div class="modal-setting-nav-bar system3-announcement-banner">
		<span class="system3-announcement">
			{plugin.networkStatus.status}
		</span>
	</div>
{/if}
<div class="vertical-tab-content">
	{#if sharedFolder}
		<ManageSharedFolder
			{plugin}
			{relayRoles}
			{sharedFolder}
			{sharedFolders}
			on:goBack={handleGoBack}
			on:close={handleClose}
			on:manageRelay={handleManageRelayEvent}
		></ManageSharedFolder>
	{:else if currentRelay}
		<ManageRelay
			{plugin}
			{relayRoles}
			relay={currentRelay}
			{sharedFolders}
			on:goBack={handleGoBack}
			on:close={handleClose}
			on:manageSharedFolder={handleManageSharedFolderEvent}
		></ManageRelay>
	{:else}
		<LoggedIn {plugin}>
			<Relays
				{relays}
				{plugin}
				on:manageRelay={handleManageRelayEvent}
				on:manageSharedFolder={handleManageSharedFolderEvent}
				on:createRelay={handleCreateRelayEvent}
				on:joinRelay={handleJoinRelay}
			></Relays>
		</LoggedIn>
	{/if}
</div>

{#if plugin.manifest.version !== plugin.version}
	<span class="relay-version">
		{plugin.version}
	</span>
{/if}

<style>
	.relay-version {
		user-select: auto;
		background: var(--color-base-10);
		color: var(--text-faint);
		position: absolute;
		bottom: 0;
		right: 0;
		font-size: xx-small;
		padding-right: 1em;
		padding-top: 0.3em;
		padding-left: 1em;
		border-top-left-radius: 1em;
	}
	.vertical-tab-content {
		max-height: var(--modal-max-height);
		position: relative;
	}
	.system3-announcement-banner {
		padding-left: 48px !important;
		background-color: var(--color-red) !important;
	}
	.system3-announcement {
		color: var(--text-on-accent);
	}
</style>
