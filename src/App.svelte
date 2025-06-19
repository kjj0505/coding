<script>
  import * as Tone from 'tone';
  import { writable, get } from 'svelte/store';

  const NOTES = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];

  let tuningNotes = ["E4", "B3", "G3", "D3", "A2", "E2"];
  let tuning = ["E", "B", "G", "D", "A", "E"];
  let selectedTuning = "standard";
  let selectedGuitar = "acoustic";

  let tab = Array(6).fill().map(() => Array(230).fill("")); 
  let currentTabPos = 0;
  let isMuted = false;
  const isPlaying = writable(false);

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
    tab = Array(6).fill().map(() => Array(230).fill(""));
    currentTabPos = 0;
  }

  function getLastActiveColumn() {
    for (let col = 229; col >= 0; col--) {
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

        await new Promise(res => setTimeout(res, 300));
      } 
    } finally {
      isPlaying.set(false);
    }
  }

  function pauseTab() {
    isPlaying.set(false);
  }

  function handleFretClick(stringIndex, fretIndex) {
    if (stringIndex === 6) return;

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
</script>

<style>
  .container {
    height: 100%; width: 100%; display: flex; flex-direction: column;
    background: white; margin: 0; padding: 0; font-family: sans-serif; box-sizing: border-box;
  }
  .top-bar { display: flex; justify-content: space-between; padding: 1rem; }
  .tab-display {
    background: #ccc; padding: 1rem; font-family: 'Courier New', Courier, monospace;
    font-size: 14px; overflow-x: auto; overflow-y: hidden; height: 22vh; margin: 0 auto;
    white-space: pre; letter-spacing: 0; line-height: 1.4; box-sizing: border-box; max-width: 100%;
  }
  .controls { display: flex; gap: 0.5rem; padding: 0.5rem 1rem; align-items: center; }
  .fretboard-wrapper {
    flex: 1; display: flex; overflow: auto; align-items: stretch; min-height: 0;
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
    display: grid;
    grid-template-rows: repeat(1, 1fr);
    margin-right: 5px;
  }
  .tunerset { display: flex; }
  .tunerset > button { margin: 8px; }
  .tunerset > button.selected { background-color: #666; color: white; font-weight: bold; }
  .fretboard {
    flex: 1; display: grid; grid-template-columns: repeat(16, 1fr);
    grid-template-rows: repeat(7, 1fr); gap: 3px; background: #999; height: 350px;
  }
  .fret {
    position: relative; background: #ccc; border: 1px solid #aaa;
  }
  .dot {
    position: absolute; top: 50%; left: 50%; width: 20px; height: 16px;
    background: black; border-radius: 50%; transform: translate(-50%, -50%);
  }
  .click-zone {
    position: absolute; bottom: -10px; left: 0; width: 100%; height: 40%;
    background: transparent; cursor: pointer;
  }
  button {
    padding: 0.5rem 1rem; border: none; border-radius: 10px; background: #ddd; font-size: 1rem;
  }
  button:disabled {
    opacity: 0.5; cursor: not-allowed;
  }
  .footer { display: flex; justify-content: space-between; }
  .setting { position: relative; }
  .tab-table {
    border-collapse: collapse;
    font-family: 'Courier New', monospace;
    font-size: 14px;
    table-layout: fixed;
    width: 100%;
  }
  .tab-table td {
    padding: 0;
    margin: 0;
    text-align: center;
    white-space: nowrap;
    width: 24px;
    position: relative;
    z-index: 1;
    height: 20px;
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

  <div class="tab-display">
    <table class="tab-table">
      <tbody>
        {#each tab as line}
          <tr>
            {#each line as fret}
              <td>{@html formatFret(fret)}</td>
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
      <button style="border-radius: 50%;" on:click={() => isMuted = !isMuted}>{isMuted ? '🔇' : '🔊'}</button>
    </div>
  </div>
</div>