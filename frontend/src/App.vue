<template>
  <div id="app" class="casino-theme">
    <div class="casino-lights top"></div>
    <div class="casino-lights bottom"></div>
    <h1 class="neon-text">The Great Bingo North </h1>
    <div class="game-controls">
      <button @click="startGame">Start New Game</button>
      <button @click="callNumber" :disabled="gameOver || autoCallActive || bingoAchieved">Call Number</button>
      <button @click="generateNewCards" :disabled="bingoAchieved || generatingCards" :class="{'generating': generatingCards}">
        <span v-if="generatingCards">Shuffling...</span>
        <span v-else>Mix numbers</span>
      </button>
      <button v-if="!autoCallActive" @click="startAutoCall" :disabled="gameOver || bingoAchieved">Auto Call</button>
      <button v-if="autoCallActive" @click="stopAutoCall" class="stop-btn">Stop Auto Call</button>
    </div>
    
    <div v-if="!autoCallActive" class="speed-control">
      <label for="call-speed">Call Speed: {{ callSpeed }}s</label>
      <input type="range" id="call-speed" v-model="callSpeed" min="1" max="10" step="1">
    </div>
    
    <div v-if="lastCalledNumber" class="last-called-number">
      Last Called: <span class="number">{{ lastCalledNumber }}</span>
      <div class="call-counter">{{ calledNumbers.length }} / 75 numbers called</div>
    </div>
    
    <!-- Card generation message -->
    <div v-if="showCardMessage" class="card-message">
      <div class="card-message-content">
        {{ cardMessageText }}
        <div v-if="generatingCards" class="shuffle-animation">
          <div class="shuffle-card"></div>
          <div class="shuffle-card"></div>
          <div class="shuffle-card"></div>
        </div>
      </div>
    </div>
    
    <!-- Card generation message -->
    <div v-if="showCardMessage" class="card-message">
      <div class="card-message-content">
        {{ cardMessageText }}
        <div v-if="generatingCards" class="shuffle-animation">
          <div class="shuffle-card"></div>
          <div class="shuffle-card"></div>
          <div class="shuffle-card"></div>
        </div>
      </div>
    </div>
    
    <div class="players">
      <div v-for="(player, index) in players" :key="index" class="player-container">
        <div class="player-controls">
          <input v-model="player.name" class="player-name-input" :placeholder="`Player ${index + 1}`" />
          <div class="color-picker">
            <label>Card Color:</label>
            <select v-model="player.cardColor" class="color-select">
              <option value="#dc3545">Red</option>
              <option value="#28a745">Green</option>
              <option value="#007bff">Blue</option>
              <option value="#fd7e14">Orange</option>
              <option value="#6f42c1">Purple</option>
              <option value="#20c997">Teal</option>
            </select>
          </div>
        </div>
        <BingoCard
          :playerName="player.name"
          :cardData="player.cardData"
          :calledNumbers="calledNumbers"
          :lastCalledNumber="lastCalledNumber"
          :cardColor="player.cardColor"
          @bingo="onPlayerBingo"
        />
      </div>
    </div>
    
    <div class="called-numbers">
      <h3>Called Numbers:</h3>
      <div class="number-grid">
        <div v-for="num in 75" :key="num" class="number-cell" :class="{ 'called': calledNumbers.includes(num) }">
          {{ num }}
        </div>
      </div>
    </div>
    <!-- Audio elements for sound effects -->
    <audio ref="bingoSound" src="https://assets.mixkit.co/active_storage/sfx/2013/2013-preview.mp3" preload="auto"></audio>
    <!-- Fallback silent audio for Safari initialization -->
    <audio ref="silentSound" src="data:audio/mpeg;base64,SUQzBAAAAAABEVRYWFgAAAAtAAADY29tbWVudABCaWdTb3VuZEJhbmsuY29tIC8gTGFTb25vdGhlcXVlLm9yZwBURU5DAAAAHQAAA1N3aXRjaCBQbHVzIMKpIE5DSCBTb2Z0d2FyZQBUSVQyAAAABgAAAzIyMzUAVFNTRQAAAA8AAANMYXZmNTcuODMuMTAwAAAAAAAAAAAAAAD/80DEAAAAA0gAAAAATEFNRTMuMTAwVVVVVVVVVVVVVUxBTUUzLjEwMFVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVf/zQsRbAAADSAAAAABVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVf/zQMSkAAADSAAAAABVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV" preload="auto"></audio>
    
    <!-- Audio context initialization message for mobile -->
    <div v-if="showAudioPrompt" class="audio-prompt">
      <div class="audio-prompt-content">
        <h3>Enable Audio</h3>
        <p>Tap the button below to enable sound effects and number announcements.</p>
        <button @click="initializeAudio" class="enable-audio-btn">Enable Audio</button>
      </div>
    </div>
  </div>
</template>

<script>
import BingoCard from './components/BingoCard.vue'

export default {
  name: 'App',
  components: {
    BingoCard
  },
  data() {
    return {
      lastCalledNumber: null,
      gameOver: false,
      bingoAchieved: false,
      autoCallActive: false,
      callSpeed: 3,
      autoCallInterval: null,
      audioInitialized: false,
      showAudioPrompt: true,
      audioContext: null,
      isSafari: false,
      safariAudioInitialized: false,
      audioInitAttempts: 0,
      generatingCards: false,
      showCardMessage: false,
      cardMessageText: '',
      players: this.loadFromLocalStorage('bingoPlayers', [
        {
          name: "Player 1",
          cardColor: "#dc3545",
          cardData: [61, 5, 9, 29, 30, 63, 11, 40, 31, 39, 53, 7, 75, 43, 12]
        },
        {
          name: "Player 2",
          cardColor: "#28a745",
          cardData: [24, 26, 65, 4, 74, 37, 75, 18, 25, 46, 62, 22, 42, 60, 33]
        },
        {
          name: "Player 3",
          cardColor: "#007bff",
          cardData: [16, 20, 15, 60, 70, 8, 63, 14, 28, 33, 35, 19, 30, 9, 45]
        },
        {
          name: "Player 4",
          cardColor: "#fd7e14",
          cardData: [62, 61, 24, 7, 66, 55, 71, 64, 57, 75, 51, 44, 2, 17, 38]
        },
        {
          name: "Player 5",
          cardColor: "#6f42c1",
          cardData: [20, 56, 49, 61, 33, 22, 48, 4, 7, 44, 9, 40, 12, 36, 54]
        }
      ]),
      calledNumbers: this.loadFromLocalStorage('bingoCalledNumbers', []),
      allNumbers: Array.from({ length: 75 }, (_, i) => i + 1)
    }
  },
  methods: {
    startGame() {
      this.calledNumbers = [];
      this.lastCalledNumber = null;
      this.gameOver = false;
      this.bingoAchieved = false;
      this.stopAutoCall();
      // Save empty calledNumbers to localStorage
      this.saveToLocalStorage('bingoCalledNumbers', this.calledNumbers);
    },
    callNumber() {
      if (this.calledNumbers.length < 75) {
        let number;
        do {
          number = Math.floor(Math.random() * 75) + 1;
        } while (this.calledNumbers.includes(number));
        this.calledNumbers.push(number);
        this.lastCalledNumber = number;
        // Save called numbers to localStorage
        this.saveToLocalStorage('bingoCalledNumbers', this.calledNumbers);
        
        // Announce the number using speech synthesis
        this.announceNumber(number);
        
        if (this.calledNumbers.length >= 75) {
          this.gameOver = true;
        }
      } else {
        this.gameOver = true;
      }
    },
    generateNewCards() {
      // Set generating state to true for visual feedback
      this.generatingCards = true;
      
      // Show a message that cards are being shuffled
      this.showCardMessage = true;
      this.cardMessageText = 'Shuffling cards...';
      
      // Use setTimeout to create a visual delay and allow UI to update
      setTimeout(() => {
        // Generate all cards at once to ensure no number repeats across cards
        const allCards = this.generateAllBingoCards(this.players.length);
        
        // Create a visual shuffling effect by updating cards with a delay
        this.players.forEach((player, index) => {
          // Stagger the updates for visual effect
          setTimeout(() => {
            player.cardData = allCards[index];
            console.log(`Player ${index + 1} card has ${player.cardData.length} numbers`);
            
            // If this is the last player, finish the process
            if (index === this.players.length - 1) {
              // Update the message
              this.cardMessageText = 'New cards ready!';
              
              // Save players to localStorage
              this.saveToLocalStorage('bingoPlayers', this.players);
              
              // Reset generating state after a short delay
              setTimeout(() => {
                this.generatingCards = false;
                
                // Hide the message after a delay
                setTimeout(() => {
                  this.showCardMessage = false;
                }, 1500);
              }, 500);
            }
          }, index * 200); // Stagger each player's card update
        });
      }, 600); // Initial delay for visual feedback
    },
    generateAllBingoCards(numCards) {
      // Traditional bingo has 5 columns (B, I, N, G, O) with specific number ranges
      const columnRanges = [
        { min: 1, max: 15 },    // B column: 1-15
        { min: 16, max: 30 },   // I column: 16-30
        { min: 31, max: 45 },   // N column: 31-45
        { min: 46, max: 60 },   // G column: 46-60
        { min: 61, max: 75 }    // O column: 61-75
      ];
      
      const cards = [];
      // Initialize a Set to track all used numbers across all cards
      const allUsedNumbers = new Set();
      
      // For each card
      for (let cardIndex = 0; cardIndex < numCards; cardIndex++) {
        // Create an array to hold 15 numbers (3 from each column)
        const card = [];
        // Track numbers used in this card to prevent duplicates
        const usedInCard = new Set();
        
        // For each column, select 3 unique numbers
        for (let colIndex = 0; colIndex < 5; colIndex++) {
          const range = columnRanges[colIndex];
          
          // Create an array of all available numbers in this column's range
          const availableNumbers = [];
          for (let num = range.min; num <= range.max; num++) {
            if (!usedInCard.has(num) && !allUsedNumbers.has(num)) {
              availableNumbers.push(num);
            }
          }
          
          // Shuffle the available numbers
          this.shuffleArray(availableNumbers);
          
          // Take 3 numbers for this column
          for (let i = 0; i < 3; i++) {
            if (availableNumbers.length > 0) {
              const num = availableNumbers.pop();
              card.push(num);
              usedInCard.add(num);
              allUsedNumbers.add(num); // Add to the set of all used numbers
            } else {
              // This should never happen since we have 15 numbers in each column
              // and we're only taking 3, but just in case:
              console.error("Ran out of numbers in column", colIndex);
              
              // Find any unused number in the entire range
              for (let n = 1; n <= 75; n++) {
                if (!usedInCard.has(n) && !allUsedNumbers.has(n)) {
                  card.push(n);
                  usedInCard.add(n);
                  allUsedNumbers.add(n);
                  break;
                }
              }
            }
          }
        }
        
        cards.push(card);
      }
      
      return cards;
    },
    shuffleArray(array) {
      // Fisher-Yates shuffle algorithm
      for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
      }
      return array;
    },
    startAutoCall() {
      if (!this.autoCallActive && !this.gameOver) {
        this.autoCallActive = true;
        this.autoCallInterval = setInterval(() => {
          if (this.gameOver) {
            this.stopAutoCall();
            return;
          }
          this.callNumber();
        }, this.callSpeed * 1000);
      }
    },
    stopAutoCall() {
      if (this.autoCallInterval) {
        clearInterval(this.autoCallInterval);
        this.autoCallInterval = null;
      }
      this.autoCallActive = false;
    },
    onPlayerBingo(playerName) {
      console.log(`BINGO EVENT RECEIVED for player: ${playerName}`);
      
      // Set bingo achieved flag to disable further calls
      this.bingoAchieved = true;
      
      // Stop auto call if active
      if (this.autoCallActive) {
        this.stopAutoCall();
      }
      
      // Play bingo sound if audio is initialized
      if (this.audioInitialized) {
        try {
          const bingoSound = this.$refs.bingoSound;
          bingoSound.currentTime = 0;
          
          // For Safari, we need to resume the AudioContext again
          if (this.isSafari && this.audioContext) {
            this.audioContext.resume()
              .then(() => {
                bingoSound.play()
                  .catch(e => console.error('Error playing bingo sound:', e));
              })
              .catch(e => console.error('Error resuming AudioContext:', e));
          } else {
            bingoSound.play()
              .catch(e => console.error('Error playing bingo sound:', e));
          }
        } catch (e) {
          console.error('Error preparing bingo sound:', e);
        }
        
        // Announce bingo using speech synthesis
        if ('speechSynthesis' in window) {
          // Cancel any ongoing speech
          window.speechSynthesis.cancel();
          
          // Create a more natural, conversational announcement
          const phrases = [
            `Wow! ${playerName} just got Bingo! Congratulations!`,
            `Amazing! ${playerName} has Bingo! Well played!`,
            `Oh my goodness! ${playerName} scored Bingo! What a game!`,
            `Fantastic! ${playerName} has won with Bingo! Great job!`
          ];
          
          // Choose a random congratulatory phrase
          const randomPhrase = phrases[Math.floor(Math.random() * phrases.length)];
          
          // Add a good luck message with a slight pause
          const goodLuckPhrases = [
            "Keep playing everyone, you might be next!",
            "Don't give up everyone else, your luck might change!",
          ];
          
          const randomGoodLuck = goodLuckPhrases[Math.floor(Math.random() * goodLuckPhrases.length)];
          
          const utterance = new SpeechSynthesisUtterance(`${randomPhrase} ${randomGoodLuck}`);
          
          // Set properties for more natural speech
          utterance.rate = 0.9; // Slightly slower for more natural sound
          utterance.pitch = 1.0; // Normal pitch
          utterance.volume = 1.0; // Full volume
          
          // Speak the announcement
          window.speechSynthesis.speak(utterance);
        }
      }
      
      // Show message with player name
      this.showWinMessage(playerName);
    },
    showWinMessage(playerName) {
      // Create a modal message instead of an alert
      const modal = document.createElement('div');
      modal.className = 'win-modal';
      
      const content = document.createElement('div');
      content.className = 'win-modal-content';
      
      const message = document.createElement('h2');
      message.textContent = `${playerName} got BINGO!`;
      
      // Create a more personalized and encouraging message
      const goodLuckPhrases = [
        "And that's a Bingo! 🎉 Thanks so much for playing — we loved having you!",
        "That's the end of the game! Big thanks to everyone who joined — you all made it fun!",
        "Congrats to our winners and a huge thank you to everyone who played. Hope to see you next time!",
        "Game over for now — but don't worry, your lucky moment could be next. Thanks for playing!"
      ];
      
      const randomGoodLuck = goodLuckPhrases[Math.floor(Math.random() * goodLuckPhrases.length)];
      
      const subMessage = document.createElement('p');
      subMessage.textContent = randomGoodLuck;
      
      const closeBtn = document.createElement('button');
      closeBtn.textContent = 'Continue Playing';
      closeBtn.onclick = () => document.body.removeChild(modal);
      
      content.appendChild(message);
      content.appendChild(subMessage);
      content.appendChild(closeBtn);
      modal.appendChild(content);
      
      document.body.appendChild(modal);
      
      // Auto close after 5 seconds
      setTimeout(() => {
        if (document.body.contains(modal)) {
          document.body.removeChild(modal);
        }
      }, 5000);
    },
    announceNumber(number) {
      // Use speech synthesis to announce the number if audio is initialized
      if (this.audioInitialized && 'speechSynthesis' in window) {
        try {
          // For Safari, we need to resume the AudioContext first
          if (this.isSafari && this.audioContext) {
            this.audioContext.resume().catch(e => console.error('Error resuming AudioContext:', e));
          }
          
          // Cancel any ongoing speech
          window.speechSynthesis.cancel();
          
          // Create a more natural number announcement
          const phrases = [
            `Number ${number}`,
            `${number}`,
            `The number is ${number}`,
            `We have ${number}`
          ];
          
          // Choose a random phrase for variety
          const randomPhrase = phrases[Math.floor(Math.random() * phrases.length)];
          
          const utterance = new SpeechSynthesisUtterance(randomPhrase);
          
          // Set properties for more natural speech
          utterance.rate = 0.9; // Slightly slower for more natural sound
          utterance.pitch = 1.0; // Normal pitch
          utterance.volume = 1.0; // Volume
          
          // Speak the number
          window.speechSynthesis.speak(utterance);
          console.log(`Announced number: ${number}`);
        } catch (e) {
          console.error('Error announcing number:', e);
        }
      }
    },
    saveToLocalStorage(key, data) {
      try {
        localStorage.setItem(key, JSON.stringify(data));
      } catch (e) {
        console.error('Error saving to localStorage:', e);
      }
    },
    loadFromLocalStorage(key, defaultValue) {
      try {
        const savedData = localStorage.getItem(key);
        return savedData ? JSON.parse(savedData) : defaultValue;
      } catch (e) {
        console.error('Error loading from localStorage:', e);
        return defaultValue;
      }
    },
    initializeAudio() {
      console.log("Initializing audio...");
      
      // Special handling for Safari
      if (this.isSafari) {
        return this.initializeSafariAudio();
      }
      
      // Create and start AudioContext (needed for iOS)
      if (!this.audioContext) {
        try {
          this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
          console.log("AudioContext created:", this.audioContext.state);
        } catch (e) {
          console.error("Failed to create AudioContext:", e);
        }
      }
      
      // Resume AudioContext if it's suspended
      if (this.audioContext && this.audioContext.state === 'suspended') {
        this.audioContext.resume()
          .then(() => console.log("AudioContext resumed successfully"))
          .catch(e => console.error("Failed to resume AudioContext:", e));
      }
      
      // First try to play the silent sound to unlock audio
      const silentSound = this.$refs.silentSound;
      silentSound.volume = 0.1;
      
      const silentPromise = silentSound.play();
      if (silentPromise !== undefined) {
        silentPromise
          .then(() => {
            console.log("Silent sound played successfully");
            // Now try to play the actual sound
            this.playActualSound();
          })
          .catch(error => {
            console.error("Silent sound failed to play:", error);
            // Try direct approach
            this.playActualSound();
          });
      } else {
        // Fallback for browsers that don't return a promise
        this.playActualSound();
      }
    },
    
    playActualSound() {
      console.log("Attempting to play actual sound...");
      // Set volume low for test
      const bingoSound = this.$refs.bingoSound;
      bingoSound.volume = 0.2;
      
      const playPromise = bingoSound.play();
      
      if (playPromise !== undefined) {
        playPromise
          .then(() => {
            console.log("Bingo sound played successfully");
            // Pause it after a short delay to ensure it's properly initialized
            setTimeout(() => {
              bingoSound.pause();
              bingoSound.currentTime = 0;
              bingoSound.volume = 1.0; // Reset volume to normal
              this.audioInitialized = true;
              this.showAudioPrompt = false;
              
              // Test speech synthesis
              if ('speechSynthesis' in window) {
                try {
                  const testUtterance = new SpeechSynthesisUtterance('Audio initialized');
                  testUtterance.volume = 0.1; // Very quiet test
                  window.speechSynthesis.speak(testUtterance);
                  console.log("Speech synthesis test successful");
                } catch (e) {
                  console.error("Speech synthesis test failed:", e);
                }
              }
            }, 100);
          })
          .catch(error => {
            console.error("Bingo sound failed to play:", error);
            this.audioInitAttempts++;
            
            if (this.audioInitAttempts < 3) {
              console.log(`Retrying audio initialization (attempt ${this.audioInitAttempts})...`);
              // Try again with user interaction
              this.showAudioPrompt = true;
            } else {
              console.log("Max audio initialization attempts reached, proceeding without audio");
              // Still mark as initialized to prevent further prompts
              this.audioInitialized = true;
              this.showAudioPrompt = false;
            }
          });
      } else {
        // For browsers that don't return a promise
        console.log("Browser didn't return a play promise, assuming success");
        this.audioInitialized = true;
        this.showAudioPrompt = false;
      }
    },
    
    initializeSafariAudio() {
      console.log("Using Safari-specific audio initialization");
      
      // Create audio context within the user gesture
      if (!this.audioContext) {
        try {
          this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
          console.log("Safari AudioContext created:", this.audioContext.state);
        } catch (e) {
          console.error("Failed to create Safari AudioContext:", e);
        }
      }
      
      // Resume the audio context (critical for Safari)
      if (this.audioContext) {
        this.audioContext.resume()
          .then(() => console.log("Safari AudioContext resumed"))
          .catch(e => console.error("Failed to resume Safari AudioContext:", e));
      }
      
      // Play both sounds in sequence with a small delay
      const silentSound = this.$refs.silentSound;
      const bingoSound = this.$refs.bingoSound;
      
      // Set volume low for initialization
      silentSound.volume = 0.1;
      bingoSound.volume = 0.1;
      
      // First play the silent sound
      silentSound.play()
        .then(() => {
          console.log("Safari silent sound played");
          
          // Then play and quickly pause the actual sound
          return bingoSound.play();
        })
        .then(() => {
          console.log("Safari bingo sound played");
          
          // Pause after a short delay
          setTimeout(() => {
            bingoSound.pause();
            bingoSound.currentTime = 0;
            bingoSound.volume = 1.0; // Reset volume
            
            this.safariAudioInitialized = true;
            this.audioInitialized = true;
            this.showAudioPrompt = false;
            
            console.log("Safari audio initialization complete");
            
            // Test speech synthesis
            if ('speechSynthesis' in window) {
              try {
                const testUtterance = new SpeechSynthesisUtterance('Audio initialized');
                testUtterance.volume = 0.1;
                window.speechSynthesis.speak(testUtterance);
              } catch (e) {
                console.error("Safari speech synthesis test failed:", e);
              }
            }
          }, 100);
        })
        .catch(error => {
          console.error("Safari audio initialization failed:", error);
          
          // Still mark as initialized to prevent further prompts
          this.audioInitialized = true;
          this.showAudioPrompt = false;
        });
    },
    checkAudioSupport() {
      // Check if we're on Safari (including iOS Safari)
      const ua = navigator.userAgent.toLowerCase();
      this.isSafari = (ua.indexOf('safari') !== -1 && ua.indexOf('chrome') === -1) ||
                      /iPad|iPhone|iPod/.test(navigator.userAgent);
      
      console.log("Browser detection - isSafari:", this.isSafari);
      
      // Check if we're on a mobile device
      const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);
      console.log("Browser detection - isMobile:", isMobile);
      
      if (isMobile || this.isSafari) {
        // On mobile or Safari, we need user interaction
        this.showAudioPrompt = true;
        this.audioInitialized = false;
      } else {
        // On desktop non-Safari, we can try to initialize automatically
        this.initializeAudio();
      }
    }
  },
  watch: {
    players: {
      handler(newPlayers) {
        this.saveToLocalStorage('bingoPlayers', newPlayers);
      },
      deep: true
    }
  },
  mounted() {
    this.checkAudioSupport();
  }
}
</script>

<style scoped>
.casino-theme {
  font-family: 'Arial', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: white;
  margin: 0;
  padding: 20px;
  background: linear-gradient(135deg, #2c003e 0%, #512b58 50%, #7b337d 100%);
  background-image: url('https://www.transparenttextures.com/patterns/diamond-upholstery.png'),
                    linear-gradient(135deg, #2c003e 0%, #512b58 50%, #7b337d 100%);
  min-height: 100vh;
  position: relative;
  overflow: hidden;
}

.neon-text {
  color: #fff;
  text-shadow: 0 0 5px #fff, 0 0 10px #fff, 0 0 15px #876483, 0 0 20px #ff00de, 0 0 25px #ff00de, 0 0 30px #ff00de, 0 0 35px #ff00de;
  animation: neon-flicker 1.5s infinite alternate;
  margin-bottom: 30px;
  font-family: 'Arial Black', Gadget, sans-serif;
  letter-spacing: 2px;
  font-size: 2.5em;
}

@keyframes neon-flicker {
  0%, 19%, 21%, 23%, 25%, 54%, 56%, 100% {
    text-shadow: 0 0 5px #fff, 0 0 10px #fff, 0 0 15px #ff00de, 0 0 20px #ff00de, 0 0 25px #ff00de, 0 0 30px #ff00de, 0 0 35px #ff00de;
  }
  20%, 24%, 55% {
    text-shadow: none;
  }
}

.casino-lights {
  position: absolute;
  left: 0;
  width: 100%;
  height: 15px;
  z-index: 1;
}

.casino-lights.top {
  top: 0;
  background: linear-gradient(90deg,
    #ff0000 0%, #ff7f00 10%, #ffff00 20%,
    #00ff00 30%, #0000ff 40%, #4b0082 50%,
    #8b00ff 60%, #ff0000 70%, #ff7f00 80%,
    #ffff00 90%, #00ff00 100%);
  animation: light-move 5s linear infinite;
  box-shadow: 0 0 20px rgba(255, 255, 255, 0.5);
}

.casino-lights.bottom {
  bottom: 0;
  background: linear-gradient(90deg,
    #00ff00 0%, #0000ff 10%, #4b0082 20%,
    #8b00ff 30%, #ff0000 40%, #ff7f00 50%,
    #ffff00 60%, #00ff00 70%, #0000ff 80%,
    #4b0082 90%, #8b00ff 100%);
  animation: light-move 5s linear infinite reverse;
  box-shadow: 0 0 20px rgba(255, 255, 255, 0.5);
}

@keyframes light-move {
  0% { background-position: 0% 50%; }
  100% { background-position: 100% 50%; }
}
.players {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  margin: 20px 0;
}
.player-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 10px;
}
.player-name-input {
  margin-bottom: 5px;
  padding: 8px;
  border: 1px solid gold;
  border-radius: 5px;
  text-align: center;
  width: 120px;
  background-color: rgba(0, 0, 0, 0.3);
  color: white;
}

.player-name-input::placeholder {
  color: rgba(255, 255, 255, 0.6);
}

.player-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 10px;
}

.color-picker {
  margin-top: 5px;
  display: flex;
  align-items: center;
  font-size: 12px;
}

.color-picker label {
  margin-right: 5px;
  color: #ddd;
}

.color-select {
  padding: 3px;
  border-radius: 3px;
  background-color: #333;
  color: white;
  border: 1px solid #555;
}
.called-numbers {
  margin-top: 20px;
  font-size: 16px;
  max-width: 600px;
  margin: 20px auto;
}
.number-grid {
  display: grid;
  grid-template-columns: repeat(15, 1fr);
  gap: 5px;
}
.number-cell {
  width: 30px;
  height: 30px;
  display: flex;
  justify-content: center;
  align-items: center;
  border: 1px solid #ddd;
  border-radius: 5px;
  font-size: 12px;
}
.number-cell.called {
  background-color: #dc3545;
  color: white;
}
.game-controls {
  margin: 15px 0;
}
button {
  background: linear-gradient(to bottom, #f7d358, #dfb30d);
  border: 2px solid #b8860b;
  color: #333;
  padding: 10px 20px;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 16px;
  font-weight: bold;
  margin: 5px;
  cursor: pointer;
  border-radius: 5px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
  transition: all 0.3s;
}

button:hover {
  background: linear-gradient(to bottom, #ffeb3b, #ffc107);
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.4);
}
button:disabled {
  background: linear-gradient(to bottom, #cccccc, #999999);
  border-color: #999999;
  color: #666666;
  cursor: not-allowed;
  box-shadow: none;
}

button:disabled:hover {
  transform: none;
  box-shadow: none;
}
.last-called-number {
  font-size: 24px;
  margin: 15px 0;
  font-weight: bold;
}
.last-called-number .number {
  display: inline-block;
  width: 50px;
  height: 50px;
  line-height: 50px;
  background-color: #dc3545;
  color: white;
  border-radius: 50%;
  margin-left: 10px;
}
.call-counter {
  font-size: 16px;
  margin-top: 5px;
  color: #666;
}
.speed-control {
  margin: 10px 0;
  display: flex;
  flex-direction: column;
  align-items: center;
}
.speed-control input {
  width: 200px;
  margin-top: 5px;
}
.stop-btn {
  background: linear-gradient(to bottom, #ff6b6b, #dc3545) !important;
  border-color: #c82333 !important;
  color: white !important;
}

.stop-btn:hover {
  background: linear-gradient(to bottom, #ff8585, #e04c59) !important;
}

.win-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.win-modal-content {
  background: linear-gradient(135deg, #540d6e, #ee4266);
  border: 5px solid gold;
  border-radius: 15px;
  padding: 30px;
  text-align: center;
  box-shadow: 0 0 30px gold, inset 0 0 20px rgba(255, 215, 0, 0.5);
  max-width: 500px;
  width: 80%;
  color: white;
  animation: modal-appear 0.5s ease-out;
  position: relative;
  overflow: hidden;
}

.win-modal-content::before {
  content: '';
  position: absolute;
  top: -50%;
  left: -50%;
  width: 200%;
  height: 200%;
  background: linear-gradient(
    to bottom right,
    rgba(255, 255, 255, 0) 0%,
    rgba(255, 255, 255, 0) 40%,
    rgba(255, 255, 255, 0.6) 50%,
    rgba(255, 255, 255, 0) 60%,
    rgba(255, 255, 255, 0) 100%
  );
  transform: rotate(45deg);
  animation: shine 3s infinite;
}

@keyframes shine {
  0% { left: -50%; top: -50%; }
  100% { left: 150%; top: 150%; }
}

@keyframes modal-appear {
  0% { transform: scale(0); opacity: 0; }
  100% { transform: scale(1); opacity: 1; }
}

.win-modal h2 {
  font-size: 28px;
  margin-bottom: 15px;
  text-shadow: 0 0 10px gold;
}

.win-modal p {
  font-size: 18px;
  margin-bottom: 25px;
}

.win-modal button {
  background: linear-gradient(to right, #ffd700, #ffaa00);
  border: 2px solid #b8860b;
  color: #333;
  padding: 12px 24px;
  font-size: 18px;
  font-weight: bold;
  border-radius: 30px;
  cursor: pointer;
  transition: all 0.3s;
  box-shadow: 0 5px 15px rgba(0,0,0,0.2);
  position: relative;
  overflow: hidden;
  z-index: 1;
}

.win-modal button::after {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg,
    rgba(255,255,255,0) 0%,
    rgba(255,255,255,0.2) 50%,
    rgba(255,255,255,0) 100%);
  transition: all 0.6s;
  z-index: -1;
}

.win-modal button:hover::after {
  left: 100%;
}

.win-modal button:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 20px rgba(0,0,0,0.3);
}

/* Audio prompt styles */
.audio-prompt {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.8);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.audio-prompt-content {
  background: linear-gradient(135deg, #512b58 0%, #7b337d 100%);
  border: 2px solid gold;
  border-radius: 15px;
  padding: 30px;
  text-align: center;
  max-width: 90%;
  box-shadow: 0 0 20px rgba(255, 215, 0, 0.5);
}

.audio-prompt-content h3 {
  color: gold;
  margin-top: 0;
  font-size: 24px;
  text-shadow: 0 0 10px rgba(255, 215, 0, 0.5);
}

.audio-prompt-content p {
  color: white;
  margin-bottom: 20px;
  font-size: 16px;
}

.enable-audio-btn {
  background: linear-gradient(to right, #ff8a00, #da1b60);
  color: white;
  border: none;
  border-radius: 25px;
  padding: 12px 30px;
  font-size: 18px;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
}

.enable-audio-btn:hover {
  transform: scale(1.05);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
}

.enable-audio-btn:active {
  transform: scale(0.98);
}
</style>