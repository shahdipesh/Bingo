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
            'last-called': number === lastCalledNumber
          }"
          :style="getNumberStyle(number)"
        >
          {{ number }}
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
      winningCells: []
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
    getNumberStyle(number) {
      if (this.calledNumbers.includes(number)) {
        return { backgroundColor: this.cardColor, color: 'white' };
      }
      return {};
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
</style>