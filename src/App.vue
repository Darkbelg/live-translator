<template>
  <div>
    <video ref="videoElement" width="640" height="480" autoplay muted></video>
    <button v-if="!mediaStream" @click="startCapture">Start Capture</button>
    <button v-if="mediaStream" @click="stopCapture">Stop Capture</button>
    <button v-if="audioUrl" :disabled="audioPlaying" @click="playAudio">Play Audio</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      mediaStream: null,
      capturedAudio: null,
      audioUrl: null,
      audioPlaying: false,
    };
  },
  methods: {
    async startCapture() {
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

      // Create an array to store audio data chunks
      const audioChunks = [];
      this.capturedAudio.ondataavailable = e => audioChunks.push(e.data);

      // When the audio recording stops, send the audio data to the API and create a URL for the audio file
      this.capturedAudio.onstop = () => {
        this.sendAudioToAPI(audioChunks);
      };

      // Start recording audio
      this.capturedAudio.start();
    },
    stopCapture() {
      console.log('stop Capture');
      if (this.mediaStream) {
        this.mediaStream.getTracks().forEach((track) => track.stop());
        this.mediaStream = null;
      }
      if (this.capturedAudio) {
        this.capturedAudio.stop();
        this.capturedAudio = null;
      }
      this.$refs.videoElement.srcObject = null;
    },
    playAudio() {
      if (this.audioUrl) {
        const audio = new Audio(this.audioUrl);
        audio.play();
        this.audioPlaying = true;
        audio.onended = () => {
          this.audioPlaying = false;
        };
      }
    },
    sendAudioToAPI(audioChunks) {
      const formData = new FormData();
      formData.append('model', 'whisper-1');
      const audioBlob = new Blob(audioChunks, { type: 'audio/webm' });
      formData.append('file', audioBlob, 'recording.webm');
      // formData.append('tranlsations','Chinese');
      fetch('https://api.openai.com/v1/audio/translations', {
        method: 'POST',
        headers: {
          'Authorization': 'Bearer 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',
        },
        body: formData,
      })
        .then(response => response.json())
        .then(data => {
          console.log('Tranlsations:', data);
        })
        .catch(error => {
          console.error('Error:', error);
        });
      const audioUrl = URL.createObjectURL(audioBlob);
      this.audioUrl = audioUrl;
      this.capturedAudio = null;
    },
  },
};
</script>