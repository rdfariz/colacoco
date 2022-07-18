<template>
  <div class="world" data-world>
    <div class="start-screen" data-start-screen>TAP TO START</div>
    <img src="./assets/ground.png" class="topground" data-topground>
    <img src="./assets/ground.png" class="topground" data-topground>
    <img src="./assets/ground.png" class="ground" data-ground>
    <img src="./assets/ground.png" class="ground" data-ground>
    <img v-show="isPlaying" src="./assets/cola-stationary.png" class="dino" data-dino>
    <div class="score">
      <div class="btn-music" @click="toggleMusic" >
        {{ isPlayMusic ? 'Stop' : 'Play' }} Music
      </div>
      <p>Highest: {{ highest }}</p>
      <p>❤: {{ life }}</p>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue';
import { Howl } from 'howler';
import {
  getCustomProperty,
  incrementCustomProperty,
  setCustomProperty,
} from "./assets/js/updateCustomProperty.js"

import dinoStationaryImg from './assets/cola-stationary.png';
import dinoLostImg from './assets/cola-lose.png';
import dinoRun0Img from './assets/cola-run-0.png';
import dinoRun1Img from './assets/cola-run-1.png';
import dinoJumpImg from './assets/cola-jump.png';
import cactusImg from './assets/yuuqi.png';
import cactus2Img from './assets/enemy.png';
import mp3Bgm from './assets/bgm.mp3';

export default {
  name: 'App',
  setup () {
    const highest = localStorage.getItem('highestScoreCola') || 0;
    const _autoPlayBgm = localStorage.getItem('autoPlayBgmCola') || 'true';
    const autoPlayBgm = JSON.parse(_autoPlayBgm) || true;
  
    const musicBgm = new Howl({
      src: [mp3Bgm],
      autoplay: autoPlayBgm,
      loop: true,
      volume: 0.2,
    });

    const isPlayMusic = ref(autoPlayBgm);

    return {
      musicBgm,
      highest,
      isPlayMusic,
    };
  },
  data(){
    return{
      isPlaying: false,
      prevElementBuff: false,
      life: 0,
      WORLD_WIDTH: 100,
      WORLD_HEIGHT: 40,
      SPEED_SCALE_INCREASE: 0.00001,
      lastTime: 0,
      speedScale: 0,
      score: 0,
      dino: {
        JUMP_SPEED: 0.19,
        GRAVITY: 0.0010,
        DINO_FRAME_COUNT: 2,
        FRAME_TIME: 100,
        isJumping: false,
        dinoFrame: 0,
        currentFrameTime: 0,
        yVelocity: 0
      },
      cactus: {
        SPEED: 0.05,
        CACTUS_INTERVAL_MIN: 500,
        CACTUS_INTERVAL_MAX: 2000,
        nextCactusTime: 0
      },
    }
  },
  mounted() {
    this.setPixelToWorldScale()
    window.addEventListener("resize", this.setPixelToWorldScale)
    // document.addEventListener("keydown", this.handleStart, { once: true })
    document.addEventListener("click", this.handleStart, { once: true })
  },
  methods: {
    toggleMusic () {
      if (this.isPlayMusic) {
        localStorage.setItem("autoPlayBgmCola", false);
        this.isPlayMusic = false;
        this.musicBgm.stop();
      } else {
        localStorage.setItem("autoPlayBgmCola", true);
        this.isPlayMusic = true;
        this.musicBgm.play();
      }
    },
    setPixelToWorldScale() {
      const worldElem = document.querySelector("[data-world]")
      let worldToPixelScale
      if (window.innerWidth / window.innerHeight < this.WORLD_WIDTH / this.WORLD_HEIGHT) {
        worldToPixelScale = window.innerWidth / this.WORLD_WIDTH
      } else {
        worldToPixelScale = window.innerHeight / this.WORLD_HEIGHT
      }

      worldElem.style.width = `${this.WORLD_WIDTH * worldToPixelScale}px`
      worldElem.style.height = `${this.WORLD_HEIGHT * worldToPixelScale}px`
    },
    handleStart() {
      const startScreenElem = document.querySelector("[data-start-screen]")
      this.isPlaying = true;
      this.lastTime = null
      this.speedScale = 1
      this.score = 0
      this.life = 0
      this.setupGround()
      this.setupTopGround()
      this.setupDino()
      this.setupCactus()
      startScreenElem.classList.add("hide")
      window.requestAnimationFrame(this.update)
    },
    update(time) {
      if (this.lastTime == null) {
        this.lastTime = time
        window.requestAnimationFrame(this.update)
        return
      }
      const delta = time - this.lastTime

      this.updateGround(delta, this.speedScale)
      this.updateTopGround(delta, this.speedScale)
      this.updateDino(delta, this.speedScale)
      this.updateCactus(delta, this.speedScale)
      this.updateSpeedScale(delta)
      window.addEventListener('focus', this.updateScore(delta));
      if (this.checkLose()) return this.handleLose()

      this.lastTime = time
      window.requestAnimationFrame(this.update)
    },
    updateScore(delta) {
      this.score += delta * 0.01
      // const scoreElem = document.querySelector("[data-score]")
      // scoreElem.textContent = Math.floor(this.score)
    },
    updateSpeedScale(delta) {
      this.speedScale += delta * this.SPEED_SCALE_INCREASE
    },
    checkLose() {
      const dinoRect = this.getDinoRect()
      if (this.getTopGroundRects().some(rect => this.isCollision(rect, dinoRect))) return true;
      if (this.getCactusRects().some(rect => this.isCollision(rect, dinoRect))) return true;
      return false;
    },
    isCollision(rect1, rect2) {
      const isCollised = (
        rect1.rect.left < rect2.rect.right &&
        rect1.rect.top < rect2.rect.bottom &&
        rect1.rect.right > rect2.rect.left &&
        rect1.rect.bottom > rect2.rect.top
      );

      if (isCollised && rect1.isBuff) {
        this.life += 1;
        if (this.life > this.highest) {
          this.highest = this.life;
          localStorage.setItem("highestScoreCola", this.highest);
        }
        rect1.el.remove();
        return false;
      } else {
        return isCollised
      }
    },
    handleLose() {
      const startScreenElem = document.querySelector("[data-start-screen]")
      this.setDinoLose()
      setTimeout(() => {
        document.addEventListener("keydown", this.handleStart, { once: true })
        document.addEventListener("click", this.handleStart, { once: true })
        startScreenElem.classList.remove("hide")
      }, 100)
    },

    setupGround () {
      const groundElems = document.querySelectorAll("[data-ground]")
      setCustomProperty(groundElems[0], "--left", 0)
      setCustomProperty(groundElems[1], "--left", 300)
    },
    updateGround(delta, speedScale) {
      const SPEED = 0.05
      const groundElems = document.querySelectorAll("[data-ground]")
      groundElems.forEach(ground => {
        incrementCustomProperty(ground, "--left", delta * speedScale * SPEED * -1)

        if (getCustomProperty(ground, "--left") <= -300) {
          incrementCustomProperty(ground, "--left", 600)
        }
      })
    },
    setupTopGround () {
      const groundElems = document.querySelectorAll("[data-topground]")
      setCustomProperty(groundElems[0], "--left", 0)
      setCustomProperty(groundElems[1], "--left", 300)
    },
    updateTopGround(delta, speedScale) {
      const SPEED = 0.05
      const groundElems = document.querySelectorAll("[data-topground]")
      groundElems.forEach(ground => {
        incrementCustomProperty(ground, "--left", delta * speedScale * SPEED * -1)

        if (getCustomProperty(ground, "--left") <= -300) {
          incrementCustomProperty(ground, "--left", 600)
        }
      })
    },
    getTopGroundRects() {
      return [...document.querySelectorAll("[data-topground]")].map(topground => {
        return {
          el: topground,
          rect: topground.getBoundingClientRect()
        }
      })
    },


    setupDino() {      
      const dinoElem = document.querySelector("[data-dino]")

      this.dino.isJumping = false
      this.dino.dinoFrame = 0
      this.dino.currentFrameTime = 0
      this.dino.yVelocity = 0
      setCustomProperty(dinoElem, "--bottom", 0)
      document.removeEventListener("click", this.onJump)
      document.addEventListener("click", this.onJump)
      document.removeEventListener("keydown", this.onJump)
      document.addEventListener("keydown", this.onJump)
    },
    updateDino(delta, speedScale) {
      this.handleRun(delta, speedScale)
      this.handleJump(delta)
    },
    getDinoRect() {
      const dinoElem = document.querySelector("[data-dino]")
      return {
        el: dinoElem,
        rect: dinoElem.getBoundingClientRect()
      }
    },
    setDinoLose() {
      const dinoElem = document.querySelector("[data-dino]")
      dinoElem.src = dinoLostImg
    },
    handleRun(delta, speedScale) {
      const dinoElem = document.querySelector("[data-dino]")
      if (this.dino.isJumping) {
        dinoElem.src = dinoStationaryImg
        return
      }

      if (this.dino.currentFrameTime >= this.dino.FRAME_TIME) {
        this.dino.dinoFrame = (this.dino.dinoFrame + 1) % this.dino.DINO_FRAME_COUNT
        dinoElem.src = this.dino.dinoFrame === 1 ? dinoRun1Img : dinoRun0Img
        this.dino.currentFrameTime -= this.dino.FRAME_TIME
      }
      this.dino.currentFrameTime += delta * speedScale
    },
    handleJump(delta) {
      const dinoElem = document.querySelector("[data-dino]")
      if (!this.dino.isJumping) return

      dinoElem.src = dinoJumpImg
      incrementCustomProperty(dinoElem, "--bottom", this.dino.yVelocity * delta)    

      if (getCustomProperty(dinoElem, "--bottom") <= 0) {
        dinoElem.src = dinoStationaryImg
        setCustomProperty(dinoElem, "--bottom", 0)
        this.dino.isJumping = false
      }

      this.dino.yVelocity -= this.dino.GRAVITY * delta
    },
    onJump(e) {
      if (e.type === 'click' || e.code === "Space" || !this.dino.isJumping) {
        this.dino.yVelocity = this.dino.JUMP_SPEED
        this.dino.isJumping = true
      }
    },

    setupCactus() {
      this.cactus.nextCactusTime = this.cactus.CACTUS_INTERVAL_MIN
      document.querySelectorAll("[data-cactus]").forEach(cactus => {
        cactus.remove()
      })
    },
    updateCactus(delta, speedScale) {
      document.querySelectorAll("[data-cactus]").forEach(cactus => {
        incrementCustomProperty(cactus, "--left", delta * speedScale * this.cactus.SPEED * -1)
        if (getCustomProperty(cactus, "--left") <= -100) {
          cactus.remove()
        }
      })

      if (this.cactus.nextCactusTime <= 0) {
        this.createCactus()
        this.cactus.nextCactusTime =
          this.randomNumberBetween(this.cactus.CACTUS_INTERVAL_MIN, this.cactus.CACTUS_INTERVAL_MAX) / speedScale
      }
      this.cactus.nextCactusTime -= delta
    },
    getCactusRects() {
      return [...document.querySelectorAll("[data-cactus]")].map(cactus => {
        return {
          el: cactus,
          isBuff: cactus.classList.contains('buff') ? true : false,
          rect: cactus.getBoundingClientRect()
        }
      })
    },
    createCactus() {
      const foo = Math.random() * 100;
      if (foo < 50 && !this.prevElementBuff) {
        // 00-50
        const worldElem = document.querySelector("[data-world]")
        const cactus = document.createElement("img")
        cactus.dataset.cactus = true
        cactus.src = cactusImg
        cactus.classList.add("buff")
        setCustomProperty(cactus, "--left", 100)
        worldElem.append(cactus)
        this.prevElementBuff = true;
      } else {
        // 50-100
        const worldElem = document.querySelector("[data-world]")
        const cactus = document.createElement("img")
        cactus.dataset.cactus = true
        cactus.src = cactus2Img

        cactus.classList.add("cactus")
        setCustomProperty(cactus, "--left", 100)
        worldElem.append(cactus)
        this.prevElementBuff = false;
      }
    },
    randomNumberBetween(min, max) {
      return Math.floor(Math.random() * (max - min + 1) + min)
    }
  }
}
</script>

<style>
@font-face {
  font-family: pixel;
  src: url('./assets/font/04B_30__.TTF');
}

#app {
  font-family: 'pixel', Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

*, *::before, *::after {
  box-sizing: border-box;
  user-select: none;
}

body {
  margin: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background-image: url('./assets/background.webp');
  background-size: cover;
  /* background-color: #f0f0f0; */
  /* display: flex;
  justify-content: center;
  align-items: flex-end; */
}

.btn-music {
  background: #2c3e50;
  cursor: pointer;
  font-size: 1.5vmin;
  width: auto;
  color: #ffffff;
  text-align: center;
  border-radius: 100px;
  padding: 5px;
  margin-bottom: 5px;
}

@media only screen and (min-width: 768px) {
  .btn-music {
    padding: 10px;
    font-size: 2.5vmin;
    margin-bottom: 15px;
  }
}

.world {
  overflow: hidden;
  position: relative;
}

.life {
  position: absolute;
  font-size: 7vmin;
  right: 0;
  top: 1vmin;
}

.score {
  position: absolute;
  font-size: 2vmin;
  right: 2vmin;
  top: 4vmin;
  text-align: right;
}
.score p {
  margin: 0 0 5px 0;
}

@media only screen and (min-width: 768px) {
  .score {
    font-size: 3.5vmin;
    top: 8vmin;
  }
  .score p {
    margin: 0 0 10px 0;
  }
}

.start-screen {
  position: absolute;
  font-size: 3vmin;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

@media only screen and (min-width: 768px) {
  .start-screen {
    font-size: 4vmin;
  }
}

.hide {
  display: none;
}


.topground {
  --left: 0;
  position: absolute;
  width: 300%;
  top: 0;
  left: calc(var(--left) * 1%);
  transform: rotate(180deg);
}

.ground {
  --left: 0;
  position: absolute;
  width: 300%;
  bottom: 0;
  left: calc(var(--left) * 1%)
}

.dino {
  --bottom: 0;
  position: absolute;
  left: 1%;
  height: 30%;
  bottom: calc(var(--bottom) * 1%);
}

.topcactus {
  position: absolute;
  left: calc(var(--left) * 1%);
  height: 30%;
  top: 0;
  transform: rotate(180deg);
}


.buff {
  position: absolute;
  left: calc(var(--left) * 1%);
  height: 30%;
  bottom: 0;
}

.cactus {
  position: absolute;
  left: calc(var(--left) * 1%);
  height: 34%;
  bottom: 0;
}
</style>
