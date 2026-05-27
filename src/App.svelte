<script>
  import { onMount } from "svelte";
  import Botones from "./components/Botones.svelte";
  import rawData from "./data/data.js";

  onMount(() => {
    function updateIframeHeight() {
      const height = Math.ceil(document.body.scrollHeight);
      const width = Math.ceil(document.body.scrollWidth);
      window.parent.postMessage(
        { type: "resize-iframe", value: height, width },
        "*",
      );
    }

    updateIframeHeight();

    if (window.ResizeObserver) {
      new ResizeObserver(() => updateIframeHeight()).observe(
        document.documentElement,
      );
    } else {
      window.addEventListener("resize", updateIframeHeight);
    }

    window.addEventListener("message", (event) => {
      if (event.data.type === "request-resize") updateIframeHeight();
    });
  });

  const years = Array.from({ length: 2024 - 2007 + 1 }, (_, i) =>
    (2007 + i).toString(),
  );

  let innerWidth;
  $: isMobile = innerWidth <= 675;

  $: ROW_HEIGHT = isMobile ? 25 : 30;
  $: ROW_GAP = isMobile ? 4 : 6;
  $: STEP = ROW_HEIGHT + ROW_GAP;

  const DURATION = 1200;

  const desktopTonalities = [
    "#ef2323",
    "#ef2d23",
    "#ef3723",
    "#ef4223",
    "#ef4c23",
    "#ef5623",
    "#ef6023",
    "#ef6a23",
    "#ef7523",
    "#ef7f23",
    "#ef8923",
    "#ef9323",
    "#ef9d23",
    "#efa823",
    "#efb223",
    "#efbc23",
    "#efc623",
    "#efd023",
    "#efdb23",
  ];
  const mobilePalette = ["#ef4423", "#ffcb04", "#3ab9d8"];
  const colorMap = {};
  const flagCodes = {
    Argentina: "ar",
    Brasil: "br",
    Bolivia: "bo",
    Chile: "cl",
    Colombia: "co",
    "Costa Rica": "cr",
    Cuba: "cu",
    Ecuador: "ec",
    "El Salvador": "sv",
    España: "es",
    Guatemala: "gt",
    Honduras: "hn",
    México: "mx",
    Nicaragua: "ni",
    Panamá: "pa",
    Paraguay: "py",
    Perú: "pe",
    Portugal: "pt",
    "R. Dominicana": "do",
    Uruguay: "uy",
    Venezuela: "ve",
  };

  const firstYear = years[0];
  const sortedInitial = [...rawData].sort(
    (a, b) => (b[firstYear] || 0) - (a[firstYear] || 0),
  );

  sortedInitial.forEach((d, i) => {
    colorMap[d.pais] = {
      desktop: desktopTonalities[i % desktopTonalities.length],
      mobile: mobilePalette[i % mobilePalette.length],
    };
  });

  function getColor(name, mobileMode) {
    if (!colorMap[name]) return "#ccc";
    return mobileMode ? colorMap[name].mobile : colorMap[name].desktop;
  }
  function formatNumber(val) {
    if (val === undefined || val === null) return "0";
    return Math.round(Number(val))
      .toString()
      .replace(/\B(?=(\d{3})+(?!\d))/g, ".");
  }

  const aggregatedData = rawData.map((d) => {
    let acc = 0;
    let history = { pais: d.pais };
    years.forEach((yr) => {
      acc += d[yr] || 0;
      history[yr] = acc;
    });
    return history;
  });

  function getRankedFrame(yearIdx) {
    const yr = years[yearIdx];
    const sorted = aggregatedData
      .map((d) => ({ name: d.pais, value: d[yr] }))
      .sort((a, b) => b.value - a.value);
    const maxVal = Math.max(...sorted.map((d) => d.value), 1);
    const result = {};
    sorted.forEach((d, i) => {
      result[d.name] = { rank: i, value: d.value, maxVal };
    });
    return result;
  }

  const frames = years.map((_, i) => getRankedFrame(i));

  // Función de cálculo de estados que usa el STEP reactivo
  function makeStatesFromFrame(frameIdx, currentStep) {
    const f = frames[frameIdx];
    const s = {};
    aggregatedData.forEach((d) => {
      s[d.pais] = {
        top: f[d.pais].rank * currentStep,
        widthPct: (f[d.pais].value / f[d.pais].maxVal) * 100,
        displayValue: f[d.pais].value,
      };
    });
    return s;
  }

  let currentIndex = 0;
  let displayYear = years[0];
  let isPlaying = false;
  let animationId;
  let startTime = null;
  let isScrubbing = false;

  $: barStates = makeStatesFromFrame(currentIndex, STEP);
  $: chartHeight = aggregatedData.length * STEP;

  function easeInOut(t) {
    return t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
  }
  function lerp(a, b, t) {
    return a + (b - a) * t;
  }

  function animateTo(fromIdx, toIdx) {
    const fromFrame = frames[fromIdx];
    const toFrame = frames[toIdx];
    startTime = null;
    function tick(now) {
      if (isScrubbing) return;
      if (!startTime) startTime = now;
      const elapsed = now - startTime;
      const rawT = Math.min(elapsed / DURATION, 1);
      const t = easeInOut(rawT);
      displayYear = years[Math.round(fromIdx + rawT * (toIdx - fromIdx))];
      const newStates = {};
      aggregatedData.forEach((d) => {
        const from = fromFrame[d.pais];
        const to = toFrame[d.pais];
        // Aquí multiplicamos por el STEP actual
        newStates[d.pais] = {
          top: lerp(from.rank * STEP, to.rank * STEP, t),
          widthPct: lerp(
            (from.value / from.maxVal) * 100,
            (to.value / to.maxVal) * 100,
            t,
          ),
          displayValue: Math.round(lerp(from.value, to.value, t)),
        };
      });
      barStates = newStates;
      if (rawT < 1) {
        animationId = requestAnimationFrame(tick);
      } else {
        currentIndex = toIdx;
        if (isPlaying && currentIndex < years.length - 1) {
          animationId = requestAnimationFrame(() =>
            animateTo(currentIndex, currentIndex + 1),
          );
        } else {
          isPlaying = false;
        }
      }
    }
    animationId = requestAnimationFrame(tick);
  }

  function toggle() {
    if (isPlaying) {
      isPlaying = false;
      cancelAnimationFrame(animationId);
    } else {
      if (currentIndex >= years.length - 1) jumpToIndex(0);
      isPlaying = true;
      animateTo(currentIndex, currentIndex + 1);
    }
  }

  function restart() {
    cancelAnimationFrame(animationId);
    isPlaying = false;
    isScrubbing = false;
    currentIndex = 0;
    displayYear = years[0];
  }
  function jumpToIndex(idx) {
    cancelAnimationFrame(animationId);
    isPlaying = false;
    isScrubbing = false;
    currentIndex = idx;
    displayYear = years[idx];
  }
  function onScrubberInput(e) {
    isScrubbing = true;
    cancelAnimationFrame(animationId);
    isPlaying = false;
    currentIndex = parseInt(e.target.value);
    displayYear = years[currentIndex];
  }
  function onScrubberChange() {
    isScrubbing = false;
  }
</script>

<svelte:window bind:innerWidth />

<main>
  <header>
    <div class="title-area">
      <h1 style="display:none">
        Iniciativas de Cooperación Sur-Sur y Triangular acumuladas por los
        países de Iberoamérica (2007-2024)
      </h1>
    </div>
  </header>

  <Botones
    {years}
    {currentIndex}
    {displayYear}
    {isPlaying}
    onToggle={toggle}
    onRestart={restart}
    onJumpToIndex={jumpToIndex}
    onStepBack={() => jumpToIndex(currentIndex - 1)}
    onStepForward={() => jumpToIndex(currentIndex + 1)}
    {onScrubberInput}
    {onScrubberChange}
  />

  <div class="chart" style="height: {chartHeight}px;">
    <div class="year-display-overlay">{displayYear}</div>

    {#each aggregatedData as item (item.pais)}
      {@const state = barStates[item.pais]}
      <div class="bar-row" style="top: {state.top}px; height: {ROW_HEIGHT}px;">
        <div class="rank">{Math.round(state.top / STEP) + 1}</div>
        <div class="label">{item.pais}</div>
        <div class="bar-container" style="height: {isMobile ? 18 : 24}px;">
          <div
            class="bar-fill"
            style="width: {state.widthPct}%; background-color: {getColor(
              item.pais,
              isMobile,
            )};"
          >
            {#if flagCodes[item.pais]}
              <img
                src="https://flagcdn.com/w80/{flagCodes[item.pais]}.png"
                alt=""
                class="flag-integrated"
              />
            {/if}
            <span class="value">{formatNumber(state.displayValue)}</span>
          </div>
        </div>
      </div>
    {/each}
  </div>
</main>

<style>
  main {
    max-width: 820px;
    margin: 0px auto;
    padding: 20px 30px;
    background: white;
    border-radius: 12px;
    position: relative;
  }

  @media (max-width: 675px) {
    main {
      padding: 15px 15px;
    }
    .label {
      width: 90px !important;
      padding-right: 8px !important;
      font-size: 0.75rem !important;
    }
    .flag-integrated {
      display: none !important;
    }
    .rank {
      width: 25px !important;
    }
    .year-display-overlay {
      font-size: 5rem !important;
      margin-bottom: 0px;
    }
    .value {
      font-size: 0.7rem !important;
    }
  }

  h1 {
    color: #212c55;
    font-size: 20px;
    margin: 0;
    line-height: 1.2;
  }
  .chart {
    position: relative;
    margin-top: 20px;
    transition: height 0.3s ease;
  }

  .year-display-overlay {
    position: absolute;
    bottom: 0;
    right: 0;
    font-size: 8rem;
    font-weight: 800;
    color: #c3c8d9;
    z-index: 1;
    pointer-events: none;
    user-select: none;
    font-variant-numeric: tabular-nums;
  }

  .bar-row {
    position: absolute;
    left: 0;
    width: 100%;
    display: flex;
    align-items: center;
    transition: top 0.2s linear;
  }

  .rank {
    width: 35px;
    font-size: 0.75rem;
    font-weight: 800;
    color: #cbd5e1;
    flex-shrink: 0;
  }

  .label {
    width: 150px;
    text-align: right;
    padding-right: 12px;
    font-size: 0.85rem;
    font-weight: 700;
    color: #334155;
    display: flex;
    align-items: center;
    justify-content: flex-end;
    height: 100%;
    flex-shrink: 0;
  }

  .bar-container {
    flex-grow: 1;
    background: #f1f5f9;
    border-radius: 4px;
    overflow: hidden;
    display: flex;
    align-items: center;
  }

  .bar-fill {
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: space-between;
    border-radius: 0 4px 4px 0;
    transition: width 0.2s linear;
    min-width: 40px;
  }

  .flag-integrated {
    height: 100%;
    width: 32px;
    object-fit: fill;
    border-right: 2px solid white;
    flex-shrink: 0;
  }

  .value {
    color: white;
    font-size: 0.8rem;
    font-weight: 800;
    text-shadow: 0 1px 2px rgba(0, 0, 0, 0.4);
    line-height: 1;
    white-space: nowrap;
    margin-right: 8px;
    margin-left: auto;
  }
</style>
