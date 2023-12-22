<script lang="ts">
  export let disabled: boolean;
  export let checked: boolean;
  export let id: string;
  export let label: string;
  export let size: string = 'normal';
</script>

<div class="switch" class:disabled>
  <input type="checkbox" {id} bind:checked {disabled} />
  <label for={id} class={size} />
  <span class="label">{label}</span>
</div>

<style>
  .switch {
    height: 1em;
    display: flex;
    align-items: center;
  }

  .switch.disabled {
    opacity: 0.5;
  }

  .label {
    margin: 0 8px;
  }

  .switch input[type='checkbox'] {
    position: absolute;
    opacity: 0;
  }

  .switch label {
    width: 2em;
    height: 1em;
    position: relative;
    cursor: pointer;
    display: block;
  }

  .switch.disabled label {
    cursor: not-allowed;
  }

  .switch label {
    display: flex;
    align-items: center;
  }

  .switch:has(label.small) span.label {
    margin-left: 2px;
    user-select: none;
  }

  .switch label:before {
    content: '';
    position: absolute;
    width: 2em;
    height: 1em;
    left: 0.1em;
    transition: background 0.1s ease;
    background: rgba(73, 80, 246, 0.5);
    border-radius: 50px;
  }

  .switch label.small:before {
    width: 1.6em;
    height: 0.8em;
  }

  .switch label:not(.small):after {
    content: '';
    position: absolute;
    width: 0.6em;
    height: 0.6em;
    border-radius: 50px;
    left: 5px;
    transition: all 0.2s ease;
    box-shadow: 0px 2px 5px 0px rgba(0, 0, 0, 0.3);
    background: #fcfff4;
    animation: switch-off 0.2s ease-out;
    z-index: 2;
  }

  .switch label.small:after {
    content: url('./icons/Vector.svg');
    position: absolute;
    border-radius: 50px;
    left: 5px;
    transition: all 0.2s ease;
    animation: switch-off 0.2s ease-out;
    z-index: 2;
    top: -2px;
  }

  .switch input[type='checkbox']:checked + label:before {
    background: #5f6dc5;
  }

  .switch input[type='checkbox']:checked + label:after {
    animation: switch-on 0.2s ease-out;
    left: 1.2em;
  }
  .switch input[type='checkbox']:checked + label.small:after {
    animation: switch-on 0.2s ease-out;
    left: 1em;
  }
</style>
