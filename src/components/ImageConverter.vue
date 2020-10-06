<template>
  <div>
    <div>Enter urls in separate lines:</div>
    <textarea v-model="urls" cols="80" rows="10">
    </textarea>
  </div>
  <br/>

  <div>
    <div>Url previews:</div>
    <div v-for="url in getImageUrls" :key="url">
      <img :src="url" :title="url" />
    </div>
  </div>
  <br/>

  <div>
    <button @click="convertToHex()">
      <span v-if="converting">Converting ...</span>
      <span v-if="!converting">Convert</span>
    </button>
  </div>
  <br/>

  <div v-show="converting">Currently processing:<br/><canvas id="c" ref="c"></canvas></div>
  <br/>

  <div v-if="result.length">
    Result bytes: {{ result.length }}, expected: {{ getExpectedSize }}
  </div>
  <div v-if="result.length">
    <textarea v-model="getHexArray"  cols="80" rows="20"></textarea>
  </div>
</template>

<script>
import * as gifuct from "gifuct-js";

export default {
  name: "ImageConverter",
  props: {},
  data() {
    return {
      urls: `/images/police.gif
/images/cat.gif
/images/minecraft.gif
/images/nyan.gif
/images/jump.gif
/images/south-park.gif
/images/deer.gif
https://64.media.tumblr.com/1bdacff5a8358b77c8d4bbe9a7c91cdc/94bda6e78a2135d0-10/s75x75_c1/21fb438b44b8f8c83fbd36f9a7780bf8f51019c7.gifv
https://stat.ameba.jp/user_images/20190604/15/mukkumama/94/04/g/o0064003214425260661.gif`,
      ctx: null,
      gifCanvas: null,
      gifCtx: null,
      tempCanvas: null,
      tempCtx: null,
      frameIndex: 0,
      frames: [],
      frameImageData: null,
      result: [],
      resultFrames: [],
      converting: false,
    };
  },
  mounted() {
    this.ctx = this.$refs.c.getContext("2d");
    this.gifCanvas = document.createElement("canvas");
    this.gifCtx = this.gifCanvas.getContext("2d");
    this.tempCanvas = document.createElement("canvas");
    this.tempCtx = this.tempCanvas.getContext("2d");
  },

  computed: {
    getHexArray() {
      let splitted = "";
      for (let i = 0; i < this.result.length; i++) {
        splitted += this.result[i] + " ";
        if (i % 10 == 9) {
          splitted += "\n";
        }
      }
      let res = `uint8_t animation_lengths[]={ ${this.resultFrames.join(", ")} };
const uint8_t animations[] PROGMEM = {
${splitted}0x00};`;
      return res;
    },
    getImageUrls() {
      let aUrls = this.urls.split("\n");
      aUrls = aUrls.map((i) => i.trim());
      aUrls = aUrls.filter((i) => Boolean(i));
      return aUrls;
    },
    getExpectedSize() {
      const totalFrames = this.resultFrames.reduce((acc, val) => acc + val);
      const frameSize = this.gifCanvas.width * this.gifCanvas.height * 3;

      const size565 = (totalFrames * frameSize) / 3 * 2
      return size565;
    }
  },

  methods: {
    convert888to565(rgb, x = 0, y = 0) {
      // See: https://www.barth-dev.de/about-rgb565-and-how-to-convert-into-it/
      const r = rgb[0];
      const g = rgb[1];
      const b = rgb[2];

      // const x1 = (r & 0xF8) | (g >> 5);
      // const x2 = ((g & 0x1C) << 3) | (b  >> 3);
      // const hexstring_1 = "0x" + x2.toString(16).toUpperCase() +",";
      // const hexstring_2 = "0x" + x1.toString(16).toUpperCase() +",";

      const r_part = (r & 0b11111000) << 8;
      const g_part = (g & 0b11111100) << 3;
      const b_part = b >> 3;

      const rgb565 = r_part + g_part + b_part;
      const hex = rgb565.toString(16).toUpperCase();
      var zerofilled_hex = ("0000" + hex).slice(-4);
      const hexstring_1 = "0x" + zerofilled_hex.slice(2, 4) + ",";
      const hexstring_2 = "0x" + zerofilled_hex.slice(0, 2) + ",";

      return [hexstring_1, hexstring_2];
    },
    getConversion() {
      let res = [];
      for (let yy = 0; yy < this.gifCanvas.height; yy++) {
        for (let xx = 0; xx < this.gifCanvas.width; xx++) {
          const data = this.gifCtx.getImageData(xx, yy, 1, 1).data;
          var rgb = [data[0], data[1], data[2]];
          res = [...res, ...this.convert888to565(rgb)];
        }
      }
      // console.log(res);
      return res;
    },
    renderGIF() {
      if (this.frames.length > 0) {
        this.$refs.c.width = this.frames[0].dims.width;
        this.$refs.c.height = this.frames[0].dims.height;

        this.gifCanvas.width = c.width;
        this.gifCanvas.height = c.height;
      }
    },
    async convertToHex() {
      this.result = [];
      this.resultFrames = [];
      this.converting = true;

      let conversionCounter = 0;

      await this.getImageUrls.forEach(async (gifURL) => {
        try {
          var promisedGif = await fetch(gifURL)
            .then((resp) => resp.arrayBuffer())
            .then((buff) => gifuct.default.parseGIF(buff))
            .then((gif) => gifuct.default.decompressFrames(gif, true));
        }
        catch (e) {
          console.error(e);
          alert(`Could not read image "${gifURL}". Check if url is correct and remote loading is allowed.`)
        }

        console.log("GIF loaded: ", gifURL, promisedGif);
        this.frames = promisedGif;
        this.renderGIF();

        this.frameIndex = 0;
        for (let i = 0; i < this.frames.length; i++) {
          this.renderFrame();
        }

        this.resultFrames.push(this.frames.length);
        conversionCounter++;

        if (conversionCounter == this.getImageUrls.length) {
          this.converting = false;
        }
      });

    },
    renderFrame() {
      // get the frame
      var frame = this.frames[this.frameIndex];

      var start = new Date().getTime();

      if (this.frameIndex === 0) {
        this.gifCtx.clearRect(0, 0, this.$refs.c.width, this.$refs.c.height);
      }

      // draw the patch
      this.drawPatch(frame);

      // perform manipulation
      this.manipulate();

      this.result = [...this.result, ...this.getConversion()];

      // update the frame index
      this.frameIndex++;
      if (this.frameIndex >= this.frames.length) {
        this.frameIndex = 0;
      }

      if (frame.disposalType === 2) {
        this.gifCtx.clearRect(0, 0, this.$refs.c.width, this.$refs.c.height);
      }

      var end = new Date().getTime();
      var diff = end - start;
    },
    drawPatch(frame) {
      var dims = frame.dims;

      if (
        !this.frameImageData ||
        dims.width != this.frameImageData.width ||
        dims.height != this.frameImageData.height
      ) {
        this.tempCanvas.width = dims.width;
        this.tempCanvas.height = dims.height;
        this.frameImageData = this.tempCtx.createImageData(
          dims.width,
          dims.height
        );
      }

      // set the patch data as an override
      this.frameImageData.data.set(frame.patch);

      // draw the patch back over the canvas
      this.tempCtx.putImageData(this.frameImageData, 0, 0);

      this.gifCtx.drawImage(this.tempCanvas, dims.left, dims.top);
    },
    manipulate() {
      var imageData = this.gifCtx.getImageData(
        0,
        0,
        this.gifCanvas.width,
        this.gifCanvas.height
      );
      var other = this.gifCtx.createImageData(
        this.gifCanvas.width,
        this.gifCanvas.height
      );

      // do pixelation
      const pixelPercent = 100; // Do not pixelate
      const width = this.$refs.c.width;
      const height = this.$refs.c.height;
      var pixelsX = 5 + Math.floor((pixelPercent / 100) * (width - 5));
      var pixelsY = (pixelsX * height) / width;

      if (pixelPercent != 100) {
        this.ctx.mozImageSmoothingEnabled = false;
        this.ctx.webkitImageSmoothingEnabled = false;
        this.ctx.imageSmoothingEnabled = false;
      }

      this.ctx.putImageData(imageData, 0, 0);
      this.ctx.drawImage(
        c,
        0,
        0,
        this.$refs.c.width,
        this.$refs.c.height,
        0,
        0,
        pixelsX,
        pixelsY
      );
      this.ctx.drawImage(
        c,
        0,
        0,
        pixelsX,
        pixelsY,
        0,
        0,
        this.$refs.c.width,
        this.$refs.c.height
      );
    },
  },
};
</script>
