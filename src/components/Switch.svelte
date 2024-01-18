<script lang="ts">
  export let disabled: boolean;
  export let checked: boolean;
  export let id: string;
  export let label: string;
  export let size: string = 'normal';

  const toggleSwitch = (
    e: MouseEvent & {
      currentTarget: EventTarget & HTMLInputElement;
    },
  ) => {
    if (id === 'autonext') {
      const checked = (e.target as HTMLInputElement).checked;
      localStorage.setItem('mida-auto-next-session', JSON.stringify(checked));
    }
  };
</script>

<div class="switch" class:disabled>
  <input
    type="checkbox"
    {id}
    bind:checked
    {disabled}
    on:click={(e) => toggleSwitch(e)}
  />
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
    font-size: 13px;
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
    background: #bdc3c7;
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
    content: url('data:image/svg+xml;utf8,<svg width="8" height="8" viewBox="0 0 8 8" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M4 0C1.79089 0 0 1.79086 0 4C0 6.20914 1.79089 8 4 8C6.20911 8 8 6.20914 8 4C8 1.79086 6.20911 0 4 0ZM5.3825 4.21203L3.3825 5.46203C3.34203 5.4873 3.29602 5.5 3.25 5.5C3.20831 5.5 3.16656 5.48963 3.12878 5.46863C3.04931 5.42456 3 5.34094 3 5.25V2.75C3 2.65906 3.04931 2.57544 3.12878 2.53137C3.20825 2.48706 3.30542 2.48975 3.3825 2.53797L5.3825 3.78797C5.45556 3.83375 5.5 3.91383 5.5 4C5.5 4.08617 5.45556 4.16627 5.3825 4.21203Z" fill="white"/></svg>');
    position: absolute;
    border-radius: 50px;
    left: 5px;
    transition: all 0.2s ease;
    animation: switch-off 0.2s ease-out;
    z-index: 2;
    top: -30%;
  }

  .switch input[type='checkbox']:checked + label:before {
    background: #707dd0;
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
