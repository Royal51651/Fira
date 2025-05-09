<script lang="ts">
    import pocket from './pocketbase.svelte';
    import { onMount, onDestroy } from "svelte"
    import { announce_message } from "./store.svelte";
    import Announcer from './reusable/announcer.svelte';
    import { push } from 'svelte-spa-router';
    import pocket_container from './pocketbase.svelte';

    let message = $state("")
    let length = $derived(200 - message.length);

    async function logout(){
        pocket.pocket.authStore.clear();
        push('/');
    }

    let messages = $state([]);

    const parseColor = (input) => {
        let output = "rgb(";
        let count = 0;
        for(let i = 0; i < input.length; i++){
            if(input[i] == ","){
                count += 1; 
            }
            if(count != 3){
                output += input[i];
            }
        }
        return output + ");";
    }

    async function send(){;
        if(message != "" && length >= 0){
            const user = await pocket.pocket.collection('users').getOne(pocket.pocket.authStore.record.id, {
                expand: 'username,color',
            });
            let messageColor = parseColor(user.color);
            const data = {
                "Body": message,
                "Username": user.username,
                "Color": messageColor
            };
            const response = await pocket.pocket.collection('messages').create(data);
            if(response){
                message = "";
            }
        } else if (length < 0){
            announce_message("Maximum message size is 200 characters")
        }
    }

    // subscribe to the thingy so it works better
    let unsubscribe: () => void;
    onMount(async () => {
        const response = await pocket.pocket.collection("messages").getList(1, 25, {
            sort: '-created',
        });
        let temp = response.items;
        messages = [];
        let last_username = "";
        // this parses the 25 most recent messages and groups based on user
        // consecutive messages by the same user get the new: false attribute
        // in addition, new messages will spawn a spacer above
        try {
            for(let i of temp){
                if(i.Username == last_username){
                    let data = {
                        "new": false,
                        "body": i,
                    }
                    messages.unshift(data);
                } else {
                    let data = {
                        "new": true,
                        "body": i,
                    }
                    messages.unshift(data);
                    last_username = i.Username;
                }
            }
            messages.reverse();
            last_username = messages[0].body.Username;
        } catch {
            console.log("No messages")
        }

        unsubscribe = await pocket.pocket
            .collection('messages')
            .subscribe("*", async ({ action, record }) => {
                if(action === "create"){
                    if(record.Username == last_username){
                        let data = {
                            "new": true,
                            "body": record
                        }
                        messages.unshift(data);
                        messages[1].new = false;
                    } else {
                        let data = {
                            "new": true,
                            "body": record
                        }
                        messages.unshift(data);
                        last_username = record.Username;
                    }
                } else if (action === "delete") {
                    let index = 0;
                    for(let i of messages){
                        if(i.data == record){
                            if(messages[i].new){
                                messages[i + 1].new = true;
                            }
                            messages.splice(i, 1);
                        }
                        index++;
                    }
                }
            });
    });

    onDestroy(() => {
        logout();
        unsubscribe?.();
    });

</script>

<Announcer />

<div class="messageBar">
    <input
        onkeydown={(e) => e.key === "Enter" && send()}
        type="text"
        placeholder="Message"
        bind:value={message}
    >
    {#if length > 15}
    <span>
        <p1>{length}</p1>
    </span>
    {:else if length >= 0}
    <span>
        <p1 class="warning">{length}</p1>
    </span>
    {:else}
    <span>
        <p1 class="error">{length}</p1>
    </span>
    {/if}
</div>

<div class="messageContainer">

    {#each messages as message}

        {#if message.new}
            <div class="message">
                <span>
                    <p1 class="username" style="color: {message.body.Color}">{message.body.Username}</p1>
                    <p1 class="subText">{message.body.created.split(".")[0]}</p1>
                </span>
            </div>

        {/if}
        <div class="message">
            <p1 class="messageBody">{message.body.Body}</p1>
        </div>
        
    {/each}

</div>


<style>

    :root {
        background-color: var(--grey-color-main);
    }

    .messageBar {
        width: 100vw;
        max-width: 100vw;
        height: 10%;
        background-color: var(--conifer-color-dark);
        justify-content: left;
        align-items: center;
        display: flex;
        color: #fff;
        min-height: 80px;
        box-sizing: border-box;
        padding-left: 20px;
        padding-right: 20px;
        gap: 10px;
    }

    .messageBar input {
        width: 95%;
    }

    .messageBar span {
        border-radius: 8px;
        border: 1px solid transparent;
        padding: 0.5em;
        background-color: #1a1a1a;
        cursor: pointer;
        transition: border-color 0.25s;
        width: 5%;
        justify-content: center;
        align-items: center;
    }

    .messageBar ::placeholder {
        color: #d1d1d1;
    }

    .messageContainer {
        width: 100%;
        height: 100%;
        align-items: left;
        display: flex;
        flex-direction: column;
        gap: 5px;
    }

    .message {
        width: 90%;
        max-width: 90%;
        display: flex;
        width: max-content;
        flex-direction: column;
        align-items: left;
        padding-left: 1em;
        padding-right: 1em;
        padding-top: .2em;
        font-weight: 500;
        font-family: inherit;
        text-align: left;
        word-wrap: anywhere;
        white-space: normal;
        gap: 10px;
    }

    .subText {
        text-wrap: pretty;
        float: left;
    }

    .message::hover {
        background-color: var(--conifer-color-dark);
    }

    .username {
        font-size: 24px;
        font-weight: 500px;
    }

    .messageBody {
        font-size: 20px;
    }

    .subText {
        font-size: 12px;
    }

    span {
        display: flex;
        align-items: left;
        padding: 2px;

        font-family: inherit;
        gap: 10px;
    }

    .warning {
        color: #ffc400;
        font-size: 20px;
    }

    .error {
        color: #ac0000;
        font-size: 24px;
    }
</style>
