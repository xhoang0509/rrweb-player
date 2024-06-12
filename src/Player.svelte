<script lang="ts">
  import { unpack } from 'rrweb';
  import { Replayer } from '@bss-sbc/rrweb';
  import type { eventWithTime } from 'rrweb/typings/types';
  import { onDestroy, onMount } from 'svelte';
  import Controller from './Controller.svelte';
  import {
    exitFullscreen,
    inlineCss,
    isFullscreen,
    onFullscreenChange,
    openFullscreen,
    setCSS,
    typeOf,
  } from './utils';

  export let width: number = 1400;
  if (width < 700) {
    width = 700;
  }
  export let height: number = 720;
  export let events: eventWithTime[] = [];
  export let skipInactive: boolean = true;
  export let autoPlay: boolean = true;
  export let speedOption: number[] = [1, 2, 4, 8];
  export let speed: number = 1;
  export let showController: boolean = true;
  export let tags: Record<string, string> = {};
  let fullScreenClass: string = '';
  export let onPrevious: () => void = () => {};
  export let onNext: () => void = () => {};
  export let onAddTag: () => void = () => {};
  export let onFullscreen: () => void = () => {};
  let replayer: Replayer;

  export const getMirror = () => replayer.getMirror();

  const controllerHeight = 80;
  let player: HTMLElement;
  let frame: HTMLElement;
  let fullscreenListener: undefined | (() => void);
  let _width: number = width;
  let _height: number = height;
  let controller: {
    toggle: () => void;
    setSpeed: (speed: number) => void;
    toggleSkipInactive: () => void;
    toggleDisablePrevious: (status: boolean) => void;
    toggleDisableNext: (status: boolean) => void;
    hiddenPopup: (event: any) => void;
  } & Controller;
  let style: string;
  let bodyWidth: number = window.innerWidth;
  let playerStyle: string;

  $: style = inlineCss({
    width: `${width - 24}px`,
    height: `${height}px`,
  });

  $: playerStyle = inlineCss({
    width: `${width - 24}px`,
    height: `${height + (showController ? controllerHeight : 0)}px`,
  });

  const updateScale = (
    el: HTMLElement,
    frameDimension: { width: number; height: number },
  ) => {
    let widthScale = width / frameDimension.width;
    let heightScale = height / frameDimension.height;
    const rrPlayerFrame = document.querySelector('.rr-player__frame');

    if (replayer?.iframe && rrPlayerFrame) {
      const iframeWidth = parseInt(replayer.iframe.getAttribute('width'));
      const iframeHeight = parseInt(replayer.iframe.getAttribute('height'));
      widthScale = rrPlayerFrame.getBoundingClientRect().width / iframeWidth;
      heightScale = rrPlayerFrame.getBoundingClientRect().height / iframeHeight;
    }

    let scale = Math.min(widthScale, heightScale, 1);
    scale = scale * 0.9;
    
    el.style.transform = `scale(${scale}) translate(-50%, -50%)`;
  };

  const fixedController = (el: HTMLElement) => {
    const styles = {
      position: 'fixed',
      bottom: 0,
      left: 0,
      right: 0,
    };
    setCSS(el, styles);
  };

  const resetFixedController = (el: HTMLElement) => {
    const styles = {
      position: 'relative',
    };
    setCSS(el, styles);
  };

  export const triggerResize = () => {
    updateScale(replayer.wrapper, {
      width: replayer.iframe.offsetWidth,
      height: replayer.iframe.offsetHeight,
    });
  };

  export const toggleFullscreen = () => {
    if (player) {
      if (isFullscreen()) {
        exitFullscreen();
      } else {
        openFullscreen(player);
      }
    }
  };

  export const addEventListener = (
    event: string,
    handler: (detail: unknown) => unknown,
  ) => {
    replayer.on(event, handler);
    switch (event) {
      case 'ui-update-current-time':
      case 'ui-update-progress':
      case 'ui-update-player-state':
        controller.$on(event, ({ detail }) => handler(detail));
      default:
        break;
    }
  };

  export const addEvent = (event: eventWithTime) => {
    replayer.addEvent(event);
  };
  export const getMetaData = () => replayer.getMetaData();
  export const getReplayer = () => replayer;

  // by pass controller methods as public API
  export const toggle = () => {
    controller.toggle();
  };
  export const setSpeed = (speed: number) => {
    controller.setSpeed(speed);
  };
  export const toggleSkipInactive = () => {
    controller.toggleSkipInactive();
  };
  export const play = () => {
    controller.play();
  };
  export const pause = () => {
    controller.pause();
  };
  export const goto = (timeOffset: number, play?: boolean) => {
    controller.goto(timeOffset, play);
  };
  export const toggleDisablePrevious = (status: boolean) => {
    controller.toggleDisablePrevious(status);
  };
  export const toggleDisableNext = (status: boolean) => {
    controller.toggleDisableNext(status);
  };
  export const hiddenPopup = (event: any) => {
    controller.hiddenPopup(event);
  };

  onMount(() => {
    // runtime type check
    if (speedOption !== undefined && typeOf(speedOption) !== 'array') {
      throw new Error('speedOption must be array');
    }
    speedOption.forEach((item) => {
      if (typeOf(item) !== 'number') {
        throw new Error('item of speedOption must be number');
      }
    });
    if (speedOption.indexOf(speed) < 0) {
      throw new Error(`speed must be one of speedOption,
        current config:
        {
          ...
          speed: ${speed},
          speedOption: [${speedOption.toString()}]
          ...
        }
        `);
    }

    replayer = new Replayer(events, {
      speed,
      root: frame,
      unpackFn: unpack,
      ...$$props,
    });

    replayer.on('resize', (dimension) => {
      updateScale(
        replayer.wrapper,
        dimension as { width: number; height: number },
      );
    });

    fullscreenListener = onFullscreenChange(() => {
      const controllerWrapper = document.querySelector('.rr-controller');
      if (isFullscreen()) {
        setTimeout(() => {
          _width = width;
          _height = height;
          width = player.offsetWidth;
          height = player.offsetHeight;
          updateScale(replayer.wrapper, {
            width: replayer.iframe.offsetWidth,
            height: replayer.iframe.offsetHeight,
          });
          fixedController(controllerWrapper as HTMLElement);
        }, 0);
      } else {
        width = _width;
        height = _height;
        updateScale(replayer.wrapper, {
          width: replayer.iframe.offsetWidth,
          height: replayer.iframe.offsetHeight,
        });
        resetFixedController(controllerWrapper as HTMLElement);
      }
    });

    document.addEventListener('fullscreenchange', () => {
      onFullscreen();
      if (isFullscreen()) {
        fullScreenClass = 'full_screen';
      } else {
        fullScreenClass = '';
      }
    });

    window.addEventListener(
      'resize',
      function (_event) {
        let newWidth = document.querySelector('body').offsetWidth;
        if (newWidth < 900) {
          width = newWidth * 0.95;
        } else {
          width = newWidth * 0.75;
        }
        bodyWidth = window.innerWidth;
        updateScale(replayer.wrapper, {
          width: replayer.iframe.offsetWidth,
          height: replayer.iframe.offsetHeight,
        });
      },
      true,
    );
  });

  onDestroy(() => {
    fullscreenListener && fullscreenListener();
  });
</script>

<div class="rr-player {fullScreenClass}" bind:this={player} style={playerStyle}>
  <div class="rr-player__frame" bind:this={frame} {style} />
  {#if replayer}
    <Controller
      bind:this={controller}
      {replayer}
      {showController}
      {autoPlay}
      {speed}
      {speedOption}
      {skipInactive}
      {tags}
      {fullScreenClass}
      {bodyWidth}
      {onPrevious}
      {onNext}
      {onAddTag}
      on:fullscreen={() => toggleFullscreen()}
    />
  {/if}
</div>

<style global>
  @import 'rrweb/dist/rrweb.min.css';
  @import './styles/global.css';

  .rr-player {
    position: relative;
    background: white;
    float: left;
    border-radius: 8px;
    box-shadow: 0 24px 48px rgba(17, 16, 62, 0.12);
    border: 2px solid #ccc;
  }

  .rr-player.full_screen {
    border: unset;
  }

  .rr-player__frame {
    overflow: hidden;
  }

  .replayer-wrapper {
    float: left;
    clear: both;
    transform-origin: top left;
    left: 50%;
    top: 50%;
  }

  .replayer-wrapper > iframe {
    border: none;
  }
</style>
