<template>
  <div>
    <md-dialog :md-active.sync="showDialog" @md-closed="onDialogClose">
      <md-dialog-title>Self-diagnosis</md-dialog-title>
      <md-steppers>
        <md-step
          id="first"
          md-label="Check Internet Speed"
          md-description="Optional"
          md-dynamic-height
        >
          <iframe
            id="first-iframe"
            frameborder="0"
            src="https://cloudcampus.speedtestcustom.com/"
          >
            <p>Your browser does not support iframes.</p>
          </iframe>
        </md-step>

        <md-step id="second" md-label="Check HTTPS Protocol">
          <div v-if="!protocol">
            Your are currently using
            <h2 style="color: #ff5252">HTTP</h2>
            protocol. You cannot use camera or microphone. Please change to
            https prefix.
          </div>
          <div v-if="protocol">
            Your are currently using
            <h2 style="color: #bbdf88">HTTPS</h2>
            protocol. You can use camera or microphone.
          </div>
        </md-step>

        <md-step id="third" md-label="Check Camera">
          <div class="container">
            <div class="row">
              <div class="col-md-6">
                <h2>Current Camera</h2>
                <code v-if="device">{{ device.label }}</code>
                <div class="border">
                  <vue-web-cam
                    ref="webcam"
                    :device-id="deviceId"
                    width="100%"
                    @started="onStarted"
                    @stopped="onStopped"
                    @error="onError"
                    @cameras="onCameras"
                    @camera-change="onCameraChange"
                  />
                </div>

                <div class="row">
                  <div class="col-md-12">
                    <md-field>
                      <label for="device">Device</label>
                      <md-select v-model="camera">
                        <md-option>-- Select Device --</md-option>
                        <md-option
                          v-for="device in devices"
                          :key="device.deviceId"
                          :value="device.deviceId"
                        >
                          {{ device.label }}
                        </md-option>
                      </md-select>
                    </md-field>
                  </div>
                  <div class="col-md-12">
                    <md-button
                      type="button"
                      class="md-raised md-accent"
                      @click="onStop"
                    >
                      Stop Camera
                    </md-button>
                    <md-button
                      class="md-raised md-primary"
                      type="button"
                      @click="onStart"
                    >
                      Start Camera
                    </md-button>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </md-step>

        <md-step id="fouth" md-label="Check Microphone">
          <md-button
            class="md-raised md-primary"
            type="button"
            @click="startAudio"
          >
            Start Microphone
          </md-button>
          <h2>Microphone</h2>
          <div class="pids-wrapper">
            <div v-for="pid in pids" v-bind:key="pid" class="pid"></div>
          </div>
        </md-step>
      </md-steppers>

      <md-dialog-actions>
        <md-button class="md-dense md-raised md-primary" @click="showDialog = false">Close</md-button>
      </md-dialog-actions>
    </md-dialog>

    <md-button class="md-primary md-raised" @click="showDialog = true">Diagnose</md-button
    >
  </div>
</template>

<script>
import Vue from 'vue';
import VueMaterial from 'vue-material';
import 'vue-material/dist/vue-material.min.css';
import 'vue-material/dist/theme/default.css';
import { WebCam } from 'vue-web-cam';

Vue.use(VueMaterial);
Vue.use(WebCam);

export default {
  name: 'DialogCustom',
  components: {
    'vue-web-cam': WebCam,
  },
  data() {
    return {
      showDialog: false,
      protocol: window.location.protocol === 'https:',
      img: null,
      camera: null,
      deviceId: null,
      devices: [],
      pids: new Array(10),
      audioInputDevices: [],
      selectedAudioDevice: {
        input: [],
        output: [],
      },
      audio: null,
    };
  },
  computed: {
    device() {
      return this.devices.find((n) => n.deviceId === this.deviceId);
    },
  },
  watch: {
    camera(id) {
      this.deviceId = id;
    },
    devices() {
      // Once we have a list select the first one
      const [first, ...tail] = this.devices;
      if (first) {
        this.camera = first.deviceId;
        this.deviceId = first.deviceId;
      }
      console.log(tail);
    },
  },
  methods: {
    onStarted(stream) {
      console.log('On Started Event', stream);
    },
    onStopped(stream) {
      console.log('On Stopped Event', stream);
    },
    onStop() {
      this.$refs.webcam.stop();
    },
    onStart() {
      this.$refs.webcam.start();
    },
    onError(error) {
      console.log('On Error Event', error);
    },
    onCameras(cameras) {
      this.devices = cameras;
      console.log('On Cameras Event', cameras);
    },
    onCameraChange(deviceId) {
      this.deviceId = deviceId;
      this.camera = deviceId;
      console.log('On Camera Change Event', deviceId);
    },
    colorPids(vol) {
      const allPids = Array.from(document.querySelectorAll('.pid'));
      const amoutOfPids = Math.round(vol / 10);
      const elemRange = allPids.slice(0, amoutOfPids);

      for (let i = 0; i < allPids.length; i += 1) {
        allPids[i].style.backgroundColor = '#e6e7e8';
      }
      for (let i = 0; i < elemRange.length; i += 1) {
        elemRange[i].style.backgroundColor = '#69ce2b';
      }
    },
    async getDevices() {
      this.audioInputDevices = await navigator.mediaDevices.enumerateDevices();
      console.log(this.audioInputDevices);
      this.audioInputDevices = this.audioInputDevices.filter((device) => device.kind === 'audioinput'
      && device.deviceId !== 'default'
      && device.deviceId !== 'communications');
    },
    startAudio() {
      navigator.mediaDevices.getUserMedia({ audio: true, video: true })
        .then((stream) => {
          this.audio = stream;
          const audioContext = new AudioContext();
          const analyser = audioContext.createAnalyser();
          const microphone = audioContext.createMediaStreamSource(stream);
          const javascriptNode = audioContext.createScriptProcessor(2048, 1, 1);

          analyser.smoothingTimeConstant = 0.8;
          analyser.fftSize = 1024;
          microphone.connect(analyser);
          analyser.connect(javascriptNode);
          javascriptNode.connect(audioContext.destination);
          javascriptNode.onaudioprocess = () => {
            const array = new Uint8Array(analyser.frequencyBinCount);
            analyser.getByteFrequencyData(array);
            let values = 0;

            const { length } = array;
            for (let i = 0; i < length; i += 1) {
              values += (array[i]);
            }

            const average = values / length;
            this.colorPids(average);
          };
        })
        .catch((err) => console.log(err));
    },
    onDialogClose() {
      // Stop camera when exit
      if (this.$refs.webcam) {
        this.$refs.webcam.stop();
      }
      // Stop mic when exit
      if (this.audio !== null) {
        this.audio.getTracks().forEach((track) => track.stop());
      }
    },
  },
  mounted() {

  },
};
</script>

<style lang="scss">
  #first-iframe {
    width: 100%;
    height: 650px;
  }

  #first-iframe {
    -ms-zoom: 1;
    -moz-transform: scale(1);
    -moz-transform-origin: 0 0;
    -o-transform: scale(1);
    -o-transform-origin: 0 0;
    -webkit-transform: scale(1);
    -webkit-transform-origin: 0 0;
  }

  .md-dialog-container {
    overflow: overlay !important;
  }

  .pids-wrapper{
    width: 100%;
  }

  .pid{
    width: calc(10% - 10px);
    height: 10px;
    display: inline-block;
    margin: 5px;
  }

  .md-dialog-actions {
    justify-content: center !important;
  }
</style>
