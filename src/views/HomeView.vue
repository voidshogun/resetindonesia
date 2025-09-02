<template>
  <div class="min-h-screen flex flex-col bg-gradient-to-t from-gray-200 to-pink-400">
    <!-- Main Content -->
    <div class="flex-1 w-full max-w-4xl p-4 mx-auto">
      <div class="bg-gradient-to-b from-pink-600 to-pink-400 rounded-2xl shadow-2xl p-6 md:p-10 border border-pink-700">
        
        <!-- Header -->
        <div class="text-center mb-8">
          <h1 class="text-3xl md:text-4xl font-bold text-white mb-2">#ResetIndonesia</h1>
          <p class="text-gray-200">Add filter #resetindonesia to your photo instantly.</p>
        </div>

        <!-- Main Content: Inputs and Canvas -->
        <div class="flex flex-col lg:flex-row gap-8">
          
          <!-- Controls -->
          <div class="lg:w-1/3 w-full flex flex-col space-y-6">
            <!-- Step 1: Upload Image -->
            <div class="bg-gray-700/20 p-5 rounded-lg">
              <div class="flex items-center mb-3">
                <div class="bg-blue-600 text-white rounded-full w-8 h-8 flex items-center justify-center font-bold text-lg mr-4">1</div>
                <h2 class="text-xl font-semibold text-white">Unggah Foto</h2>
              </div>
              <label for="image-upload" class="btn btn-primary w-full text-center cursor-pointer">
                Pilih Foto
              </label>
              <input id="image-upload" type="file" @change="handleImageUpload" accept="image/*" class="file-input">
              <p v-if="imageFileName" class="text-xs text-gray-400 mt-2 text-center truncate">{{ imageFileName }}</p>
            </div>

            <!-- Step 2: Download -->
            <div class="bg-gray-700/20 p-5 rounded-lg">
              <div class="flex items-center mb-3">
                <div class="bg-emerald-600 text-white rounded-full w-8 h-8 flex items-center justify-center font-bold text-lg mr-4">2</div>
                <h2 class="text-xl font-semibold text-white">Unduh</h2>
              </div>
              <button @click="downloadImage" :disabled="!isDone" :class="['btn', 'w-full', isDone ? 'btn-success' : 'btn-disabled']">
                Unduh Foto
              </button>
            </div>
          </div>

          <!-- Canvas / Image Display -->
          <div class="lg:w-2/3 w-full bg-gray-900/20 rounded-lg flex items-center justify-center p-4 min-h-[300px] md:min-h-[450px] border-2 border-dashed border-pink-800">
            <div v-if="statusMessage" class="text-center text-gray-500">
              <svg v-if="isProcessing" class="animate-spin h-8 w-8 text-white mx-auto mb-3" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
              </svg>
              <p class="text-white">{{ statusMessage }}</p>
            </div>
            <canvas ref="canvas" class="max-w-full max-h-full rounded-md shadow-lg" :class="{'hidden': statusMessage}"></canvas>
          </div>
        </div>
      </div>
    </div>

    <!-- Sticky Footer -->
    <footer class="bg-gradient-to-r from-pink-600 to-pink-400 text-gray-400 py-4 mt-6">
      <div class="max-w-4xl mx-auto px-4 flex flex-col items-center text-center">
        <h3 class="text-lg font-semibold text-white">#ResetIndonesia</h3>
        <p class="text-sm mt-2 text-white">&copy; @unbothered__space | luts by @ramandadwiputra</p>
        <p class="text-sm mt-2 text-white">Make this site even better, Contribute on <a target="_blank" class="underline text-sm text-white" href="https://github.com/voidshogun/resetindonesia.git">Github.</a></p>
      </div>
    </footer>
  </div>
</template>


<script setup>
import { ref, watch, onMounted } from 'vue';

// DOM element reference
const canvas = ref(null);

// Reactive state
const originalImage = ref(null);
const lutData = ref(null);
const imageFileName = ref('');
const statusMessage = ref('Upload an image to get started.');
const isProcessing = ref(false);
const isDone = ref(false);

onMounted(async () => {
  try {
    // Fetch the default LUT from the public folder
    const response = await fetch('/luts/resetindonesia.cube');
    if (!response.ok) {
      throw new Error(`Failed to load the default LUT file. Status: ${response.status}`);
    }
    const cubeText = await response.text();
    lutData.value = parseCubeFile(cubeText);
    statusMessage.value = 'LUT loaded. Please upload an image.';
  } catch (error) {
    console.error(error);
    statusMessage.value = 'Error: Could not load default LUT.';
  }
});


// --- File Handling ---
const handleImageUpload = (event) => {
    const file = event.target.files[0];
    if (!file) return;
    
    const reader = new FileReader();
    reader.onload = (e) => {
        originalImage.value = new Image();
        originalImage.value.onload = () => {
            imageFileName.value = file.name;
            isDone.value = false;
            // Draw original image to canvas temporarily if no LUT is selected yet
            if (!lutData.value) {
                drawImageToCanvas(originalImage.value);
                statusMessage.value = 'Waiting for LUT to load...';
            }
        };
        originalImage.value.src = e.target.result;
    };
    reader.readAsDataURL(file);
};

const drawImageToCanvas = (image) => {
    if (!canvas.value) return;
    const ctx = canvas.value.getContext('2d');
    canvas.value.width = image.width;
    canvas.value.height = image.height;
    ctx.drawImage(image, 0, 0);
    statusMessage.value = ''; // Hide status to show canvas
};

// --- Core LUT Logic ---
const parseCubeFile = (fileContent) => {
    const lines = fileContent.split('\n');
    const lut = [];
    let size = 0;

    for (const line of lines) {
        const trimmedLine = line.trim();
        if (trimmedLine.startsWith('#') || trimmedLine.length === 0) {
            continue; // Skip comments and empty lines
        }
        
        if (trimmedLine.startsWith('LUT_3D_SIZE')) {
            size = parseInt(trimmedLine.split(' ')[1], 10);
            if (isNaN(size) || size <= 1) {
                throw new Error('Invalid or unsupported LUT_3D_SIZE.');
            }
        } else if (trimmedLine.startsWith('TITLE')) {
            // Optional: could display title in UI
        }
        else {
            const parts = trimmedLine.split(' ').map(parseFloat);
            if (parts.length === 3 && !parts.some(isNaN)) {
                lut.push({ r: parts[0], g: parts[1], b: parts[2] });
            }
        }
    }

    if (size === 0 || lut.length !== size * size * size) {
        throw new Error('Failed to parse .cube file. Size mismatch or not found.');
    }
    
    return { size, lut };
};

// This string contains the code that will run in the background thread.
const workerScript = `
self.onmessage = function(e) {
    const { imageData, lutData } = e.data;
    const data = imageData.data;
    const { size, lut } = lutData;
    const sizeMinus1 = size - 1;

    for (let i = 0; i < data.length; i += 4) {
        const r = data[i] / 255;
        const g = data[i + 1] / 255;
        const b = data[i + 2] / 255;

        const x = r * sizeMinus1;
        const y = g * sizeMinus1;
        const z = b * sizeMinus1;

        const x0 = Math.floor(x);
        const y0 = Math.floor(y);
        const z0 = Math.floor(z);

        const x1 = Math.min(x0 + 1, sizeMinus1);
        const y1 = Math.min(y0 + 1, sizeMinus1);
        const z1 = Math.min(z0 + 1, sizeMinus1);
        
        const xd = x - x0;
        const yd = y - y0;
        const zd = z - z0;
        
        const c000 = lut[z0 * size * size + y0 * size + x0];
        const c100 = lut[z0 * size * size + y0 * size + x1];
        const c010 = lut[z0 * size * size + y1 * size + x0];
        const c110 = lut[z0 * size * size + y1 * size + x1];
        const c001 = lut[z1 * size * size + y0 * size + x0];
        const c101 = lut[z1 * size * size + y0 * size + x1];
        const c011 = lut[z1 * size * size + y1 * size + x0];
        const c111 = lut[z1 * size * size + y1 * size + x1];

        const c00 = { r: c000.r * (1 - xd) + c100.r * xd, g: c000.g * (1 - xd) + c100.g * xd, b: c000.b * (1 - xd) + c100.b * xd };
        const c10 = { r: c010.r * (1 - xd) + c110.r * xd, g: c010.g * (1 - xd) + c110.g * xd, b: c010.b * (1 - xd) + c110.b * xd };
        const c01 = { r: c001.r * (1 - xd) + c101.r * xd, g: c001.g * (1 - xd) + c101.g * xd, b: c001.b * (1 - xd) + c101.b * xd };
        const c11 = { r: c011.r * (1 - xd) + c111.r * xd, g: c011.g * (1 - xd) + c111.g * xd, b: c011.b * (1 - xd) + c111.b * xd };

        const c0 = { r: c00.r * (1 - yd) + c10.r * yd, g: c00.g * (1 - yd) + c10.g * yd, b: c00.b * (1 - yd) + c10.b * yd };
        const c1 = { r: c01.r * (1 - yd) + c11.r * yd, g: c01.g * (1 - yd) + c11.g * yd, b: c01.b * (1 - yd) + c11.b * yd };

        const finalColor = { r: c0.r * (1 - zd) + c1.r * zd, g: c0.g * (1 - zd) + c1.g * zd, b: c0.b * (1 - zd) + c1.b * zd };

        data[i] = finalColor.r * 255;
        data[i + 1] = finalColor.g * 255;
        data[i + 2] = finalColor.b * 255;
    }

    self.postMessage(imageData);
};
`;

const applyLut = () => {
    if (!originalImage.value || !lutData.value || !canvas.value) return;

    isProcessing.value = true;
    isDone.value = false;
    statusMessage.value = 'Applying LUT... this may take a moment.';

    setTimeout(() => {
        const ctx = canvas.value.getContext('2d', { willReadFrequently: true });
        drawImageToCanvas(originalImage.value);
        const imageData = ctx.getImageData(0, 0, canvas.value.width, canvas.value.height);

        const blob = new Blob([workerScript], { type: 'application/javascript' });
        const workerUrl = URL.createObjectURL(blob);
        const worker = new Worker(workerUrl);

        worker.onmessage = (e) => {
            const processedImageData = e.data;
            ctx.putImageData(processedImageData, 0, 0);
            
            isProcessing.value = false;
            isDone.value = true;
            statusMessage.value = '';
            
            worker.terminate();
            URL.revokeObjectURL(workerUrl);
        };

        worker.onerror = (e) => {
            console.error('Error in worker:', e);
            statusMessage.value = 'An error occurred during processing.';
            isProcessing.value = false;
            worker.terminate();
            URL.revokeObjectURL(workerUrl);
        };
        
        // ** THE FIX IS HERE **
        // Convert the reactive lutData to a plain JS object before sending it to the worker.
        const plainLutData = JSON.parse(JSON.stringify(lutData.value));
        worker.postMessage({ imageData, lutData: plainLutData }, [imageData.data.buffer]);
    }, 50); 
};

// --- Download ---
const downloadImage = () => {
    if (!isDone.value || !canvas.value) return;

    const link = document.createElement('a');
    link.download = `edited-${imageFileName.value || 'image'}.png`;
    link.href = canvas.value.toDataURL('image/png');
    link.click();
};

// Watch for changes in image or LUT to trigger the application
watch([originalImage, lutData], ([newImage, newLut]) => {
    if (newImage && newLut) {
        applyLut();
    }
});
</script>

<style scoped>
/* Hide the default file input */
.file-input {
    display: none;
}
.btn {
  @apply inline-block px-6 py-3 text-sm font-semibold tracking-wide text-white uppercase transition-colors duration-300 transform rounded-lg shadow-md focus:outline-none focus:ring focus:ring-opacity-50;
}
.btn-primary {
  @apply bg-blue-600 hover:bg-blue-700 focus:ring-blue-500;
}
.btn-secondary {
  @apply bg-indigo-600 hover:bg-indigo-700 focus:ring-indigo-500;
}
.btn-success {
  @apply bg-emerald-600 hover:bg-emerald-700 focus:ring-emerald-500;
}
.btn-disabled {
  @apply bg-gray-500 cursor-not-allowed;
}
</style>

