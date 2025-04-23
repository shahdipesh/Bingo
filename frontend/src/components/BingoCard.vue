<template>
  <div class="bingo-card" :class="{ 'winner': hasBingo }" :style="cardStyle">
    <div class="card-header" :style="headerStyle">{{ playerName }}</div>
    <div v-if="hasBingo" class="bingo-badge">BINGO!</div>
    <div v-if="hasBingo" class="jackpot-lights">
      <div v-for="n in 20" :key="`light-${n}`" class="light" :style="getLightStyle(n)"></div>
    </div>
    <div v-if="hasBingo" class="confetti-container">
      <div v-for="n in 100" :key="`confetti-${n}`" class="confetti" :style="getConfettiStyle(n)"></div>
    </div>
    <div v-if="hasBingo" class="win-overlay">
      <div class="win-text">WINNER!</div>
    </div>
    <div class="card-body" :style="bodyStyle">
      <div class="card-grid">
        <div
          v-for="(number, index) in cardNumbers"
          :key="index"
          class="card-number"
          :class="{
            'winning-cell': isWinningCell(index),
            'last-called': number === lastCalledNumber,
            'revealed': isRevealed(number)
          }"
          :style="getNumberStyle(number)"
        >
          <span class="number-text">{{ number }}</span>
          <div v-if="!isRevealed(number) || number === lastCalledNumber"
               class="scratch-overlay"
               :class="{'scratching': number === lastCalledNumber}">
            <div class="scratch-texture"></div>
          </div>
          <div v-if="number === lastCalledNumber" class="scratch-particles">
            <div v-for="p in 12" :key="`particle-${p}`" class="particle" :style="getParticleStyle(p)"></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'BingoCard',
  props: {
    playerName: {
      type: String,
      required: true
    },
    cardData: {
      type: Array,
      required: true
    },
    calledNumbers: {
      type: Array,
      default: () => []
    },
    lastCalledNumber: {
      type: Number,
      default: null
    },
    cardColor: {
      type: String,
      default: "#dc3545" // Default red color
    }
  },
  data() {
    return {
      winningCells: [],
      revealedNumbers: []
    }
  },
  computed: {
    cardNumbers() {
      // Ensure cardData is an array of numbers
      if (!Array.isArray(this.cardData)) {
        console.error('cardData is not an array:', this.cardData);
        return [];
      }
      
      // Filter out any non-numeric values
      return this.cardData.filter(num => typeof num === 'number');
    },
    hasBingo() {
      return this.winningCells.length > 0;
    },
    cardStyle() {
      return {
        borderColor: this.cardColor
      };
    },
    headerStyle() {
      return {
        backgroundColor: this.cardColor,
        color: 'white'
      };
    },
    bodyStyle() {
      // Light version of the card color (20% opacity)
      return {
        backgroundColor: `${this.cardColor}22`
      };
    }
  },
  methods: {
    isWinningCell(index) {
      // Make sure we don't have duplicate indices in winningCells
      return this.winningCells.indexOf(index) !== -1;
    },
    isRevealed(number) {
      // Check if the number has been called and it's not the current last called number
      return this.calledNumbers.includes(number) && number !== this.lastCalledNumber;
    },
    getNumberStyle(number) {
      if (this.calledNumbers.includes(number)) {
        // Add this number to revealed numbers if not already there
        if (!this.revealedNumbers.includes(number)) {
          this.revealedNumbers.push(number);
        }
        return { backgroundColor: this.cardColor, color: 'white' };
      }
      return {};
    },
    getParticleStyle(index) {
      const size = Math.random() * 3 + 2; // 2-5px
      const angle = (index / 12) * 360;
      const distance = Math.random() * 25 + 15; // 15-40px
      const duration = Math.random() * 0.6 + 0.4; // 0.4-1s
      const delay = Math.random() * 0.3 + 0.1; // 0.1-0.4s
      const color = Math.random() > 0.5 ? '#aaa' : '#999';
      
      return {
        width: `${size}px`,
        height: `${size}px`,
        backgroundColor: color,
        animation: `particle-fly ${duration}s ease-out ${delay}s forwards`,
        '--angle': `${angle}deg`,
        '--distance': `${distance}px`
      };
    },
    getConfettiStyle(n) {
      const colors = ['#f44336', '#e91e63', '#9c27b0', '#673ab7', '#3f51b5', '#2196f3', '#03a9f4', '#00bcd4', '#009688', '#4CAF50', '#8BC34A', '#CDDC39', '#FFEB3B', '#FFC107', '#FF9800', '#FF5722', 'gold', 'silver'];
      const color = colors[Math.floor(Math.random() * colors.length)];
      const size = Math.floor(Math.random() * 10) + 5; // 5-15px
      const left = Math.floor(Math.random() * 100); // 0-100%
      const animationDelay = Math.random() * 3; // 0-3s
      const animationDuration = 3 + Math.random() * 2; // 3-5s
      
      return {
        backgroundColor: color,
        width: `${size}px`,
        height: `${size}px`,
        left: `${left}%`,
        animationDelay: `${animationDelay}s`,
        animationDuration: `${animationDuration}s`
      };
    },
    getLightStyle(n) {
      const angle = (n / 20) * 360;
      const radius = 80; // Distance from center
      const x = Math.cos(angle * Math.PI / 180) * radius;
      const y = Math.sin(angle * Math.PI / 180) * radius;
      const animationDelay = (n / 20) * 2; // Staggered animation
      
      return {
        transform: `translate(${x}px, ${y}px)`,
        animationDelay: `${animationDelay}s`
      };
    },
    checkForBingo() {
      this.winningCells = [];
      
      // Only check for blackout bingo (all numbers called)
      const allNumbersCalled = this.cardNumbers.every(num => {
        const called = this.calledNumbers.includes(num);
        return called;
      });
      
      if (allNumbersCalled) {
        console.log('BLACKOUT BINGO! All numbers called:', this.cardNumbers);
        
        // Add all indices to winning cells
        for (let i = 0; i < this.cardNumbers.length; i++) {
          this.winningCells.push(i);
        }
        
        // Emit event when bingo is achieved
        this.$emit('bingo', this.playerName);
      }
    }
  },
  watch: {
    calledNumbers: {
      handler() {
        this.checkForBingo();
      },
      deep: true
    }
  }
}
</script>

<style scoped>
.bingo-card {
  border: 2px solid;
  border-radius: 10px;
  margin: 10px;
  width: 150px;
  box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2);
  position: relative;
  transition: all 0.3s ease;
}

.bingo-card.winner {
  box-shadow: 0 0 20px 10px gold;
  transform: scale(1.05);
  z-index: 10;
  animation: winner-pulse 1.5s infinite alternate;
}

@keyframes winner-pulse {
  0% { box-shadow: 0 0 20px 5px gold; }
  100% { box-shadow: 0 0 30px 15px gold; }
}

.card-header {
  text-align: center;
  font-weight: bold;
  padding: 8px;
  border-top-left-radius: 8px;
  border-top-right-radius: 8px;
}

.card-body {
  padding: 5px;
  border-bottom-left-radius: 8px;
  border-bottom-right-radius: 8px;
}

.card-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: repeat(5, 1fr);
  gap: 5px;
  padding: 5px;
  justify-items: center;
}

.card-number {
  width: 30px;
  height: 30px;
  display: flex;
  justify-content: center;
  align-items: center;
  border: 1px solid #ddd;
  font-size: 12px;
  border-radius: 5px;
  transition: all 0.2s ease;
  background-color: white;
  color: #333;
  position: relative;
  overflow: hidden;
}

.number-text {
  position: relative;
  z-index: 1;
}

.scratch-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(135deg, #888 0%, #ccc 50%, #aaa 100%);
  z-index: 2;
  transition: transform 0.5s ease, opacity 0.5s ease;
  border-radius: 5px;
  box-shadow: inset 0 0 3px rgba(0,0,0,0.3);
  overflow: hidden;
}

.scratch-texture {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image:
    radial-gradient(circle at 10px 10px, rgba(255,255,255,0.2) 1px, transparent 1px),
    radial-gradient(circle at 20px 20px, rgba(255,255,255,0.2) 1px, transparent 1px),
    radial-gradient(circle at 30px 10px, rgba(255,255,255,0.2) 1px, transparent 1px),
    radial-gradient(circle at 10px 30px, rgba(255,255,255,0.2) 1px, transparent 1px),
    url('data:image/svg+xml;utf8,<svg width="40" height="40" xmlns="http://www.w3.org/2000/svg"><path d="M0 0 L40 40 M40 0 L0 40 M20 0 L20 40 M0 20 L40 20" stroke-width="1" stroke="rgba(0,0,0,0.1)"/></svg>');
}

.card-number.revealed .scratch-overlay {
  transform: translateX(100%) rotate(5deg);
  opacity: 0;
  visibility: hidden; /* Completely hide the element after animation */
}

.scratch-overlay.scratching {
  animation: scratch-off 0.8s 0.1s forwards;
}

.scratch-overlay.scratching .scratch-texture {
  animation: texture-move 0.8s 0.1s forwards;
}

.scratch-overlay::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 40%;
  height: 100%;
  background: linear-gradient(
    90deg,
    rgba(255,255,255,0) 0%,
    rgba(255,255,255,0.3) 50%,
    rgba(255,255,255,0) 100%
  );
  transform: skewX(-20deg);
  animation: shine 1s infinite;
  display: none;
}

.scratch-overlay.scratching::before {
  display: block;
}

@keyframes shine {
  0% { left: -100%; }
  100% { left: 200%; }
}

@keyframes scratch-off {
  0% {
    transform: translateX(0) rotate(0deg);
    opacity: 1;
  }
  20% {
    transform: translateX(10%) rotate(2deg);
    opacity: 0.9;
  }
  40% {
    transform: translateX(30%) rotate(3deg);
    opacity: 0.7;
  }
  60% {
    transform: translateX(50%) rotate(5deg);
    opacity: 0.5;
  }
  80% {
    transform: translateX(70%) rotate(7deg);
    opacity: 0.3;
  }
  100% {
    transform: translateX(120%) rotate(10deg);
    opacity: 0;
    visibility: hidden; /* Ensure it's completely hidden */
  }
}

@keyframes texture-move {
  0% {
    background-position: 0 0;
  }
  20% {
    background-position: 2px 0;
  }
  40% {
    background-position: 3px 1px;
  }
  60% {
    background-position: 5px 2px;
  }
  80% {
    background-position: 7px 3px;
  }
  100% {
    background-position: 10px 5px;
  }
}

.card-number.winning-cell {
  background-color: gold !important;
  color: #333 !important;
  animation: pulse 1.5s infinite;
}

.card-number.last-called {
  background-color: #ff9800 !important;
  color: white !important;
  box-shadow: 0 0 10px #ff9800;
  transform: scale(1.15);
  z-index: 2;
}

.bingo-badge {
  position: absolute;
  top: -15px;
  right: -15px;
  background: linear-gradient(45deg, #FFD700, #FFA500);
  color: #333;
  padding: 8px 15px;
  border-radius: 20px;
  font-weight: bold;
  font-size: 14px;
  box-shadow: 0 2px 10px rgba(255,215,0,0.6);
  z-index: 20;
  animation: badge-animation 1s infinite alternate;
}

@keyframes badge-animation {
  0% { transform: scale(1) rotate(-5deg); }
  100% { transform: scale(1.1) rotate(5deg); }
}

.confetti-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  pointer-events: none;
  z-index: 15;
}

.confetti {
  position: absolute;
  top: -10px;
  border-radius: 50%;
  animation: confetti-fall linear forwards;
}

@keyframes confetti-fall {
  0% {
    transform: translateY(0) rotate(0deg);
    opacity: 1;
  }
  50% {
    opacity: 1;
  }
  100% {
    transform: translateY(200px) rotate(720deg);
    opacity: 0;
  }
}

.jackpot-lights {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  z-index: 12;
  pointer-events: none;
}

.light {
  position: absolute;
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background-color: yellow;
  box-shadow: 0 0 10px 2px yellow;
  animation: light-flash 0.5s infinite alternate;
}

@keyframes light-flash {
  0% { opacity: 0.3; }
  100% { opacity: 1; }
}

.win-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.3);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 14;
  pointer-events: none;
}

.win-text {
  font-size: 24px;
  font-weight: bold;
  color: gold;
  text-shadow: 0 0 10px gold, 0 0 20px red;
  animation: win-text-animation 0.5s infinite alternate;
  transform: rotate(-15deg);
}

@keyframes win-text-animation {
  0% { transform: scale(1) rotate(-15deg); }
  100% { transform: scale(1.2) rotate(-15deg); }
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.1); }
  100% { transform: scale(1); }
}

.scratch-particles {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  z-index: 3;
  pointer-events: none;
}

.particle {
  position: absolute;
  top: 0;
  left: 0;
  background-color: #ccc;
  border-radius: 50%;
  transform-origin: center;
  box-shadow: 0 0 2px rgba(0,0,0,0.1);
  opacity: 0.8;
}

@keyframes particle-fly {
  0% {
    opacity: 1;
    transform: translate(0, 0) scale(1);
  }
  50% {
    opacity: 0.7;
  }
  100% {
    opacity: 0;
    transform: rotate(var(--angle, 0deg)) translateX(var(--distance, 20px)) scale(0);
  }
}
</style>