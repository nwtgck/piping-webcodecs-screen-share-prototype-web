<template>
  <button @click="share()" style="width: 5rem; height: 5rem;">share</button><br>
  <br>
  <hr>
  <button @click="view()" style="width: 5rem; height: 5rem;">view</button><br>
  <hr>
  <canvas ref="myCanvas" width="640" height="480"></canvas>
</template>

<script setup lang="ts">
// (base: https://developer.chrome.com/articles/webcodecs/)

import {ref} from "vue";
import urlJoin from "url-join";
import * as cborX from "cbor-x";

// const serverUrl = ref("http://localhost:8080/screensharetest/");
const serverUrl = ref("https://localhost:8443/screensharetest/");
const path = ref("path1");
const myCanvas = ref<HTMLCanvasElement>();

async function share() {
  const mediaStream = await navigator.mediaDevices.getDisplayMedia();
  let chunkNum = 0;

  const videoEncoderConfig: VideoEncoderConfig = {
    codec: "vp8",
    width: 640,
    height: 480,
    bitrate: 2_000_000, // 2 Mbps
    framerate: 30,
  };

  const videoEncoder = new VideoEncoder({
    output: handleChunk,
    error: (e: any) => {
      console.log(e.message);
    },
  });
  videoEncoder.configure(videoEncoderConfig);

  let frameCounter = 0;
  const videoTrack = mediaStream.getVideoTracks()[0];
  const videoTrackProcessor = new MediaStreamTrackProcessor({ track: videoTrack });

  const reader = videoTrackProcessor.readable.getReader();
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
      frameCounter++;
      const keyFrame = frameCounter % 150 == 0;
      videoEncoder.encode(frame, { keyFrame });
      frame.close();
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

    fetch(urlJoin(serverUrl.value, path.value, chunkNum.toString()), {
      method: "POST",
      headers: { "Content-Type": "application/octet-stream" },
      body: chunk,
    });
  }
}

async function view() {
  const ctx = myCanvas.value!.getContext("2d")!;
  const pendingVideoFrames: VideoFrame[] = [];
  let underflow = true;
  let baseTime = 0;

  const videoDecoderConfig: VideoDecoderConfig = {
    codec: "vp8",
    codedWidth: 640,
    codedHeight: 480,
  };
  const videoDecoder = new VideoDecoder({
    output: handleFrame,
    error: (e: any) => {
      console.log(e.message);
    },
  });
  videoDecoder.configure(videoDecoderConfig);

  for await (const x of downloadVideoChunks()) {
    const chunk = new EncodedVideoChunk({
      timestamp: x.timestamp,
      type: x.type,
      data: x.data,
    });
    videoDecoder.decode(chunk);
  }
  await videoDecoder.flush();

  async function* downloadVideoChunks() {
    for (let chunkNum = 1; ; chunkNum++) {
      const res = await fetch(urlJoin(serverUrl.value, path.value, chunkNum.toString()), {
      });
      if (!res.ok) {
        throw new Error(`status is not OK: ${res.status}`);
      }
      const chunk = new Uint8Array(await res.arrayBuffer());
      const decoded = cborX.decode(chunk);
      yield {
        timestamp: decoded.timestamp,
        type: decoded.type,
        data: decoded.data,
        decoderConfigDescription: decoded.decoderConfigDescription,
      };
    }
  }

  function handleFrame(frame: VideoFrame) {
    pendingVideoFrames.push(frame);
    if (underflow) {
      setTimeout(renderFrame, 0);
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
