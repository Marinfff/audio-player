<script>
import fs from 'fs'
import {ipcRenderer} from 'electron'
import dataurl from "dataurl";
import Wave from 'wavesurfer.js';

export default {
  name: 'Window',
  data() {
    return {
      path: '',
      points: [],
      wave: null
    }
  },
  methods: {
    getFilePath() {
      this.path = ipcRenderer.sendSync('getPath', 'path', 10);
      this.read(this.path.filePaths[0])
    },
    read(path) {
      fs.readFile(path, (error, data) => {
        this.createAudio(dataurl.convert({data, mimetype: 'audio/mp3'}));
      });
    },
    createAudio(base64) {
      this.wave = Wave.create({
        container: this.$refs.wave,
        waveColor: '#41b883',
        progressColor: '#088044'
      });

      this.wave.load(base64);

      this.wave.on('ready', () => {
        this.wave.play();
      });

      let EQ = []

      this.wave.on('ready', () => {
        EQ = [
          {
            f: 32,
            type: 'lowshelf'
          },
          {
            f: 64,
            type: 'peaking'
          },
          {
            f: 125,
            type: 'peaking'
          },
          {
            f: 250,
            type: 'peaking'
          },
          {
            f: 500,
            type: 'peaking'
          },
          {
            f: 1000,
            type: 'peaking'
          },
          {
            f: 2000,
            type: 'peaking'
          },
          {
            f: 4000,
            type: 'peaking'
          },
          {
            f: 8000,
            type: 'peaking'
          },
          {
            f: 16000,
            type: 'highshelf'
          }
        ];

        const filters = EQ.map((band) => {
          const filter = this.wave.backend.ac.createBiquadFilter();
          filter.type = band.type;
          filter.gain.value = 0;
          filter.Q.value = 1;
          filter.frequency.value = band.f;
          return filter;
        });

        this.wave.backend.setFilters(filters);

        const dataArray = new Uint8Array(this.wave.backend.analyser.frequencyBinCount);

        this.wave.on('audioprocess', () => {
          this.wave.backend.analyser.getByteFrequencyData(dataArray);
          this.points = [...dataArray];
        });

        const container = this.$refs.equalizer
        filters.forEach((filter) => {
          const input = document.createElement('input');
          Object.assign(input, {
            type: 'range',
            min: -40,
            max: 40,
            value: 0,
            title: filter.frequency.value
          });
          input.style.display = 'inline-block';
          input.setAttribute('orient', 'vertical');
          this.wave.util.style(input, {
            webkitAppearance: 'slider-vertical',
            width: '50px',
            height: '150px'
          });
          container.appendChild(input);

          const onChange = (e) => {
            filter.gain.value = ~~e.target.value;
          };

          input.addEventListener('input', onChange);
          input.addEventListener('change', onChange);
        });

        this.wave.filters = filters;
      });
    },
  }
};
</script>

<template>
  <div>
    <div v-if="path">
      <div ref="wave" class="mt-10"></div>
      <div class="d-flex align-center">
        <svg width="50%" height="300px">
          <circle cx="150" cy="150" :r="points.reduce((a, b)=> a + b, 0) / 1000" fill="#41b883"/>
        </svg>
        <div ref="equalizer" class="mt-2"></div>
      </div>
      <v-btn @click="wave.playPause()" class="player">
        Play/Pause
      </v-btn>
    </div>
    <div v-else class="position_center">
      <div @click="getFilePath()" class="d-flex align-center">
        <v-icon large>mdi-file-upload</v-icon>
        <span class="pl-5">Выберите аудио файл для воспроизведения</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
.position_center {
  position: fixed;
  top: 50%;
  cursor: pointer;
  transform: translateY(-50%);
  left: 50%;
  transform: translateX(-50%);
}

.player {
  position: fixed;
  width: calc(100% - 20px);
  bottom: 10px;
  left: 10px;
  right: 10px;
}
</style>

