<script>
    import redLed from '$lib/assets/led-red-black.svg';
    import greenLed from '$lib/assets/led-green-black.svg';
    import yellowLed from '$lib/assets/led-amber-black.svg';

    let props = $props()

    const LEDs = {
        red: redLed,
        green: greenLed,
        yellow: yellowLed
    };
</script>

{#if LEDs[props.color]}
    <img src={LEDs[props.color]} alt="{props.color} status led" width="25" height="25"/>
{/if}