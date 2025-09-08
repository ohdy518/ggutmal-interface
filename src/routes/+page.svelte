<script>
    import ky from "ky";
    import {browser} from "$app/environment";
    import Status from "./Status.svelte";

    const baseAPIAddress = "http://localhost:8000/api";
    const cAPIHealthPoint = "/health"
    const cAPISubmitPoint = "/submit";
    const cAPIResetPoint = "/reset";

    let userInput;

    let serverHealth = false;
    let lastResetTimeStamp;
    let recentReset = false;

    let currentWord = "";
    let wordHistory = [];
    let explicitHistory = [];
    $: wordHistoryLength = explicitHistory.length;
    $: wordHistoryConcat = explicitHistory.join(` -> `)

    if (browser) {
        userInput = document.getElementById('user-input');
        checkAPIHealth()
        resetServer()
    }

    function on_key_down(event) {
        // `keydown` event is fired while the physical key is held down.

        // Assuming you only want to handle the first press, we early
        // return to skip.
        if (event.repeat) return;

        // In the switch-case we're updating our boolean flags whenever the
        // desired bound keys are pressed.
        switch (event.key) {
            case "Enter":
                console.log("enter key detection")
                submit()
        }
    }

    async function resetServer() {
        const data = await ky.get(baseAPIAddress + cAPIResetPoint).json();
        serverHealth = data['status'] === 'ok'
        if (serverHealth) {
            console.log("reset successful")
            lastResetTimeStamp = Date.now()
            recentReset = true
            setTimeout(() => {recentReset = false}, 1000)
            return;
        }
        console.error("failed to reset")
    }

    async function checkAPIHealth() {
        const data = await ky.get(baseAPIAddress + cAPIHealthPoint).json();
        if (data['status'] === 'ok') {
            console.log("API health ok")
            return;
        }
        console.error("API health error")

    }

    async function submit() {
        console.group("submit call trace");
        if (!userInput) { console.error("variable userInput is undefined"); return; }
        let value = userInput.value.trim();
        console.log(`value: ${value}`)
        if (!value) { console.info("early return: no user input"); return; }
        userInput.value = "";
        if (value.length <= 1) {
            console.info("early return: minimum char");
        }

        wordHistory.push(currentWord);
        currentWord = value

        console.group("request trace")

        const data = await ky.post(baseAPIAddress + cAPISubmitPoint, {
            json: { word: value }
        }).json();
        console.log(`status: ${data['status']}; newWord: ${data['newWord']}; gameOver: ${data['gameOver']}`);
        console.groupEnd();
        console.groupEnd();

        if (data['status'] === 'rejected') {
            currentWord = wordHistory.pop()
        } else { // accepted
            wordHistory.push(currentWord);
            currentWord = data['newWord'];
            if (data['gameOver'] === true) {
                gameOver()
            }
        }
        explicitHistory = wordHistory;
    }

    function gameOver() {

    }

</script>

<div class="p-3 w-screen h-screen grid grid-cols-1 grid-rows-8 gap-3 bg-slate-900 text-slate-50 ptd">
    <div id='header' class="debug p-2 jbm text-sm">
        <span class="{serverHealth ? 'bg-green-500' : 'bg-rose-500'} p-2">health {serverHealth ? 'ok' : 'bad'}</span>
        <span class="{lastResetTimeStamp ? (recentReset ? 'bg-green-500' : 'bg-slate-500') : 'bg-rose-500'} p-2">{lastResetTimeStamp ? (recentReset ? 'now reset' : 'once reset') : 'never reset'}</span>
        <span class="bg-amber-500 p-2">agent unknown</span>
<!--        <Status color="red"/>-->
    </div>
    <div class="row-span-2"></div>
    <div id="content" class="row-span-1 flex flex-col debug text-6xl">
        <div class="my-auto debug relative flex items-baseline">
            <div id="container-left" class="debug font-regular text-slate-300 whitespace-nowrap absolute right-1/2 bottom-px pr-2">
                {wordHistoryConcat} ->
            </div>
            <div id="container-right" class="debug font-semibold text-left text-8xl ml-[50%]">&nbsp;{currentWord}</div>
        </div>
    </div>
    <div id='definition' class="row-span-2">definition</div>
    <div id="controls" class="row-span-1 grid grid-cols-6 debug">
        <span id="status-message" class="debug col-span-3"></span>
        <input
                id="user-input"
                class="debug border-slate-300 text-slate-300 outline-slate-300 border-2 col-span-2 outline-offset-4 outline-2 focus:outline-solid outline-none p-3 text-6xl"
                placeholder="{currentWord.slice(-1)}..."
        >
    </div>
    <div id="footer" class="row-span-1">
        <span>version 0.1</span>
        <span>by ohdy518</span>
    </div>
</div>

<svelte:window
    on:keydown={on_key_down}
/>