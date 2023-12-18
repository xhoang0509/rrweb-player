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

  const dispatch = createEventDispatcher();

  export let replayer: Replayer;
  export let showController: boolean;
  export let autoPlay: boolean;
  export let skipInactive: boolean;
  export let speedOption: number[];
  export let speed = speedOption.length ? speedOption[0] : 1;
  export let tags: Record<string, string> = {};

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
  };

  export const toggleSkipInactive = () => {
    skipInactive = !skipInactive;
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
      <div class="rr-controler-btn__left">
        <button on:click={toggle}>
          {#if playerState === 'playing'}
            <svg
              width="16"
              height="16"
              viewBox="0 0 48 48"
              xmlns="http://www.w3.org/2000/svg"
              ><g id="Play"
                ><path
                  d="m37.324 20.026-22-12.412a4.685 4.685 0 0 0 -4.711.036 4.528 4.528 0 0 0 -2.28 3.938v24.824a4.528 4.528 0 0 0 2.28 3.938 4.687 4.687 0 0 0 4.711.036l22-12.412a4.543 4.543 0 0 0 0-7.948z"
                /></g
              ></svg
            >
          {:else}
            <svg
              width="16"
              height="16"
              viewBox="0 0 24 24"
              fill="none"
              xmlns="http://www.w3.org/2000/svg"
            >
              <g clip-path="url(#clip0_908_391)">
                <path d="M3 22H9V2H3V22ZM15 2V22H21V2H15Z" fill="black" />
              </g>
              <defs>
                <clipPath id="clip0_908_391">
                  <rect width="24" height="24" fill="white" />
                </clipPath>
              </defs>
            </svg>
          {/if}
        </button>
      </div>
      <div class="rr-controler-btn__right">
        {#each speedOption as s}
          <button
            class:active={s === speed && speedState !== 'skipping'}
            on:click={() => setSpeed(s)}
          >
            {s}x
          </button>
        {/each}
        <div class="rr-fullscreen" on:click={() => dispatch('fullscreen')}>
          <button>
            <svg
              width="16"
              height="16"
              viewBox="0 0 24 24"
              xmlns="http://www.w3.org/2000/svg"
              ><g id="Layer_6" data-name="Layer 6"
                ><path
                  d="m21 9a1 1 0 0 1 -1-1v-4h-4a1 1 0 0 1 0-2h4a2 2 0 0 1 2 2v4a1 1 0 0 1 -1 1z"
                /><path
                  d="m20 22h-4a1 1 0 0 1 0-2h4v-4a1 1 0 0 1 2 0v4a2 2 0 0 1 -2 2z"
                /><path
                  d="m8 22h-4a2 2 0 0 1 -2-2v-4a1 1 0 0 1 2 0v4h4a1 1 0 0 1 0 2z"
                /><path
                  d="m3 9a1 1 0 0 1 -1-1v-4a2 2 0 0 1 2-2h4a1 1 0 0 1 0 2h-4v4a1 1 0 0 1 -1 1z"
                /></g
              ></svg
            >
          </button>
          <span>Fullscreen</span>
        </div>
        <Switch
          id="skip"
          bind:checked={skipInactive}
          disabled={false}
          label="skip inactive"
        />
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
    justify-content: space-around;
    align-items: space-between;
    border-radius: 0 0 5px 5px;
  }

  .rr-timeline {
    width: 80%;
    display: flex;
    align-items: center;
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
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 13px;
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
    border-radius: 50%;
    cursor: pointer;
  }

  .rr-controller__btns button:active {
    background: #e0e1fe;
  }

  .rr-controller__btns button.active {
    color: #fff;
    background: #5f6dc5;
  }

  .rr-controller__btns button:disabled {
    cursor: not-allowed;
  }
  .rr-controler-btn__right {
    display: flex;
    align-items: center;
  }

  .rr-fullscreen {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-right: 8px;
    cursor: pointer;
  }
</style>
