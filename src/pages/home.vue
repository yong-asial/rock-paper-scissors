<template>
  <f7-page name="home">
    <!-- Top Navbar -->
    <f7-navbar>
      <f7-nav-title>Fruit Freshness</f7-nav-title>
    </f7-navbar>
    <f7-block>
      <f7-row class="justify-content-center" v-show="loaded">
        <f-card>
          <div class="camera" v-show="!isMobile()">
            <video ref="videoRef" playsinline autoplay></video>
          </div>
          <div class="camera" v-show="isMobile()">
            <img ref="imageRef">
          </div>
          <div class="camera" v-show="showCanvas">
            <canvas id="canvas" ref="canvasRef"></canvas>
          </div>
          <div class="result">{{ result }}</div>
        </f-card>
      </f7-row>
      <f7-row v-show="!loaded">
        <f7-col>
          <f7-preloader :size="42"></f7-preloader>
        </f7-col>
      </f7-row>
    </f7-block>
    <f7-block strong v-show="loaded">
      <f7-row class="justify-content-center">
        <f7-col v-show="!isMobile()">
          <f7-button fill raised @click="predictWithUsermedia">
            <f7-icon :f7="cameraIcon" :size="36"></f7-icon>
          </f7-button>
        </f7-col>
        <f7-col v-show="isMobile()">
          <f7-button fill raised @click="predictWithCanvasCamera">
            <f7-icon :f7="cameraIcon" :size="36"></f7-icon>
          </f7-button>
        </f7-col>
      </f7-row>
    </f7-block>
  </f7-page>
</template>
<script>
import * as tf from '@tensorflow/tfjs';
import { onMounted, ref, onBeforeUnmount } from 'vue';
import { fetch as fetchPolyfill } from "whatwg-fetch";

export default {
  setup() {
    // variable and DOM
    const image = ref("");
    const result = ref(null);
    const videoRef = ref(null);
    const imageRef = ref(null);
    const canvasRef = ref(null);
    const loaded = ref(false);
    const showCanvas = ref(false);
    const cameraIcon = ref('camera');
    const w = 480; // width for canvas, video, image
    const h = 480; // height for canvas, video, image
    let model;
    let tensor;
    let metadata;
    let predictionInterval = 1000 * 5; // set it lower if the pc and model could run faster
    let predictionTimer = null;

    onBeforeUnmount(() => {
      if (model) model.dispose();
      if (tensor) tensor.dispose();
    })

    onMounted(async () => {
      if (isAndroid()) {
        // overwrite fetch so that it can load model from file system
        window.fetch = fetchPolyfill;
        // set prediction interval for android little bit longer
        predictionInterval = 1000 * 10;
      }
      // load model
      model = await tf.loadLayersModel('static/model/model.json');
      const res = await fetch('static/model/metadata.json');
      metadata = await res.json();
      canvasRef.value.width = w;
      canvasRef.value.height = h;
      // start usermedia stream on web and if browser support it
      if (!isMobile() && navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        startUserMedia();
      } else {
        loaded.value = true;
      }
    });

    /**
     * Web functions
     */

    const predictWithUsermedia = async () => {
      if (!model || !videoRef.value || !canvasRef.value) {
        return;
      }
      // toggle camera icon and prediction
      if (cameraIcon.value === 'camera') {
        cameraIcon.value = 'xmark_circle';
        // start prediction
        predictionTimer = setInterval(() => {
          // draw canvas from usermedia stream
          canvasRef.value.width = videoRef.value.videoWidth;
          canvasRef.value.height = videoRef.value.videoHeight;
          canvasRef.value.getContext('2d').drawImage(videoRef.value, 0, 0, canvasRef.value.width, canvasRef.value.height);
          // run classification with the canvas
          classify(canvasRef.value.getContext('2d').getImageData(0, 0, canvasRef.value.width, canvasRef.value.height));
        }, predictionInterval);
      } else {
        cameraIcon.value = 'camera';
        stopPredictionTimer();
      }
    }

    const startUserMedia = () => {
      const constraint = {
        audio: false,
        video: {
          facingMode: 'environment',
          width: { min: w, ideal: w, max: w, exact: w },
          height: { min: h, ideal: h, max: h, exact: h },
        }
      };

      const handleSuccess = (stream) => {
        loaded.value = true;
        videoRef.value.srcObject = stream;
      }

      const handleError = (error) => {
        alert(error.message);
        loaded.value = true;
        console.log('navigator.MediaDevices.getUserMedia error: ', error.message, error.name);
      }

      navigator.mediaDevices.getUserMedia(constraint).then(handleSuccess).catch(handleError);
    }

    /** 
     * Mobile functions
     */

    const predictWithCanvasCamera = () => {
      if (!model || !window.plugin.CanvasCamera || !imageRef) {
        return;
      }
      // toggle camera icon and prediction
      if (cameraIcon.value === 'camera') {
        cameraIcon.value = 'xmark_circle';
        startCanvasCamera();
        // start predicting
        predictionTimer = setInterval(() => {
          if (!imageRef || !imageRef.value || !imageRef.value.src) return;
          classify(imageRef.value);
        }, predictionInterval);
      } else {
        cameraIcon.value = 'camera';
        stopCanvasCamera();
        stopPredictionTimer();
      }
    }

    const startCanvasCamera = () => {
      const options = {
          canvas: {
            width: w,
            height: h
          },
          capture: {
            width: w,
            height: h
          },
          use: 'file',
          fps: 30,
          hasThumbnail: false,
          cameraFacing: 'back'
      };
      window.plugin.CanvasCamera.start(options, async (error) => {
        console.log('[CanvasCamera start]', 'error', error);
      }, (data) => {
        // set file protocol
        const protocol = 'file://';
        let filepath = '';
        if (isAndroid()) {
          filepath = protocol + data.output.images.fullsize.file;
        } else {
          filepath = data.output.images.fullsize.file;
        }
        // read image from local file and assign to image element
        window.resolveLocalFileSystemURL(filepath, async (fileEntry) => {
          fileEntry.file((file) => {
            const reader = new FileReader();
            reader.onloadend = async () => {
                const blob = new Blob([new Uint8Array(reader.result)], { type: "image/png" });
                imageRef.value.src = window.URL.createObjectURL(blob);
            };
            reader.readAsArrayBuffer(file);
          }, (err) => {
            console.log('read', err);
          });
        }, (error) => {
          console.log(error);
        })
      });
    }

    const stopCanvasCamera = () => {
        // stop canvas camera
        window.plugin.CanvasCamera.stop((error) => {
            console.log('[CanvasCamera stop]', 'error', error);
        }, (data) => {
            console.log('[CanvasCamera stop]', 'data', data);
        });
    }

    /**
     * Helper functions
     */

    const classify = (image) => {
      // the first prediction takes a few seconds
      result.value = false;
      tensor = tf.browser
        .fromPixels(image)
        .resizeNearestNeighbor([224, 224])
        .expandDims()
        .toFloat();

      model.predict(tensor).data()
        .then(res => {
          const index = tf.argMax(res).dataSync();
          result.value = metadata.labels[index];
        });

    }

    const stopPredictionTimer = () => {
        clearInterval(predictionTimer);
        predictionTimer = null;
    }

    const isMobile = () => {
      return isAndroid() || isIos();
    }

    const isAndroid = () => {
      const userAgent = navigator.userAgent || navigator.vendor || window.opera;
      if (/android/i.test(userAgent)) return true;
      return false;
    }

    const isIos = () => {
      const userAgent = navigator.userAgent || navigator.vendor || window.opera;
      if (/iPad|iPhone|iPod/i.test(userAgent)) return true;
      return false;
    }

    return {
      image,
      result,
      videoRef,
      imageRef,
      canvasRef,
      predictWithUsermedia,
      loaded,
      predictWithCanvasCamera,
      showCanvas,
      cameraIcon,
      isMobile
    }
  },
}
</script>
<style scoped>
  div[class*="col"] {
    background: #fff;
    text-align: center;
    color: #000;
    padding: 5px;
    margin-bottom: 16px;
    font-size: 12px;
  }

  .result {
    font-weight: bold;
    font-size: 15px;
    text-align: center;
    padding: 5px;
    margin-top: 10px;
  }

  div.camera {
    width: 100%;
    height: 100%;
    overflow: hidden;
  }

  canvas, video, img {
    height: 480px;
    width: 480px;
  }
</style>