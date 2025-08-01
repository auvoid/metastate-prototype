<script lang="ts">
	import { goto } from '$app/navigation';
	import { onMount } from 'svelte';
	import { Message } from '$lib/fragments';
	import Group from '$lib/fragments/Group/Group.svelte';
	import { Button, Avatar, Input } from '$lib/ui';
	import { clickOutside } from '$lib/utils';
	import { heading } from '../../store';
	import { apiClient } from '$lib/utils/axios';

	import {
		searchUsers,
		searchResults,
		isSearching,
		searchError
	} from '$lib/stores/users';
	import type { GroupInfo } from '$lib/types';

	let messages = $state([]);
	let groups: GroupInfo[] = $state([]);
	let allMembers = $state([]);
	let selectedMembers = $state<string[]>([]);
	let currentUserId = '';
	let openNewChatModal = $state(false);
	let searchValue = $state('');
	let debounceTimer: NodeJS.Timeout;

	async function loadMessages() {
		const { data } = await apiClient.get('/api/chats');
		const { data: userData } = await apiClient.get('/api/users');
		currentUserId = userData.id;

		messages = data.chats.map((c) => {
			const members = c.participants.filter((u) => u.id !== userData.id);
			const memberNames = members.map((m) => m.name ?? m.handle ?? m.ename);
			const avatar = members.length > 1 ? '/images/group.png' : members[0].avatarUrl;
			return {
				id: c.id,
				avatar,
				username: memberNames.join(', '),
				unread: c.latestMessage ? c.latestMessage.isRead : false,
				text: c.latestMessage?.text ?? 'No message yet'
			};
		});
	}

	onMount(async () => {
		await loadMessages();

		const memberRes = await apiClient.get('/api/members');
		allMembers = memberRes.data;
	});

	function toggleMemberSelection(id: string) {
		if (selectedMembers.includes(id)) {
			selectedMembers = selectedMembers.filter((m) => m !== id);
		} else {
			selectedMembers = [...selectedMembers, id];
		}
	}

	function handleSearch(value: string) {
		searchValue = value;
		clearTimeout(debounceTimer);
		debounceTimer = setTimeout(() => {
			searchUsers(value);
		}, 300);
	}

	async function createChat() {
		if (selectedMembers.length === 0) return;

		try {
			if (selectedMembers.length === 1) {
				await apiClient.post(`/api/chats/`, {
					name:
						allMembers.find((m) => m.id === selectedMembers[0])?.name ??
						'New Chat',
					participantIds: [selectedMembers[0]]
				});
				await loadMessages(); // 🛠️ Refresh to include the new direct message
			} else {
				const groupMembers = allMembers.filter((m) =>
					selectedMembers.includes(m.id)
				);
				const groupName = groupMembers.map((m) => m.name ?? m.handle ?? m.ename).join(', ');
				groups = [
					...groups,
					{
						id: Math.random().toString(36).slice(2),
						name: groupName,
						avatar: '/images/group.png'
					}
				];
			}
		} catch (err) {
			console.error('Failed to create chat:', err);
		} finally {
			openNewChatModal = false;
			selectedMembers = [];
			searchValue = '';
		}
	}
</script>


<section class="px-4 py-4">
	<div class="flex justify-end mb-4">
		<Button variant="secondary" size="sm" callback={() => {(openNewChatModal = true)}}>
			New Chat
		</Button>
	</div>

	{#if messages.length > 0}
		<h3 class="text-md font-semibold text-gray-700 mb-2">Messages</h3>
		{#each messages as message}
			<Message
				class="mb-2"
				avatar={message.avatar}
				username={message.username}
				text={message.text}
				unread={!message.unread}
				callback={() => {
					heading.set(message.username);
					goto(`/messages/${message.id}`);
				}}
			/>
		{/each}
	{/if}

	{#if groups.length > 0}
		<h3 class="text-md font-semibold text-gray-700 mb-2 mt-6">Groups</h3>
		{#each groups as group}
			<Group
				name={group.name || "New Group"}
				avatar={group.avatar}
				unread={true}
				callback={() => goto(`/group/${group.id}`)}
			/>
		{/each}
	{:else if messages.length === 0}
		<div class="w-full px-5 py-5 text-center text-sm text-gray-500">
			You don't have any messages yet. Start a Direct Message by searching a name.
		</div>
	{/if}

	<dialog
		open={openNewChatModal}
		use:clickOutside={() => (openNewChatModal = false)}
		onclose={() => (openNewChatModal = false)}
		class="w-[90vw] md:max-w-[40vw] z-50 absolute start-[50%] top-[50%] translate-x-[-50%] translate-y-[-50%] p-4 border border-gray-400 rounded-3xl bg-white shadow-xl"
	>
		<div class="bg-white rounded-xl w-full p-6 space-y-6 relative">
			<h2 class="text-xl font-semibold">Start a New Chat</h2>

			<Input
				type="text"
				bind:value={searchValue}
				placeholder="Search users..."
				oninput={(e: Event) => handleSearch((e.target as HTMLInputElement).value)}
			/>

			{#if $isSearching}
				<div class="text-gray-500 mt-2">Searching...</div>
			{:else if $searchError}
				<div class="text-red-500 mt-2">{$searchError}</div>
			{/if}

			<div class="max-h-[250px] overflow-y-auto space-y-3">
				{#each $searchResults.filter(m => m.id !== currentUserId) as member}
					<label class="flex items-center space-x-3 cursor-pointer">
						<input
							type="checkbox"
							checked={selectedMembers.includes(member.id)}
							onchange={() => toggleMemberSelection(member.id)}
							class="accent-brand focus:ring"
						/>
						<Avatar src={member.avatarUrl} size="sm" />
						<span class="text-sm">{member.name ?? member.handle ?? member.ename}</span>
					</label>
				{/each}
			</div>

			<div class="flex justify-end gap-2 pt-4">
				<Button size="sm" variant="secondary" callback={() => {(openNewChatModal = false)}}>Cancel</Button>
				<Button size="sm" variant="primary" callback={createChat}>Start Chat</Button>
			</div>
		</div>
	</dialog>
</section>
