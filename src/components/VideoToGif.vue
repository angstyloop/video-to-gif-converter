<template>
<div class='video-to-gif'>
  <template v-if="isDone('video')">
    <div class="margin-bottom">
      <video autoplay loop width="278" :src="videoUrl"></video>
    </div>
    <button class="margin-right" @click="convert">Convert</button>
  </template>
  <template v-else-if="isPending('video')">
    <div>
      <span>Loading...</span>
    </div>
  </template>
  <template v-else>
    <div class="margin-bottom">
      <h1>Convert to GIF</h1>
    </div>
  </template>
  <input type="file" @change="onVideoChange($event)">
  <template v-if="isDone('gif')">
    <div>
      <h2>Enjoy your GIF!</h2>
      <img :src="gifUrl" width=278>
    </div>
  </template>
  <div class="margin-top">
    <span class="italic">Powered by <a href="https://github.com/ffmpegwasm/ffmpeg.wasm">ffmpegwasm by jeromewu</a>.</span>
  </div>
</div>
</template>

<script>
import {createFFmpeg, fetchFile} from '@ffmpeg/ffmpeg'

const NOT_STARTED = 0;
const PENDING = 1;
const DONE = 2;

export default {
  name: 'video-to-gif',
  data () {
    return {
      loadStatus: {
          ffmpeg: NOT_STARTED,
          video: NOT_STARTED,
          gif: NOT_STARTED,
      },
      ffmpeg: null,
      gifUrl: '',
      data: {buffer: null},
      videoName: 'in.mp4',
      gifName: 'out.gif',
    }
  },
  async mounted() {
    this.ffmpeg = await createFFmpeg({log: true})
    this.setPending('ffmpeg')
    await this.ffmpeg.load()
    this.setDone('ffmpeg')
  },
  destroyed() {
      URL.revokeObjectURL(this.videoUrl)
      URL.revokeObjectURL(this.gifUrl)
  },
  methods: {
    isPending(name) {
        return this.loadStatus[name] == PENDING
    },
    isDone(name) {
        return this.loadStatus[name] == DONE
    },
    setPending(name) {
        this.$set(this.loadStatus, name, PENDING)
    },
    setDone(name) {
        this.$set(this.loadStatus, name, DONE)
    },
    onVideoChange(event) {
      this.setPending('video')
      this.videoFile = event.target.files[0]
      const videoBlob = new Blob([this.videoFile], {type: 'video/mp4'})
      this.videoUrl = URL.createObjectURL(videoBlob)
      this.setDone('video')
    },
    async convert() {
      if (! this.isDone('video')) {
        console.warn("FFMPEG isn't ready.")
        return 
      }
      this.setPending('gif')
      await this.ffmpeg.FS('writeFile', this.videoName, await fetchFile(this.videoFile))
      await this.ffmpeg.run('-i', this.videoName, '-t', '2.5', '-ss', '2.0', '-f', 'gif', this.gifName)
      this.data = await this.ffmpeg.FS('readFile', this.gifName)
      this.gifUrl = await URL.createObjectURL(new Blob([this.data.buffer, {type: 'image/gif'}]))
      this.setDone('gif')
    }
  }
}
</script>
<style scoped>
.margin-top {
  margin-top: 1em;
}
.margin-bottom {
  margin-bottom: 1em;
}
.margin-right {
  margin-right: 1em;
}
.italic {
    font-style: italic;
    font-size: 1em;
}
</style>
