<script lang="ts">
	import { tick } from 'svelte';

	let messages: { text: string; sender: string }[] = $state([]);
	let userInput: string = $state('');
	let showChat = $state(false);
	let currentThread = $state('');

	let element: any = $state('');

	const scrollToBottom = async (node: any) => {
		node.scroll({ top: node.scrollHeight, behavior: 'smooth' });
	};

	// function to toggle chat interface
	const toggleChat = () => {
		showChat = !showChat;
		createThread();
	};

	// reset chat
	const resetChat = () => {
		currentThread = '';
		messages = [];
		userInput = '';
		createThread();
	};

	$inspect(currentThread);

	// create thread to start new session
	const createThread = async () => {
		if (currentThread.length > 0) return;
		try {
			const response = await fetch('https://api.openai.com/v1/threads', {
				method: 'POST',
				headers: {
					Authorization: `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`,
					'Content-Type': 'application/json',
					'OpenAI-Beta': 'assistants=v2'
				},
				body: JSON.stringify({})
			});

			if (!response.ok) {
				throw new Error('Network response was not ok');
			}

			const data = await response.json();
			currentThread = data.id;
		} catch (error) {
			console.error('Error:', error);
		}
	};

	// function to check run status
	const checkRunStatus = async (runId: string) => {
		try {
			const response = await fetch(
				`https://api.openai.com/v1/threads/${currentThread}/runs/${runId}`,
				{
					method: 'GET',
					headers: {
						Authorization: `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`,
						'OpenAI-Beta': 'assistants=v2'
					}
				}
			);

			if (!response.ok) {
				throw new Error('Network response was not ok');
			}

			const data = await response.json();
			if (data.status != 'completed') {
				checkRunStatus(runId);
			} else {
				getMessages();
			}
		} catch (error) {
			console.error('Error:', error);
			messages.push({
				text: 'Error: Unable to fetch response, please retry!',
				sender: 'assistant'
			});
		}
	};

	// add message to thread
	const addMessageToThread = async () => {
		try {
			const response = await fetch(`https://api.openai.com/v1/threads/${currentThread}/messages`, {
				method: 'POST',
				headers: {
					Authorization: `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`,
					'Content-Type': 'application/json',
					'OpenAI-Beta': 'assistants=v2'
				},
				body: JSON.stringify({
					role: 'user',
					content: userInput
				})
			});

			if (!response.ok) {
				throw new Error('Network response was not ok');
			}

			const data = await response.json();
		} catch (error) {
			console.error('Error:', error);
			messages.push({
				text: 'Error: Unable to fetch response, please retry!',
				sender: 'assistant'
			});
		}
	};

	// function to run the assistant
	const runAssistant = async () => {
		try {
			const response = await fetch(`https://api.openai.com/v1/threads/${currentThread}/runs`, {
				method: 'POST',
				headers: {
					Authorization: `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`,
					'Content-Type': 'application/json',
					'OpenAI-Beta': 'assistants=v2'
				},
				body: JSON.stringify({
					assistant_id: import.meta.env.VITE_OPENAI_ASSISTANT_ID
				})
			});

			if (!response.ok) {
				throw new Error('Network response was not ok');
			}

			const data = await response.json();
			// check run status till its completed
			checkRunStatus(data.id);
		} catch (error) {
			console.error('Error:', error);
			messages.push({
				text: 'Error: Unable to fetch response, please retry!',
				sender: 'assistant'
			});
		}
	};

	// functioin to get messages
	const getMessages = async () => {
		try {
			const response = await fetch(`https://api.openai.com/v1/threads/${currentThread}/messages`, {
				method: 'GET',
				headers: {
					Authorization: `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`,
					'Content-Type': 'application/json',
					'OpenAI-Beta': 'assistants=v2'
				}
			});

			if (!response.ok) {
				throw new Error('Network response was not ok');
			}

			const data = await response.json();
			const botMessage = data.data[0].content[0].text.value;
			messages.push({ text: botMessage, sender: 'assistant' });
			await tick();
			scrollToBottom(element);
		} catch (error) {
			console.error('Error:', error);
			messages.push({
				text: 'Error: Unable to fetch response, please retry!',
				sender: 'assistant'
			});
		}
	};

	// function to user input
	const sendMessage = async () => {
		if (userInput.trim() === '') return;

		messages.push({ text: userInput, sender: 'user' });

		await tick();
		scrollToBottom(element);

		createThread();

		addMessageToThread();

		runAssistant();

		userInput = '';
	};
</script>

{#if showChat}
	<div class="relative h-96 w-80 border border-indigo-500 rounded-md p-2">
		<div class="absolute bottom-0 left-0 h-auto w-full p-2">
			{#if messages.length == 0}
				<h1 class="my-2">Welcome to Megha Bot!</h1>
			{:else}
				<div bind:this={element} class="overflow-auto max-h-64 my-2">
					{#each messages as message}
						<div
							class="text-xs rounded-md p-2 m-2 flex w-fit {message.sender == 'user'
								? 'border border-indigo-700 items-end rtl:items-start'
								: 'border border-gray-700 items-start'}"
						>
							{message.text}
						</div>
					{/each}
				</div>
			{/if}
			<input
				type="text"
				bind:value={userInput}
				onkeydown={(e) => e.key === 'Enter' && sendMessage()}
				placeholder="Type a message..."
				class="w-full rounded-md text-xs"
			/>
			<div class="flex flex-row my-2 items-end">
				<button
					class="w-24 py-2 px-3 mr-2 bg-indigo-500 text-white text-sm font-semibold rounded-md shadow focus:outline-none"
					onclick={sendMessage}>Send</button
				>
				<button
					class="py-2 px-3 mr-2 bg-gray-400 text-white text-sm font-semibold rounded-md shadow focus:outline-none"
					onclick={toggleChat}>Close</button
				>
				<button class="text-xs underline" onclick={resetChat}>Reset chat</button>
			</div>
		</div>
	</div>
{:else}
	<button
		class="py-2 px-3 bg-indigo-500 text-white text-sm font-semibold rounded-md shadow focus:outline-none"
		onclick={toggleChat}>Chat</button
	>
{/if}
