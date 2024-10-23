<script lang="ts">
	let messages: { text: string; sender: string }[] = $state([]);
	let userInput: string = $state('');
	let showChat = $state(true);

	const toggleChat = () => {
		showChat = !showChat;
	};

	const sendMessage = async () => {
		if (userInput.trim() === '') return;

		messages.push({ text: userInput, sender: 'user' });

		try {
			const response = await fetch('https://api.openai.com/v1/chat/completions', {
				method: 'POST',
				headers: {
					Authorization: `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`,
					'Content-Type': 'application/json'
				},
				body: JSON.stringify({
					model: 'gpt-3.5-turbo',
					messages: [
						...messages.map((msg: { sender: string; text: string }) => ({
							role: msg.sender === 'user' ? 'user' : 'assistant',
							content: msg.text
						}))
					]
				})
			});

			if (!response.ok) {
				throw new Error('Network response was not ok');
			}

			const data = await response.json();
			const botMessage = data.choices[0].message.content;
			messages.push({ text: botMessage, sender: 'assistant' });
		} catch (error) {
			console.error('Error:', error);
			messages.push({ text: 'Error: Unable to fetch response', sender: 'assistant' });
		}

		userInput = '';
	};
</script>

{#if showChat}
	<div class="relative h-96 w-80 border border-indigo-500 rounded-md p-2">
		<div class="absolute bottom-0 left-0 h-auto w-full p-2">
			{#if messages.length == 0}
				<h1 class="my-2">Welcome to Megha Bot!</h1>
			{:else}
				<div class="overflow-auto max-h-64 my-2">
					{#each messages as message}
						<div class={message.sender}>
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
				class="w-full rounded-md"
			/>
			<div class="flex flex-row my-2">
				<button
					class="w-24 py-2 px-3 mr-2 bg-indigo-500 text-white text-sm font-semibold rounded-md shadow focus:outline-none"
					onclick={sendMessage}>Send</button
				>
				<button
					class="py-2 px-3 bg-gray-400 text-white text-sm font-semibold rounded-md shadow focus:outline-none"
					onclick={toggleChat}>Close</button
				>
			</div>
		</div>
	</div>
{:else}
	<button
		class="py-2 px-3 bg-indigo-500 text-white text-sm font-semibold rounded-md shadow focus:outline-none"
		onclick={toggleChat}>Chat</button
	>
{/if}
