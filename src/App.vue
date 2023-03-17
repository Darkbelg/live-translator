<template>
  <div>
    <label for="apiKey">API Key:</label>
    <input id="apiKey" type="text" v-model="apiKey">
    <video ref="videoElement" width="640" height="480" autoplay muted></video>
    <button v-if="!mediaStream" @click="startCapture">Start Capture</button>
    <button v-if="mediaStream" @click="stopCapture">Stop Capture</button>
    <ul>
      <li v-for="line, index in translation" v-bind:key="index">
        {{ line }}
      </li>
    </ul>
  </div>
</template>
<script>
export default {
  data() {
    return {
      mediaStream: null,
      capturedAudio: null,
      audioPlaying: false,
      intervalId: null, // Keep track of the interval id
      intervalSend: null,
      translation: [],
      audioChunksArray: [],
      apiKey: localStorage.getItem('apiKey') || '',
    };
  },
  methods: {
    async startCapture() {
      this.translation = [];
      const displayMediaOptions = {
        audio: true,
        video: true,
      };
      this.mediaStream = await navigator.mediaDevices.getDisplayMedia(displayMediaOptions);
      this.$refs.videoElement.srcObject = this.mediaStream;

      // Create separate streams for audio and video
      const audioStream = new MediaStream();
      const videoStream = new MediaStream();
      this.mediaStream.getTracks().forEach(track => {
        if (track.kind === 'audio') {
          audioStream.addTrack(track);
        } else {
          videoStream.addTrack(track);
        }
      });

      // Configure the audio recorder to exclude the video track
      const audioTrack = audioStream.getAudioTracks()[0];
      const audioSettings = audioTrack.getSettings();
      audioSettings.sampleRate = 48000; // Set the sample rate to 48 kHz
      const audioRecorderOptions = {
        mimeType: 'audio/webm',
        audioBitsPerSecond: 128000, // Set the bitrate to 128 kbps
        audioChannels: 2, // Set the number of audio channels to 2 (stereo)
      };
      this.capturedAudio = new MediaRecorder(audioStream, audioRecorderOptions);

      // Start recording audio
      this.capturedAudio.start();
      console.log('start');

      // Set an interval to stop recording after 1 seconds
      this.intervalId = setInterval(() => {
        this.capturedAudio.stop();
        // console.log(this.audioChunksArray.slice(-20));
        this.sendAudioToAPI();
        console.log('stop');
      }, 6000);


      this.capturedAudio.ondataavailable = e => {
        console.log("restart");
        this.audioChunksArray.push(e.data);
        // console.log( this.audioChunksArray);
        this.capturedAudio.start();
      };
    },
    stopCapture() {
      console.log('stop Capture');

      if (this.mediaStream) {
        this.mediaStream.getTracks().forEach((track) => track.stop());
        this.mediaStream = null;
      }
      if (this.capturedAudio) {
        this.capturedAudio.ondataavailable = null; // Remove the event listener
        this.capturedAudio.stop();
        this.capturedAudio = null;
      }
      if (this.intervalId) {
        clearInterval(this.intervalId);
        this.intervalId = null;
      }
      this.$refs.videoElement.srcObject = null;
    },
    sendAudioToAPI() {
      const mergedAudioBlob = new Blob(this.audioChunksArray, { type: 'audio/webm' });
      const formData = new FormData();
      formData.append('model', 'whisper-1');
      formData.append('file', mergedAudioBlob, 'recording.webm');

      fetch('https://api.openai.com/v1/audio/translations', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${this.apiKey}`,
        },
        body: formData,
      })
        .then(response => response.json())
        .then(data => {
          console.log('Translations:', data);
          this.translation.unshift(data.text);
        })
        .catch(error => {
          console.error('Error:', error);
        });
      this.audioChunksArray = [];
    },
  },
  watch: {
    apiKey: function (newApiKey) {
      localStorage.setItem('apiKey', newApiKey);
    }
  }
};
</script>