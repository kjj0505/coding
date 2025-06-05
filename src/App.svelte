<script>
  import * as Tone from 'tone';

  const NOTES = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];

  // 줄의 음과 프렛 번호로 전체 음 이름을 구하는 함수
  let tuningNotes = ["E4", "B3", "G3", "D3", "A2", "E2"];

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
    Tone.start();
    const synth = new Tone.Synth().toDestination();
    synth.triggerAttackRelease(note, '8n');
  }

  function handleFretClick(stringIndex, fretIndex) {
  const stringNote = tuningNotes[5 - stringIndex]; // ← 고음줄을 위에 배치한 것처럼 처리
  const note = getNoteFromString(stringNote, fretIndex);
  play(note);
}



  

  let showTuningOptions = false;
  let showTuningOptions2 = false;

  function toggleTuner() {
    showTuningOptions = !showTuningOptions;
  }

  function toggleTuner2() {
    showTuningOptions2 = !showTuningOptions2;
  }


  let tuning = ["E", "B", "G", "D", "A", "E"];
  let tuning2 = ["D", "A", "D", "G", "D", "E"]
  let notes = [
    { string: 1, fret: 3 },
    { string: 2, fret: 2 },
    { string: 3, fret: 0 },
    { string: 4, fret: 0 },
    { string: 5, fret: 0 },
    { string: 6, fret: 0 },
  ];

  let tab = Array(5).fill().map(() => Array(230).fill("")); 
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
    font-family: monospace;
    overflow-y: hidden; 
    height: 25vh;
    white-space: pre;
    margin: 0 auto;
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
    grid-template-rows: repeat(1, 1fr); /* fretboard와 동일하게 */
  }

  .tunerset{
    display: flex;
  }

  .tunerset > button{
    margin: 8px;
  }

  .fretboard {
    flex: 1;
    display: grid;
    grid-template-columns: repeat(16, 1fr);
    grid-template-rows: repeat(6, 1fr);
    gap: 2px;
    background: #999;
    height: 300px;
  }

  .fret {
    position: relative;   /* .dot이 위치 기준 삼도록 */
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

  button {
    padding: 0.5rem 1rem;
    border: none;
    border-radius: 10px;
    background: #ddd;
    font-size: 1rem;
  }

  .footer{
    display: flex;
    justify-content: space-between;
  }

  .setting{
    position: relative;
  }
</style>

<div class="container">
  <div class="top-bar">
    <button>guitar site</button>
    <div>
      <button>reset</button>
      <button>play</button>
    </div>
  </div>

  <div class="tab-display">
    {#each tab as line, i}
    <div class="tab-line" style="margin-bottom: {i === 4 ? '1rem' : '0'}">
      {@html line.map(n => n || '-').join('')}
    </div>
  {/each}

    {#each tab as line, i}
    <div class="tab-line" style="margin-bottom: {i === 4 ? '1rem' : '0'}">
      {@html line.map(n => n || '-').join('')}
    </div>
  {/each}
  </div>

  <div class="controls">
    <!-- <div class="group"> -->
      <button on:click="{toggleTuner}" class="setting">tuner</button>
      {#if showTuningOptions}
        <div class="tunerset">
          <button>standard</button>
          <button>drop d</button>
        </div>
      {/if}
    <!-- </div> -->
  </div>

  <div class="fretboard-wrapper">
    <div class="string-labels">
      {#each tuning as t}
        <div class="string-label">{t}</div>
      {/each}
    </div>
    <div class="fretboard">
  {#each Array(6) as _, stringIndex}
    {#each Array(16) as _, fretIndex}
      <div class="fret" on:click={() => handleFretClick(stringIndex, fretIndex)}>
        {#if [2, 4, 6, 8, 14].includes(fretIndex) && stringIndex === 2}
          <div class="dot"></div>
        {:else if fretIndex === 11 && (stringIndex === 1 || stringIndex === 3)}
          <div class="dot"></div>
        {/if}
      </div>
    {/each}
  {/each}
</div>

</div>

  <div class="footer">
    <div class="controls">
      <button on:click="{toggleTuner2}" class="setting">guitar</button>
    {#if showTuningOptions2}
      <div class="tunerset">
        <button>acoistic guitar</button>
        <button>electric guitar</button>
      </div>
    {/if}
    </div>  

    <div>
      <button style="border-radius: 50%;">&#128264;</button>
    </div>
  </div>

    
  
</div>