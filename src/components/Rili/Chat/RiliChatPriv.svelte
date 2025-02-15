<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import autoAnimate from "@formkit/auto-animate";
  import { Icon, PaperAirplane } from "svelte-hero-icons";
  import moment from "moment";

  import { pb, currentUser } from "@/lib/pocketbase";
  import { cn, getImageURL } from "@/lib/utils";
  import { parseMsg } from "./utils";

  let newMessage: string;
  let messages = [];
  let unsubscribe: () => void;
  let sending = false;
  let latestMessage: string;

  onMount(async () => {
    if (!$currentUser || !$currentUser.admin) return;

    messages = await pb
      .collection("privateMessages")
      .getList(1, 100, {
        sort: "created",
        expand: "sender",
      })
      .then((records) => records.items);

    unsubscribe = await pb
      .collection("privateMessages")
      .subscribe("*", async ({ action, record }) => {
        if (action === "create") {
          const sender = await pb.collection("users").getOne(record.sender);
          record.expand = { sender };
          messages = [...messages, record];
          if (sender.id === $currentUser.id) {
            scroll();
          }
        }
      });

    document.addEventListener("keydown", handleEnter);
    scroll();
    document.getElementById("message-bar").focus();
  });

  onDestroy(() => {
    document.removeEventListener("keydown", handleEnter);
    unsubscribe();
  });

  function handleEnter(event: KeyboardEvent) {
    if (event.key === "Enter") {
      console.log("fire");
      sendMessage();
    }
  }

  function scroll() {
    window.scrollTo({
      top: document.body.scrollHeight,
      left: 0,
      behavior: "smooth",
    });
  }

  async function sendMessage() {
    if (!sending && newMessage.trim() && latestMessage !== newMessage) {
      sending = true;
      latestMessage = newMessage;
      await pb.collection("privateMessages").create({
        text: newMessage.trim(),
        sender: $currentUser.id,
      });
      setTimeout(() => {
        sending = false;
      }, 1000);
    }
    newMessage = undefined;
    document.getElementById("message-bar").focus();
  }

  async function showAll() {
    messages = await pb
      .collection("privateMessages")
      .getList(1, 1000000, {
        sort: "created",
        expand: "sender",
      })
      .then((records) => records.items);
  }

  function shouldCreateBorder(msg, msgArr: Array<any>): boolean {
    const index = messages.findIndex((_message) => _message.id === msg.id);
    const before = msgArr[index - 1 < 0 ? 0 : index - 1];

    return moment(before.created).day() !== moment(msg.created).day();
  }
</script>

{#if !$currentUser}
  <h1 class="text-error">Not logged in</h1>
{:else if !$currentUser.admin}
  <h1 class="text-error">Unauthorized</h1>
{:else}
  <div class="flex min-h-screen w-screen flex-row justify-center">
    <div class="mb-8 mt-8 w-2/3 pb-8 pt-8">
      <div use:autoAnimate>
        {#each messages as message (message.id)}
          {#if shouldCreateBorder(message, messages)}
            <div class="divider divider-secondary w-full">
              <time class="text-secondary"
                >{moment(message.created).format("DD/MM/YYYY")}</time
              >
            </div>
          {/if}
          <div
            class={cn("chat", {
              "chat-start": message.expand?.sender?.id !== $currentUser.id,
              "chat-end": message.expand?.sender?.id === $currentUser.id,
            })}
          >
            <div class="avatar chat-image">
              <div class="w-10 rounded-full">
                <img
                  src={message.expand?.sender?.avatar
                    ? getImageURL(
                        message.expand?.sender?.collectionId,
                        message.expand?.sender?.id,
                        message.expand?.sender?.avatar,
                      )
                    : `https://ui-avatars.com/api/?name=${message.expand?.sender?.name}`}
                  alt=""
                />
              </div>
            </div>
            <div class="chat-header">
              {message.expand?.sender?.name}
              <time class="text-xs opacity-50"
                >{moment(message.created).format("HH:mm")}</time
              >
            </div>
            <div class="chat-bubble chat-bubble-accent">
              {@html parseMsg(message)}
            </div>
          </div>
        {/each}
      </div>
      <div
        class="mt-4 flex w-full items-center gap-2 rounded-full bg-base-200 p-4"
      >
        <input
          class="input input-bordered inline-flex w-full max-w-lg flex-1 items-center rounded-full bg-base-100 px-4 py-2"
          type="text"
          placeholder="Soy un castor"
          id="message-bar"
          bind:value={newMessage}
        />
        <button
          class="btn btn-primary inline-flex max-w-12 flex-1 items-center rounded-full px-3 py-1"
          on:click={sendMessage}
          ><div class="h-6 w-6"><Icon src={PaperAirplane} /></div></button
        >
        <button
          class="btn btn-secondary inline-flex items-center rounded-full"
          on:click={showAll}>Show all</button
        >
      </div>
    </div>
  </div>
{/if}
