<template>
  <f7-page name="home">
    <!-- Top Navbar -->
    <f7-navbar>
      <f7-nav-title>Rock Paper Scissors</f7-nav-title>
    </f7-navbar>
    <div v-show="!loaded" class="text-align-center">
      <div class="preloader"></div>
    </div>
    <f7-block v-show="loaded">
      <f7-row class="justify-content-center text-align-center camera-container">
        <f-card>
          <div class="camera" v-show="!isMobile() && showCamera">
            <video ref="videoRef" playsinline autoplay></video>
          </div>
          <div class="camera" v-show="isMobile() && showCamera">
            <img ref="imageRef" src="static/icons/Nothing.svg">
          </div>
        </f-card>
      </f7-row>
      <f7-row class="justify-content-center">
        <f7-col class="hand-container">
          <div>
            <h1>Player</h1>
            <img class="hand" :src="playerImage">
          </div>
        </f7-col>
        <f7-col class="hand-container">
          <div>
            <h1>Computer</h1>
            <img class="hand" :src="computerImage">
          </div>
        </f7-col>
      </f7-row>
      <f7-row class="justify-content-center">
        <f7-col>
          <f7-button fill raised @click="play">
            <f7-icon :f7="playIcon" :size="36"></f7-icon>
          </f7-button>
        </f7-col>
        <f7-col>
          <f7-button fill raised @click="toggleCamera">
            <f7-icon :f7="toggleCameraIcon" :size="36"></f7-icon>
          </f7-button>
        </f7-col>
      </f7-row>
    </f7-block>
    <f7-block class="result-block">
        <h2 class="vs">{{ vs }}</h2>
        <h3 class="result">{{ result }}</h3>
    </f7-block>
  </f7-page>
</template>

<script>
import { onMounted, ref, computed } from 'vue';
import * as tmImage from '@teachablemachine/image';
import { fetch as fetchPolyfill } from 'whatwg-fetch';

export default {
  setup() {
    // variable and DOM
    const videoRef = ref(null);
    const imageRef = ref(null);
    const showCamera = ref(true);
    const usermediaLoaded = ref(false);
    const firstTimePredicted = ref(false);
    const playIcon = ref('play');
    const toggleCameraIcon = ref('nosign');
    const player = ref('Nothing');
    const computer = ref('Nothing');
    const result = ref(null);
    const vs = ref(null);
    const w = 224; // width for video, image
    const h = 224; // height for video, image
    const predictionInterval = 1000 * 1; // set it lower if the pc and model could run faster
    const gamePlayInterval = 1000 * 1.5; // will play with computer every this interval
    let model;
    let predictionTimer = null;
    let gamePlayTimer = null;
    let totalClasses; // classes from model - nothing, rock, paper, scissors
    let labels;

    onMounted(async () => {
      if (isAndroid()) {
        // overwrite fetch so that it can load model from file system
        window.fetch = fetchPolyfill;
      }
      // load model
      const URL = 'static/model/rock-paper-scissors/';
      const modelURL = URL + 'model.json';
      const metadataURL = URL + 'metadata.json';
      model = await tmImage.load(modelURL, metadataURL);
      // run the first prediction since it takes few seconds. the subsequent predictions are almost realtime.
      totalClasses = model.getTotalClasses();
      labels = model.getClassLabels();
      classify(imageRef.value);
      // start usermedia stream on web and if browser support it
      if (!isMobile() && navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        startUserMedia();
      }
    });

    const play = () => {
      if (isMobile()) {
        predictWithCanvasCamera()
      } else {
        predictWithUsermedia();
      }
    }

    const toggleCamera = () => {
      showCamera.value = !showCamera.value;
      if (!showCamera.value) {
        toggleCameraIcon.value = 'camera';
      } else {
        toggleCameraIcon.value = 'nosign';
      }
    }

    /**
     * Web functions
     */

    const predictWithUsermedia = async () => {
      if (!model || !videoRef.value) {
        return;
      }
      // toggle camera icon and prediction
      if (playIcon.value === 'play') {
        playIcon.value = 'pause';
        // start prediction
        predictionTimer = setInterval(() => {
          classify(videoRef.value);
        }, predictionInterval);
        startGamePlayTimer();
      } else {
        playIcon.value = 'play';
        stopPredictionTimer();
        stopGamePlayTimer();
        resetGame();
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
        usermediaLoaded.value = true;
        videoRef.value.srcObject = stream;
      }

      const handleError = (error) => {
        alert(error.message);
        usermediaLoaded.value = true;
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
      if (playIcon.value === 'play') {
        playIcon.value = 'pause';
        startCanvasCamera();
        // start predicting
        predictionTimer = setInterval(() => {
          if (!imageRef || !imageRef.value || !imageRef.value.src) return;
          classify(imageRef.value);
        }, predictionInterval);
        startGamePlayTimer();
      } else {
        playIcon.value = 'play';
        stopCanvasCamera();
        stopPredictionTimer();
        stopGamePlayTimer();
        resetGame();
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
          cameraFacing: 'front'
      };
      window.plugin.CanvasCamera.start(options, async (error) => {
        alert('could not start canvas camera');
      }, (data) => {
        readImageFile(data);
      });
    }

    const readImageFile = (data) => {
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
                const blob = new Blob([new Uint8Array(reader.result)], { type: 'image/png' });
                imageRef.value.src = window.URL.createObjectURL(blob);
            };
            reader.readAsArrayBuffer(file);
          }, (err) => {
            console.log('read', err);
          });
        }, (error) => {
          console.log(error);
        })
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

    const startGamePlayTimer = () => {
      gamePlayTimer = setInterval(() => {
        computer.value = labels[getRandomClassIndex()];
        calculate();
        setTimeout(() => {
          // change to loading gif after shown what computer has chosen
          computer.value = 'loading';
        }, 500);
      }, gamePlayInterval);
    }

    const classify = (image) => {
      model.predict(image)
        .then(prediction => {
          let maxProbab = 0;
          let index = 0;
          for (let i = 0; i < totalClasses; i++) {
            if (prediction[i].probability.toFixed(2) > maxProbab) {
              maxProbab = prediction[i].probability.toFixed(2);
              index = i;
            }
          }
          if (!firstTimePredicted.value) {
            firstTimePredicted.value = true;
          } else {
            player.value = prediction[index].className;
          }
        });
    }

    const loaded = computed(() => {
      if (isMobile()) {
        return firstTimePredicted.value;
      } else {
        // on web, the app is ready when both usermedia and model is loaded
        return usermediaLoaded.value && firstTimePredicted.value;
      }
    })

    const playerImage = computed(() => {
      return getImage(player.value);
    })

    const computerImage = computed(() => {
      return getImage(computer.value);
    })

    const calculate = () => {
      if (!player.value || player.value === 'Nothing') {
        vs.value = 'Player has not chosen the hand';
        result.value = null;
        return;
      }
      if (!computer.value || computer.value === 'Nothing') {
        vs.value = 'Computer has not chosen the hand';
        result.value = null;
        return;
      }
      vs.value = player.value + ' vs. ' + computer.value;
      let whoWin;
      if (player.value === computer.value) {
        whoWin = 'Draw';
      } else if (player.value === 'Paper') {
        if (computer.value === 'Rock') {
          whoWin = 'Player Win';
        }
      } else if (player.value === 'Rock') {
        if (computer.value === 'Scissors') {
          whoWin = 'Player Win';
        }
      } else if (play.value === 'Scissors') {
        if (computer.value === 'Paper') {
          whoWin = 'Player Win';
        }
      }
      if (!whoWin) {
        whoWin = 'Computer Win';
      }
      result.value = whoWin;
    }

    const resetGame = () => {
      player.value = 'Nothing';
      vs.value = null;
      result.value = null;
      setTimeout(() => {
        // change computer hand to nothing
        computer.value = 'Nothing';
      }, 1000);
    }

    const getImage = (icon) => {
      let extension = '.svg';
      if (icon === 'loading') {
        extension = '.gif';
      }
      return 'static/icons/' + icon + extension;
    }

    const getRandomClassIndex = () => {
      const max = totalClasses - 1;
      return Math.floor(Math.random() * max) + 1;
    }

    const stopPredictionTimer = () => {
        clearInterval(predictionTimer);
        predictionTimer = null;
    }

    const stopGamePlayTimer = () => {
        clearInterval(gamePlayTimer);
        gamePlayTimer = null;
    }

    const isMobile = () => {
      return window.cordova && (isAndroid() || isIos());
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
      videoRef,
      imageRef,
      showCamera,
      playIcon,
      isMobile,
      loaded,
      playerImage,
      computerImage,
      player,
      play,
      toggleCamera,
      toggleCameraIcon,
      computer,
      result,
      vs,
      isIos
    }
  }
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

  div.camera {
    height: 224px;
    width: 224px;
    overflow: hidden;
  }

  img.hand {
    height: 100px;
    width: 100px;
  }

  .hand-container {
    margin-right: 1px;
    margin-left: 1px;
  }

  .camera-container {
    margin-bottom: 5px;
  }

  .result-block {
    text-align: center;
  }

</style>