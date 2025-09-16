<script>
    import ky from "ky";
    import {browser} from "$app/environment";
    import Status from "./Status.svelte";

    const baseAPIAddress = "http://localhost:8000/api";
    const cAPIHealthPoint = "/health"
    const cAPISubmitPoint = "/submit";
    const cAPIResetPoint = "/reset";
    const cAPIDefinitionPoint = "/define";

    let userInput;

    let submitting = false;
    let serverHealth = false;
    let lastResetTimeStamp;
    let recentReset = false;
    let clientGameOver = false;

    let currentDefinition = "";

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
                if (!submitting) {
                    submit()
                }
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
        console.log("pinging server...")
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

        submitting = true

        wordHistory.push(currentWord);
        currentWord = value

        console.group("request trace")

        await getDefinition(currentWord)

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
            currentDefinition = data['definition'];
            if (data['gameOver'] === true) {
                setGameOver()
            }
        }
        explicitHistory = wordHistory;
        submitting = false
    }

    function setGameOver() {
        clientGameOver = true
    }

    async function getDefinition(value) {
        const data = await ky.post(baseAPIAddress + cAPIDefinitionPoint, {
            json: { word: value }
        }).json()

        currentDefinition = data['definition']
    }

</script>

<div class="p-3 w-screen h-screen grid grid-cols-1 grid-rows-8 gap-3 bg-slate-900 text-slate-50 ptd">
    <div id='header' class="flex flex-col gap-2 p-2 jbm text-sm">
        <div class="flex flex-row gap-2">
            <span class="ring-2 ring-inset ring-slate-500 p-2">technical status</span>
            <span class="{serverHealth ? 'bg-green-500' : 'bg-rose-500'} p-2">health {serverHealth ? 'ok' : 'bad'}</span>
            <span class="{lastResetTimeStamp ? (recentReset ? 'bg-green-500' : 'bg-slate-500') : 'bg-rose-500'} p-2">{lastResetTimeStamp ? (recentReset ? 'now reset' : 'once reset') : 'never reset'}</span>
            <span class="bg-amber-500 p-2">agent unknown</span>
        </div>
        <div class="flex flex-row gap-2">
            <span class="ring-2 ring-inset ring-slate-500 p-2">game status</span>
            <span class="{wordHistoryLength === 0 ? 'bg-slate-500' : (!clientGameOver ? 'bg-green-500' : 'bg-rose-500')} p-2">game {wordHistoryLength === 0 ? 'not started' : (!clientGameOver ? 'underway' : 'over')}</span>
            <span class="bg-slate-500 p-2">rally count {wordHistoryLength}</span>
        </div>

            <!--        <Status color="red"/>-->
    </div>
    <div class="row-span-2"></div>
    <div id="content" class="row-span-1 flex flex-col  text-6xl">
        <div class="my-auto  relative flex items-baseline">
            <div id="container-left" class=" font-regular text-slate-300 whitespace-nowrap absolute right-1/2 bottom-px pr-2">
                {wordHistoryConcat} ->
            </div>
            <div id="container-right" class=" font-semibold text-left text-8xl ml-[50%]">&nbsp;{currentWord}</div>
        </div>
    </div>
    <div id='definition' class="row-span-2 relative flex flex-col">
<!--        <div id="filler-container" class="right-1/2 absolute h-3 debug"></div>-->
        <div class="kopub text-2xl leading-9 ml-[50%] pr-45">
            <span class="font-bold text-3xl">{currentWord}</span><br/> {currentDefinition}
        </div>
    </div>
    <div id="controls" class="row-span-1 grid grid-cols-6 ">
        <span id="status-message" class=" col-span-3"></span>
        <input
                id="user-input"
                class=" border-slate-300 text-slate-300 outline-slate-300 border-2 col-span-2 outline-offset-4 outline-2 focus:outline-solid outline-none p-3 text-6xl"
                placeholder="{currentWord.slice(-1)}..."
        >
    </div>
    <div id="footer" class="row-span-1">
        <span>agent: random</span>
        <span>development</span>
    </div>
</div>

<svelte:window
    on:keydown={on_key_down}
/>
