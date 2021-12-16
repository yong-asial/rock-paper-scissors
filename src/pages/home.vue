<template>
  <f7-page name="home">
    <!-- Top Navbar -->
    <f7-navbar :sliding="false">
      <f7-nav-title sliding>Fruit freshness</f7-nav-title>
    </f7-navbar>
    <f7-block strong v-show="cameraLoaded">
      <f7-row class="justify-content-center">
        <f7-col v-show="!isAndroid() && !isIphone()">
          <f7-button fill :disabled="!loaded" raised @click="runPrediction">
            <f7-icon f7="camera" :size="36"></f7-icon>
          </f7-button>
        </f7-col>
        <f7-col v-show="isAndroid() || isIphone()">
          <f7-button fill :disabled="!loaded" raised @click="openCamera">
            <f7-icon f7="camera_circle" :size="36"></f7-icon>
          </f7-button>
        </f7-col>
        <f7-col v-show="isAndroid() || isIphone()">
          <f7-button fill :disabled="!loaded" raised @click="openFilePicker">
            <f7-icon f7="folder" :size="36"></f7-icon>
          </f7-button>
        </f7-col>
        <f7-col v-show="isAndroid() || isIphone()">
          <f7-button fill :disabled="!loaded" raised @click="canvasCamera">
            <f7-icon f7="camera_viewfinder" :size="36"></f7-icon>
          </f7-button>
        </f7-col>
      </f7-row>
      <f7-row v-if="result">
        <f7-col class="predict-result">
          <p style="font-size: 1.5rem">{{ result }}</p>
        </f7-col>
      </f7-row>
    </f7-block>
    <f7-block>
      <f7-row class="justify-content-center" v-show="cameraLoaded">
        <f-card class="predict-image">
          <div class="camera" v-show="!isAndroid() && !isIphone()">
            <p>Usermedia</p>
            <video ref="videoRef" playsinline autoplay></video>
          </div>
          <div class="camera" v-show="isAndroid() || isIphone()">
            <p>Image</p>
						<img style="width:480px;height:480px" id="imageFile">
          </div>
          <div class="camera" v-show="!isAndroid() && !isIphone()">
            <p>Canvas</p>
            <canvas id="canvas" ref="canvasRef"></canvas>
          </div>
        </f-card>
      </f7-row>
      <f7-row v-show="!cameraLoaded">
        <f7-col>
          <f7-preloader :size="42"></f7-preloader>
        </f7-col>
      </f7-row>
    </f7-block>
  </f7-page>
</template>
<script>
import { f7, f7ready } from 'framework7-vue';
import * as tf from '@tensorflow/tfjs';
import { onMounted, ref, onBeforeUnmount } from 'vue';
import { fetch as fetchPolyfill } from "whatwg-fetch";

export default {
  setup() {
    const image = ref("");
    const result = ref(null);
    const loaded = ref(false);
    const videoRef = ref(null);
    const canvasRef = ref(null);
    const inPrediction = ref(false);
    const cameraLoaded = ref(false);
    let model;
    let tensor;
    let metadata;

    const items = [
      'Fresh apple',
      'Fresh banana',
      'Fresh orange',
      'Rotten apple',
      'Rotten banana',
      'Rotten orange'
    ];

    const runPrediction = async () => {
      if (!model) {
        return;
      }

      canvasRef.value.width = videoRef.value.videoWidth;
      canvasRef.value.height = videoRef.value.videoHeight;
      canvasRef.value.getContext('2d').drawImage(videoRef.value, 0, 0, canvasRef.value.width, canvasRef.value.height);

      loaded.value = false;
      result.value = false;
      inPrediction.value = true;

      tensor = tf.browser
        .fromPixels(canvasRef.value.getContext('2d').getImageData(0, 0, canvasRef.value.width, canvasRef.value.height))
        .resizeNearestNeighbor([224, 224])
        .expandDims()
        .toFloat();

      const res = await model.predict(tensor).data();
      const index = tf.argMax(res).dataSync();

      result.value = metadata.labels[index];

      setTimeout(() => {
        inPrediction.value = false;
      }, 1000);

      loaded.value = true;
    }

    const constraints = {
      audio: false,
      video: {
        facingMode: 'environment',
        width: { min: 480, ideal: 480, max: 480, exact: 480 },
        height: { min: 480, ideal: 480, max: 480, exact: 480 },
      }
    };

    const handleSuccess = (stream) => {
      cameraLoaded.value = true;
      videoRef.value.srcObject = stream;
    }

    const handleError = (error) => {
      alert(error.message);
      cameraLoaded.value = true;
      console.log('navigator.MediaDevices.getUserMedia error: ', error.message, error.name);
    }

    const isAndroid = () => {
      const userAgent = navigator.userAgent || navigator.vendor || window.opera;
      if (/android/i.test(userAgent)) return true;
      return false;
    }

    const isIphone = () => {
      const userAgent = navigator.userAgent || navigator.vendor || window.opera;
      if (/iPad|iPhone|iPod/i.test(userAgent)) return true;
      return false;
    }

    onMounted(async () => {
      if (isAndroid()) {
        // overwrite fetch so that it can load model from file system
        window.fetch = fetchPolyfill;
      }
      model = await tf.loadLayersModel('static/model/model.json');
      const res = await fetch('static/model/metadata.json');
      metadata = await res.json();
      canvasRef.value.width = 480;
      canvasRef.value.height = 480;
      if ((!isAndroid() && !isIphone()) && navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        navigator.mediaDevices.getUserMedia(constraints).then(handleSuccess).catch(handleError);
      } else {
        cameraLoaded.value = true;
      }
      loaded.value = true;
    });

    onBeforeUnmount(() => {
      if (model) model.dispose();
      if (tensor) tensor.dispose();
    })

    /* - - - - - - - - - - - - - - - -
     Methods for the camera plugin
    - - - - - - - - - - - - - - - - - */
    // Set options for camera
    function setOptions(srcType) {
      const options = {
        quality: 50,
        destinationType: Camera.DestinationType.DATA_URL,
        sourceType: srcType,
        encodingType: Camera.EncodingType.JPEG,
        mediaType: Camera.MediaType.PICTURE,
        allowEdit: false,
        correctOrientation: true,
      };
      return options;
    }

    // Take picture with camera
    function openCamera() {
      try {
        const srcType = Camera.PictureSourceType.CAMERA;
        const options = setOptions(srcType);
        navigator.camera.getPicture(
          function cameraSuccess(imageUrl) {
            displayImage(imageUrl);
          },
          function cameraError(error) {
            console.debug("Unable to take picture: " + error, "app");
          },
          options
        );
      }
      catch {
        console.error("Something went wrong.");
      }
    }

    // Open gallery and choose a picture
    function openFilePicker() {
      try {
        const srcType = Camera.PictureSourceType.SAVEDPHOTOALBUM;
        const options = setOptions(srcType);
        navigator.camera.getPicture(
          function cameraSuccess(imageUrl) {
            // Cordova bug getPicture doesn't work properly
            displayImage(imageUrl);
          },
          function cameraError(error) {
            console.debug("Unable to obtain picture: " + error, "app");
          },
          options
        );
      } catch {
        console.error("Something went wrong.");
      }
    }

    // Add picture to DOM
    async function displayImage(img) {
      let image = document.getElementById("imageFile");
      image.src = "data:image/jpeg;base64," + img;
      // predict
      tensor = tf.browser
        .fromPixels(image)
        .resizeNearestNeighbor([224, 224])
        .expandDims()
        .toFloat();
      const res = await model.predict(tensor).data();
      const index = tf.argMax(res).dataSync();
      result.value = metadata.labels[index];
    }

    document.addEventListener('deviceready', function () {
      console.log('device is ready');
    }, false);

    async function canvasCamera() {
      const canvasEl = document.getElementById('canvas');
      window.plugin.CanvasCamera.initialize(canvasEl);
      const options = {
          canvas: {
            width: 480,
            height: 480
          },
          capture: {
            width: 480,
            height: 480
          },
          use: 'file',
          fps: 30,
          hasThumbnail: false,
          //thumbnailRatio: 1/6,
          cameraFacing: 'back',
          onAfterDraw: function(frame) {
            console.log('afterDraw', frame)
          }
      };
      let predictionInterval = 1000 * 5;
      if (isAndroid()) {
        predictionInterval = 1000 * 10;
      }
      setInterval(async () => {
        // predict
        let image = document.getElementById("imageFile");
        if (!image || !image.src) return;
        tensor = tf.browser
          .fromPixels(image)
          .resizeNearestNeighbor([224, 224])
          .expandDims()
          .toFloat();
        const res = await model.predict(tensor).data();
        const index = tf.argMax(res).dataSync();
        result.value = metadata.labels[index];
      }, predictionInterval);
      window.plugin.CanvasCamera.start(options, async function(error) {
          console.log('[CanvasCamera start]', 'error', error);
      }, function(data) {
        const protocol = 'file://';
        let filepath = '';
        if (isAndroid()) {
          filepath = protocol + data.output.images.fullsize.file;
        } else {
          filepath = data.output.images.fullsize.file;
        }
        window.resolveLocalFileSystemURL(filepath, async function success(fileEntry) {
          fileEntry.file(function (file) {
            let reader = new FileReader();
            reader.onloadend = async function() {
                let blob = new Blob([new Uint8Array(reader.result)], { type: "image/png" });
                let image = document.getElementById("imageFile");
                image.src = window.URL.createObjectURL(blob);
            };
            reader.readAsArrayBuffer(file);
          }, function errorReadFile(err) {
            console.log('read', err);
          });
        }, function error(error) {
          console.log(error);
        })
      });
    }

    return {
      image,
      result,
      loaded,
      videoRef,
      canvasRef,
      runPrediction,
      inPrediction,
      cameraLoaded,
      openCamera,
      openFilePicker,
      canvasCamera,
      isAndroid,
      isIphone
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

  .predict-result {
    border: 1px solid #ddd;
    text-transform: uppercase;
  }

  .predict-image {
    overflow: hidden;
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