<script>

import pocket from './pocketbase.svelte';
import { announce_message, ip } from "./store.svelte";
import Announcer from './reusable/announcer.svelte';
import IpControlBar from './reusable/ip-control-bar.svelte';
import { ClientResponseError } from 'pocketbase';
import { push } from 'svelte-spa-router';

let passwordConfirm = $state("");
let username = $state("");
let email = $state("");
let password = $state("");
let message = $state("Signup for");
let file = $state(null);
let pfp_message = $state("Select Profile Picture");

const randomColor = () => {
    let color = "";
    for(let i = 0; i < 3; i++){
        color += (Math.floor(Math.random() * 192) + 64).toString() + ",";
    }
    return color;
}

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

async function signUp() {
    try {
        if(passwordConfirm == password && email != "" && username != "" && /^\S*$/.test(password) && password.length >= 8 && file != null){
            let color = parseColor(randomColor());
            console.log(file);
            const data = {
                "email": email,
                "emailVisibility": true,
                "username": username,
                "color": color,
                "avatar": file,
                "password": password,
                "passwordConfirm": passwordConfirm
            };
            const newUser = await pocket.pocket.collection('users').create(data);
            if(newUser){
                message = "Profile created for ";
            }

        } else if (email == "" || username == "" || password == "" || passwordConfirm == "") {
            announce_message("Fill out all fields");
        } else if (!/^[a-zA-Z0-9+_.-]+@[a-zA-Z0-9.-]+$/.test(email)) {
            announce_message("Invalid E-Mail. Try again");
        } else if (!/^\S*$/.test(password) || password.length < 8) {
            announce_message("Password must be 8 characters or longer with no whitespace");
        } else if (password != passwordConfirm) {
            announce_message("Passwords do not match");
        } else if (file == null) {
            announce_message("Choose a profile picture");
        } else {
            announce_message("Sign-up Failed. Try again");
        }
    } catch (err) {
        console.log(err);
        if(err instanceof ClientResponseError){
            if (err.status == 400){
                announce_message("Invalid signup credentials");
            } else if (err.status == 0){
                announce_message("Failed to connect to server at " + ip.ip + ":" + ip.port);
            }
        }
    }
}

const handleFileChange = (e) => { 
    file = e.target.files[0]; 
    pfp_message = file.name;

};

</script>

<Announcer />

<div class="header">
    <span>
        <a href="https://github.com/Royal51651/Conifer">
            <img src="src\assets\conifer-logo-nobg-512.png" alt="Conifer Logo" class="logo1"> 
        </a>
        <h1>
            {message}
        </h1>
    </span>
    
    <h1 style="color: var(--conifer-color-lightest);">
        &nbsp;{ip.ip + ":" + ip.port}
    </h1>
</div>
    

<div class="logicBox">

    {#if message== "Profile created for "}

    <button onclick={() => {
        push('/')
    }}>
        Go To Login
    </button>

    {:else}

    <IpControlBar />

    <input 
    type="text"
    placeholder="Username"
    bind:value={username}
    />
    <input 
        type="text"
        placeholder="Email"
        bind:value={email}
    />
    <input 
        type="password"
        placeholder="Password"
        bind:value={password}
    />
    <input 
        type="password"
        placeholder="Confirm Password"
        bind:value={passwordConfirm}
    />

    <input 
        style="display: none" 
        type="file" 
        onchange={handleFileChange} 
        accept="image/*" 
        id="uploadButton"
    />
    {#if pfp_message == "Select Profile Picture"}
        <label
            class="divWrapper fileUploader" 
            for='uploadButton'
        >
            {pfp_message}
        </label>
    {:else}
        <label
            class="divWrapper fileUploader" 
            for='uploadButton'
            style="color: white;"
        >
            {pfp_message}
        </label>
    {/if}

    <button onclick={signUp}>Submit</button>
    {/if}

</div>

<style>
    span {
        display: flex;
        flex-direction: row;
        word-wrap: break-word;
        width: 100%;
        justify-content: center;
        align-items: center;
        box-sizing: border-box;
    }

    .fileUploader {
        text-align: left;
        color: var(--conifer-color);
        cursor: pointer;
    }

    .fileUploader:hover {
        border-color: #dcdcdc;
    }

    .header {
        display: flex;
        flex-direction: row;
        justify-content: center;
        align-items: center;
    }

    h1 {
        display: flex;
        word-wrap: normal;
    }

    @media only screen and (max-width: 1200px) {
        .header {
            flex-direction: column;
        }
        span {
            justify-content: left;
            align-items: left;
        }
    }
</style>

