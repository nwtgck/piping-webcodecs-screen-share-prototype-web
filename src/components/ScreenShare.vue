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
declare const MediaStreamTrackProcessor: any;

import {ref} from "vue";
import urlJoin from "url-join";
import * as cborX from "cbor-x";

// const serverUrl = ref("http://localhost:8080/screensharetest/");
const serverUrl = ref("https://localhost:8443/screensharetest/");
const path = ref("path1");
const myCanvas = ref<HTMLCanvasElement>();

async function share() {
  const stream = await navigator.mediaDevices.getDisplayMedia();
  let chunkNum = 0;

  const init = {
    output: handleChunk,
    error: (e: any) => {
      console.log(e.message);
    },
  };

  const config = {
    codec: "vp8",
    width: 640,
    height: 480,
    bitrate: 2_000_000, // 2 Mbps
    framerate: 30,
  };

  const encoder = new VideoEncoder(init);
  encoder.configure(config);

  let frameCounter = 0;

  const track = stream.getVideoTracks()[0];
  const trackProcessor = new MediaStreamTrackProcessor(track);

  const reader = trackProcessor.readable.getReader();
  while (true) {
    const result = await reader.read();
    if (result.done) break;

    const frame = result.value;
    if (encoder.encodeQueueSize > 2) {
      // Too many frames in flight, encoder is overwhelmed
      // let's drop this frame.
      frame.close();
    } else {
      frameCounter++;
      const keyFrame = frameCounter % 150 == 0;
      encoder.encode(frame, { keyFrame });
      frame.close();
    }
  }

  async function handleChunk(chunk: EncodedVideoChunk, metadata: EncodedVideoChunkMetadata) {
    chunkNum++;

    const chunkData = new Uint8Array(chunk.byteLength);
    chunk.copyTo(chunkData);

    const encoded = cborX.encode({
      timestamp: chunk.timestamp,
      type: chunk.type,
      data: chunkData,
      decoderConfigDescription: metadata.decoderConfig?.description,
    });

    fetch(urlJoin(serverUrl.value, path.value, chunkNum.toString()), {
      method: "POST",
      headers: { "Content-Type": "application/octet-stream" },
      body: encoded,
    });
  }
}

async function view() {
  const ctx = myCanvas.value!.getContext("2d")!;
  let pendingFrames: VideoFrame[] = [];
  let underflow = true;
  let baseTime = 0;
  let chunkNum = 0;
  const init = {
    output: handleFrame,
    error: (e: any) => {
      console.log(e.message);
    },
  };

  const config = {
    codec: "vp8",
    codedWidth: 640,
    codedHeight: 480,
  };
  const decoder = new VideoDecoder(init);
  decoder.configure(config);

  for await (const x of downloadVideoChunks()) {
    const chunk = new EncodedVideoChunk({
      timestamp: x.timestamp,
      type: x.type,
      data: x.data,
    });
    decoder.decode(chunk);
  }
  await decoder.flush();

  async function* downloadVideoChunks() {
    for (let chunkNum = 1; ; chunkNum++) {
      const res = await fetch(urlJoin(serverUrl.value, path.value, chunkNum.toString()), {
      });
      if (!res.ok) {
        throw new Error(`status is not OK: ${res.status}`);
      }
      const arr = new Uint8Array(await res.arrayBuffer());
      const decoded = cborX.decode(arr);
      yield {
        timestamp: decoded.timestamp,
        type: decoded.type,
        data: decoded.data,
        decoderConfigDescription: decoded.decoderConfigDescription,
      };
    }
  }

  function handleFrame(frame: VideoFrame) {
    pendingFrames.push(frame);
    if (underflow) setTimeout(renderFrame, 0);
  }

  function calculateTimeUntilNextFrame(timestamp: number) {
    if (baseTime == 0) baseTime = performance.now();
    let mediaTime = performance.now() - baseTime;
    return Math.max(0, timestamp / 1000 - mediaTime);
  }

  async function renderFrame() {
    underflow = pendingFrames.length == 0;
    if (underflow) return;

    const frame = pendingFrames.shift()!;

    // Based on the frame's timestamp calculate how much of real time waiting
    // is needed before showing the next frame.
    const timeUntilNextFrame = calculateTimeUntilNextFrame(frame.timestamp);
    await new Promise((r) => {
      setTimeout(r, timeUntilNextFrame);
    });
    ctx.drawImage(frame, 0, 0);
    frame.close();

    // Immediately schedule rendering of the next frame
    setTimeout(renderFrame, 0);
  }
}
</script>
