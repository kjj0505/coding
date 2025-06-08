<script>
  import * as Tone from 'tone';

  const NOTES = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];

  let tuningNotes = ["E4", "B3", "G3", "D3", "A2", "E2"];
  let tuning = ["E", "B", "G", "D", "A", "E"];

  let tab = Array(6).fill().map(() => Array(230).fill(""));
  let currentTabPos = 0;
  let isMuted = false;

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

  async function playTab() {
    for (let col = 0; col < 230; col++) {
      for (let stringIndex = 0; stringIndex < 6; stringIndex++) {
        const fret = tab[stringIndex][col];
        if (fret !== "" && !isNaN(fret)) {
          const stringNote = tuningNotes[5 - stringIndex];
          const note = getNoteFromString(stringNote, parseInt(fret));
          play(note);
        }
      }
      await Tone.start();
      await new Promise((res) => setTimeout(res, 300));
    }
  }

  function handleFretClick(stringIndex, fretIndex) {
    if (stringIndex === 6) return; // 마지막 7번째 줄은 무시

    const stringNote = tuningNotes[5 - stringIndex];
    const note = getNoteFromString(stringNote, fretIndex);
    play(note);
    if (tab[stringIndex]) {
      tab[stringIndex][currentTabPos] = fretIndex.toString();
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
  if (n === "") return "----";       // 빈 칸은 4자
  const s = n.toString();
  if (s.length === 1) return `--${s}-`;  // ex: 3 → --3-
  if (s.length === 2) return `-${s}-`;   // ex: 12 → -12-
  return s.slice(0, 4);                 // safety
}
</script>

<style>
  .container { 
    height: 100%;
    width: 100%;
    display: flex;
    flex-direction: column;
    background: white;
    margin: 0;
    padding: 0;
    font-family: sans-serif;
    box-sizing: border-box; 
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
    letter-spacing: 0;
    line-height: 1.4;
    box-sizing: border-box;
    max-width: 100%;
  }
  .tab-line { 
    font-family: inherit;
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
    justify-content: space-between;
    padding: 0.5rem;
    margin: 2px;
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
  }
  .tunerset {
    display: flex;
  }
  .tunerset > button {
    margin: 8px;
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
    font-size: 1rem;
  }
  .footer {
    display: flex;
    justify-content: space-between;
  }
  .setting {
    position: relative; 
  }
</style>

<div class="container">
  <div class="top-bar">
    <button>guitar site</button>
    <div>
      <button on:click={resetTab}>reset</button>
      <button on:click={playTab}>play</button>
    </div>
  </div>

  <div class="tab-display">
    {#each tab as line, i}  
      <div class="tab-line">
        {@html line.map(n => formatFret(n)).join('')}
      </div>
    {/each}
  </div>

  <div class="controls">
    <button on:click={toggleTuner} class="setting">tuner</button>
    {#if showTuningOptions}
      <div class="tunerset">
        <button>standard</button>
        <button>drop d</button>
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
          <button>acoustic guitar</button>
          <button>electric guitar</button>
        </div>
      {/if}
    </div>
    <div>
      <button style="border-radius: 50%;" on:click={() => isMuted = !isMuted}>{isMuted ? '🔇' : '🔊'}</button>
    </div>
  </div>
</div>
