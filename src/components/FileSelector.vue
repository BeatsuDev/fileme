<template>
    <label class="file-drop-selector" for="file" @dragover.prevent @drop.prevent="onFileDropped">
        <input ref="fileInputEl" @change="onFileInputChanged" type="file" id="file" style="display: none" />
        Select or drop a file here.
    </label>
</template>

<script setup lang="ts">
import { ref } from "vue";

const selectedFile = defineModel<File | null>({ default: null, required: true });

function onFileDropped(e: DragEvent) {
    const file = e.dataTransfer?.files[0];
    if (file) selectedFile.value = file;
}

const fileInputEl = ref<HTMLInputElement | null>(null);
function onFileInputChanged(e: Event) {
    if (!(e.target instanceof HTMLInputElement)) 
        throw new Error("onFileInputChanged called with non-input target");

    const file = e.target.files?.[0];
    selectedFile.value = file ?? null;
}
</script>

<style scoped>
.file-drop-selector {
  padding: 20px;
  border: 2px dashed #ccc;
  border-radius: 5px;
  width: 400px;
  height: 200px;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 20px;
  cursor: pointer;
}
</style>