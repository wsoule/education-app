<script lang="ts">
  import { invoke } from '@tauri-apps/api/core';
  import {
    isPermissionGranted,
    requestPermission,
    sendNotification
  } from '@tauri-apps/plugin-notification';
  import { marked } from 'marked';
  import { afterUpdate, beforeUpdate } from 'svelte';

  const model = 'llama3.1';
  type Message = {
    role: 'assistant' | 'user' | 'system';
    content: string;
  };
  let messages: Message[] = [
    {
      role: 'system',
      content:
        'You are Professor Dumbledore. Answer as Dumbledore, the assistant, only and give guidance about Hogwarts and wizardry.'
    }
  ];

  let loading = false;
  async function chat(messages: Message[]): Promise<Message> {
    const body = {
      model: model,
      messages: messages
    };

    new Promise((resolve) => setTimeout(() => (loading = true), 200));
    // loading = true;

    const response = await fetch('http://localhost:11434/api/chat', {
      method: 'POST',
      body: JSON.stringify(body)
    });

    const reader = response.body?.getReader();
    if (!reader) {
      throw new Error('Failed to read response body');
    }
    let content = '';
    while (true) {
      const { done, value } = await reader.read();
      if (done) {
        break;
      }
      const rawjson = new TextDecoder().decode(value);
      const json = JSON.parse(rawjson);

      if (json.done === false) {
        content += json.message.content;
      }
    }
    loading = false;
    return { role: 'assistant', content: content };
  }

  async function askQuestion(input: string): Promise<void> {
    return new Promise<void>(async (resolve) => {
      if (input.trim() === '') {
        console.log('Thankyou. Goodbye.\n');
        console.log(
          '=======\nHere is the message history that was used in this conversation.\n=======\n'
        );
        messages.forEach((message) => {
          console.log(message);
        });
        resolve();
      } else {
        console.log('Done =', input);
        messages = [...messages, { role: 'user', content: input }];
        messages = [...messages, await chat(messages)];
      }
      console.log('done with gen');
    });
  }

  async function checkPermission() {
    if (!(await isPermissionGranted())) {
      return (await requestPermission()) === 'granted';
    }
    return true;
  }

  export async function enqueueNotification(title: string, body: string = 'User') {
    if (!(await checkPermission())) {
      return;
    }
    sendNotification({ title, body });
  }
  let name = '';
  let greetMsg = '';
  let div: HTMLDivElement | null = null;
  // let div = $state();
  let autoscroll = false;

  async function greet() {
    // Learn more about Tauri commands at https://tauri.app/v1/guides/features/command
    console.log('message = ', messages);
    await askQuestion(name);
    greetMsg = await invoke('greet', { name });
  }
  beforeUpdate(() => {
    if (div) {
      const scrollableDistance = div.scrollHeight - div.offsetHeight;
      autoscroll = div.scrollTop > scrollableDistance - 20;
    }
  });

  afterUpdate(() => {
    if (autoscroll && div) {
      // div.scrollTo(0, div.scrollHeight);
      window.scrollTo(0, div.scrollHeight);
    }
  });
</script>

<div class="container">
  <h1>Welcome to Tauri!</h1>

  <div class="row">
    <a href="https://vitejs.dev" target="_blank">
      <img src="/vite.svg" class="logo vite" alt="Vite Logo" />
    </a>
    <a href="https://tauri.app" target="_blank">
      <img src="/tauri.svg" class="logo tauri" alt="Tauri Logo" />
    </a>
    <a href="https://kit.svelte.dev" target="_blank">
      <img src="/svelte.svg" class="logo svelte-kit" alt="SvelteKit Logo" />
    </a>
  </div>

  <p>Click on the Tauri, Vite, and SvelteKit logos to learn more.</p>
  <p>{greetMsg}</p>
  <button on:click={() => enqueueNotification('hello', name)}>Notif</button>
  <div class="chat" bind:this={div}>
    {#each messages as message}
      <p>{message.role}: {@html marked(message.content)}</p>
    {/each}
    {#if loading}
      <p>Loading...</p>
    {/if}
  </div>

  <form class="row" on:submit|preventDefault={greet}>
    <input id="greet-input" placeholder="Enter a name..." bind:value={name} />
    <button type="submit">Greet</button>
  </form>
</div>

<style>
  .logo.vite:hover {
    filter: drop-shadow(0 0 2em #747bff);
  }

  .logo.svelte-kit:hover {
    filter: drop-shadow(0 0 2em #ff3e00);
  }

  :root {
    font-family: Inter, Avenir, Helvetica, Arial, sans-serif;
    font-size: 16px;
    line-height: 24px;
    font-weight: 400;

    color: #0f0f0f;
    background-color: #f6f6f6;

    font-synthesis: none;
    text-rendering: optimizeLegibility;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    -webkit-text-size-adjust: 100%;
  }

  .container {
    margin: 0;
    padding-top: 10vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    text-align: center;
  }

  .logo {
    height: 6em;
    padding: 1.5em;
    will-change: filter;
    transition: 0.75s;
  }

  .logo.tauri:hover {
    filter: drop-shadow(0 0 2em #24c8db);
  }

  .row {
    display: flex;
    justify-content: center;
  }

  a {
    font-weight: 500;
    color: #646cff;
    text-decoration: inherit;
  }

  a:hover {
    color: #535bf2;
  }

  h1 {
    text-align: center;
  }

  input,
  button {
    border-radius: 8px;
    border: 1px solid transparent;
    padding: 0.6em 1.2em;
    font-size: 1em;
    font-weight: 500;
    font-family: inherit;
    color: #0f0f0f;
    background-color: #ffffff;
    transition: border-color 0.25s;
    box-shadow: 0 2px 2px rgba(0, 0, 0, 0.2);
  }

  button {
    cursor: pointer;
  }

  button:hover {
    border-color: #396cd8;
  }
  button:active {
    border-color: #396cd8;
    background-color: #e8e8e8;
  }

  input,
  button {
    outline: none;
  }

  #greet-input {
    margin-right: 5px;
  }

  @media (prefers-color-scheme: dark) {
    :root {
      color: #f6f6f6;
      background-color: #2f2f2f;
    }

    a:hover {
      color: #24c8db;
    }

    input,
    button {
      color: #ffffff;
      background-color: #0f0f0f98;
    }
    button:active {
      background-color: #0f0f0f69;
    }
  }
</style>
