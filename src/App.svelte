<script>
  let tuning = ["D", "A", "D", "G", "B", "E"];
  let notes = [
    { string: 1, fret: 3 },
    { string: 2, fret: 2 },
    { string: 3, fret: 0 },
    { string: 4, fret: 0 },
    { string: 5, fret: 0 },
    { string: 6, fret: 0 },
  ];
</script>

<style>
  html, body {
    overflow: hidden;
  }

  *, *::before, *::after {
    box-sizing: inherit;
  }

  .container {
    height: 100vh;
    width: 100%;
    overflow: hidden;
    display: flex;
    flex-direction: column;
    background: white;

    margin: 0;
    padding: 0;
    height: 100vh;
    width: 100vw;
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
    overflow-y: auto;
    max-height: 20vh;
    height: 20vh;
  }

  .controls {
    display: flex;
    gap: 0.5rem;
    padding: 0.5rem 1rem;
  }

  .fretboard-wrapper {
    flex: 1;
    display: flex;
    overflow: auto;
    align-items: stretch;
  }

  .string-labels {
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    padding: 0.5rem;
  }

  .string-label {
    background: red;
    color: white;
    padding: 0.2rem 0.4rem;
    border-radius: 4px;
    text-align: center;
    font-weight: bold;
  }

  .tunerset{
    display: flex;
  }

  .tunerset > div{
    margin: 10px;
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
    background: #ccc;
    border: 1px solid #aaa;
  }

  .dot {
    background: black;
    border-radius: 50%;
    width: 16px;
    height: 16px;
    margin: auto;
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
  }

  .guitarset{
    display: flex;
  }

  .guitarset > div{
    margin: 10px;
  }
  
  .setting{

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
    </div>

  <div class="controls">
    <button class="setting">tuner</button>
    <div class="tunerset">
      <div>standard</div>
      <div>drop d</div>
    </div>
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
          <div class="fret">
            {#if notes.find(n => n.string === 6 - stringIndex && n.fret === fretIndex)}
              <div class="dot"></div>
            {/if}
          </div>
        {/each}
      {/each}
    </div>
  </div>

  <div class="footer">
    <button class="setting">guitar</button>
    <div class="guitarset">
      <div>acoustic guitar</div>
      <div>electric guitar</div>
    </div>
  </div>

    <div>
      <button style="border-radius: 50%;">&#128264;</button>
    </div>
  
</div>