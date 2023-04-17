<template>
  <div id="app">
    <h1>Audio Recorder</h1>
    <button @click="startRecording">Start Recording</button>
    <button @click="stopRecording">Stop Recording</button>
    <audio v-if="audioBlob" controls :src="audioUrl"></audio>
  </div>
</template>

<script>
import lamejs from "lamejs";

export default {
  data() {
    return {
      audioBlob: null,
      audioUrl: "",
      mediaRecorder: null,
      recordedChunks: [],
    };
  },
  methods: {
    async startRecording() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({
          audio: true,
        });
        this.mediaRecorder = new MediaRecorder(stream);
        this.recordedChunks = [];
        this.mediaRecorder.addEventListener("dataavailable", (event) => {
          if (event.data.size > 0) {
            this.recordedChunks.push(event.data);
          }
        });
        this.mediaRecorder.start();
      } catch (error) {
        console.error("Error starting recording:", error);
      }
    },
    stopRecording() {
      this.mediaRecorder.addEventListener("stop", async () => {
        const audioBuffer = await this.concatenateChunksToBuffer(
          this.recordedChunks
        );
        this.audioBlob = await this.convertToMp3(audioBuffer);
        this.audioUrl = URL.createObjectURL(this.audioBlob);
      });
      this.mediaRecorder.stop();
    },
    async concatenateChunksToBuffer(chunks) {
      const audioBuffer = new ArrayBuffer(
        chunks.reduce((totalLength, chunk) => totalLength + chunk.length, 0)
      );
      const audioBufferView = new Uint8Array(audioBuffer);
      let currentPosition = 0;
      chunks.forEach((chunk) => {
        audioBufferView.set(chunk, currentPosition);
        currentPosition += chunk.length;
      });
      return audioBuffer;
    },
    async convertToMp3(audioBuffer) {
      const audioContext = new AudioContext();
      const audioBufferSource = await audioContext.decodeAudioData(audioBuffer);
      const numberOfChannels = audioBufferSource.numberOfChannels;
      const sampleRate = audioBufferSource.sampleRate;
      const samples = audioBufferSource.getChannelData(0);

      const mp3encoder = new lamejs.Mp3Encoder(numberOfChannels, sampleRate, 128);
      const mp3Data = [];

      const blockSize = 1152;
      for (let i = 0; i < samples.length; i += blockSize) {
        const sampleChunk = samples.subarray(i, i + blockSize);
        const mp3Buffer = mp3encoder.encodeBuffer(sampleChunk);

        if (mp3Buffer.length > 0) {
          mp3Data.push(new Int8Array(mp3Buffer));
        }
      }

      const endBuffer = mp3encoder.flush();
      if (endBuffer.length > 0) {
        mp3Data.push(new Int8Array(endBuffer));
      }

      const blob = new Blob(mp3Data, { type: "audio/mp3" });
      return blob;
    }
  },
};
</script>

