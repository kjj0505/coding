<script>
  import * as Tone from 'tone';
  import { writable, get } from 'svelte/store';
  import { tick } from 'svelte';

  const NOTES = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];

  let tuningNotes = ["E4", "B3", "G3", "D3", "A2", "E2"];
  let tuning = ["E", "B", "G", "D", "A", "E"];
  let selectedTuning = "standard";
  let selectedGuitar = "acoustic";

  let tab = Array(6).fill().map(() => Array(60).fill(""));
  let currentTabPos = 0;
  let isMuted = false;
  const isPlaying = writable(false);

  let tabContainer;

  let timeSignature = "4/4";

  // 박자 편집용 상태
  // 편집 중 여부
  let editingTimeTop = false;
  let editingTimeBottom = false;

  // 임시 입력값
  let timeTopInput = timeSignature.split('/')[0];
  let timeBottomInput = timeSignature.split('/')[1];

  function setGuitar(type) {
    selectedGuitar = type;
  }

  function setTuning(mode) {
    selectedTuning = mode;
    if (mode === "standard") {
      tuningNotes = ["E4", "B3", "G3", "D3", "A2", "E2"];
      tuning = ["E", "B", "G", "D", "A", "E"];
    } else if (mode === "drop d") {
      tuningNotes = ["D4", "A3", "D3", "G3", "D2", "E2"];
      tuning = ["D", "A", "D", "G", "D", "E"];
    }
  }

  function getNoteFromString(stringNote, fret) {
    const baseNote = stringNote.slice(0, -1);
    const baseOctave = parseInt(stringNote.slice(-1));
    const baseIndex = NOTES.indexOf(baseNote);
    const totalSemitone = baseIndex + fret;
    const note = NOTES[totalSemitone % 12];
    const octave = baseOctave + Math.floor(totalSemitone / 12);
    return `${note}${octave}`;
  }

  function play(note) {
    if (isMuted) return;
    Tone.start();
    const synth = new Tone.Synth().toDestination();
    synth.triggerAttackRelease(note, '8n');
  }

  function resetTab() {
    tab = Array(6).fill().map(() => Array(60).fill(""));
    currentTabPos = 0;
    if (tabContainer) tabContainer.scrollLeft = 0;
  }

  function getLastActiveColumn() {
    for (let col = tab[0].length - 1; col >= 0; col--) {
      for (let i = 0; i < 6; i++) {
        if (tab[i][col] !== "") return col;
      }
    }
    return -1;
  }

  async function playTab() {
    if (get(isPlaying)) return;

    const lastCol = getLastActiveColumn();
    if (lastCol === -1) return;

    isPlaying.set(true);
    await Tone.start();

    let delayMs = 300;
    if (timeSignature === "3/4") delayMs = 400;
    else if (timeSignature === "6/8") delayMs = 200;
    else if (timeSignature === "2/4") delayMs = 350;
    else if (timeSignature === "5/4") delayMs = 300;

    try {
      for (let col = 0; col <= lastCol; col++) {
        if (!get(isPlaying)) break;

        for (let stringIndex = 0; stringIndex < 6; stringIndex++) {
          const fret = tab[stringIndex][col];
          if (fret !== "" && !isNaN(fret)) {
            const stringNote = tuningNotes[5 - stringIndex];
            const note = getNoteFromString(stringNote, parseInt(fret));
            play(note);
          }
        }

        await new Promise(res => setTimeout(res, delayMs));
      }
    } finally {
      isPlaying.set(false);
    }
  }

  function pauseTab() {
    isPlaying.set(false);
  }

  async function handleFretClick(stringIndex, fretIndex) {
    if (stringIndex === 6) return;

    if (currentTabPos + 1 >= tab[0].length) {
      tab = tab.map(row => [...row, "", ""]);
      await tick();
      if (tabContainer) tabContainer.scrollLeft = tabContainer.scrollWidth;
    }

    const stringNote = tuningNotes[5 - stringIndex];
    const note = getNoteFromString(stringNote, fretIndex);
    play(note);
    if (tab[stringIndex]) {
      tab[stringIndex][currentTabPos] = (fretIndex + 1).toString();
    }
    currentTabPos += 2;
  }

  let showTuningOptions = false;
  let showTuningOptions2 = false;

  function toggleTuner() {
    showTuningOptions = !showTuningOptions;
  }

  function toggleTuner2() {
    showTuningOptions2 = !showTuningOptions2;
  }

  function formatFret(n) {
    return n === "" ? "" : n.toString();
  }

  function hasAnyNotes() {
    return tab.some(row => row.some(cell => cell !== ""));
  }

  // 박자 분수 클릭 시 편집 시작
  function startEditTop() {
    editingTimeTop = true;
    timeTopInput = timeSignature.split('/')[0];
  }

  function startEditBottom() {
    editingTimeBottom = true;
    timeBottomInput = timeSignature.split('/')[1];
  }

  // 박자 변경 적용
  function applyTimeSignature() {
    // bottom은 2,4,8,16만 허용
    if (!/^\d+$/.test(timeTopInput)) return; // 숫자 아닌 경우 무시
    if (!["2", "4", "8", "16"].includes(timeBottomInput)) return;

    timeSignature = `${timeTopInput}/${timeBottomInput}`;
    editingTimeTop = false;
    editingTimeBottom = false;
  }

  // 박자 입력 중 엔터 적용
  function onTimeTopKeydown(e) {
    if (e.key === "Enter") {
      if (timeTopInput === "") return;
      applyTimeSignature();
    } else if (e.key === "Escape") {
      editingTimeTop = false;
    }
  }

  function onTimeBottomChange(e) {
    timeBottomInput = e.target.value;
    applyTimeSignature();
  }

  // 편집 중이면 박자 분수 표시 안 함
</script>

<style>
  /* 기존 스타일 그대로 유지 */

  .container {
    height: 100%;
    width: 100%;
    display: flex;
    flex-direction: column;
    background: white;
    margin: 0;
    padding: 0;
    font-family: sans-serif;
  }

  .top-bar {
    display: flex;
    justify-content: space-between;
    padding: 1rem;
  }

  .tab-display {
    background: #ccc;
    padding: 1rem;
    font-family: 'Courier New', Courier, monospace;
    font-size: 14px;
    overflow-x: auto;
    overflow-y: hidden;
    height: 22vh;
    margin: 0 auto;
    white-space: pre;
    box-sizing: border-box;
    max-width: 100%;
  }

  .controls {
    display: flex;
    gap: 0.5rem;
    padding: 0.5rem 1rem;
    align-items: center;
  }

  .fretboard-wrapper {
    flex: 1;
    display: flex;
    overflow: auto;
    align-items: stretch;
    min-height: 0;
  }

  .string-labels {
    display: flex;
    flex-direction: column;
    justify-content: center;
    padding: 0.1rem;
    margin: 0;
    gap: 24px;
  }

  .string-label {
    background: red;
    color: white;
    padding: 0.1rem 0.4rem;
    border-radius: 4px;
    text-align: center;
    font-weight: bold;
    margin-right: 5px;
  }

  .tunerset {
    display: flex;
  }

  .tunerset > button {
    margin: 8px;
  }

  .tunerset > button.selected {
    background-color: #666;
    color: white;
    font-weight: bold;
  }

  .fretboard {
    flex: 1;
    display: grid;
    grid-template-columns: repeat(16, 1fr);
    grid-template-rows: repeat(7, 1fr);
    gap: 3px;
    background: #999;
    height: 350px;
  }

  .fret {
    position: relative;
    background: #ccc;
    border: 1px solid #aaa;
  }

  .dot {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 20px;
    height: 16px;
    background: black;
    border-radius: 50%;
    transform: translate(-50%, -50%);
  }

  .click-zone {
    position: absolute;
    bottom: -10px;
    left: 0;
    width: 100%;
    height: 40%;
    background: transparent;
    cursor: pointer;
  }

  button {
    padding: 0.5rem 1rem;
    border: none;
    border-radius: 10px;
    background: #ddd;
  }

  button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .footer {
    display: flex;
    justify-content: space-between;
  }

  .tab-table {
    border-collapse: collapse;
    font-family: 'Courier New', monospace;
    font-size: 14px;
    table-layout: auto;
    width: max-content;
  }

  .tab-table td {
    padding: 0;
    margin: 0;
    text-align: center;
    white-space: nowrap;
    min-width: 24px;
    height: 20px;
    position: relative;
    z-index: 1;
  }

  .tab-table td::after {
    content: "";
    position: absolute;
    top: 50%;
    left: 0;
    height: 1px;
    width: 100%;
    background-color: black;
    transform: translateY(-50%);
    z-index: 0;
  }

  .tab-table td:not(:empty)::after {
    display: none;
  }

  /* 박자 표시용 셀 스타일 */
  .time-signature-cell {
    font-weight: bold;
    font-size: 16px;
    vertical-align: middle;
    padding: 0 4px;
    user-select: none;
  }

  /* 박자 편집 input 스타일 */
  .time-top-input {
    width: 24px;
    font-size: 16px;
    font-weight: bold;
    text-align: center;
    border: none;
    border-bottom: 1.5px solid black;
    background: transparent;
  }
  .time-top-input:focus {
    outline: none;
  }

  .time-bottom-select {
    font-size: 16px;
    font-weight: bold;
    border: none;
    border-top: 1.5px solid black;
    background: transparent;
    text-align-last: center;
  }
  .time-bottom-select:focus {
    outline: none;
  }

</style>

<div class="container">
  <div class="top-bar">
    <button>guitar site</button>
    <div>
      <button on:click={resetTab}>reset</button>
      <button on:click={pauseTab} disabled={!$isPlaying}>pause</button>
      <button on:click={playTab} disabled={$isPlaying || !hasAnyNotes()}>play</button>
    </div>
  </div>

  <!-- 박자 선택 컨트롤 -->

  <div class="tab-display" bind:this={tabContainer}>
    <table class="tab-table" spellcheck="false">
      <tbody>
        {#each tab as line, i}
          <tr>
            {#if i === 0}
              <!-- 첫 줄 첫 칸에 박자 분수 표시 -->
              <td class="time-signature-cell" rowspan="6" style="border:none; padding-right: 10px;">
                <div style="line-height: 1; text-align:center; cursor:pointer; user-select:none;">
                  {#if editingTimeTop}
                    <input
                      class="time-top-input"
                      type="text"
                      bind:value={timeTopInput}
                      on:keydown={onTimeTopKeydown}
                      on:blur={applyTimeSignature}
                      maxlength="2"
                      autofocus
                    />
                  {:else}
                    <div on:click={startEditTop}>{timeTopInput}</div>
                  {/if}

                  {#if editingTimeBottom}
                    <select
                      class="time-bottom-select"
                      bind:value={timeBottomInput}
                      on:change={onTimeBottomChange}
                      on:blur={applyTimeSignature}
                      autofocus
                    >
                      <option value="2">2</option>
                      <option value="4">4</option>
                      <option value="8">8</option>
                      <option value="16">16</option>
                    </select>
                  {:else}
                    <div style="border-top: 1px solid black; margin-top: 2px;" on:click={startEditBottom}>
                      {timeBottomInput}
                    </div>
                  {/if}
                </div>
              </td>
            {/if}
            {#each line as fret, j}
              <!-- 박자 표시 셀 뒷쪽부터 표시 -->
              {#if j === 0 && i !== 0}
                <!-- 첫 줄의 셀은 이미 위에서 박자 표시 때문에 열 하나 차지했으므로 건너뜀 -->
              {:else}
                <td>{@html formatFret(fret)}</td>
              {/if}
            {/each}
          </tr>
        {/each}
      </tbody>
    </table>
  </div>

  <div class="controls">
    <button on:click={toggleTuner} class="setting">tuner</button>
    {#if showTuningOptions}
      <div class="tunerset">
        <button class:selected={selectedTuning === "standard"} on:click={() => setTuning("standard")}>standard</button>
        <button class:selected={selectedTuning === "drop d"} on:click={() => setTuning("drop d")}>drop d</button>
      </div>
    {/if}
  </div>

  <div class="fretboard-wrapper">
    <div class="string-labels">
      {#each tuning as t}
        <div class="string-label">{t}</div>
      {/each}
    </div>
    <div class="fretboard">
      {#each Array(7) as _, stringIndex}
        {#each Array(16) as _, fretIndex}
          <div class="fret">
            {#if stringIndex < 6}
              <div class="click-zone" on:click={() => handleFretClick(stringIndex, fretIndex)}></div>
            {/if}
            {#if [2, 4, 6, 8, 14].includes(fretIndex) && stringIndex === 3}
              <div class="dot"></div>
            {:else if fretIndex === 11 && (stringIndex === 2 || stringIndex === 4)}
              <div class="dot"></div>
            {/if}
          </div>
        {/each}
      {/each}
    </div>
  </div>

  <div class="footer">
    <div class="controls">
      <button on:click={toggleTuner2} class="setting">guitar</button>
      {#if showTuningOptions2}
        <div class="tunerset">
          <button class:selected={selectedGuitar === 'acoustic'} on:click={() => setGuitar('acoustic')}>acoustic guitar</button>
          <button class:selected={selectedGuitar === 'electric'} on:click={() => setGuitar('electric')}>electric guitar</button>
        </div>
      {/if}
    </div>
    <div>
      <button style="border-radius: 50%;" on:click={() => isMuted = !isMuted}>
        {#if isMuted}
          🔇
        {:else}
          🔊
        {/if}
      </button>
    </div>
  </div>
</div>
