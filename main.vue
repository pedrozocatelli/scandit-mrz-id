<script setup>
import {
  Camera,
  CameraSwitchControl,
  DataCaptureContext,
  DataCaptureView,
  FrameSourceState,
  configure,
} from 'scandit-web-datacapture-core';
import {
  IdCapture,
  IdCaptureOverlay,
  IdCaptureSettings,
  IdDocumentType,
  IdImageType,
  idCaptureLoader,
} from 'scandit-web-datacapture-id';

import { onMounted, ref, getCurrentInstance, onUnmounted } from 'vue';

import IconCaretOutline from '@components/icons/IconBackCaret.vue';

const elRefs = ref(null);
const view = ref(null);
const context = ref(null);
const idCapture = ref(null);
const camera = ref(null);

const supportedDocuments = ref([
  IdDocumentType.DLVIZ,
  IdDocumentType.IdCardVIZ,
  IdDocumentType.PassportMRZ,
  IdDocumentType.USUSIdBarcode,
]);

const removeIdCaptureListener = (idCaptureInstance) => {
  if (idCaptureInstance) {
    idCaptureInstance.listeners = [];
  }
};

onMounted(async () => {
  try {
    context.value = null;
    idCapture.value = null;
    camera.value = null;
    view.value = null;

    const internalInstance = getCurrentInstance();
    elRefs.value = internalInstance?.refs.elRefs;

    view.value = new DataCaptureView();
    view.value.connectToElement(elRefs.value);
    view.value.showProgressBar();

    await configure({
      licenseKey:
        'MY-KEY',
      libraryLocation: new URL(
        'library/engine',
        window.location.origin,
      ).toString(),
      moduleLoaders: [idCaptureLoader({ enableVIZDocuments: true })],
    });

    view.value.hideProgressBar();

    context.value = await DataCaptureContext.create();
    camera.value = Camera.default;

    await camera.value.applySettings(IdCapture.recommendedCameraSettings);
    await context.value.setFrameSource(camera.value);

    await view.value.setContext(context.value);

    view.value.addControl(new CameraSwitchControl());

    // Create the IdCapture settings needed for the selected mode
    const settings = new IdCaptureSettings();
    settings.supportedDocuments = supportedDocuments.value;
    settings.setShouldPassImageTypeToResult(IdImageType.Face, true);

    // Create the IdCapture mode with the new settings
    idCapture.value = await IdCapture.forContext(context.value, settings);

    idCapture.value.addListener({
      didCaptureId: async (idCaptureInstance, session) => {
        // Disable the IdCapture mode to handle the current result
        await idCapture.value?.setEnabled(false);

        const capturedId = session.newlyCapturedId;
        console.log(capturedId);
        void idCapture.value.reset();
      },
      didRejectId: async () => {
        console.log('it did not read properly');
        // await idCapture.value?.setEnabled(false);
      },
    });

    await IdCaptureOverlay.withIdCaptureForView(idCapture.value, view.value);

    // Disable the IdCapture mode until the camera is accessed
    await idCapture.value?.setEnabled(false);

    // Finally, switch on the camera
    await camera.value.switchToDesiredState(FrameSourceState.On);
    await idCapture.value?.setEnabled(true);
  } catch (error) {
    console.error(error);
  }
});

onUnmounted(async () => {
  if (idCapture.value) {
    await camera.value.switchToDesiredState(FrameSourceState.Off);
    removeIdCaptureListener(idCapture.value);
    context.value.removeMode(idCapture.value);
    void idCapture.value.reset();
    view.value.detachFromElement(elRefs.value);
  }
});
</script>

<template>
  <div class="relative">
    <div class="dce-scanarea border-radius mb-10" style="min-height: 80px">
      <a class="top left" @click="$router.go(-1)">
        <div class="camera-button">
          <IconCaretOutline fill="#fff" />
        </div>
      </a>
      <div class="camera-title">
        <h1>Scan the barcode on the <b>back</b> of the ID or the Passport</h1>
      </div>
    </div>
  </div>
  <div ref="elRefs" class="component-barcode-scanner"></div>
</template>

<style scoped lang="scss">
*:focus {
  outline: none;
}

.container {
  max-width: 100%;
}

h1 {
  font-size: 1.5rem;
  font-weight: 400;
}
.component-barcode-scanner {
  width: 100%;
  height: 100%;
  /* min-width: 640px; */
  min-height: 480px;
  /* background: #eee; */
  position: relative;
  resize: both;
  border-radius: 20px;
}

.dce-video-container {
  /* position: absolute;
  left: 0;
  top: 0; */
  width: 100%;
  /* min-width: 80%; */
  /* width: auto; */
  /* max-width: 100%; */
  height: 100%;
  border-radius: 20px !important;
}

.border-radius {
  width: 100%;
  height: 100%;
  border-radius: 20px !important;
  -webkit-border-radius: 20px;
  -moz-border-radius: 20px;
  border-radius: 20px;
  -webkit-transform: translateZ(0);
  -webkit-mask-image: -webkit-radial-gradient(circle, white 100%, black 100%);
  -khtml-border-radius: 20px;
}

.dce-scanarea {
  width: 100%;
  height: 100%;
  /* position: absolute; */
  left: 0;
  top: 0;
  overflow: hidden;
  border-radius: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  margin: auto;
  -webkit-border-radius: 20px;
  -moz-border-radius: 20px;
  border-radius: 20px;
  -khtml-border-radius: 20px;
}

.dce-scanlight {
  display: none;
  width: 100%;
  height: 70px;
  margin-top: -55px;
  position: absolute;
  animation: dce-scanlight 3s infinite;
  background-image: linear-gradient(
    rgba(255, 255, 255, 0),
    rgb(189, 243, 212, 0.5),
    rgb(189, 243, 212)
  );
  border-bottom-width: 3px;
  border-bottom-style: solid;
  border-bottom-color: rgb(82, 177, 134);
}

.div-select-container {
  /* position: absolute; */
  left: 0;
  top: 0;
}

.dce-sel-camera {
  display: block;
}

.select {
  font-size: 1rem;
  width: 300px;
  border: 1px solid #fff;
  border-radius: 10px;
  padding: 1rem;
  background-size: 30px;
  background-position: calc(100% - 10px);
}

.camera-select {
  height: 50px;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 10px;
  background-color: rgba(0, 0, 0, 0.25);
  select {
    height: 50px;
    padding: 0 2rem 0 1rem;
    border-radius: 10px;
    background-position: calc(100% - 10px);
    background-size: 12px;
    &:hover {
      cursor: pointer;
    }
  }
  &:hover {
    background-color: rgba(256, 256, 256, 0.25);
    cursor: pointer;
    border: none;
    outline: none;
  }
  &:focus {
    background-color: #121212;
  }
  &:active {
    background-color: #121212;
  }
}

.dce-sel-camera {
  border: none;
  &:active {
    background-color: #121212;
    border: none;
  }
}

.camera-link {
  height: 36px;
  &:hover {
    background-color: rgba(0, 0, 0, 0.25);
    color: #fff;
  }
  &:active {
    border: none;
    box-shadow: none;
  }
}

.dce-sel-resolution {
  display: block;
  margin-top: 5px;
}

.camera-title {
  position: absolute;
  // top: 7%;
  bottom: 6%;
  padding: 0 0.5rem;
  border-radius: 5px;
  /* background-color: rgba(256, 256, 256, .25); */
  background-color: rgba(0, 0, 0, 0.25);
  max-width: 60%;
  /* transform: translateX(-50%); */
}

.camera-button {
  position: absolute;
  top: 6%;
  // left: 5%;
  height: 50px;
  width: 50px;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 10px;
  background-color: rgba(0, 0, 0, 0.25);
  &:hover {
    background-color: rgba(256, 256, 256, 0.25);
    cursor: pointer;
  }
  &:active {
    background-color: #121212;
  }
  /* transform: translateX(-50%); */
}

.top {
  top: 6%;
  position: absolute;
  margin-bottom: 1rem;
}

.bottom {
  bottom: 6%;
  position: absolute;
  margin-top: 1rem;
}
.right {
  right: 4%;
  position: absolute;
  margin-left: 1rem;
}
.left {
  left: 4%;
  position: absolute;
  margin-right: 1rem;
}

.center {
  left: 50%;
}

@keyframes dce-scanlight {
  from {
    top: 0;
  }

  to {
    top: 97%;
  }
}

@keyframes dbrScanner-scanlight {
  from {
    top: 0;
  }

  to {
    top: 97%;
  }
}
</style>
