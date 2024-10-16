<script lang="ts">
  import { invoke } from '@tauri-apps/api/core';
  import {
    isPermissionGranted,
    requestPermission,
    sendNotification
  } from '@tauri-apps/plugin-notification';
  import { marked } from 'marked';
  import { tick } from 'svelte';

  const model = 'llama3.1';
  type Message = {
    role: 'assistant' | 'user' | 'system';
    content: string;
  };

  let messages: Message[] = $state([
    {
      role: 'system',
      content:
        'You are Professor Dumbledore. Answer as Dumbledore, the assistant, only and give guidance about Hogwarts and wizardry. And answer in only 3 words.'
    }
  ]);

  let loading = $state(false);
  let name = $state('');
  let greetMsg = $state('');
  let div: HTMLDivElement | undefined = $state();
  let currentResponse = $state('');

  const chat = async (messages: Message[]): Promise<void> => {
    const body = {
      model: model,
      messages: messages,
      stream: true // Enable streaming
    };

    loading = true;
    currentResponse = '';

    const response = await fetch('http://localhost:11434/api/chat', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(body)
    });

    const reader = response.body?.getReader();
    if (!reader) {
      throw new Error('Failed to read response body');
    }

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = new TextDecoder().decode(value);
      const lines = chunk.split('\n');

      for (const line of lines) {
        if (line.trim() !== '') {
          try {
            const parsed = JSON.parse(line);
            if (!parsed.done) {
              currentResponse += parsed.message.content;
            } else {
              // Final message received
              messages.push({ role: 'assistant', content: currentResponse });
              loading = false;
              name = '';
              return;
            }
          } catch (e) {
            console.error('Error parsing JSON:', e);
          }
        }
      }
    }
  };

  const askQuestion = async (input: string): Promise<void> => {
    if (input.trim() === '') {
      console.log('Thank you. Goodbye.\n');
      console.log(
        '=======\nHere is the message history that was used in this conversation.\n=======\n'
      );
      messages.forEach((message) => {
        console.log(message);
      });
      return;
    }

    console.log('User input =', input);
    messages = [...messages, { role: 'user', content: input }];
    await chat(messages);
    console.log('Response completed');
  };

  const checkPermission = async (): Promise<boolean> => {
    if (!(await isPermissionGranted())) {
      return (await requestPermission()) === 'granted';
    }
    return true;
  };

  const enqueueNotification = async (title: string, body: string = 'User'): Promise<void> => {
    if (!(await checkPermission())) {
      return;
    }
    sendNotification({ title, body });
  };

  const greet = async (): Promise<void> => {
    await askQuestion(name);
    greetMsg = await invoke('greet', { name });
  };

  $effect.pre(() => {
    if (!div) return;

    messages.length;
    currentResponse;

    if (div.offsetHeight + div.scrollTop > div.scrollHeight - 20) {
      tick().then(() => {
        if (div) {
          window.scrollTo(0, div.scrollHeight);
        }
      });
    }
  });

  const preventDefault = (fn: Function) => {
    return function (event: Event) {
      event.preventDefault();
      fn.call(this, event);
    };
  };
</script>

<div class="container">
  <header>
    <h1>Hogwarts Chat with Professor Dumbledore</h1>
    <div class="logo-container">
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
  </header>

  <main>
    <div class="chat-container" bind:this={div}>
      {#each messages as message}
        <div class="message {message.role}">
          <strong>{message.role}:</strong>
          {@html marked(message.content)}
        </div>
      {/each}
      {#if loading}
        <div class="message assistant">
          <strong>Assistant:</strong>
          {@html marked(currentResponse)}
        </div>
      {/if}
    </div>

    <form class="input-form" onsubmit={preventDefault(greet)}>
      <input id="greet-input" placeholder="Ask Professor Dumbledore..." bind:value={name} />
      <button type="submit">Send</button>
    </form>
  </main>

  <footer>
    <p>{greetMsg}</p>
    <button
      class="notification-btn"
      onclick={() => enqueueNotification('Hello from Hogwarts', name)}
    >
      Send Owl Post
    </button>
  </footer>
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
