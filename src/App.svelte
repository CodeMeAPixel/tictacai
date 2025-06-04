<script>
  import Field from "./Field.svelte";
  import { onMount } from 'svelte';
  import { fade, fly, scale } from 'svelte/transition';

  // Game configuration
  let winCombos = [
    [0, 1, 2],
    [3, 4, 5],
    [6, 7, 8],
    [0, 3, 6],
    [1, 4, 7],
    [2, 5, 8],
    [0, 4, 8],
    [2, 4, 6],
  ];

  // Game state
  let arr = [];
  let usersTurn = true;
  let message = "";
  let difficulty = 'Medium'; // Default difficulty level
  let gameMode = 'singleplayer'; // Default game mode
  let currentPlayer = 'X'; // Track current player for multiplayer
  let showHelpModal = false;
  let showSettingsModal = false;
  let player1Name = 'Player 1';
  let player2Name = 'Player 2';
  let soundEnabled = true;
  let confettiEnabled = true;
  let darkTheme = true;
  let statsVisible = false;

  // Game stats
  let stats = {
    wins: 0,
    losses: 0,
    ties: 0,
    totalGames: 0,
    easyWins: 0,
    mediumWins: 0,
    hardWins: 0,
    // Add AI character stats
    aiCharacterStats: {
      balanced: { wins: 0, losses: 0, ties: 0 },
      aggressive: { wins: 0, losses: 0, ties: 0 },
      defensive: { wins: 0, losses: 0, ties: 0 },
      tricky: { wins: 0, losses: 0, ties: 0 }
    }
  };

  // AI Characters with different strategies
  let aiCharacter = 'balanced'; // Default AI character
  const aiCharacters = {
    balanced: { 
      name: 'Balanced',
      description: 'Plays with equal focus on offense and defense',
      icon: '⚖️',
      color: '#4D7CFF'
    },
    aggressive: { 
      name: 'Aggressive',
      description: 'Prioritizes offensive moves to win quickly',
      icon: '🔥',
      color: '#e94560'
    },
    defensive: { 
      name: 'Defensive',
      description: 'Focuses on blocking your winning moves',
      icon: '🛡️',
      color: '#58D68D'
    },
    tricky: { 
      name: 'Tricky',
      description: 'Uses unexpected strategies to confuse opponents',
      icon: '🎭',
      color: '#9B59B6'
    }
  };

  // Background music controls
  let musicEnabled = true; // Default to enabled
  let musicVolume = 0.3; // Initial volume at 30%
  let backgroundMusic;
  
  onMount(() => {
    // Load stats from localStorage
    const savedStats = localStorage.getItem('tictactoe_stats');
    if (savedStats) {
      stats = JSON.parse(savedStats);
    }
    
    // Setup background music - fixed implementation
    backgroundMusic = document.getElementById('background-music');
    if (backgroundMusic) {
      backgroundMusic.volume = musicVolume;
      // We need to unmute the audio programmatically
      backgroundMusic.muted = false;
      
      // Try to play after a short delay to ensure DOM is fully loaded
      setTimeout(() => {
        // Only attempt to play if music is enabled
        if (musicEnabled) {
          const playPromise = backgroundMusic.play();
          
          // Handle autoplay restrictions
          if (playPromise !== undefined) {
            playPromise.catch(err => {
              console.log('Auto-play prevented:', err);
              // Set up a one-time click handler to start music
              const startAudio = () => {
                backgroundMusic.play();
                document.removeEventListener('click', startAudio);
              };
              document.addEventListener('click', startAudio);
            });
          }
        }
      }, 500);
    }
    
    const savedSettings = localStorage.getItem('tictactoe_settings');
    if (savedSettings) {
      const settings = JSON.parse(savedSettings);
      soundEnabled = settings.soundEnabled;
      confettiEnabled = settings.confettiEnabled;
      darkTheme = settings.darkTheme;
      player1Name = settings.player1Name || 'Player 1';
      player2Name = settings.player2Name || 'Player 2';
      aiCharacter = settings.aiCharacter || 'balanced';
      musicEnabled = settings.musicEnabled !== undefined ? settings.musicEnabled : true;
      musicVolume = settings.musicVolume || 0.3;
      
      // Apply music settings
      toggleBackgroundMusic(musicEnabled);
      if (backgroundMusic) {
        backgroundMusic.volume = musicVolume;
      }
      
      // Apply theme from saved settings
      applyTheme();
    }
  });
  
  // Save settings to localStorage
  function saveSettings() {
    localStorage.setItem('tictactoe_settings', JSON.stringify({
      soundEnabled,
      confettiEnabled,
      darkTheme,
      player1Name,
      player2Name,
      aiCharacter,
      musicEnabled,
      musicVolume
    }));
    
    // Apply theme changes immediately
    applyTheme();
    
    showSettingsModal = false;
  }
  
  // Apply theme based on settings
  function applyTheme() {
    if (darkTheme) {
      document.documentElement.removeAttribute('data-theme');
    } else {
      document.documentElement.setAttribute('data-theme', 'light');
    }
  }

  // Update stats with AI character tracking
  function updateStats(result) {
    stats.totalGames++;
    
    if (result === 'win') {
      stats.wins++;
      if (difficulty === 'Easy') stats.easyWins++;
      if (difficulty === 'Medium') stats.mediumWins++;
      if (difficulty === 'Hard') stats.hardWins++;
      // Track by AI character
      if (gameMode === 'singleplayer') {
        stats.aiCharacterStats[aiCharacter].losses++;
      }
    } else if (result === 'loss') {
      stats.losses++;
      // Track by AI character
      if (gameMode === 'singleplayer') {
        stats.aiCharacterStats[aiCharacter].wins++;
      }
    } else if (result === 'tie') {
      stats.ties++;
      // Track by AI character
      if (gameMode === 'singleplayer') {
        stats.aiCharacterStats[aiCharacter].ties++;
      }
    }
    
    localStorage.setItem('tictactoe_stats', JSON.stringify(stats));
  }

  // Calculate win/loss ratio
  function getWinLossRatio() {
    if (stats.losses === 0) return stats.wins > 0 ? '∞' : '0';
    return (stats.wins / stats.losses).toFixed(2);
  }

  // Get win rate percentage
  function getWinRate() {
    if (stats.totalGames === 0) return '0%';
    return Math.round((stats.wins / stats.totalGames) * 100) + '%';
  }

  // Play sound effect
  function playSound(soundName) {
    if (!soundEnabled) return;
    
    const sounds = {
      move: new Audio('/sounds/move.mp3'),
      win: new Audio('/sounds/win.mp3'),
      loss: new Audio('/sounds/loss.mp3'),
      tie: new Audio('/sounds/tie.mp3'),
      click: new Audio('/sounds/click.mp3')
    };
    
    if (sounds[soundName]) {
      sounds[soundName].volume = 0.5;
      sounds[soundName].play().catch(err => console.log('Audio error:', err));
    }
  }

  // Enhance showConfetti function
  function showConfetti() {
    if (!confettiEnabled) return;
    
    if (typeof confetti !== "undefined") {
      // Create a more elaborate confetti effect
      const duration = 3000;
      const end = Date.now() + duration;
      
      const colors = ['#e94560', '#0f3460', '#ff7f50', '#4D7CFF', '#58D68D'];
      
      (function frame() {
        confetti({
          particleCount: 5,
          angle: 60,
          spread: 55,
          origin: { x: 0 },
          colors: colors
        });
        
        confetti({
          particleCount: 5,
          angle: 120,
          spread: 55,
          origin: { x: 1 },
          colors: colors
        });
        
        if (Date.now() < end) {
          requestAnimationFrame(frame);
        }
      }());
      
      // Also fire from the middle for a bigger burst
      setTimeout(() => {
        confetti({
          particleCount: 100,
          spread: 100,
          origin: { y: 0.6 },
          colors: colors
        });
      }, 250);
    }
  }

  const winner = (perc) => {
    if (arr.length >= 5) {
      let arr2 = arr.filter((val, i) => i % 2 == perc);

      for (let array of winCombos) {
        if (array.every((element) => arr2.includes(element))) {
          if (gameMode === 'singleplayer') {
            if (perc == 0) {
              message = "You Won!";
              playSound('win');
              showConfetti(); // Call confetti effect
              updateStats('win');
              
              const game = document.querySelector(".game-frame");
              array.forEach((arr, index) => {
                game.childNodes[arr].style.backgroundColor = "#58D68D";
              });
            } else {
              message = "PC Won!";
              playSound('loss');
              updateStats('loss');
              
              const game = document.querySelector(".game-frame");
              array.forEach((arr, index) => {
                game.childNodes[arr].style.backgroundColor = "#EC7063";
              });
            }
          } else {
            // Multiplayer mode
            if (perc == 0) {
              message = `${player1Name} Won!`;
              playSound('win');
              showConfetti(); // Call confetti effect
              
              const game = document.querySelector(".game-frame");
              array.forEach((arr, index) => {
                game.childNodes[arr].style.backgroundColor = "#58D68D";
              });
            } else {
              message = `${player2Name} Won!`;
              playSound('win');
              showConfetti(); // Call confetti effect
              
              const game = document.querySelector(".game-frame");
              array.forEach((arr, index) => {
                game.childNodes[arr].style.backgroundColor = "#4D7CFF";
              });
            }
          }
          return true;
        }
      }
    }
    return false;
  };

  const userMove = (e) => {
    if (e.target.firstChild.innerHTML == "" && !message) {
      if (gameMode === 'singleplayer') {
        if (usersTurn) {
          // Single player mode - user's turn
          e.target.firstChild.innerHTML = "X";
          e.target.firstChild.dataset.value = "X";
          arr.push(parseInt(e.target.getAttribute("number")));
          playSound('move');
          
          if (!winner(0)) {
            usersTurn = false;
            if (arr.length != 9) {
              pcMove();
            } else {
              message = "Game Tie!";
              playSound('tie');
              updateStats('tie');
            }
          }
        }
      } else {
        // Multiplayer mode
        e.target.firstChild.innerHTML = currentPlayer;
        e.target.firstChild.dataset.value = currentPlayer;
        arr.push(parseInt(e.target.getAttribute("number")));
        playSound('move');
        
        if (!winner(currentPlayer === 'X' ? 0 : 1)) {
          if (arr.length == 9) {
            message = "Game Tie!";
            playSound('tie');
          } else {
            // Switch player
            currentPlayer = currentPlayer === 'X' ? 'O' : 'X';
          }
        }
      }
    }
  };

  const pcMove = () => {
    if (!message) {
      setTimeout(() => {
        if (difficulty === 'Easy') {
          pcMoveEasy();
        } else if (difficulty === 'Medium') {
          pcMoveMedium();
        } else if (difficulty === 'Hard') {
          pcMoveHard();
        }
      }, 500);
    }
  };
  
  const pcMoveEasy = () => {
    let num;
    do {
      num = Math.floor(Math.random() * 9);
    } while (arr.includes(num));

    const game = document.querySelector(".game-frame");
    game.childNodes[num].firstChild.innerHTML = "O";
    game.childNodes[num].firstChild.dataset.value = "O";

    arr.push(num);
    usersTurn = true;
    winner(1);
  };

  const pcMoveMedium = () => {
    let num;
    let arr2 = arr.filter((val, i) => i % 2 == 0);
    let arr3 = arr.filter((val, i) => i % 2 == 1);

    // Apply AI character strategy
    if (aiCharacter === 'aggressive') {
      // Aggressive AI prioritizes its own winning moves over blocking
      
      // First try to find winning move for AI
      for (let array of winCombos) {
        if (
          array.some((element) => arr3.includes(element)) &&
          !array.some((element) => arr2.includes(element))
        ) {
          const findCommon = (a, b) => a.filter((n) => b.indexOf(n) !== -1);
          const common = findCommon(array, arr3);

          const findMissing = (a, b) => a.filter((n) => b.indexOf(n) == -1);
          const missing = findMissing(array, common);

          if (common.length == 2 && !arr.includes(missing[0])) {
            num = missing[0];
            break;
          }
        }
      }
      
      // If no winning move, then look for blocking moves
      if (num === undefined) {
        for (let array of winCombos) {
          if (
            array.some((element) => arr2.includes(element)) &&
            !array.some((element) => arr3.includes(element))
          ) {
            const findCommon = (a, b) => a.filter((n) => b.indexOf(n) !== -1);
            const common = findCommon(array, arr2);

            const findMissing = (a, b) => a.filter((n) => b.indexOf(n) == -1);
            const missing = findMissing(array, common);

            if (common.length == 2 && !arr.includes(missing[0])) {
              num = missing[0];
              break;
            }
          }
        }
      }
    } else if (aiCharacter === 'defensive') {
      // Defensive AI prioritizes blocking opponent's moves
      
      // First try to block player's winning move
      for (let array of winCombos) {
        if (
          array.some((element) => arr2.includes(element)) &&
          !array.some((element) => arr3.includes(element))
        ) {
          const findCommon = (a, b) => a.filter((n) => b.indexOf(n) !== -1);
          const common = findCommon(array, arr2);

          const findMissing = (a, b) => a.filter((n) => b.indexOf(n) == -1);
          const missing = findMissing(array, common);

          if (common.length == 2 && !arr.includes(missing[0])) {
            num = missing[0];
            break;
          }
        }
      }
      
      // If no blocking move, look for own winning moves
      if (num === undefined) {
        for (let array of winCombos) {
          if (
            array.some((element) => arr3.includes(element)) &&
            !array.some((element) => arr2.includes(element))
          ) {
            const findCommon = (a, b) => a.filter((n) => b.indexOf(n) !== -1);
            const common = findCommon(array, arr3);

            const findMissing = (a, b) => a.filter((n) => b.indexOf(n) == -1);
            const missing = findMissing(array, common);

            if (common.length == 2 && !arr.includes(missing[0])) {
              num = missing[0];
              break;
            }
          }
        }
      }
    } else if (aiCharacter === 'tricky') {
      // Tricky AI sometimes makes unexpected moves
      // First check if center is available (position 4)
      if (!arr.includes(4)) {
        num = 4;
      } else {
        // Try to create a fork situation
        const corners = [0, 2, 6, 8];
        const availableCorners = corners.filter(corner => !arr.includes(corner));
        
        if (availableCorners.length > 0 && Math.random() > 0.3) {
          // 70% chance to pick a corner
          num = availableCorners[Math.floor(Math.random() * availableCorners.length)];
        } else {
          // Otherwise, look for regular winning or blocking moves
          for (let array of winCombos) {
            if (
              array.some((element) => arr3.includes(element)) &&
              !array.some((element) => arr2.includes(element))
            ) {
              const findCommon = (a, b) => a.filter((n) => b.indexOf(n) !== -1);
              const common = findCommon(array, arr3);

              const findMissing = (a, b) => a.filter((n) => b.indexOf(n) == -1);
              const missing = findMissing(array, common);

              if (common.length == 2 && !arr.includes(missing[0])) {
                num = missing[0];
                break;
              }
            }
          }
          
          if (num === undefined) {
            for (let array of winCombos) {
              if (
                array.some((element) => arr2.includes(element)) &&
                !array.some((element) => arr3.includes(element))
              ) {
                const findCommon = (a, b) => a.filter((n) => b.indexOf(n) !== -1);
                const common = findCommon(array, arr2);

                const findMissing = (a, b) => a.filter((n) => b.indexOf(n) == -1);
                const missing = findMissing(array, common);

                if (common.length == 2 && !arr.includes(missing[0])) {
                  num = missing[0];
                  break;
                }
              }
            }
          }
        }
      }
    } else {
      // Balanced AI (default)
      for (let array of winCombos) {
        if (
          array.some((element) => arr2.includes(element)) &&
          !array.some((element) => arr3.includes(element))
        ) {
          const findCommon = (a, b) => a.filter((n) => b.indexOf(n) !== -1);
          const common = findCommon(array, arr2);

          const findMissing = (a, b) => a.filter((n) => b.indexOf(n) == -1);
          const missing = findMissing(array, common);

          if (common.length == 2 && !arr.includes(missing[0])) {
            num = missing[0];
            break;
          }
        }
      }
    }

    if (num === undefined) {
      do {
        num = Math.floor(Math.random() * 9);
      } while (arr.includes(num));
    }

    const game = document.querySelector(".game-frame");
    game.childNodes[num].firstChild.innerHTML = "O";
    game.childNodes[num].firstChild.dataset.value = "O";

    arr.push(num);
    usersTurn = true;
    winner(1);
  };

  const minimax = (newBoard, depth, isMaximizing) => {
    let result = checkWinner(newBoard);
    if (result !== null) {
      if (result === "O") return 10 - depth;
      else if (result === "X") return depth - 10;
      else return 0;
    }

    if (isMaximizing) {
      let bestScore = -Infinity;
      for (let i = 0; i < 9; i++) {
        if (!newBoard.includes(i)) {
          newBoard.push(i);
          let score = minimax(newBoard, depth + 1, false);
          newBoard.pop();
          bestScore = Math.max(score, bestScore);
        }
      }
      return bestScore;
    } else {
      let bestScore = Infinity;
      for (let i = 0; i < 9; i++) {
        if (!newBoard.includes(i)) {
          newBoard.push(i);
          let score = minimax(newBoard, depth + 1, true);
          newBoard.pop();
          bestScore = Math.min(score, bestScore);
        }
      }
      return bestScore;
    }
  };

  const checkWinner = (board) => {
    let arr2 = board.filter((val, i) => i % 2 == 0);
    let arr3 = board.filter((val, i) => i % 2 == 1);

    for (let combo of winCombos) {
      if (combo.every((element) => arr2.includes(element))) {
        return "X";
      } else if (combo.every((element) => arr3.includes(element))) {
        return "O";
      }
    }
    return board.length === 9 ? "tie" : null;
  };

  const reset = () => {
    message = "";
    arr = [];
    usersTurn = true;
    currentPlayer = 'X';

    const game = document.querySelector(".game-frame").childNodes;

    game.forEach((el) => {
      el.style.backgroundColor = "";
      el.firstChild.innerHTML = "";
      el.firstChild.dataset.value = "";
    });
    
    playSound('click');
  };
  
  function toggleGameMode() {
    gameMode = gameMode === 'singleplayer' ? 'multiplayer' : 'singleplayer';
    reset();
    playSound('click');
  }
  
  function openHelp() {
    showHelpModal = true;
    playSound('click');
  }
  
  function openSettings() {
    showSettingsModal = true;
    playSound('click');
  }
  
  function closeModal() {
    // Fix modal closing by explicitly setting each modal state to false
    showHelpModal = false;
    showSettingsModal = false;
    statsVisible = false;
    playSound('click');
  }
  
  function toggleStats() {
    statsVisible = !statsVisible;
    playSound('click');
  }

  // Calculate AI win rate directly
  function calculateAIWinRate(aiCharacterId) {
    const aiStat = stats.aiCharacterStats[aiCharacterId];
    const aiTotal = aiStat.wins + aiStat.losses + aiStat.ties;
    return aiTotal > 0 ? Math.round((aiStat.losses / aiTotal) * 100) : 0;
  }
  
  // Toggle background music
  function toggleBackgroundMusic(enabled) {
    if (!backgroundMusic) return;
    
    if (enabled) {
      const playPromise = backgroundMusic.play();
      if (playPromise !== undefined) {
        playPromise.catch(err => console.log('Play error:', err));
      }
    } else {
      backgroundMusic.pause();
    }
  }
  
  // Update background music volume
  function updateMusicVolume() {
    if (backgroundMusic) {
      backgroundMusic.volume = musicVolume;
    }
  }
  
  $: if (musicEnabled && backgroundMusic && backgroundMusic.paused) {
    backgroundMusic.play().catch(err => console.log('Audio error:', err));
  } else if (!musicEnabled && backgroundMusic && !backgroundMusic.paused) {
    backgroundMusic.pause();
  }
  
  $: if (backgroundMusic) {
    backgroundMusic.volume = musicVolume;
  }
</script>

<style>
  main {
    text-align: center;
    padding: 1em;
    max-width: 100%;
    margin: 0 auto;
    font-family: 'Poppins', sans-serif;
    background: var(--bg-primary);
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    position: relative;
    overflow: hidden;
    transition: background 0.3s ease;
  }

  main::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: 
      radial-gradient(circle at 25% 25%, rgba(77, 124, 255, 0.1) 0%, transparent 50%),
      radial-gradient(circle at 75% 75%, rgba(233, 69, 96, 0.1) 0%, transparent 50%);
    pointer-events: none;
  }

  .game-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    width: 100%;
    max-width: 800px;
    margin: 0 auto;
    position: relative;
    z-index: 2;
    padding: 2rem;
    border-radius: 20px;
    background: var(--bg-glass);
    backdrop-filter: blur(10px);
    box-shadow: 0 15px 35px var(--shadow-color);
    border: 1px solid var(--border-color);
    transition: background 0.3s ease, box-shadow 0.3s ease;
  }

  .game-frame {
    width: 100%;
    max-width: 500px;
    aspect-ratio: 1/1;
    margin: 0 auto;
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-template-rows: repeat(3, 1fr);
    gap: 15px;
    perspective: 1000px;
  }

  .title {
    color: var(--text-primary);
    margin-bottom: 20px;
    font-size: 3.5rem;
    font-weight: 700;
    text-shadow: 0 0 15px var(--accent-primary);
    letter-spacing: 2px;
    position: relative;
    display: inline-block;
    transition: color 0.3s ease, text-shadow 0.3s ease;
  }

  .title::after {
    content: '';
    position: absolute;
    bottom: -10px;
    left: 50%;
    transform: translateX(-50%);
    width: 50%;
    height: 3px;
    background: linear-gradient(90deg, transparent, var(--accent-primary), transparent);
    transition: background 0.3s ease;
  }

  .button {
    color: var(--text-primary);
    background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
    padding: 12px 28px;
    border-radius: 50px;
    margin-top: 25px;
    font-size: 1.2rem;
    font-weight: 600;
    cursor: pointer;
    border: none;
    box-shadow: 0 8px 16px var(--shadow-color);
    transition: all 0.3s ease;
    position: relative;
    overflow: hidden;
    z-index: 1;
  }

  .button::before {
    content: '';
    position: absolute;
    top: 0;
    left: -100%;
    width: 100%;
    height: 100%;
    background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
    transition: all 0.5s ease;
    z-index: -1;
  }

  .button:hover::before {
    left: 100%;
  }

  .button:hover {
    transform: translateY(-3px);
    box-shadow: 0 12px 24px rgba(0, 0, 0, 0.4);
  }

  .button:active {
    transform: translateY(2px);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
  }

  /* Game mode selector - updated to include difficulty */
  .mode-selector {
    display: flex;
    background: var(--bg-tertiary);
    border-radius: 50px;
    padding: 5px;
    margin-bottom: 20px;
    width: fit-content;
    margin: 0 auto 20px;
    box-shadow: 0 4px 12px var(--shadow-color);
    border: 1px solid var(--border-color);
    position: relative;
    transition: background 0.3s ease;
  }

  .mode-option {
    padding: 8px 16px;
    cursor: pointer;
    border-radius: 50px;
    transition: all 0.3s ease;
    color: var(--text-primary);
    font-weight: 500;
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .mode-active {
    background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
    box-shadow: 0 4px 8px var(--shadow-color);
  }

  /* Updated turn indicator with theme variables */
  .turn-indicator {
    background: var(--bg-tertiary);
    border-radius: 50px;
    padding: 8px 16px;
    color: var(--text-primary);
    margin-bottom: 15px;
    font-size: 1rem;
    font-weight: 500;
    display: inline-flex;
    align-items: center;
    gap: 8px;
    box-shadow: 0 4px 8px var(--shadow-color);
    border: 1px solid var(--border-color);
    transition: background 0.3s ease, color 0.3s ease;
  }

  /* Player icon styles for turn indicator */
  .player-icon {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    border-radius: 50%;
  }

  .player-x {
    background: rgba(233, 69, 96, 0.2);
    color: #e94560;
  }

  .player-o {
    background: rgba(77, 124, 255, 0.2);
    color: #4D7CFF;
  }

  .highlight {
    color: #e94560;
    font-weight: 600;
  }

  .ai-name {
    background: linear-gradient(90deg, var(--ai-color, #4D7CFF), transparent);
    background-clip: text;
    -webkit-background-clip: text;
    color: transparent;
    font-weight: 600;
  }

  /* Toggle switch styles */
  .toggle {
    position: relative;
    display: inline-block;
    width: 50px;
    height: 24px;
  }

  .toggle input {
    opacity: 0;
    width: 0;
    height: 0;
  }

  .toggle-slider {
    position: absolute;
    cursor: pointer;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(30, 41, 59, 0.6);
    transition: .4s;
    border-radius: 24px;
  }

  .toggle-slider:before {
    position: absolute;
    content: "";
    height: 16px;
    width: 16px;
    left: 4px;
    bottom: 4px;
    background-color: white;
    transition: .4s;
    border-radius: 50%;
  }

  input:checked + .toggle-slider {
    background-color: #4D7CFF;
  }

  input:checked + .toggle-slider:before {
    transform: translateX(26px);
  }

  /* Stats modal styles */
  .stats-container {
    display: flex;
    flex-direction: column;
    gap: 20px;
  }

  .stat-summary {
    display: flex;
    justify-content: space-between;
    background: rgba(30, 41, 59, 0.6);
    border-radius: 15px;
    padding: 15px;
    margin-bottom: 15px;
  }

  .stat-card {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 15px;
    background: rgba(15, 23, 42, 0.8);
    border-radius: 10px;
    border: 1px solid rgba(255, 255, 255, 0.05);
    transition: all 0.3s ease;
    flex: 1;
    margin: 0 5px;
  }

  .stat-card:hover {
    transform: translateY(-5px);
    box-shadow: 0 10px 20px rgba(0, 0, 0, 0.3);
  }

  .stat-value {
    font-size: 2rem;
    font-weight: 700;
    margin: 5px 0;
  }

  .stat-label {
    font-size: 0.9rem;
    color: rgba(255, 255, 255, 0.7);
  }

  .win { color: #58D68D; }
  .loss { color: #e94560; }
  .tie { color: #4D7CFF; }
  .ratio { color: #F4D03F; }
  .rate { color: #9B59B6; }

  .ai-stats-section {
    margin-top: 20px;
  }

  .ai-stats-title {
    font-size: 1.2rem;
    color: #4D7CFF;
    margin-bottom: 15px;
    text-align: left;
  }

  .ai-stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
    gap: 15px;
  }

  .ai-stats-card {
    background: rgba(30, 41, 59, 0.6);
    border-radius: 10px;
    padding: 15px;
    border-left: 4px solid;
    transition: all 0.3s ease;
  }

  .ai-stats-card:hover {
    transform: translateY(-3px);
    box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
  }

  .ai-stats-header {
    display: flex;
    align-items: center;
    margin-bottom: 10px;
  }

  .ai-stats-icon {
    font-size: 1.5rem;
    margin-right: 10px;
  }

  .ai-stats-name {
    font-weight: 600;
    font-size: 1.1rem;
  }

  .ai-stats-data {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 10px;
    text-align: center;
  }

  .ai-stat-item {
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .ai-stat-value {
    font-weight: 700;
    font-size: 1.2rem;
  }

  .ai-stat-label {
    font-size: 0.8rem;
    color: rgba(255, 255, 255, 0.7);
  }

  /* Remove the original difficulty selector */
  .difficulty-selector {
    display: none;
  }

  .message {
    animation: glowPulse 2s infinite alternate;
  }

  @keyframes glowPulse {
    0% {
      text-shadow: 0 0 5px rgba(255, 255, 255, 0.5);
    }
    100% {
      text-shadow: 0 0 20px rgba(255, 255, 255, 0.8), 0 0 30px rgba(233, 69, 96, 0.6);
    }
  }

  /* Grid lines */
  .game-frame::before,
  .game-frame::after {
    content: '';
    position: absolute;
    background: rgba(255, 255, 255, 0.05);
    z-index: 0;
  }

  /* Tech pattern background */
  .tech-pattern {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-image: 
      linear-gradient(rgba(255, 255, 255, 0.03) 1px, transparent 1px),
      linear-gradient(90deg, rgba(255, 255, 255, 0.03) 1px, transparent 1px);
    background-size: 20px 20px;
    pointer-events: none;
    z-index: 1;
  }

  /* Floating particles */
  .particles {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    overflow: hidden;
    z-index: 0;
  }

  /* GitHub Corner */
  .github-corner {
    position: absolute;
    top: 0;
    right: 0;
    z-index: 10;
  }

  .github-corner:hover .octo-arm {
    animation: octocat-wave 560ms ease-in-out;
  }

  .github-corner svg {
    fill: #e94560;
    color: #0f172a;
    position: absolute;
    top: 0;
    border: 0;
    right: 0;
  }

  @keyframes octocat-wave {
    0%, 100% { transform: rotate(0) }
    20%, 60% { transform: rotate(-25deg) }
    40%, 80% { transform: rotate(10deg) }
  }

  /* Open Source Banner */
  .opensource-banner {
    position: fixed;
    bottom: 60px;
    left: 0;
    background: linear-gradient(135deg, #e94560, #0f3460);
    color: white;
    padding: 8px 16px;
    border-radius: 0 8px 8px 0;
    font-size: 0.9rem;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    display: flex;
    align-items: center;
    gap: 8px;
    z-index: 100;
    transition: all 0.3s ease;
    text-decoration: none;
  }

  .opensource-banner:hover {
    transform: translateX(5px);
    box-shadow: 0 6px 16px rgba(0, 0, 0, 0.4);
  }

  .opensource-banner svg {
    width: 20px;
    height: 20px;
  }

  .credit {
    position: absolute;
    top: 20px;
    right: 20px;
    font-size: 0.9rem;
    color: #e94560;
    font-weight: 500;
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
    z-index: 10;
  }

  @media (max-width: 600px) {
    .credit {
      font-size: 0.7rem;
      top: 10px;
      right: 10px;
    }
  }

  @media (max-width: 768px) {
    .game-container {
      padding: 1.5rem;
    }
    
    .game-frame {
      max-width: 350px;
    }
    
    .title {
      font-size: 2.5rem;
    }
    
    .button {
      font-size: 1rem;
      padding: 10px 20px;
    }
    
    .mode-selector {
      flex-wrap: wrap;
      gap: 8px;
      justify-content: center;
      padding: 8px;
    }
    
    .mode-option {
      padding: 8px 12px;
      font-size: 0.9rem;
    }
    
    .difficulty-inline {
      margin-left: 0;
      padding-left: 0;
      border-left: none;
      width: 100%;
      justify-content: center;
      margin-top: 8px;
      border-top: 1px solid rgba(255, 255, 255, 0.2);
      padding-top: 8px;
    }
    
    .github-corner svg {
      width: 60px;
      height: 60px;
    }
    
    .opensource-banner {
      bottom: 70px;
      font-size: 0.8rem;
      padding: 6px 12px;
    }
  }

  /* Button styles */
  .icon-button {
    background: rgba(30, 41, 59, 0.6);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 50%;
    width: 40px;
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    transition: all 0.3s ease;
    margin: 0 5px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  }

  .icon-button:hover {
    background: rgba(77, 124, 255, 0.4);
    transform: translateY(-2px);
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.3);
  }

  .button-bar {
    display: flex;
    justify-content: center;
    margin-top: 20px;
    gap: 10px;
  }

  /* Modal styles */
  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.7);
    backdrop-filter: blur(5px);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
  }

  .modal {
    background: rgba(15, 23, 42, 0.95);
    border-radius: 15px;
    padding: 25px;
    width: 90%;
    max-width: 600px;
    max-height: 85vh;
    overflow-y: auto;
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.5);
    border: 1px solid rgba(255, 255, 255, 0.1);
    position: relative;
  }

  .modal-close {
    position: absolute;
    top: 15px;
    right: 15px;
    background: none;
    border: none;
    color: white;
    font-size: 1.5rem;
    cursor: pointer;
    transition: all 0.3s ease;
  }

  .modal-close:hover {
    color: #e94560;
    transform: rotate(90deg);
  }

  .modal-title {
    font-size: 1.8rem;
    margin-bottom: 20px;
    color: var(--text-primary);
    text-align: center;
    position: relative;
    padding-bottom: 10px;
    transition: color 0.3s ease;
  }

  .modal-title::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 25%;
    width: 50%;
    height: 2px;
    background: linear-gradient(90deg, transparent, rgba(233, 69, 96, 0.7), transparent);
  }

  /* Enhanced settings modal styles */
  .settings-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
    gap: 15px;
    margin-top: 20px;
  }

  .settings-card {
    background: rgba(30, 41, 59, 0.6);
    border-radius: 10px;
    padding: 15px;
    border-left: 4px solid var(--accent-secondary);
    height: 100%;
    transition: all 0.3s ease;
  }

  .settings-card:hover {
    transform: translateY(-3px);
    box-shadow: 0 10px 20px rgba(0, 0, 0, 0.2);
  }

  .settings-card h3 {
    font-size: 1.2rem;
    color: #4D7CFF;
    margin-bottom: 15px;
    display: flex;
    align-items: center;
    position: relative;
  }

  .settings-card h3 svg {
    margin-right: 10px;
  }

  .settings-card h3::after {
    content: '';
    position: absolute;
    bottom: -5px;
    left: 0;
    width: 50px;
    height: 2px;
    background: linear-gradient(90deg, #4D7CFF, transparent);
  }

  /* Enhanced player name inputs */
  .player-input {
    position: relative;
    margin-bottom: 20px;
  }

  .player-input input {
    width: 100%;
    background: rgba(15, 23, 42, 0.5);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 8px;
    padding: 12px 15px;
    padding-left: 40px;
    color: white;
    font-size: 1rem;
    transition: all 0.3s ease;
  }

  .player-input input:focus {
    outline: none;
    border-color: var(--accent-primary);
    box-shadow: 0 0 0 2px rgba(233, 69, 96, 0.2);
    background: rgba(15, 23, 42, 0.8);
  }

  .player-input svg {
    position: absolute;
    left: 12px;
    top: 50%;
    transform: translateY(-50%);
    color: rgba(255, 255, 255, 0.5);
  }

  /* Enhanced toggle switches */
  .toggle-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 15px;
    padding: 8px 10px;
    border-radius: 8px;
    transition: background 0.3s ease;
  }

  .toggle-row:hover {
    background: rgba(255, 255, 255, 0.05);
  }

  .toggle-label {
    display: flex;
    align-items: center;
    gap: 10px;
    color: white;
    font-weight: 500;
  }

  .toggle-description {
    color: rgba(255, 255, 255, 0.6);
    font-size: 0.8rem;
    margin-top: 5px;
  }

  /* Enhanced AI character selection */
  .ai-character-selector {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
    gap: 10px;
    margin-top: 15px;
  }

  .ai-character-option {
    background: rgba(15, 23, 42, 0.5);
    border: 2px solid transparent;
    border-radius: 10px;
    padding: 15px 10px;
    cursor: pointer;
    transition: all 0.3s ease;
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;
  }

  .ai-character-option:hover {
    transform: translateY(-3px);
    background: rgba(30, 41, 59, 0.8);
    box-shadow: 0 8px 16px rgba(0, 0, 0, 0.3);
  }

  .ai-character-selected {
    border-color: currentColor;
    box-shadow: 0 0 15px rgba(255, 255, 255, 0.1);
  }

  .ai-character-icon {
    font-size: 2rem;
    margin-bottom: 8px;
  }

  .ai-character-name {
    font-weight: 600;
    margin-bottom: 5px;
  }

  .ai-character-desc {
    font-size: 0.75rem;
    color: rgba(255, 255, 255, 0.7);
    line-height: 1.3;
  }

  /* Save button enhancement */
  .settings-save-btn {
    background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
    color: white;
    border: none;
    border-radius: 50px;
    padding: 12px 25px;
    font-size: 1.1rem;
    font-weight: 600;
    margin-top: 20px;
    cursor: pointer;
    display: block;
    width: 100%;
    max-width: 200px;
    margin-left: auto;
    margin-right: auto;
    transition: all 0.3s ease;
    position: relative;
    overflow: hidden;
    box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
  }

  .settings-save-btn::before {
    content: '';
    position: absolute;
    top: 0;
    left: -100%;
    width: 100%;
    height: 100%;
    background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
    transition: all 0.5s ease;
  }

  .settings-save-btn:hover::before {
    left: 100%;
  }

  .settings-save-btn:hover {
    transform: translateY(-3px);
    box-shadow: 0 12px 24px rgba(0, 0, 0, 0.3);
  }

  /* Help modal improvements */
  .help-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
    gap: 15px;
    margin-top: 20px;
  }

  .help-card {
    background: rgba(30, 41, 59, 0.6);
    border-radius: 10px;
    padding: 15px;
    border-left: 4px solid #4D7CFF;
    height: 100%;
  }

  .help-card h3 {
    font-size: 1.2rem;
    color: #4D7CFF;
    margin-bottom: 10px;
    display: flex;
    align-items: center;
  }

  .help-card h3 svg {
    margin-right: 8px;
  }

  .help-card p, .help-card ul {
    color: rgba(255, 255, 255, 0.8);
    line-height: 1.6;
  }

  .help-card ul {
    padding-left: 20px;
  }

  .help-card li {
    margin-bottom: 5px;
  }

  .ai-character-help {
    display: flex;
    align-items: center;
    margin-bottom: 8px;
    padding: 6px;
    border-radius: 6px;
    transition: all 0.3s ease;
  }

  .ai-character-help:hover {
    background: rgba(255, 255, 255, 0.05);
  }

  .ai-character-help-icon {
    font-size: 1.2rem;
    margin-right: 8px;
    width: 25px;
    text-align: center;
  }

  .ai-character-help-content {
    flex: 1;
  }

  .ai-character-help-name {
    font-weight: 600;
    margin-bottom: 2px;
  }

  .ai-character-help-desc {
    font-size: 0.8rem;
    color: rgba(255, 255, 255, 0.7);
  }
  
  /* Enhanced select styles */
  .styled-select {
    position: relative;
    display: inline-block;
    width: 100%;
  }
  
  .styled-select select {
    appearance: none;
    -webkit-appearance: none;
    width: 100%;
    padding: 10px 16px;
    padding-right: 40px;
    font-size: 1rem;
    border-radius: 10px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    background: rgba(15, 23, 42, 0.7);
    color: white;
    cursor: pointer;
    outline: none;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    transition: all 0.3s ease;
  }
  
  .styled-select::after {
    content: '';
    position: absolute;
    top: 50%;
    right: 15px;
    transform: translateY(-50%);
    width: 0;
    height: 0;
    border-left: 6px solid transparent;
    border-right: 6px solid transparent;
    border-top: 6px solid rgba(255, 255, 255, 0.7);
    pointer-events: none;
    transition: all 0.3s ease;
  }
  
  .styled-select:hover select {
    background: rgba(30, 41, 59, 0.8);
    border-color: rgba(255, 255, 255, 0.3);
  }
  
  .styled-select:hover::after {
    border-top-color: white;
  }
  
  .styled-select select:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
  
  /* Volume slider styles */
  .volume-slider {
    width: 100%;
    margin-top: 5px;
  }
  
  .volume-slider input[type="range"] {
    -webkit-appearance: none;
    appearance: none;
    width: 100%;
    height: 6px;
    border-radius: 3px;
    background: rgba(15, 23, 42, 0.8);
    outline: none;
    transition: all 0.3s ease;
  }
  
  .volume-slider input[type="range"]::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: #4D7CFF;
    cursor: pointer;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
    transition: all 0.3s ease;
  }
  
  .volume-slider input[type="range"]::-moz-range-thumb {
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: #4D7CFF;
    cursor: pointer;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
    border: none;
    transition: all 0.3s ease;
  }
  
  .volume-slider input[type="range"]:hover::-webkit-slider-thumb {
    background: #e94560;
    transform: scale(1.1);
  }
  
  .volume-slider input[type="range"]:hover::-moz-range-thumb {
    background: #e94560;
    transform: scale(1.1);
  }
  
  .volume-level {
    display: flex;
    justify-content: space-between;
    margin-top: 5px;
    font-size: 0.7rem;
    color: rgba(255, 255, 255, 0.7);
  }

  /* Update the difficulty-inline to use our styled select */
  .difficulty-inline {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-top: 10px;
  }
  
  .difficulty-inline .styled-select {
    width: auto;
  }
</style>

<main>
  <!-- Replace credit div with GitHub corner -->
  <a href="https://github.com/CodeMeAPixel/tictactoe" class="github-corner" aria-label="View source on GitHub" target="_blank" rel="noopener noreferrer">
    <svg width="80" height="80" viewBox="0 0 250 250" aria-hidden="true">
      <path d="M0,0 L115,115 L130,115 L142,142 L250,250 L250,0 Z"></path>
      <path d="M128.3,109.0 C113.8,99.7 119.0,89.6 119.0,89.6 C122.0,82.7 120.5,78.6 120.5,78.6 C119.2,72.0 123.4,76.3 123.4,76.3 C127.3,80.9 125.5,87.3 125.5,87.3 C122.9,97.6 130.6,101.9 134.4,103.2" fill="currentColor" style="transform-origin: 130px 106px;" class="octo-arm"></path>
      <path d="M115.0,115.0 C114.9,115.1 118.7,116.5 119.8,115.4 L133.7,101.6 C136.9,99.2 139.9,98.4 142.2,98.6 C133.8,88.0 127.5,74.4 143.8,58.0 C148.5,53.4 154.0,51.2 159.7,51.0 C160.3,49.4 163.2,43.6 171.4,40.1 C171.4,40.1 176.1,42.5 178.8,56.2 C183.1,58.6 187.2,61.8 190.9,65.4 C194.5,69.0 197.7,73.2 200.1,77.6 C213.8,80.2 216.3,84.9 216.3,84.9 C212.7,93.1 206.9,96.0 205.4,96.6 C205.1,102.4 203.0,107.8 198.3,112.5 C181.9,128.9 168.3,122.5 157.7,114.1 C157.9,116.9 156.7,120.9 152.7,124.9 L141.0,136.5 C139.8,137.7 141.6,141.9 141.8,141.8 Z" fill="currentColor" class="octo-body"></path>
    </svg>
  </a>
  
  <div class="game-container">
    <h1 class="title" class:message={message}>{!message ? 'Tic Tac Toe' : message}</h1>
    
    <!-- Turn indicator with AI character info -->
    {#if !message}
      <div class="turn-indicator">
        {#if gameMode === 'singleplayer'}
          {#if usersTurn}
            <span class="player-icon player-x">X</span>
            <span>Your turn</span>
          {:else}
            <span class="player-icon player-o">O</span>
            <span>
              <span class="ai-name" style="--ai-color: {aiCharacters[aiCharacter].color}">
                {aiCharacters[aiCharacter].icon} {aiCharacters[aiCharacter].name}
              </span> 
              thinking...
            </span>
          {/if}
        {:else}
          <!-- Multiplayer mode -->
          {#if currentPlayer === 'X'}
            <span class="player-icon player-x">X</span>
            <span><span class="highlight">{player1Name}'s</span> turn</span>
          {:else}
            <span class="player-icon player-o">O</span>
            <span><span class="highlight">{player2Name}'s</span> turn</span>
          {/if}
        {/if}
      </div>
    {/if}
    
    <div class="mode-selector">
      <div 
        class="mode-option" 
        class:mode-active={gameMode === 'singleplayer'} 
        on:click={toggleGameMode}
      >
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path>
          <circle cx="12" cy="7" r="4"></circle>
        </svg>
        Singleplayer
      </div>
      
      <div 
        class="mode-option" 
        class:mode-active={gameMode === 'multiplayer'} 
        on:click={toggleGameMode}
      >
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"></path>
          <circle cx="9" cy="7" r="4"></circle>
          <path d="M23 21v-2a4 4 0 0 0-3-3.87"></path>
          <path d="M16 3.13a4 4 0 0 1 0 7.75"></path>
        </svg>
        Multiplayer
      </div>
      
      {#if gameMode === 'singleplayer'}
        <div class="difficulty-inline">
          <div class="styled-select">
            <select bind:value={difficulty} disabled={!!message}>
              <option value="Easy">Easy</option>
              <option value="Medium">Medium</option>
              <option value="Hard">Hard</option>
            </select>
          </div>
        </div>
      {/if}
    </div>
    
    <div class="game-frame">
      {#each Array(9) as field, index}
        <Field onClick={userMove} number={index}/>
      {/each}
    </div>
    
    <button class="button" on:click={reset}>
      {!message ? 'Reset Game' : 'Play Again'}
    </button>
    
    <div class="button-bar">
      <button class="icon-button" on:click={openHelp} title="Help">
        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <circle cx="12" cy="12" r="10"></circle>
          <path d="M9.09 9a3 3 0 0 1 5.83 1c0 2-3 3-3 3"></path>
          <line x1="12" y1="17" x2="12.01" y2="17"></line>
        </svg>
      </button>
      
      <button class="icon-button" on:click={openSettings} title="Settings">
        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <circle cx="12" cy="12" r="3"></circle>
          <path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"></path>
        </svg>
      </button>
      
      <button class="icon-button" on:click={toggleStats} title="Statistics">
        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <line x1="18" y1="20" x2="18" y2="10"></line>
          <line x1="12" y1="20" x2="12" y2="4"></line>
          <line x1="6" y1="20" x2="6" y2="14"></line>
        </svg>
      </button>
    </div>
  </div>

  {#if showHelpModal}
    <div class="modal-overlay" transition:fade>
      <div class="modal" transition:scale={{ start: 0.95 }}>
        <button class="modal-close" on:click={closeModal}>×</button>
        <h2 class="modal-title">How to Play</h2>
        
        <div class="help-grid">
          <div class="help-card">
            <h3>
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                <polyline points="14 2 14 8 20 8"></polyline>
                <line x1="16" y1="13" x2="8" y2="13"></line>
                <line x1="16" y1="17" x2="8" y2="17"></line>
                <polyline points="10 9 9 9 8 9"></polyline>
              </svg>
              Game Rules
            </h3>
            <p>The goal is to be the first to get three of your marks in a row (horizontally, vertically, or diagonally).</p>
            <p>Players take turns putting their marks in empty squares.</p>
          </div>
          
          <div class="help-card">
            <h3>
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"></path>
                <circle cx="9" cy="7" r="4"></circle>
                <path d="M23 21v-2a4 4 0 0 0-3-3.87"></path>
                <path d="M16 3.13a4 4 0 0 1 0 7.75"></path>
              </svg>
              Game Modes
            </h3>
            <ul>
              <li><strong>Singleplayer:</strong> Play against the computer with different difficulty levels and AI characters.</li>
              <li><strong>Multiplayer:</strong> Play with a friend on the same device, taking turns.</li>
            </ul>
          </div>
          
          <div class="help-card">
            <h3>
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <line x1="4" y1="21" x2="4" y2="14"></line>
                <line x1="12" y1="21" x2="12" y2="12"></line>
                <line x1="20" y1="21" x2="20" y2="16"></line>
                <line x1="4" y1="14" x2="4" y2="14"></line>
                <line x1="12" y1="12" x2="12" y2="12"></line>
                <line x1="20" y1="16" x2="20" y2="16"></line>
              </svg>
              Difficulty Levels
            </h3>
            <ul>
              <li><strong>Easy:</strong> The computer makes random moves.</li>
              <li><strong>Medium:</strong> The computer can block your winning moves and find its own.</li>
              <li><strong>Hard:</strong> The computer plays perfectly using the minimax algorithm.</li>
            </ul>
          </div>
          
          <div class="help-card">
            <h3>
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path>
                <circle cx="12" cy="7" r="4"></circle>
              </svg>
              AI Characters
            </h3>
            <p>Choose from different AI personalities with unique strategies:</p>
            
            {#each Object.entries(aiCharacters) as [id, character]}
              <div class="ai-character-help" style="border-left: 3px solid {character.color}; padding-left: 8px;">
                <div class="ai-character-help-icon">{character.icon}</div>
                <div class="ai-character-help-content">
                  <div class="ai-character-help-name" style="color: {character.color}">{character.name}</div>
                  <div class="ai-character-help-desc">{character.description}</div>
                </div>
              </div>
            {/each}
          </div>
        </div>
      </div>
    </div>
  {/if}
  
  {#if showSettingsModal}
    <div class="modal-overlay" transition:fade>
      <div class="modal" transition:scale={{ start: 0.95 }}>
        <button class="modal-close" on:click={closeModal}>×</button>
        <h2 class="modal-title">Settings</h2>
        
        <div class="settings-grid">
          <div class="settings-card">
            <h3>
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"></path>
                <circle cx="9" cy="7" r="4"></circle>
                <path d="M23 21v-2a4 4 0 0 0-3-3.87"></path>
                <path d="M16 3.13a4 4 0 0 1 0 7.75"></path>
              </svg>
              Players
            </h3>
            
            <div class="player-input">
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path>
                <circle cx="12" cy="7" r="4"></circle>
              </svg>
              <input type="text" placeholder="Player 1 Name" bind:value={player1Name}>
            </div>
            
            <div class="player-input">
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path>
                <circle cx="12" cy="7" r="4"></circle>
              </svg>
              <input type="text" placeholder="Player 2 Name" bind:value={player2Name}>
            </div>
          </div>
          
          <div class="settings-card">
            <h3>
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M18 8A6 6 0 0 0 6 8c0 7-3 9-3 9h18s-3-2-3-9"></path>
                <path d="M13.73 21a2 2 0 0 1-3.46 0"></path>
              </svg>
              Effects
            </h3>
            
            <div class="toggle-row">
              <div>
                <div class="toggle-label">
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M3 18v-6a9 9 0 0 1 18 0v6"></path>
                    <path d="M21 19a2 2 0 0 1-2 2h-1a2 2 0 0 1-2-2v-3a2 2 0 0 1 2-2h3zM3 19a2 2 0 0 0 2 2h1a2 2 0 0 0 2-2v-3a2 2 0 0 0-2-2H3z"></path>
                  </svg>
                  Sound Effects
                </div>
                <div class="toggle-description">Play sound effects during the game</div>
              </div>
              <label class="toggle">
                <input type="checkbox" bind:checked={soundEnabled}>
                <span class="toggle-slider"></span>
              </label>
            </div>
            
            <!-- New background music toggle -->
            <div class="toggle-row">
              <div>
                <div class="toggle-label">
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                    <circle cx="12" cy="12" r="10"></circle>
                    <circle cx="12" cy="12" r="4"></circle>
                    <line x1="4.93" y1="4.93" x2="9.17" y2="9.17"></line>
                    <line x1="14.83" y1="14.83" x2="19.07" y2="19.07"></line>
                    <line x1="14.83" y1="9.17" x2="19.07" y2="4.93"></line>
                    <line x1="14.83" y1="9.17" x2="18.36" y2="5.64"></line>
                    <line x1="4.93" y1="19.07" x2="9.17" y2="14.83"></line>
                  </svg>
                  Background Music
                </div>
                <div class="toggle-description">Play ambient background music</div>
              </div>
              <label class="toggle">
                <input type="checkbox" bind:checked={musicEnabled} on:change={() => toggleBackgroundMusic(musicEnabled)}>
                <span class="toggle-slider"></span>
              </label>
            </div>
            
            <!-- Music volume slider -->
            {#if musicEnabled}
              <div style="margin-left: 15px; margin-top: 5px;">
                <div class="toggle-description" style="margin-bottom: 8px;">Music Volume</div>
                <div class="volume-slider">
                  <input 
                    type="range" 
                    min="0" 
                    max="1" 
                    step="0.01" 
                    bind:value={musicVolume} 
                    on:input={updateMusicVolume}
                  >
                  <div class="volume-level">
                    <span>Low</span>
                    <span>High</span>
                  </div>
                </div>
              </div>
            {/if}
            
            <div class="toggle-row">
              <div>
                <div class="toggle-label">
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                    <polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"></polygon>
                  </svg>
                  Confetti Effect
                </div>
                <div class="toggle-description">Show celebration confetti when you win</div>
              </div>
              <label class="toggle">
                <input type="checkbox" bind:checked={confettiEnabled}>
                <span class="toggle-slider"></span>
              </label>
            </div>
            
            <!-- Dark theme toggle -->
            <div class="toggle-row">
              <div>
                <div class="toggle-label">
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                    <circle cx="12" cy="12" r="5"></circle>
                    <line x1="12" y1="1" x2="12" y2="3"></line>
                    <line x1="12" y1="21" x2="12" y2="23"></line>
                    <line x1="4.22" y1="4.22" x2="5.64" y2="5.64"></line>
                    <line x1="18.36" y1="18.36" x2="19.78" y2="19.78"></line>
                    <line x1="1" y1="12" x2="3" y2="12"></line>
                    <line x1="21" y1="12" x2="23" y2="12"></line>
                    <line x1="4.22" y1="19.78" x2="5.64" y2="18.36"></line>
                    <line x1="18.36" y1="5.64" x2="19.78" y2="4.22"></line>
                  </svg>
                  Dark Theme
                </div>
                <div class="toggle-description">Use dark color scheme</div>
              </div>
              <label class="toggle">
                <input type="checkbox" bind:checked={darkTheme}>
                <span class="toggle-slider"></span>
              </label>
            </div>
          </div>
        </div>
        
        <div class="settings-card" style="margin-top: 15px;">
          <h3>
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path>
              <circle cx="12" cy="7" r="4"></circle>
            </svg>
            AI Personality
          </h3>
          
          <p style="color: rgba(255, 255, 255, 0.7); margin-bottom: 10px;">Choose an AI opponent with a unique playing style:</p>
          
          <div class="ai-character-selector">
            {#each Object.entries(aiCharacters) as [id, character]}
              <div 
                class="ai-character-option" 
                class:ai-character-selected={aiCharacter === id}
                style="color: {character.color}"
                on:click={() => aiCharacter = id}
              >
                <div class="ai-character-icon">{character.icon}</div>
                <div class="ai-character-name">{character.name}</div>
                <div class="ai-character-desc">{character.description}</div>
              </div>
            {/each}
          </div>
        </div>
        
        <button class="settings-save-btn" on:click={saveSettings}>
          Save Settings
        </button>
      </div>
    </div>
  {/if}

  {#if statsVisible}
    <div class="modal-overlay" transition:fade>
      <div class="modal" transition:scale={{ start: 0.95 }}>
        <button class="modal-close" on:click={closeModal}>×</button>
        <h2 class="modal-title">Game Statistics</h2>
        
        <div class="stats-container">
          <!-- Summary Stats -->
          <div class="stat-summary">
            <div class="stat-card">
              <div class="stat-value win">{stats.wins}</div>
              <div class="stat-label">Wins</div>
            </div>
            
            <div class="stat-card">
              <div class="stat-value loss">{stats.losses}</div>
              <div class="stat-label">Losses</div>
            </div>
            
            <div class="stat-card">
              <div class="stat-value tie">{stats.ties}</div>
              <div class="stat-label">Ties</div>
            </div>
            
            <div class="stat-card">
              <div class="stat-value ratio">{getWinLossRatio()}</div>
              <div class="stat-label">W/L Ratio</div>
            </div>
            
            <div class="stat-card">
              <div class="stat-value rate">{getWinRate()}</div>
              <div class="stat-label">Win Rate</div>
            </div>
          </div>
          
          <!-- AI Character Stats -->
          <div class="ai-stats-section">
            <h3 class="ai-stats-title">Performance Against AI Characters</h3>
            <div class="ai-stats-grid">
              {#each Object.entries(aiCharacters) as [id, character]}
                {#if stats.aiCharacterStats[id]}
                  <div class="ai-stats-card" style="border-color: {character.color}">
                    <div class="ai-stats-header">
                      <div class="ai-stats-icon">{character.icon}</div>
                      <div class="ai-stats-name" style="color: {character.color}">{character.name}</div>
                    </div>
                    
                    <div class="ai-stats-data">
                      <div class="ai-stat-item">
                        <div class="ai-stat-value win">{stats.aiCharacterStats[id].losses}</div>
                        <div class="ai-stat-label">You Won</div>
                      </div>
                      
                      <div class="ai-stat-item">
                        <div class="ai-stat-value loss">{stats.aiCharacterStats[id].wins}</div>
                        <div class="ai-stat-label">AI Won</div>
                      </div>
                      
                      <div class="ai-stat-item">
                        <div class="ai-stat-value tie">{stats.aiCharacterStats[id].ties}</div>
                        <div class="ai-stat-label">Ties</div>
                      </div>
                    </div>
                    
                    <div style="margin-top: 10px; text-align: center;">
                      <div class="ai-stat-label">Your Win Rate</div>
                      <div class="ai-stat-value rate">{calculateAIWinRate(id)}%</div>
                    </div>
                  </div>
                {/if}
              {/each}
            </div>
          </div>
          
          <!-- Difficulty Stats -->
          <div class="ai-stats-section">
            <h3 class="ai-stats-title">Wins by Difficulty</h3>
            <div class="ai-stats-grid" style="grid-template-columns: repeat(3, 1fr);">
              <div class="ai-stats-card" style="border-color: #58D68D">
                <div class="ai-stats-header">
                  <div class="ai-stats-icon">🌱</div>
                  <div class="ai-stats-name" style="color: #58D68D">Easy</div>
                </div>
                <div class="ai-stat-value win" style="text-align: center; font-size: 2.5rem; margin: 10px 0;">
                  {stats.easyWins}
                </div>
              </div>
              
              <div class="ai-stats-card" style="border-color: #F4D03F">
                <div class="ai-stats-header">
                  <div class="ai-stats-icon">🔄</div>
                  <div class="ai-stats-name" style="color: #F4D03F">Medium</div>
                </div>
                <div class="ai-stat-value win" style="text-align: center; font-size: 2.5rem; margin: 10px 0;">
                  {stats.mediumWins}
                </div>
              </div>
              
              <div class="ai-stats-card" style="border-color: #e94560">
                <div class="ai-stats-header">
                  <div class="ai-stats-icon">🔥</div>
                  <div class="ai-stats-name" style="color: #e94560">Hard</div>
                </div>
                <div class="ai-stat-value win" style="text-align: center; font-size: 2.5rem; margin: 10px 0;">
                  {stats.hardWins}
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  {/if}
  
  <div class="tech-pattern"></div>
</main>

<audio id="background-music" loop>
  <source src="/sounds/background-music.mp3" type="audio/mpeg">
  Your browser does not support the audio element.
</audio>