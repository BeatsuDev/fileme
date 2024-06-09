<script setup lang="ts">
import { generateSlug } from "random-word-slugs";
import { Peer } from "peerjs";
import { ref, onMounted } from "vue";
import FileSelector from "./components/FileSelector.vue";

// File selection logic
const selectedFile = ref<File | null>(null);

// Peer logic
const progressElement = ref<HTMLProgressElement | null>(null);
const slugInput = ref<HTMLInputElement | null>(null);
const isReadyForReceiving = ref(false);
const MAGIC_SALT = "bUYIODVF87bdfvsidf6vb78sd76vbA96BS7DV6a7d";
const peerSlug = ref(generateSlug(2));
const peerSlugId = ref<string | null>(null);

function slugToId(slug: string) {
  const base64 = btoa(MAGIC_SALT + slug);
  return base64.replace(/=+$/, "");
}

let receivedFiles = ref<HTMLAnchorElement[]>([]);
let receivingPeer: Peer | null = null;
let metadata: {
  name: string;
  size: number;
  type: string;
  lastModified: number;
} | null = null;
let receiveBuffer: Uint8Array[] = [];
let receiveProgress = 0;
let receiveSize = 0;
onMounted(() => {
  setTimeout(() => {
    receivingPeer = new Peer(slugToId(peerSlug.value), {debug: 3});
  
    receivingPeer.on("open", id => {
      peerSlugId.value = id;
      isReadyForReceiving.value = true;
    });
    
    receivingPeer.on("connection", conn => {
      conn.on("data", data => {
        if (typeof data === "string") {
          metadata = JSON.parse(data.replace(/^data: /, ""));
          receiveSize = metadata!.size;
          progressElement.value!.max = receiveSize;
          console.log("Received metadata", metadata);
        } else {
          if (data instanceof ArrayBuffer) data = new Uint8Array(data);

          if (!(data instanceof Uint8Array)) {
            console.error("Received data is not a Uint8Array", data);
            return;
          }

          receiveBuffer.push(data);
          receiveProgress += data.byteLength;
          progressElement.value!.value = receiveProgress;
          
          if (receiveProgress === receiveSize) {
            const blob = new Blob(receiveBuffer, {type: metadata!.type});
            const url = URL.createObjectURL(blob);
            const a = document.createElement("a");
            a.href = url;
            a.download = metadata!.name;
            receiveBuffer = [];
            receiveProgress = 0;

            receivedFiles.value.push(a);
          }
        }
      });
    });
  }, 0);
});

// Send file logic
let sendingPeer: Peer | null = null;
onMounted(() => {
  sendingPeer = new Peer(
    {
      debug: 3,
      config: {
        iceServers: [
          {urls: "stun:stun.l.google.com:19302"},
          {urls: "stun:stun1.l.google.com:19302"},
          {urls: "stun:stun2.l.google.com:19302"},
          {urls: "stun:stun3.l.google.com:19302"},
          {urls: "stun:stun4.l.google.com:19302"},
        ],
      }
    }
  );
})

function sendFileClicked() {
  if (!selectedFile.value) return alert("No file selected");
  sendFile(selectedFile.value);
}

function sendFile(file: File) {
  if (!sendingPeer) return alert("Peer not ready");
  if (!slugInput.value) return alert("No receiver ID element on the page. Please refresh the page.");
  if (!slugInput.value?.value) return alert("Receiver ID is empty");

  console.log("Connecting to", slugToId(slugInput.value.value));
  const conn = sendingPeer.connect(slugToId(slugInput.value.value));
  conn.on("open", () => {
    // Send metadata first
    const data = `data: ${JSON.stringify({
      name: file.name,
      size: file.size,
      type: file.type,
      lastModified: file.lastModified,
    })}`;
    console.log("Sending metadata", data);
    conn.send(data);

    // Send file chunks
    const reader = chunkedReader(file);
    for (const chunk of reader()) {
      console.log("Sending chunk", chunk.size);
      conn.send(chunk);
    }
  });
}

function chunkedReader(file: File, chunkSize: number = 1024 * 1024 * 1024) {
  return function* () {
    let offset = 0;
    while (offset < file.size) {
      const chunk = file.slice(offset, offset + chunkSize);
      offset += chunkSize;
      yield chunk;
    }
  };
}
</script>

<template>
  <main>
    <h1>Peer-to-peer file sharing</h1>
    <div id="container">
      <div id="receiving-container">
        <div style="font-size: 1.5em;">Your peer ID: <code style="font-size: 1.5em" v-if="isReadyForReceiving">{{ peerSlug }}</code></div>
        <div id="progress-container">
          Receiving progress:
          <progress ref="progressElement" min="0"></progress>
        </div>
        <a v-for="file in receivedFiles" :href="file.href" :download="file.download">Save: {{ file.download }}</a>
      </div>
      <div id="sending-container">
        <label for="slug-input">Receivers ID: <input ref="slugInput" type="text"></label>
        <FileSelector v-model="selectedFile" />
        <div>Selected file: <span id="selected-files-span">{{ selectedFile?.name ?? "" }}</span></div>
        <button @click="sendFileClicked">Send file</button>
      </div>
    </div>
  </main>
</template>

<style scoped>
#container {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20px;
}

@media (max-width: 800px) {
  #container {
    flex-direction: column;
    gap: 150px;
  }
}

#receiving-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

#sending-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
}
</style>
