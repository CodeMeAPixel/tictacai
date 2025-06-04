<script>
  export let onClick, number;
  let hover = false;
</script>

<style>
  .field {
    background: var(--field-bg);
    backdrop-filter: blur(5px);
    cursor: pointer;
    border-radius: 15px;
    display: grid;
    place-items: center;
    transition: all 0.3s ease;
    box-shadow: 0 8px 16px var(--shadow-color);
    transform-style: preserve-3d;
    position: relative;
    overflow: hidden;
    height: 100%;
    width: 100%;
    border: 1px solid var(--border-color);
  }

  .field::after {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: linear-gradient(135deg, rgba(255,255,255,0.1), rgba(255,255,255,0));
    pointer-events: none;
    z-index: 1;
  }

  .field:hover {
    transform: translateY(-5px) scale(1.03);
    box-shadow: 0 15px 35px var(--shadow-color);
    background: var(--hover-bg);
    border-color: var(--border-color);
  }

  .field:active {
    transform: translateY(2px) scale(0.98);
    box-shadow: 0 5px 15px var(--shadow-color);
  }

  .field p {
    font-size: 64px;
    font-weight: bold;
    margin: 0;
    transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
    position: relative;
    z-index: 2;
    color: var(--text-primary);
    text-shadow: 0 0 15px rgba(255, 255, 255, 0.5);
  }

  .field p:empty {
    transform: scale(0);
  }

  .field p:not(:empty) {
    transform: scale(1) rotate(360deg);
  }

  /* X styling with glow effect */
  .field p:not(:empty)[data-value="X"] {
    text-shadow: 0 0 15px rgba(233, 69, 96, 0.7);
    animation: pulse-x 2s infinite alternate;
  }

  /* O styling with glow effect */
  .field p:not(:empty)[data-value="O"] {
    text-shadow: 0 0 15px rgba(0, 119, 255, 0.7);
    animation: pulse-o 2s infinite alternate;
  }

  @keyframes pulse-x {
    0% { text-shadow: 0 0 15px rgba(233, 69, 96, 0.7); }
    100% { text-shadow: 0 0 25px rgba(233, 69, 96, 0.9); }
  }

  @keyframes pulse-o {
    0% { text-shadow: 0 0 15px rgba(0, 119, 255, 0.7); }
    100% { text-shadow: 0 0 25px rgba(0, 119, 255, 0.9); }
  }

  /* Fix for specific browser display */
  @media (max-width: 600px) {
    .field p {
      font-size: 48px;
    }
  }
</style>

<div 
  class="field" 
  on:click={onClick} 
  {number} 
  on:mouseenter={() => hover = true}
  on:mouseleave={() => hover = false}
  style="transform: {hover ? 'scale(1.05)' : 'scale(1)'}">
  <p></p>
</div>
