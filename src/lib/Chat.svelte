<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import * as tmi from "tmi.js";
  import { fade } from "svelte/transition";
  import { flip } from "svelte/animate";

  type Message = {
    userstate: tmi.ChatUserstate;
    message: string;
    channel: string;
  };

  let messages: Message[] = [];
  let client: tmi.Client;
  let channels: string[] = [];
  let value: string;
  $: joinedChannels = channels;

  function joinChannels(client: tmi.Client) {
    client.getChannels().forEach(async (channel) => {
      await client.part(channel).catch((err) => console.log(err));
    });
    channels.forEach(async (channel) => {
      await client
        .join(channel)
        .catch((err) => console.log("Could not join channel:", channel));
    });
  }

  async function handleSubmit() {
    channels = value
      .toLowerCase()
      .trim()
      .split(/\s*[,.\s]+\s*/);
    value = "";

    channels = channels.filter((channel) => channel !== "");
    console.log(channels);
    joinChannels(client);
  }

  onMount(async () => {
    client = new tmi.Client({});
    client.on("connected", () => {
      console.log("connected");
    });

    client.on("join", (channel) => {
      console.log("Joined: ", channel);
      console.log(client.getChannels());
    });

    client.on("message", (channel, userstate, message, self) => {
      messages.unshift({ channel, userstate, message });
      messages = messages;
    });

    client.on("disconnected", (channel) => {
      console.log("Disconnected from twitch servers");
    });
    await client.connect().catch((err) => {
      console.log(err);
    });
  });

  onDestroy(async () => {
    client.disconnect();
  });
</script>

<div>
  <form on:submit|preventDefault={handleSubmit}>
    <input bind:value />
  </form>

  <div class="chat-container">
    {#each messages as message (message)}
      <p transition:fade|local animate:flip>
        [{message.channel}] {message.userstate.username}: {message.message}
      </p>
    {/each}
  </div>
  <div>
    {#each joinedChannels as channel (channel)}
      <div>{channel}</div>
    {/each}
  </div>
</div>

<style>
  .chat-container {
    border: 1px solid black;
    min-height: 30em;
    max-height: 30em;
    overflow: auto;
    scroll-behavior: auto;
  }
</style>
