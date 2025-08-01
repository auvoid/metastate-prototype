<script lang="ts">
	import { page } from '$app/state';
	import { BottomNav, Comment, Header, MessageInput, SideBar } from '$lib/fragments';
	import { showComments } from '$lib/store/store.svelte';
	import { openCreatePostModal, isCreatePostModalOpen } from '$lib/stores/posts';
	import { comments, fetchComments, createComment, activePostId } from '$lib/stores/comments';
	import CreatePostModal from '$lib/fragments/CreatePostModal/CreatePostModal.svelte';
	import { onMount } from 'svelte';
	import { apiClient, getAuthId, getAuthToken } from '$lib/utils';
	import type { userProfile } from '$lib/types';
	import { heading } from '../store';
	import { goto } from '$app/navigation';
	import type { AxiosError } from 'axios';

	let { children } = $props();
	let ownerId: string | null = $state(null);
	let route = $derived(page.url.pathname);

	let commentValue: string = $state('');
	let commentInput: HTMLInputElement | undefined = $state();
	let activeReplyToId: string | null = $state(null);
	let idFromParams = $state();
	let isCommentsLoading = $state(false);
	let commentsError = $state<string | null>(null);
	let profile = $state<userProfile | null>(null);

	const handleSend = async () => {
		console.log($activePostId, commentValue);
		if (!$activePostId || !commentValue.trim()) return;

		try {
			await createComment($activePostId, commentValue);
			commentValue = '';
			activeReplyToId = null;
		} catch (err) {
			console.error('Failed to create comment:', err);
		}
	};

	$effect(() => {
		idFromParams = page.params.id;

		console.log(route);

		if (route.includes('home')) {
			heading.set('Feed');
		} else if (route.includes('discover')) {
			heading.set('Search');
		} else if (route.includes('/post/audience')) {
			heading.set('Audience');
		} else if (route.includes('post')) {
			heading.set('Upload photo');
		} else if (route === '/messages') {
			heading.set('Messages');
		} else if (route.includes('settings')) {
			heading.set('Settings');
		} else if (route.includes('profile')) {
			heading.set('Profile');
		}
	});

	// Watch for changes in showComments to fetch comments when opened
	$effect(() => {
		ownerId = getAuthId();
		if (showComments.value && activePostId) {
			isCommentsLoading = true;
			commentsError = null;
			fetchComments($activePostId)
				.catch((err) => {
					commentsError = err.message;
				})
				.finally(() => {
					isCommentsLoading = false;
				});
		}
	});

	async function fetchProfile() {
		try {
			if (!getAuthToken()) {
				goto('/auth');
			}
			const response = await apiClient.get(`/api/users/${ownerId}`).catch((e: AxiosError) => {
				if (e.response?.status === 401) {
					goto('/auth');
				}
			});
			if (!response) return;
			profile = response.data;
		} catch (err) {
			console.log(err instanceof Error ? err.message : 'Failed to load profile');
		}
	}

	onMount(fetchProfile);
</script>

<main
	class={`block h-[100dvh] ${route !== '/home' && route !== '/messages' && route !== '/profile' && !route.includes('settings') && !route.includes('/profile') ? 'grid-cols-[20vw_auto]' : 'grid-cols-[20vw_auto_30vw]'} md:grid`}
>
	<SideBar
		profileSrc={profile?.avatarUrl ?? '/images/user.png'}
		handlePost={async () => {
			openCreatePostModal();
		}}
	/>
	<section class="hide-scrollbar h-[100dvh] overflow-y-auto px-4 pb-8 md:px-8 md:pt-8">
		<div class="flex flex-col">
			<Header
				variant={route === `/messages/${idFromParams}` || route.includes('/post')
					? 'secondary'
					: route.includes('profile')
						? 'tertiary'
						: 'primary'}
				heading={$heading}
				isCallBackNeeded={route.includes('profile')}
				callback={() => alert('Ads')}
				options={[
					{ name: 'Report', handler: () => alert('report') },
					{ name: 'Clear chat', handler: () => alert('clear') }
				]}
			/>
			{@render children()}
		</div>
	</section>
	{#if route === '/home' || route === '/messages'}
		<aside
			class="hide-scrollbar relative hidden h-[100dvh] overflow-y-scroll border border-e-0 border-t-0 border-b-0 border-s-gray-200 px-8 pt-14 md:block"
		>
			{#if route === '/home'}
				{#if showComments.value}
					<ul class="pb-4">
						<h3 class="text-black-600 mb-6 text-center">{$comments.length} Comments</h3>
						{#if isCommentsLoading}
							<li class="text-center text-gray-500">Loading comments...</li>
						{:else if commentsError}
							<li class="text-center text-red-500">{commentsError}</li>
						{:else}
							{#each $comments as comment}
								<li class="mb-4">
									<Comment
										comment={{
											userImgSrc: comment.author.avatarUrl,
											name: comment.author.name || comment.author.handle,
											commentId: comment.id,
											comment: comment.text,
											isUpVoted: false,
											isDownVoted: false,
											upVotes: 0,
											time: new Date(comment.createdAt).toLocaleDateString(),
											replies: []
										}}
										handleReply={() => {
											activeReplyToId = comment.id;
											commentInput?.focus();
										}}
									/>
								</li>
							{/each}
						{/if}
						<MessageInput
							class="sticky start-0 bottom-4 mt-4 w-full px-2"
							variant="comment"
							src={profile?.avatarUrl ?? '/images/user.png'}
							bind:value={commentValue}
							{handleSend}
							bind:input={commentInput}
						/>
					</ul>
				{/if}
			{/if}
		</aside>
	{/if}

	{#if route !== `/messages/${idFromParams}`}
		<BottomNav class="btm-nav" profileSrc={profile?.avatarUrl ?? ''} />
	{/if}
</main>

{#if $isCreatePostModalOpen}
	<CreatePostModal />
{/if}
