<script lang="ts">
  import { EventType } from 'rrweb';
  import type { Replayer } from 'rrweb';
  import type { playerMetaData } from 'rrweb/typings/types';
  import type {
    PlayerMachineState,
    SpeedMachineState,
  } from 'rrweb/typings/replay/machine';
  import {
    onMount,
    onDestroy,
    createEventDispatcher,
    afterUpdate,
  } from 'svelte';
  import { formatTime } from './utils';
  import Switch from './components/Switch.svelte';
  import {
    IconAddTag,
    MobileVerticalDotsMajor,
    btnForward,
    btnFullscreen,
    btnNext,
    btnPause,
    btnPlaying,
    btnPrevious,
    btnRewind,
  } from './constants/buttons.const';
  const dispatch = createEventDispatcher();

  export let replayer: Replayer;
  export let showController: boolean;
  export let autoPlay: boolean;
  export let skipInactive: boolean;
  export let speedOption: number[];
  export let speed = speedOption.length ? speedOption[0] : 1;
  export let tags: Record<string, string> = {};
  export let activeSpeed: boolean = false;
  export let onPrevious: () => void = () => {};
  export let onNext: () => void = () => {};
  export let onAddTag: () => void = () => {};
  let disablePrevious: boolean = false;
  let disableNext: boolean = false;
  let more: boolean = false;
  let autonext: boolean = false;
  let size: string = 'small';
  let currentTime = 0;
  $: {
    dispatch('ui-update-current-time', { payload: currentTime });
  }
  let timer: number | null = null;
  let playerState: 'playing' | 'paused' | 'live';
  $: {
    dispatch('ui-update-player-state', { payload: playerState });
  }
  let speedState: 'normal' | 'skipping';
  let progress: HTMLElement;
  let step: HTMLElement;
  let finished: boolean;

  let meta: playerMetaData;
  $: meta = replayer.getMetaData();
  let percentage: string;
  $: {
    const percent = Math.min(1, currentTime / meta.totalTime);
    percentage = `${100 * percent}%`;
    dispatch('ui-update-progress', { payload: percent });
  }
  type CustomEvent = {
    name: string;
    background: string;
    position: string;
  };
  let customEvents: CustomEvent[];
  $: customEvents = (() => {
    const { context } = replayer.service.state;
    const totalEvents = context.events.length;
    const start = context.events[0].timestamp;
    const end = context.events[totalEvents - 1].timestamp;
    const customEvents: CustomEvent[] = [];

    // calculate tag position.
    const position = (startTime: number, endTime: number, tagTime: number) => {
      const sessionDuration = endTime - startTime;
      const eventDuration = endTime - tagTime;
      const eventPosition = 100 - (eventDuration / sessionDuration) * 100;

      return eventPosition.toFixed(2);
    };

    // loop through all the events and find out custom event.
    context.events.forEach((event) => {
      /**
       * we are only interested in custom event and calculate it's position
       * to place it in player's timeline.
       */
      if (event.type === EventType.Custom) {
        const customEvent = {
          name: event.data.tag,
          background: tags[event.data.tag] || '#5f6dc5',
          position: `${position(start, end, event.timestamp)}%`,
        };
        customEvents.push(customEvent);
      }
    });

    return customEvents;
  })();

  const loopTimer = () => {
    stopTimer();

    function update() {
      currentTime = replayer.getCurrentTime();

      if (currentTime < meta.totalTime) {
        timer = requestAnimationFrame(update);
      }
    }

    timer = requestAnimationFrame(update);
  };

  const stopTimer = () => {
    if (timer) {
      cancelAnimationFrame(timer);
      timer = null;
    }
  };

  export const toggle = () => {
    switch (playerState) {
      case 'playing':
        pause();
        break;
      case 'paused':
        play();
        break;
      default:
        break;
    }
  };

  export const play = () => {
    if (playerState !== 'paused') {
      return;
    }
    if (finished) {
      replayer.play();
      finished = false;
    } else {
      replayer.play(currentTime);
    }
  };

  export const pause = () => {
    if (playerState !== 'playing') {
      return;
    }
    replayer.pause();
  };

  export const goto = (timeOffset: number, play?: boolean) => {
    currentTime = timeOffset;
    const resumePlaying =
      typeof play === 'boolean' ? play : playerState === 'playing';
    if (resumePlaying) {
      replayer.play(timeOffset);
    } else {
      replayer.pause(timeOffset);
    }
  };

  const handleProgressClick = (event: MouseEvent) => {
    if (speedState === 'skipping') {
      return;
    }
    const progressRect = progress.getBoundingClientRect();
    const x = event.clientX - progressRect.left;
    let percent = x / progressRect.width;
    if (percent < 0) {
      percent = 0;
    } else if (percent > 1) {
      percent = 1;
    }
    const timeOffset = meta.totalTime * percent;
    goto(timeOffset);
  };

  export const setSpeed = (newSpeed: number) => {
    let needFreeze = playerState === 'playing';
    speed = newSpeed;
    if (needFreeze) {
      replayer.pause();
    }
    replayer.setConfig({ speed });
    if (needFreeze) {
      replayer.play(currentTime);
    }
    toggleOptionsPopup();
  };

  export const toggleSkipInactive = () => {
    skipInactive = !skipInactive;
  };

  const toggleOptionsPopup = () => {
    activeSpeed = !activeSpeed;
  };

  const handleRewindTime = () => {
    if (currentTime - 10000 > 0) {
      goto(currentTime - 10000);
    } else {
      goto(0);
    }
  };

  const handleForwardTime = () => {
    if (currentTime + 10000 < meta.totalTime) {
      goto(currentTime + 10000);
    } else {
      goto(meta.totalTime);
    }
  };

  const handlePrevious = () => {
    if (disablePrevious) {
      return;
    }
    onPrevious();
  };

  const handleNext = () => {
    if (disableNext) {
      return;
    }
    onNext();
  };

  export const toggleDisablePrevious = (status: boolean) => {
    disablePrevious = status;
  };
  export const toggleDisableNext = (status: boolean) => {
    disableNext = status;
  };

  const toggleMore = () => {
    more = !more;
  };

  const handleFullscreen = () => {
    dispatch('fullscreen');
    activeSpeed = false;
    more = false;
  };

  onMount(() => {
    playerState = replayer.service.state.value as typeof playerState;
    speedState = replayer.speedService.state.value as typeof speedState;
    replayer.on(
      'state-change',
      (states: { player?: PlayerMachineState; speed?: SpeedMachineState }) => {
        const { player, speed } = states;
        if (player?.value && playerState !== player.value) {
          playerState = player.value as typeof playerState;
          switch (playerState) {
            case 'playing':
              loopTimer();
              break;
            case 'paused':
              stopTimer();
              break;
            default:
              break;
          }
        }
        if (speed?.value && speedState !== speed.value) {
          speedState = speed.value as typeof speedState;
        }
      },
    );
    replayer.on('finish', () => {
      finished = true;
    });

    if (autoPlay) {
      replayer.play();
    }

    const autoNextCache =
      JSON.parse(localStorage.getItem('mida-auto-next-session')) || '';

    if (autoNextCache) {
      autonext = autoNextCache;
    }
  });

  afterUpdate(() => {
    if (skipInactive !== replayer.config.skipInactive) {
      replayer.setConfig({ skipInactive });
    }
  });

  onDestroy(() => {
    replayer.pause();
    stopTimer();
  });
</script>

{#if showController}
  <div class="rr-controller">
    <div class="rr-timeline">
      <span class="rr-timeline__time">{formatTime(currentTime)}</span>
      <div
        class="rr-progress"
        bind:this={progress}
        on:click={(event) => handleProgressClick(event)}
      >
        <div
          class="rr-progress__step"
          bind:this={step}
          style="width: {percentage}"
        />
        {#each customEvents as event}
          <div
            title={event.name}
            style="width: 10px;height: 5px;position: absolute;top:
            2px;transform: translate(-50%, -50%);background: {event.background};left:
            {event.position};"
          />
        {/each}

        <div class="rr-progress__handler" style="left: {percentage}" />
      </div>
      <span class="rr-timeline__time">{formatTime(meta.totalTime)}</span>
    </div>
    <div class="rr-controller__btns">
      <div class="rr-controller-btn__left">
        <button on:click={toggle}>
          {#if playerState === 'playing'}
            {@html btnPause}
          {:else}
            {@html btnPlaying}
          {/if}
        </button>
        <div class="rr-time">
          {formatTime(currentTime)} / {formatTime(meta.totalTime)}
        </div>
        <button
          class="rr-btn-common-action rr-button-rewind"
          on:click={handleRewindTime}
        >
          <span>10s</span>
          &nbsp;
          {@html btnRewind}
        </button>
        <button
          class="rr-btn-common-action rr-button-forward"
          on:click={handleForwardTime}
        >
          {@html btnForward}
          &nbsp;
          <span>10s</span>
        </button>
        <button
          class="rr-btn-common-action rr-button-previous {disablePrevious
            ? 'disable'
            : ''}"
          on:click={handlePrevious}
        >
          <span>Previous</span>
          &nbsp;
          {@html btnPrevious}
        </button>
        <button
          class="rr-btn-common-action rr-button-next {disableNext
            ? 'disable'
            : ''}"
          on:click={handleNext}
        >
          {@html btnNext}
          &nbsp;
          <span>Next</span>
        </button>
      </div>
      <div class="rr-controller-btn__right">
        {#if activeSpeed}
          <div class="rr-list-speed">
            {#each speedOption as s}
              <button
                class:active={s === speed && speedState !== 'skipping'}
                on:click={() => setSpeed(s)}
              >
                {s}x
              </button>
            {/each}
          </div>
        {/if}

        <div class="rr-speed-wrapper" on:click={() => toggleOptionsPopup()}>
          {speed}x Speed
        </div>
        <div class="rr-fullscreen" on:click={() => handleFullscreen()}>
          <button>
            {@html btnFullscreen}
          </button>
          <span>Fullscreen</span>
        </div>
        <Switch
          id="skip"
          bind:checked={skipInactive}
          disabled={false}
          label="Skip inactive"
        />
        <div class="rr-more">
          <button
            class="rr-more-button rr-btn-common-action"
            on:click={toggleMore}
          >
            {@html MobileVerticalDotsMajor}
          </button>
          {#if more}
            <div class="rr-more-popup">
              <div class="rr-more-add-tag" on:click={onAddTag}>
                <span>{@html IconAddTag}</span>
                <span>Add tags</span>
              </div>
              <div class="rr-more-autonext">
                <Switch
                  id="autonext"
                  bind:checked={autonext}
                  bind:size
                  disabled={false}
                  label="Autoplay"
                />
              </div>
            </div>
          {/if}
        </div>
      </div>
    </div>
  </div>
{/if}

<style>
  .rr-controller {
    width: 100%;
    height: 80px;
    background: #fff;
    display: flex;
    flex-direction: column;
    justify-content: unset;
    align-items: unset;
    border-radius: 0 0 5px 5px;
  }

  .rr-timeline {
    width: 80%;
    display: flex;
    align-items: center;
    flex: 1;
  }

  .rr-timeline__time {
    display: inline-block;
    width: 100px;
    text-align: center;
    color: #11103e;
  }

  .rr-progress {
    flex: 1;
    height: 12px;
    background: #eee;
    position: relative;
    border-radius: 3px;
    cursor: pointer;
    box-sizing: border-box;
    border-top: solid 4px #fff;
    border-bottom: solid 4px #fff;
  }

  .rr-progress.disabled {
    cursor: not-allowed;
  }

  .rr-progress__step {
    height: 100%;
    position: absolute;
    left: 0;
    top: 0;
    background: #e0e1fe;
  }

  .rr-progress__handler {
    width: 20px;
    height: 20px;
    border-radius: 10px;
    position: absolute;
    top: 2px;
    transform: translate(-50%, -50%);
    background: #5f6dc5;
  }

  .rr-controller__btns {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: space-between;
    background-color: #303335;
    color: #fff;
    height: 44px;
    font-size: 14px;
    padding: 0 8px;
  }

  .rr-controller__btns button {
    width: 32px;
    height: 32px;
    display: flex;
    padding: 0;
    align-items: center;
    justify-content: center;
    background: none;
    border: none;
    border-radius: 8px;
    cursor: pointer;
  }

  .rr-controller__btns button.active {
    background-color: #f1f1f1;
  }

  .rr-controller__btns .rr-list-speed button:hover {
    background-color: #f1f1f1;
  }

  /* .rr-controller__btns button.active {
    color: #fff;
    background: #5f6dc5;
  } */

  .rr-controller__btns button:disabled {
    cursor: not-allowed;
  }
  .rr-controller-btn__left {
    display: flex;
    align-items: center;
  }

  .rr-controller-btn__right {
    display: flex;
    align-items: center;
    position: relative;
  }

  .rr-fullscreen {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-right: 8px;
    cursor: pointer;
  }

  .rr-speed-wrapper {
    cursor: pointer;
    user-select: none;
    margin-right: 8px;
  }

  .rr-list-speed {
    position: absolute;
    background-color: #fff;
    bottom: 36px;
    left: 18px;
    border-radius: 8px;
    padding: 8px, 0px, 8px, 0px;
    z-index: 2;
    box-shadow: -1px 1px 16px 0px rgba(0, 0, 0, 0.75);
    -webkit-box-shadow: -1px 1px 16px 0px rgba(0, 0, 0, 0.75);
    -moz-box-shadow: -1px 1px 16px 0px rgba(0, 0, 0, 0.75);
  }

  /* button actions */
  /* button skip 10s */

  .rr-btn-common-action {
    width: auto !important;
    height: 26px !important;
    padding: 2px 4px !important;
    background-color: #373a3e !important;
  }

  .rr-btn-common-action:hover {
    background-color: #4a4a4a !important;
  }

  .rr-btn-common-action:active {
    background-color: #616161 !important;
  }

  .rr-btn-common-action.disable {
    opacity: 0.8;
    cursor: default;
  }

  .rr-btn-common-action.disable:hover {
    background-color: initial;
  }

  .rr-btn-common-action span {
    color: white;
  }

  .rr-button-rewind {
    margin-left: 8px;
    margin-right: 4px;
  }

  .rr-button-forward {
  }

  .rr-button-rewind span {
    margin-right: 4px;
  }
  .rr-button-forward span {
    margin-left: 4px;
  }

  /* button previous & next */
  .rr-button-previous {
    margin-left: 12px;
  }
  .rr-button-next {
    margin-left: 4px;
  }
  /* times */
  .rr-time {
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 0 4px;
    margin-left: 8px;
  }

  /* more */
  .rr-more {
    position: relative;
  }
  .rr-more-button {
    border: 1px solid #fff !important;
  }
  .rr-more-popup {
    position: absolute;
    bottom: calc(100% + 4px);
    right: 0;
    background-color: #fff;
    color: #000;
    padding: 8px, 0px, 8px, 0px;
    border-radius: 8px;
    box-shadow: -1px 1px 16px 0px rgba(0, 0, 0, 0.75);
    -webkit-box-shadow: -1px 1px 16px 0px rgba(0, 0, 0, 0.75);
    -moz-box-shadow: -1px 1px 16px 0px rgba(0, 0, 0, 0.75);
  }
  .rr-more-add-tag,
  .rr-more-autonext {
    width: 114px;
    height: 36px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
  }

  .rr-more-add-tag {
    user-select: none;
  }

  .rr-more-add-tag > span:first-child {
    padding-top: 4px !important;
    padding-right: 4px;
  }
</style>
