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
  let targetChannels: string[] = [];
  let joinedChannels: string[] = [];
  let value: string;

  async function leaveChannel(client: tmi.Client, channel: string) {
    await client
      .part(channel)
      .then(() => {
        joinedChannels = joinedChannels.filter(
          (joinedChannel) => joinedChannel != channel
        );
      })
      .catch((err) => console.log(err));
  }

  async function joinChannels(client: tmi.Client) {
    targetChannels.forEach(async (channel) => {
      if (joinedChannels.includes(channel)) {
        console.log("channel already joined");
        return;
      }
      await client
        .join(channel)
        .then(() => joinedChannels.push(channel))
        .catch((err) => console.log("Could not join channel:", channel));
      joinedChannels = joinedChannels;
    });
  }

  async function handleSubmit() {
    targetChannels = value
      .toLowerCase()
      .trim()
      .split(/\s*[,.\s]+\s*/);
    value = "";

    targetChannels = targetChannels.filter((channel) => channel !== "");
    console.log(targetChannels);
    joinChannels(client);
  }

  onMount(async () => {
    client = new tmi.Client({});
    client.on("connected", () => {
      console.log("connected");
    });

    client.on("part", (channel, self) => {
      if (!self) return;
      console.log("Left :", channel);
    });

    client.on("join", (channel, self) => {
      if (!self) return;

      console.log("Joined: ", channel);
      console.log(client.getChannels());
    });

    client.on("chat", (channel, userstate, message, self) => {
      messages.unshift({ channel, userstate, message });
      messages = messages;
    });

    client.on("disconnected", () => {
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

<div id="container">
  <form on:submit|preventDefault={handleSubmit}>
    <input bind:value />
  </form>

  <div id="chat-container">
    <div id="messages-container">
      {#each messages as message (message)}
        <div
          transition:fade|local={{ duration: 50 }}
          animate:flip={{ duration: 100 }}
        >
          <p>
            <span>[{message.channel}] </span>
            <span style="color: {message.userstate.color};"
              >{message.userstate.username}:
            </span>
            {message.message}
          </p>
        </div>
      {/each}
    </div>
  </div>
  <div>
    {#each joinedChannels as channel (channel)}
      <div>{channel}</div>
      <button
        on:click={() => {
          leaveChannel(client, channel);
        }}>Leave</button
      >
    {/each}
  </div>
</div>

<style>
  :root {
    background-color: black;
    color: white;
  }
  #container {
    display: flex;
    flex-direction: column;
    align-items: normal;
  }
  #messages-container {
    border: 1px solid gray;
    min-height: 30em;
    max-height: 30em;
    overflow: auto;
    scroll-behavior: auto;
    color: white;
  }
</style>
