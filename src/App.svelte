<script>
  let theme = 'dark';
  let channelStatus = '—';
  let phaseStatus = '—';
  let micRunning = false;
  let meterFill = '0%';
  let sweepRunning = false;
  let sweepFreq = '—';
  let sweepTime = '—';

  let audioCtx = null;
  let sweepOsc = null;
  let sweepGain = null;
  let sweepTimeout = null;
  let micStream = null;
  let micAnalyser = null;
  let micAnimation = null;

  function getAudioContext() {
    if (!audioCtx) {
      audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    }
    if (audioCtx.state === 'suspended') {
      audioCtx.resume();
    }
    return audioCtx;
  }

  function stopSweep() {
    if (sweepOsc) {
      sweepOsc.stop();
      sweepOsc.disconnect();
      sweepOsc = null;
    }
    if (sweepGain) {
      sweepGain.disconnect();
      sweepGain = null;
    }
    if (sweepTimeout) {
      clearTimeout(sweepTimeout);
      sweepTimeout = null;
    }
    sweepRunning = false;
  }

  function playChannel(channel) {
    const ctx = getAudioContext();
    stopSweep();

    const panner = ctx.createStereoPanner();
    panner.connect(ctx.destination);

    const osc = ctx.createOscillator();
    const gainNode = ctx.createGain();

    osc.connect(gainNode);
    gainNode.connect(panner);

    osc.frequency.value = 440;
    gainNode.gain.setValueAtTime(0.5, ctx.currentTime);
    gainNode.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 1.5);

    if (channel === 'left') {
      panner.pan.value = -1;
      channelStatus = 'Left';
    } else if (channel === 'right') {
      panner.pan.value = 1;
      channelStatus = 'Right';
    } else {
      panner.pan.value = 0;
      channelStatus = 'Both';
    }

    osc.start(ctx.currentTime);
    osc.stop(ctx.currentTime + 1.5);
  }

  function playPhase() {
    const ctx = getAudioContext();
    stopSweep();

    const osc1 = ctx.createOscillator();
    const osc2 = ctx.createOscillator();
    const gain1 = ctx.createGain();
    const gain2 = ctx.createGain();

    osc1.connect(gain1);
    gain1.connect(ctx.destination);
    osc2.connect(gain2);
    gain2.connect(ctx.destination);

    osc1.frequency.value = 200;
    osc2.frequency.value = 200;

    gain1.gain.setValueAtTime(0.5, ctx.currentTime);
    gain2.gain.setValueAtTime(0.5, ctx.currentTime);
    gain1.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 2);
    gain2.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 2);

    osc1.start(ctx.currentTime);
    osc2.start(ctx.currentTime);
    osc1.stop(ctx.currentTime + 2);
    osc2.stop(ctx.currentTime + 2);

    setTimeout(() => {
      phaseStatus = 'In Phase';
    }, 800);
  }

  function toggleMic() {
    if (micRunning) {
      micStream.getTracks().forEach(track => track.stop());
      micStream = null;
      micAnalyser = null;
      cancelAnimationFrame(micAnimation);
      micRunning = false;
      meterFill = '0%';
      return;
    }

    navigator.mediaDevices.getUserMedia({ audio: true }).then(stream => {
      micStream = stream;
      const source = getAudioContext().createMediaStreamSource(stream);
      micAnalyser = getAudioContext().createAnalyser();
      micAnalyser.fftSize = 256;
      source.connect(micAnalyser);

      micRunning = true;
      updateMeter();
    }).catch(() => {
      alert('Microphone access denied');
    });
  }

  function updateMeter() {
    if (!micAnalyser) return;

    const data = new Uint8Array(micAnalyser.frequencyBinCount);
    micAnalyser.getByteFrequencyData(data);

    let sum = 0;
    for (let i = 0; i < data.length; i++) {
      sum += data[i];
    }
    const avg = sum / data.length;
    const percent = Math.min(100, (avg / 128) * 100);

    meterFill = percent + '%';
    micAnimation = requestAnimationFrame(updateMeter);
  }

  function toggleSweep() {
    if (sweepRunning) {
      stopSweep();
      sweepFreq = '—';
      sweepTime = '—';
      return;
    }

    const ctx = getAudioContext();
    const startFreq = 20;
    const endFreq = 20000;
    const duration = 10;
    const steps = 100;
    const stepDuration = duration / steps;
    const freqStep = (endFreq - startFreq) / steps;

    sweepOsc = ctx.createOscillator();
    sweepGain = ctx.createGain();

    sweepOsc.type = 'sine';
    sweepOsc.connect(sweepGain);
    sweepGain.connect(ctx.destination);
    sweepGain.gain.value = 0.3;

    sweepOsc.frequency.setValueAtTime(startFreq, ctx.currentTime);

    let currentStep = 0;
    let startTime = ctx.currentTime;

    sweepOsc.start(ctx.currentTime);
    sweepRunning = true;

    function scheduleNextStep() {
      if (!sweepRunning || currentStep >= steps) {
        stopSweep();
        sweepFreq = 'Done';
        return;
      }

      currentStep++;
      const nextFreq = startFreq + (freqStep * currentStep);
      const nextTime = ctx.currentTime + stepDuration;

      sweepOsc.frequency.setValueAtTime(nextFreq, nextTime);

      sweepFreq = Math.round(nextFreq) + ' Hz';
      sweepTime = ((ctx.currentTime - startTime)).toFixed(1) + 's / ' + duration + 's';

      sweepTimeout = setTimeout(scheduleNextStep, stepDuration * 1000);
    }

    scheduleNextStep();
  }

  function toggleTheme() {
    theme = theme === 'dark' ? 'light' : 'dark';
  }
</script>

<main data-theme={theme}>
  <button class="theme-toggle" on:click={toggleTheme}>
    {theme === 'dark' ? 'Light' : 'Dark'}
  </button>

  <h1>Sound Test</h1>

  <div class="grid">
    <div class="cell">
      <span class="cell-label">Channels</span>
      <div class="btn-row">
        <button class="btn" on:click={() => playChannel('left')}>L</button>
        <button class="btn" on:click={() => playChannel('both')}>L+R</button>
        <button class="btn" on:click={() => playChannel('right')}>R</button>
      </div>
      <span class="status">{channelStatus}</span>
    </div>

    <div class="cell">
      <span class="cell-label">Phase</span>
      <div class="btn-row">
        <button class="btn" on:click={playPhase}>Test</button>
      </div>
      <span class="status">{phaseStatus}</span>
      <span class="cell-sublabel">Check speaker polarity</span>
    </div>

    <div class="cell">
      <span class="cell-label">Volume</span>
      <button class="btn" on:click={toggleMic}>
        {micRunning ? 'Stop' : 'Start'}
      </button>
      <div class="meter">
        <div class="meter-fill" style:width={meterFill}></div>
      </div>
      <span class="cell-sublabel">Microphone input</span>
    </div>

    <div class="cell" style="grid-column: span 3;">
      <span class="cell-label">Frequency Sweep</span>
      <div class="freq-control">
        <div class="btn-row">
          <button class="btn" on:click={toggleSweep}>
            {sweepRunning ? 'Stop' : 'Start Sweep'}
          </button>
        </div>
        <span class="status">{sweepFreq}</span>
        <span class="status-dim">{sweepTime}</span>
      </div>
    </div>
  </div>

  <p class="desc">
    Test your audio setup with different sound tests.<br><br>
    If you hear no sound, check your system volume and ensure the page has permission to play audio.<br><br>
    <span class="highlight">Use headphones for channel and phase tests.</span>
  </p>
</main>

<style>
  :root {
    --bg: #0a0a0a;
    --fg: #ededed;
    --fg-dim: #737373;
    --border: #262626;
    --accent: #fff;
    --accent-dim: #404040;
  }

  [data-theme="light"] {
    --bg: #ffffff;
    --fg: #0a0a0a;
    --fg-dim: #737373;
    --border: #e5e5e5;
    --accent: #000;
    --accent-dim: #cccccc;
  }

  :global(body) {
    margin: 0;
    padding: 0;
    font-family: 'Geist', -apple-system, sans-serif;
    background: var(--bg);
    color: var(--fg);
    min-height: 100vh;
  }

  main {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 48px 24px;
    background: var(--bg);
    color: var(--fg);
  }

  h1 {
    font-size: 14px;
    font-weight: 500;
    color: var(--fg-dim);
    margin-bottom: 24px;
    letter-spacing: 0.5px;
  }

  .theme-toggle {
    position: absolute;
    top: 24px;
    right: 24px;
    background: transparent;
    border: 1px solid var(--border);
    color: var(--fg-dim);
    padding: 8px 12px;
    font-family: 'Geist Mono', monospace;
    font-size: 12px;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .theme-toggle:hover {
    border-color: var(--fg-dim);
    color: var(--fg);
  }

  .grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 1px;
    background: var(--border);
    border: 1px solid var(--border);
    width: 100%;
    max-width: 800px;
  }

  .cell {
    background: var(--bg);
    padding: 32px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 24px;
    min-height: 180px;
  }

  .cell-label {
    font-size: 12px;
    color: var(--fg-dim);
    text-transform: uppercase;
    letter-spacing: 1px;
  }

  .cell-sublabel {
    font-size: 11px;
    color: var(--fg-dim);
    opacity: 0.6;
  }

  .btn {
    font-family: 'Geist Mono', monospace;
    font-size: 13px;
    padding: 10px 20px;
    background: transparent;
    color: var(--fg);
    border: 1px solid var(--border);
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .btn:hover {
    border-color: var(--fg-dim);
    background: var(--fg-dim);
    color: var(--bg);
  }

  .btn.active {
    background: var(--fg);
    color: var(--bg);
    border-color: var(--fg);
  }

  .btn-row {
    display: flex;
    gap: 8px;
  }

  .status {
    font-family: 'Geist Mono', monospace;
    font-size: 24px;
    color: var(--fg);
    min-height: 32px;
  }

  .status-dim {
    color: var(--fg-dim);
    font-size: 14px;
  }

  .meter {
    width: 100%;
    max-width: 200px;
    height: 8px;
    background: var(--border);
    position: relative;
    overflow: hidden;
  }

  .meter-fill {
    height: 100%;
    background: var(--fg);
    width: 0%;
    transition: width 0.05s ease;
  }

  .freq-control {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 16px;
  }

  .desc {
    font-size: 13px;
    color: var(--fg-dim);
    text-align: center;
    max-width: 500px;
    line-height: 1.6;
    margin-top: 32px;
  }

  .highlight {
    color: var(--fg);
  }

  @media (max-width: 600px) {
    main {
      padding: 32px 16px;
    }

    .grid {
      grid-template-columns: 1fr;
    }

    .cell {
      padding: 24px 16px;
      min-height: 140px;
    }

    .btn-row {
      flex-wrap: wrap;
      justify-content: center;
    }

    .btn {
      padding: 12px 16px;
    }
  }
</style>
