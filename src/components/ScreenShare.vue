<template>
  <input v-model="serverUrl" size="60"><br>
  <input v-model="path" size="60"><br>
  <button @click="share()" style="width: 5rem; height: 5rem;">share</button><br>
  <hr>
  <button @click="view()" style="width: 5rem; height: 5rem;">view</button><br>
  <hr>
  <canvas ref="myCanvas" :width="width" :height="height"></canvas>
</template>

<script setup lang="ts">
// (base: https://developer.chrome.com/articles/webcodecs/)

import {ref} from "vue";
import urlJoin from "url-join";
import * as cborX from "cbor-x";
import {AsyncSemaphore} from "@esfx/async-semaphore";

// const serverUrl = ref("http://localhost:8080/screensharetest/");
// const serverUrl = ref("https://localhost:8443/screensharetest/");
const serverUrl = ref("https://ppng.io/screensharetest1/");
const path = ref("path1");
const myCanvas = ref<HTMLCanvasElement>();

const width = 1280;
const height = 720;

async function share() {
  const mediaStream = await navigator.mediaDevices.getDisplayMedia();
  let chunkNum = 0;

  const videoEncoderConfig: VideoEncoderConfig = {
    codec: "vp8",
    width: width,
    height: height,
    bitrate: 5_000_000, // 4 Mbps
    framerate: 30,
  };

  const videoEncoder = new VideoEncoder({
    output: handleChunk,
    error: (e: any) => {
      console.log(e.message);
    },
  });
  videoEncoder.configure(videoEncoderConfig);

  let keyFrameDecider = 0;
  const videoTrack = mediaStream.getVideoTracks()[0];
  const videoTrackProcessor = new MediaStreamTrackProcessor({ track: videoTrack });

  receiveKeyFrameRequestLoop();

  const reader = videoTrackProcessor.readable.getReader();
  let lastFrame: VideoFrame | undefined;
  while (true) {
    const result = await reader.read();
    if (result.done) {
      break;
    }
    const frame = result.value;
    if (videoEncoder.encodeQueueSize > 2) {
      // Too many frames in flight, encoder is overwhelmed
      // let's drop this frame.
      frame.close();
    } else {
      const keyFrame = keyFrameDecider === 0;
      keyFrameDecider = (keyFrameDecider + 1) % 150;
      videoEncoder.encode(frame, { keyFrame });
      lastFrame?.close();
      lastFrame = frame;
    }
  }

  async function receiveKeyFrameRequestLoop() {
    while (true) {
      try {
        const res = await fetch(urlJoin(serverUrl.value, path.value, "keyframe"));
        if (res.status !== 200) {
          throw new Error("not OK status");
        }
        await res.text();
        console.log("Encoding key frame...");
        if (lastFrame !== undefined) {
          videoEncoder.encode(lastFrame, { keyFrame: true });
          // Avoid too many key frames
          keyFrameDecider = 1;
        }
      } catch (e) {
      }
      await new Promise(resolve => setTimeout(resolve, 2000));
    }
  }

  async function handleChunk(encodedVideoChunk: EncodedVideoChunk, metadata: EncodedVideoChunkMetadata) {
    chunkNum++;

    const encodedVideoChunkData = new Uint8Array(encodedVideoChunk.byteLength);
    encodedVideoChunk.copyTo(encodedVideoChunkData);

    const chunk: Uint8Array = cborX.encode({
      timestamp: encodedVideoChunk.timestamp,
      type: encodedVideoChunk.type,
      data: encodedVideoChunkData,
      decoderConfigDescription: metadata.decoderConfig?.description,
    });

    try {
      const res = await fetch(urlJoin(serverUrl.value, path.value, chunkNum.toString()), {
        method: "POST",
        headers: { "Content-Type": "application/octet-stream" },
        body: chunk,
      });
      if (res.status !== 200) {
        console.debug(`failed to send chunk #${chunkNum}. status=${res.status}`);
      }
    } catch (e) {
      // Discarding chunk is OK because it is a frame
    }
  }
}

async function view() {
  const ctx = myCanvas.value!.getContext("2d")!;
  const pendingVideoFrames: VideoFrame[] = [];
  let underflow = true;
  let baseTime = 0;
  const downloadSemaphore = new AsyncSemaphore(3);
  let lastHandledTimestamp = -1; // first frame timestamp can be 0
  let keyFrameRequesting = false;

  const videoDecoderConfig: VideoDecoderConfig = {
    codec: "vp8",
    codedWidth: width,
    codedHeight: height,
  };
  const videoDecoder = new VideoDecoder({
    output: handleFrame,
    error: (e: any) => {
      console.log(e.message);
    },
  });
  videoDecoder.configure(videoDecoderConfig);

  (async () => {
    for (let chunkNum = 1; ; chunkNum++) {
      await downloadSemaphore.wait();
      downloadVideoChunk(chunkNum).then(chunk => {
        if (chunk === undefined) {
          return;
        }
        const encodedVideoChunk = new EncodedVideoChunk({
          timestamp: chunk.timestamp,
          type: chunk.type,
          data: chunk.data,
        });
        videoDecoder.decode(encodedVideoChunk);
      }).finally(() => {
        downloadSemaphore.release();
      });
    }
  })().then();
  await videoDecoder.flush();

  async function downloadVideoChunk(chunkNum: number): Promise<{ timestamp: number, type: "key" | "delta", data: Uint8Array, decoderConfigDescription?: unknown } | undefined> {
    try {
      const res = await fetch(urlJoin(serverUrl.value, path.value, chunkNum.toString()), {
      });
      if (!res.ok) {
        console.debug(`failed to receive a chunk #${chunkNum}. status=${res.status}`);
        // Discard chunk
        return
      }
      const chunk = new Uint8Array(await res.arrayBuffer());
      const decoded = cborX.decode(chunk);
      if (decoded.type === "key") {
        console.log(decoded.timestamp, "key frame chunk arrived");
      }
      return {
        timestamp: decoded.timestamp,
        type: decoded.type,
        data: decoded.data,
        decoderConfigDescription: decoded.decoderConfigDescription,
      };
    } catch (e) {
      console.debug(`failed to receive a chunk #${chunkNum}`, e);
      // Discard chunk
    }
  }

  function handleFrame(frame: VideoFrame) {
    if (lastHandledTimestamp > frame.timestamp) {
      requestKeyFrame();
    }
    lastHandledTimestamp = frame.timestamp;
    pendingVideoFrames.push(frame);
    if (underflow) {
      setTimeout(renderFrame, 0);
    }
  }

  async function requestKeyFrame() {
    if (keyFrameRequesting) {
      console.log("already key frame requesting");
      return;
    }
    keyFrameRequesting = true;
    while (true) {
      try {
        const res = await fetch(urlJoin(serverUrl.value, path.value, "keyframe"), {
          method: "POST",
        });
        if (res.status !== 200) {
          throw new Error("not OK status");
        }
        await res.text();
        console.log("key frame requested");
        keyFrameRequesting = false;
        break;
      } catch (e) {
        console.warn("failed to request a key frame", e);
      }
      await new Promise(resolve => setTimeout(resolve, 2000));
    }
  }

  function calculateTimeUntilNextFrame(timestamp: number): number {
    if (baseTime == 0) {
      baseTime = performance.now();
    }
    const mediaTime = performance.now() - baseTime;
    return Math.max(0, timestamp / 1000 - mediaTime);
  }

  async function renderFrame() {
    underflow = pendingVideoFrames.length == 0;
    if (underflow) {
      return;
    }

    const videoFrame = pendingVideoFrames.shift()!;

    // Based on the frame's timestamp calculate how much of real time waiting
    // is needed before showing the next frame.
    const timeUntilNextFrame = calculateTimeUntilNextFrame(videoFrame.timestamp);
    await new Promise((r) => {
      setTimeout(r, timeUntilNextFrame);
    });
    ctx.drawImage(videoFrame, 0, 0);
    videoFrame.close();

    // Immediately schedule rendering of the next frame
    setTimeout(renderFrame, 0);
  }
}
</script>
