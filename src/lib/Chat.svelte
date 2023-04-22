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
        .then(() => joinedChannels.push("#" + channel))
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
    const channels = JSON.parse(localStorage.getItem("channels"));
    client = new tmi.Client({ channels: channels });
    joinedChannels = channels;
    client.on("connected", () => {
      console.log("connected");
    });

    client.on("part", (channel, username, self) => {
      if (self) {
        console.log("Left :", channel);
        localStorage.setItem("channels", JSON.stringify(client.getChannels()));
      }
    });

    client.on("join", (channel, username, self) => {
      if (self) {
        console.log("Joined: ", channel);
        localStorage.setItem("channels", JSON.stringify(client.getChannels()));
      }
    });

    client.on("chat", (channel, userstate, message) => {
      messages = [{ channel, userstate, message }, ...messages];
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
    <label for="channels">Enter channel(s)</label>
    <input name="channels" bind:value />
  </form>

  <div id="chat-container">
    <div id="messages-container">
      {#each messages as message (message)}
        <div
          transition:fade|local={{ duration: 50 }}
          animate:flip={{ duration: 100 }}
        >
          <span>[{message.channel}] </span>
          <span style="color: {message.userstate.color};"
            >{message.userstate.username}:
          </span>
          <span>
            {message.message}
          </span>
        </div>
      {/each}
    </div>
  </div>
  <div>
    <div>Joined Channels</div>
    {#each joinedChannels as channel (channel)}
      <div>
        <span>{channel}</span>
        <button
          on:click={() => {
            leaveChannel(client, channel);
          }}>Leave</button
        >
      </div>
    {/each}
  </div>
</div>

<style>
  :root {
    background-color: black;
    color: white;
    font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
      Roboto, Oxygen, Ubuntu, Cantarell, "Open Sans", "Helvetica Neue",
      sans-serif;
  }
  #container {
    max-height: 100%;
    display: flex;
    flex-direction: column;
    align-items: normal;
    gap: 4px;
  }
  #messages-container::-webkit-scrollbar {
    display: none;
  }
  #messages-container {
    display: flex;
    flex-direction: column-reverse;
    border: 1px solid gray;
    min-height: 30em;
    max-height: 30em;
    overflow-y: scroll;
    color: white;
    padding: 1em 1em;
    gap: 1rem;
    scrollbar-width: none;
  }
</style>
