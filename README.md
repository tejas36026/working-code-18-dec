<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Optimized Image Processing</title>
    <link rel="stylesheet" href="styles.css">
    <script src="https://cdn.jsdelivr.net/npm/tesseract.js@2.1.4/dist/tesseract.min.js"></script>
    <script  src = "effects.js"></script>
<script src = "removewatermark.js"></script>
<script src = "objremovelipsync.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/body-pix"></script>
</head>
<body>
    <img id="sourceImage" crossorigin="anonymous" style="display: none;">
    <div id="mainContainer" style="display: none;"></div>
    <canvas id="visualizationCanvas" style="display: none;"></canvas>
    <div id="imageDisplay" class="image-container"></div>
<div id = "canvasContainer"></div>
    <div id="sidebar">
        <div class="controls">
            <input type="file" id="imageUpload" accept="image/*">

            <label for="imageCount">Number of images per effect:</label>
            <input type="number" id="imageCount" min="1" max="100" value="">
            <button id="processButton">Process Image</button>
            <button id="fastProcessButton">Fast Process</button>
            <button id="visualizeBtn">Visualize</button>
            <button id="keypointsButton">Hand</button> <!-- New button -->
            <button  id = "legButton" > Leg</button>
            <button  id = "legButton1" > leftfoot</button>
            <button  id = "legButton2" > rightfoot</button>
            <button id = "leftlegonlyid">leftlegonlyid </button>
            <button id = "rightlegonlyid">rightlegonlyid </button>
            <button id ="rightarmonlyid">rightarmonlyid </button>
            <button id ="leftarmonlyid">leftarmonlyid</button>

            <div>
                <label for="widthInput">Target Width:</label>
                <input type="number" id="widthInput" min="1" step="1">
            </div>
            <div>
                <label for="heightInput">Target Height:</label>
                <input type="number" id="heightInput" min="1" step="1">
            </div>
            <button id="resizeButton">Resize Image</button>
            <button id="toggleDraw">Toggle Draw Mode</button>
            <button id="removeObject">Remove Object</button>
            <button id="lipSyncButton">Lip Sync Animation</button> <!-- New button -->
            <button id="generateImages">Generate Images</button> <!-- Brightness button -->
            <button id="removeWatermarkButton">Remove Watermark</button>
            <label for="brightness">Max Brightness Change:</label>
            <input type="range" id="brightness" min="0" max="255" value="100">
            <input type="number" id="value1" value="0">
            <input type="number" id="value2" value="0">
            <input type="number" id="value3" value="0">
            <input type="number" id="value4" value="0">
            <input type="number" id="value5" value="0">
        </div>
        <div id="masterCheckboxControl">
            <input type="checkbox" id="masterCheckbox" checked>
            <label for="masterCheckbox">Select/Unselect All</label>
        </div>
        <div id="effectControls"></div>
    </div>
    <div id="mainContent">
        <div id="resultsContainer"></div>
        <canvas id="imageCanvas"></canvas>
        <div id="generatedImages"></div>
        <div id="segmentsContainer"></div>
        <div id="progress"></div> <!-- Added progress div -->
        <div id="imageContainer">
            <div class="image-wrapper">
                <h3>Original Image</h3>
                <img id="originalImage" alt="Original Image">
            </div>
            <div class="image-wrapper">
                <h3>Segmented Image</h3>
                <img id="segmentedImage" alt="Segmented Image">
            </div>
        </div>
    </div>

<script src= "main.js"></script>
<script>

let segmentationResult;
let processedData = null; 
let net;
let imageArray = []; 
const viewedImages = []; // Array to store images for further processing
let generatedImages = [];


function processSegmentVariations(imageData, partName) {
    return new Promise((resolve) => {

        segmentationWorker.postMessage({
            imageData: imageData.data,
            partName: partName,
            width: imageData.width,
            height: imageData.height
        });

        segmentationWorker.onmessage = function(e) {
            const { type, extremePoints, averages, partName } = e.data;

            const variations = [{
                data: new Uint8ClampedArray(imageData.data),
                extremePoints: extremePoints,
                points: {}
            }];
            
            if (!collectedPoints.has(partName)) {
                collectedPoints.set(partName, []);
            }

            if (extremePoints && extremePoints.top) collectedPoints.get(partName).push(extremePoints.top);
            if (extremePoints && extremePoints.bottom) collectedPoints.get(partName).push(extremePoints.bottom);

            // Initialize points object with missing properties
            Object.keys(BODY_PARTS).forEach(part => {
                variations[0].points[part] = {
                    top: null,
                    bottom: null
                };
            });

            // Assign the extreme points to the correct properties
            if (extremePoints) {
                variations[0].points[partName] = {
                    top: extremePoints.top,
                    bottom: extremePoints.bottom
                };
            }

            resolve(variations);
        };
    });
}


document.getElementById('imageUpload').addEventListener('change', function(event) {
    const file = event.target.files[0];
    const reader = new FileReader();
    
    reader.onload = function(e) {
        const img = document.getElementById('sourceImage');
        img.src = e.target.result;
        img.onload = async () => {
            await loadModels();
            await prepareSegmentation();
        };
    };
    
    reader.readAsDataURL(file);
});

document.getElementById('keypointsButton').addEventListener('click', handcode);
// document.getElementById('keypointsButton').addEventListener('mouseover', handcode);

document.getElementById('legButton').addEventListener('click', legcode);
// document.getElementById('legButton').addEventListener('mouseover', legcode);

document.getElementById('legButton1').addEventListener('click', legfootcode);
// document.getElementById('legButton1').addEventListener('mouseover', legfootcode);
document.getElementById('legButton2').addEventListener('click', rightfootcode);
document.getElementById('leftlegonlyid').addEventListener('click', leftlegonly);
document.getElementById('rightlegonlyid').addEventListener('click', rightlegonly);
document.getElementById('rightarmonlyid').addEventListener('click', rightarmonly);
document.getElementById('leftarmonlyid').addEventListener('click', leftarmonly);


async function prepareSegmentation() {
    const img = document.getElementById('sourceImage');
    segmentationResult = await net.segmentPersonParts(img);
}

const segmentationWorker = new Worker('keypoints-worker.js');
let collectedPoints = new Map();



async function handcode() {

    const img = document.getElementById('sourceImage');
    const mainContainer = document.getElementById('mainContainer');
    mainContainer.innerHTML = '';

    const imageGrid = document.createElement('div');
    imageGrid.className = 'image-grid';

    const segmentation = segmentationResult;
    const bodyPartImages = {};
    // collectedPoints.clear();

    for (let partId = 0; partId < 24; partId++) {
        const partName = Object.keys(BODY_PARTS)[partId];
        if (!partName) continue;
        if (!segmentation.data.includes(partId)) {

            continue; 
        }

        const segmentCanvas = document.createElement('canvas');
        segmentCanvas.width = img.width;
        segmentCanvas.height = img.height;
        // const segmentCtx = segmentCanvas.getContext('2d');
        const segmentCtx = segmentCanvas.getContext('2d', { willReadFrequently: true });
        segmentCtx.drawImage(img, 0, 0);

        const imageData = segmentCtx.getImageData(0, 0, img.width, img.height);
      
        for (let i = 0; i < segmentation.data.length; i++) {
            if (segmentation.data[i] !== partId) imageData.data[i * 4 + 3] = 0;
        }

        const variations = await processSegmentVariations(imageData, partName);
        bodyPartImages[partName] = variations.map(v => ({
            imageData: v.data,
            width: img.width,
            height: img.height,
            extremePoints: v.extremePoints
        }));
    }
    
    const pointsToProcess = {
        leftFace: collectedPoints.get('left_face'),
        rightFace: collectedPoints.get('right_face'),
        leftUpperArmFront: collectedPoints.get('left_upper_arm_front'),
        leftUpperArmBack: collectedPoints.get('left_upper_arm_back'),
        leftLowerArmFront: collectedPoints.get('left_lower_arm_front'),
        leftLowerArmBack: collectedPoints.get('left_lower_arm_back'),
        leftHand: collectedPoints.get('left_hand'),
        rightUpperArmFront: collectedPoints.get('right_upper_arm_front'),
        rightUpperArmBack: collectedPoints.get('right_upper_arm_back'),
        rightLowerArmFront: collectedPoints.get('right_lower_arm_front'),
        rightLowerArmBack: collectedPoints.get('right_lower_arm_back'),
        rightHand: collectedPoints.get('right_hand'),
        torsoFront: collectedPoints.get('torso_front'),
        torsoBack: collectedPoints.get('torso_back'),
        leftUpperLegFront: collectedPoints.get('left_upper_leg_front'),
        leftUpperLegBack: collectedPoints.get('left_upper_leg_back'),
        leftLowerLegFront: collectedPoints.get('left_lower_leg_front'),
        leftLowerLegBack: collectedPoints.get('left_lower_leg_back'),
        rightUpperLegFront: collectedPoints.get('right_upper_leg_front'),
        rightUpperLegBack: collectedPoints.get('right_upper_leg_back'),
        rightLowerLegFront: collectedPoints.get('right_lower_leg_front'),
        rightLowerLegBack: collectedPoints.get('right_lower_leg_back'),
        leftFoot: collectedPoints.get('left_foot'),
        rightFoot: collectedPoints.get('right_foot')
    };

    segmentationWorker.postMessage({
        type: 'calculateAverage',
        points: pointsToProcess,
        bodyPartImages,
        partNames: {
            leftUpperArmFront: 'left_upper_arm_front',
            leftUpperArmBack: 'left_upper_arm_back',
            leftLowerArmFront: 'left_lower_arm_front',
            leftLowerArmBack: 'left_lower_arm_back',
            leftHand: 'left_hand',
            rightUpperArmFront: 'right_upper_arm_front',
            rightUpperArmBack: 'right_upper_arm_back',
            rightLowerArmFront: 'right_lower_arm_front',
            rightLowerArmBack: 'right_lower_arm_back',
            rightHand: 'right_hand',
            leftFoot: 'left_foot',
            rightFoot: 'right_foot',
            leftUpperFoot: 'left_upper_foot',
            leftLowerFoot: 'left_lower_foot',
            rightUpperFoot: 'right_upper_foot',
            rightLowerFoot: 'right_lower_foot',
            leftUpperLegFront: 'left_upper_leg_front',
            leftUpperLegBack: 'left_upper_leg_back',
            leftLowerLegFront: 'left_lower_leg_front',
            leftLowerLegBack: 'left_lower_leg_back',
            rightUpperLegFront: 'right_upper_leg_front',
            rightUpperLegBack: 'right_upper_leg_back',
            rightLowerLegFront: 'right_lower_leg_front',
            rightLowerLegBack: 'right_lower_leg_back'
        },
        offset: { x: 100, y: 50 },
        imageArray
    }); 

    segmentationWorker.onmessage = e => {
        const { type, averages, extremePoints, partNames } = e.data;
        console.log(e.data);
        if (type === 'combinedResults' && (averages || extremePoints)) {
            
            processedData = {
                averages,
                extremePoints,
                partNames,
                timestamp: Date.now()
            };

            const canvas = document.createElement('canvas');
            canvas.width = img.width;
            canvas.height = img.height;
            // const ctx = canvas.getContext('2d');
            const ctx = canvas.getContext('2d', { willReadFrequently: true });
            ctx.drawImage(img, 0, 0);

        const wrapper = document.createElement('div');
        wrapper.className = 'image-wrapper';
        wrapper.id = "wrapperid";
        wrapper.appendChild(canvas);
        canvas.id = "canvasid1";
        wrapper.appendChild(document.createElement('div')).className = 'keypoints-label';
        mainContainer.appendChild(wrapper);

// Determine which workers to use based on extreme points
const handParts = [
    'leftHand', 'rightHand', 
    'leftUpperArmFront', 'leftUpperArmBack', 
    'leftLowerArmFront', 'leftLowerArmBack',
    'rightUpperArmFront', 'rightUpperArmBack', 
    'rightLowerArmFront', 'rightLowerArmBack'
];

// Check if any hand parts are present
const hasHandParts = handParts.some(part => 
    extremePoints[part] && 
    Object.keys(extremePoints[part]).length > 0
);

// Create workers conditionally
const workersToUse = [];
if (hasHandParts) {
    const postprocessingWorker = new Worker('handworker.js');
    workersToUse.push({
        worker: postprocessingWorker,
        type: 'hand'
    });
}

numberOfVariations = document.getElementById('imageCount').value;

const workerMessages = workersToUse.map(({type}) => ({
    type: e.data.type,
    imageData: ctx.getImageData(0, 0, canvas.width, canvas.height),
    width: canvas.width,
    height: canvas.height,
    extremePoints,
    averages,
    timestamp: Date.now(),
    partNames,
    numberOfVariations, 
    bodyPartImages,
    rotationAngles: [0, 45, 90, 135, 180],
    imageArray
}));

let accumulatedVariations = [];

let totalVariationsExpected = numberOfVariations;

let totalworkerimages = 2 * numberOfVariations;

const generatedImages = [];
let combinedImagesProcessed = 0;

// Track already processed pairs
const processedPairs = new Set();

function attemptCombination() {
    for (let i = 0; i < totalworkerimages; i += 2) {
        // Stop if we already have the required number of combined images
        if (combinedImagesProcessed >= numberOfVariations) {
            break;
        }

        // Check if the pair has already been processed
        const pairKey = `${i}-${i + 1}`;
        if (generatedImages.length >= i + 2 && !processedPairs.has(pairKey)) {
            combineImagesByIndexes(i, i + 1); // Combine images in pairs
            processedPairs.add(pairKey); // Mark the pair as processed
            combinedImagesProcessed++;
        }
    }
}

// Overwrite the `push` method of the generatedImages array to trigger combination checks dynamically
const originalPush = generatedImages.push;
generatedImages.push = function (...args) {
    originalPush.apply(this, args);
    attemptCombination(); // Check if combinations can be processed
};

function safelyCombineImages(img1, img2) {
    return new Promise((resolve, reject) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');

        canvas.width = img1.width;
        canvas.height = img1.height;

        ctx.drawImage(img1, 0, 0);

        ctx.globalAlpha = 1;

        const x = (canvas.width - img2.width) / 2;
        const y = (canvas.height - img2.height) / 2;
        ctx.drawImage(img2, x, y);

        ctx.globalAlpha = 1;

        const combinedImg = new Image();
        combinedImg.onload = () => resolve(combinedImg);
        combinedImg.onerror = reject;
        combinedImg.src = canvas.toDataURL();
    });
}

function combineImagesByIndexes(index1, index2) {
    const firstImageType = generatedImages[index1].type;
    const secondImageType = generatedImages[index2].type;

    console.log(`Attempting to combine - First Type: ${firstImageType}, Second Type: ${secondImageType}`);

    // Prevent combination if types are the same
    if (firstImageType === secondImageType) {
        console.error(`Cannot combine images of the same type: ${firstImageType}`);
        return; // Exit the function immediately
    }

    if (generatedImages.length > Math.max(index1, index2)) {
        const imagePromises = [index1, index2].map(idx => {
            return new Promise((resolve, reject) => {
                const imgInfo = generatedImages[idx];
                const img = new Image();

                if (imgInfo.imageUrl) {
                    img.src = imgInfo.imageUrl;
                } else {
                    try {
                        img.src = createImageFromData(
                            imgInfo.imageData,
                            imgInfo.width,
                            imgInfo.height
                        );
                    } catch (error) {
                        console.error('Error creating image:', error);
                        reject(error);
                        return;
                    }
                }

                img.onload = () => resolve(img);
                img.onerror = reject;
            });
        });

        Promise.all(imagePromises)
            .then(([img1, img2]) => safelyCombineImages(img1, img2))
            .then(combinedImg => {
                const combinedContainer = document.createElement('div');
                combinedContainer.className = 'combined-images-container';

                const combinedWrapper = document.createElement('div');
                combinedWrapper.className = 'image-wrapper combined';
                combinedWrapper.appendChild(combinedImg);

                const verificationInfo = document.createElement('div');
                verificationInfo.className = 'verification-info';
                verificationInfo.innerHTML = `
                    Combination Verification:
                    <br>First Image Type: ${generatedImages[index1].type}
                    <br>Second Image Type: ${generatedImages[index2].type}
                    <br>First Image Dimensions: ${generatedImages[index1].width}x${generatedImages[index1].height}
                    <br>Second Image Dimensions: ${generatedImages[index2].width}x${generatedImages[index2].height}
                    <br>Combined Image Dimensions: ${combinedImg.width}x${combinedImg.height}
                `;

                combinedContainer.appendChild(verificationInfo);
                combinedContainer.appendChild(combinedWrapper);

                const container = document.getElementById('imageContainer');
                container.appendChild(combinedContainer);

                console.log(`Images at indexes ${index1} and ${index2} combined successfully`);
            })
            .catch(error => {
                console.error(`Error combining images at indexes ${index1} and ${index2}:`, error);

                const errorContainer = document.createElement('div');
                errorContainer.className = 'verification-info';
                errorContainer.textContent = `Error combining images: ${error.message}`;

                const container = document.getElementById('imageContainer');
                container.appendChild(errorContainer);
            });
      
        }
}

// Worker processing
Promise.allSettled(
    workersToUse.map(({ worker }, index) =>
        new Promise((resolve, reject) => {
            worker.onmessage = (e) => {
                if (e.data.type === 'processedVariations') {
                    if (!e.data.variations || e.data.variations.length === 0) {
                        console.warn(`No variations for ${workersToUse[index].type} worker`);
                        return;
                    }

                    workersToUse[index].variations = e.data.variations;

                    e.data.variations.forEach((variation, variationIndex) => {
                        const img = document.createElement('img');
                        img.src = variation.imageUrl || createImageFromData(
                            variation.imageData,
                            variation.width,
                            variation.height
                        );

                        const wrapper = document.createElement('div');
                        wrapper.className = 'image-wrapper';

                        wrapper.appendChild(img);

                        const container = document.getElementById('imageContainer');
                        container.appendChild(wrapper);
                        console.log(workersToUse[index].type);
                        generatedImages.push({
                            index: variationIndex,
                            type: workersToUse[index].type,
                            image: img,
                            imageUrl: variation.imageUrl,
                            imageData: variation.imageData,
                            width: variation.width,
                            height: variation.height
                        });
                        console.log('generatedImages :>> ', generatedImages);
                    });

                    resolve({
                        type: workersToUse[index].type,
                        variations: e.data.variations
                    });
                }
            };

            worker.onerror = (error) => {
                console.error('Worker error:', error);
                reject(error);
            };

            worker.postMessage(workerMessages[index]);
        })
    )
    ).catch(error => {
        console.error('Overall worker processing error:', error);
    });

    // Helper function to create an image from ImageData
    function createImageFromData(imageData, width, height) {
        const canvas = document.createElement('canvas');
        canvas.width = width;
        canvas.height = height;
        const ctx = canvas.getContext('2d');
        ctx.putImageData(new ImageData(imageData, width, height), 0, 0);
        return canvas.toDataURL();
    }


        }
    
    
    }


}


async function legcode() {


const img = document.getElementById('sourceImage');
const mainContainer = document.getElementById('mainContainer');
mainContainer.innerHTML = '';
if (!window.imageArray) {
        window.imageArray = [];
    }
    
    // Push the current image to the array
    window.imageArray.push(img);
const imageGrid = document.createElement('div');
imageGrid.className = 'image-grid';

const segmentation = segmentationResult;
const bodyPartImages = {};
// collectedPoints.clear();

for (let partId = 0; partId < 24; partId++) {
    const partName = Object.keys(BODY_PARTS)[partId];
    if (!partName) continue;
    if (!segmentation.data.includes(partId)) {

        continue; 
    
    }

    const segmentCanvas = document.createElement('canvas');
    segmentCanvas.width = img.width;
    segmentCanvas.height = img.height;
    // const segmentCtx = segmentCanvas.getContext('2d');
    const segmentCtx = segmentCanvas.getContext('2d', { willReadFrequently: true });
    segmentCtx.drawImage(img, 0, 0);

    const imageData = segmentCtx.getImageData(0, 0, img.width, img.height);
  
    for (let i = 0; i < segmentation.data.length; i++) {
        if (segmentation.data[i] !== partId) imageData.data[i * 4 + 3] = 0;
    }

    const variations = await processSegmentVariations(imageData, partName);
    bodyPartImages[partName] = variations.map(v => ({
        imageData: v.data,
        width: img.width,
        height: img.height,
        extremePoints: v.extremePoints
    }));
}

const pointsToProcess = {
    leftFace: collectedPoints.get('left_face'),
    rightFace: collectedPoints.get('right_face'),
    leftUpperArmFront: collectedPoints.get('left_upper_arm_front'),
    leftUpperArmBack: collectedPoints.get('left_upper_arm_back'),
    leftLowerArmFront: collectedPoints.get('left_lower_arm_front'),
    leftLowerArmBack: collectedPoints.get('left_lower_arm_back'),
    leftHand: collectedPoints.get('left_hand'),
    rightUpperArmFront: collectedPoints.get('right_upper_arm_front'),
    rightUpperArmBack: collectedPoints.get('right_upper_arm_back'),
    rightLowerArmFront: collectedPoints.get('right_lower_arm_front'),
    rightLowerArmBack: collectedPoints.get('right_lower_arm_back'),
    rightHand: collectedPoints.get('right_hand'),
    torsoFront: collectedPoints.get('torso_front'),
    torsoBack: collectedPoints.get('torso_back'),
    leftUpperLegFront: collectedPoints.get('left_upper_leg_front'),
    leftUpperLegBack: collectedPoints.get('left_upper_leg_back'),
    leftLowerLegFront: collectedPoints.get('left_lower_leg_front'),
    leftLowerLegBack: collectedPoints.get('left_lower_leg_back'),
    rightUpperLegFront: collectedPoints.get('right_upper_leg_front'),
    rightUpperLegBack: collectedPoints.get('right_upper_leg_back'),
    rightLowerLegFront: collectedPoints.get('right_lower_leg_front'),
    rightLowerLegBack: collectedPoints.get('right_lower_leg_back'),
    leftFoot: collectedPoints.get('left_foot'),
    rightFoot: collectedPoints.get('right_foot')
};

segmentationWorker.postMessage({
    type: 'calculateAverage',
    points: pointsToProcess,
    bodyPartImages,
    partNames: {
        leftUpperArmFront: 'left_upper_arm_front',
        leftUpperArmBack: 'left_upper_arm_back',
        leftLowerArmFront: 'left_lower_arm_front',
        leftLowerArmBack: 'left_lower_arm_back',
        leftHand: 'left_hand',
        rightUpperArmFront: 'right_upper_arm_front',
        rightUpperArmBack: 'right_upper_arm_back',
        rightLowerArmFront: 'right_lower_arm_front',
        rightLowerArmBack: 'right_lower_arm_back',
        rightHand: 'right_hand',
        leftFoot: 'left_foot',
        rightFoot: 'right_foot',
        leftUpperFoot: 'left_upper_foot',
        leftLowerFoot: 'left_lower_foot',
        rightUpperFoot: 'right_upper_foot',
        rightLowerFoot: 'right_lower_foot',
        leftUpperLegFront: 'left_upper_leg_front',
        leftUpperLegBack: 'left_upper_leg_back',
        leftLowerLegFront: 'left_lower_leg_front',
        leftLowerLegBack: 'left_lower_leg_back',
        rightUpperLegFront: 'right_upper_leg_front',
        rightUpperLegBack: 'right_upper_leg_back',
        rightLowerLegFront: 'right_lower_leg_front',
        rightLowerLegBack: 'right_lower_leg_back'
    },
    offset: { x: 100, y: 50 },
    imageArray
}); 

segmentationWorker.onmessage = e => {
    const { type, averages, extremePoints, partNames } = e.data;
    console.log(e.data);
    if (type === 'combinedResults' && (averages || extremePoints)) {
        
        processedData = {
            averages,
            extremePoints,
            partNames,
            timestamp: Date.now()
        };

        const canvas = document.createElement('canvas');
        canvas.width = img.width;
        canvas.height = img.height;
        // const ctx = canvas.getContext('2d');
        const ctx = canvas.getContext('2d', { willReadFrequently: true });
        ctx.drawImage(img, 0, 0);

        const wrapper = document.createElement('div');
        wrapper.className = 'image-wrapper';
        wrapper.id = "wrapperid";
        wrapper.appendChild(canvas);
        canvas.id = "canvasid1";
        wrapper.appendChild(document.createElement('div')).className = 'keypoints-label';
        mainContainer.appendChild(wrapper);

const legParts = [
    'leftFoot', 'rightFoot',
    'leftUpperLegFront', 'leftUpperLegBack', 
    'leftLowerLegFront', 'leftLowerLegBack',
    'rightUpperLegFront', 'rightUpperLegBack', 
    'rightLowerLegFront', 'rightLowerLegBack'
];

// Check if any leg parts are present
const hasLegParts = legParts.some(part => 
    extremePoints[part] && 
    Object.keys(extremePoints[part]).length > 0
);

// Create workers conditionally
const workersToUse = [];
if (hasLegParts) {
    const legWorker = new Worker('legworker.js');
    workersToUse.push({
        worker: legWorker,
        type: 'leg'
    });
}

numberOfVariations = document.getElementById('imageCount').value;

const workerMessages = workersToUse.map(({type}) => ({
    type: e.data.type,
    imageData: ctx.getImageData(0, 0, canvas.width, canvas.height),
    width: canvas.width,
    height: canvas.height,
    extremePoints,
    averages,
    timestamp: Date.now(),
    partNames,
    numberOfVariations, 
    bodyPartImages,
    rotationAngles: [0, 45, 90, 135, 180],
    imageArray
}));

let accumulatedVariations = [];

let totalVariationsExpected = numberOfVariations;

let totalworkerimages = 2 * numberOfVariations;

// const generatedImages = [];
let combinedImagesProcessed = 0;

// Track already processed pairs
const processedPairs = new Set();

function attemptCombination() {
    for (let i = 0; i < totalworkerimages; i += 2) {
        // Stop if we already have the required number of combined images
        if (combinedImagesProcessed >= numberOfVariations) {
            break;
        }

        // Check if the pair has already been processed
        const pairKey = `${i}-${i + 1}`;
        if (generatedImages.length >= i + 2 && !processedPairs.has(pairKey)) {
            combineImagesByIndexes(i, i + 1); // Combine images in pairs
            processedPairs.add(pairKey); // Mark the pair as processed
            combinedImagesProcessed++;
        }
    }
}

// Overwrite the `push` method of the generatedImages array to trigger combination checks dynamically
const originalPush = generatedImages.push;
generatedImages.push = function (...args) {
    originalPush.apply(this, args);
    attemptCombination(); // Check if combinations can be processed
};

function safelyCombineImages(img1, img2) {
    return new Promise((resolve, reject) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');

        canvas.width = img1.width;
        canvas.height = img1.height;

        ctx.drawImage(img1, 0, 0);

        ctx.globalAlpha = 1;

        const x = (canvas.width - img2.width) / 2;
        const y = (canvas.height - img2.height) / 2;
        ctx.drawImage(img2, x, y);

        ctx.globalAlpha = 1;

        const combinedImg = new Image();
        combinedImg.onload = () => resolve(combinedImg);
        combinedImg.onerror = reject;
        combinedImg.src = canvas.toDataURL();
    });
}

function combineImagesByIndexes(index1, index2) {
    const firstImageType = generatedImages[index1].type;
    const secondImageType = generatedImages[index2].type;

    console.log(`Attempting to combine - First Type: ${firstImageType}, Second Type: ${secondImageType}`);

    // Prevent combination if types are the same
    if (firstImageType === secondImageType) {
        console.error(`Cannot combine images of the same type: ${firstImageType}`);
        return; // Exit the function immediately
    }

    if (generatedImages.length > Math.max(index1, index2)) {
        const imagePromises = [index1, index2].map(idx => {
            return new Promise((resolve, reject) => {
                const imgInfo = generatedImages[idx];
                const img = new Image();

                if (imgInfo.imageUrl) {
                    img.src = imgInfo.imageUrl;
                } else {
                    try {
                        img.src = createImageFromData(
                            imgInfo.imageData,
                            imgInfo.width,
                            imgInfo.height
                        );
                    } catch (error) {
                        console.error('Error creating image:', error);
                        reject(error);
                        return;
                    }
                }

                img.onload = () => resolve(img);
                img.onerror = reject;
            });
        });

        Promise.all(imagePromises)
            .then(([img1, img2]) => safelyCombineImages(img1, img2))
            .then(combinedImg => {
                const combinedContainer = document.createElement('div');
                combinedContainer.className = 'combined-images-container';

                const combinedWrapper = document.createElement('div');
                combinedWrapper.className = 'image-wrapper combined';
                combinedWrapper.appendChild(combinedImg);

                const verificationInfo = document.createElement('div');
                verificationInfo.className = 'verification-info';
                verificationInfo.innerHTML = `
                    Combination Verification:
                    <br>First Image Type: ${generatedImages[index1].type}
                    <br>Second Image Type: ${generatedImages[index2].type}
                    <br>First Image Dimensions: ${generatedImages[index1].width}x${generatedImages[index1].height}
                    <br>Second Image Dimensions: ${generatedImages[index2].width}x${generatedImages[index2].height}
                    <br>Combined Image Dimensions: ${combinedImg.width}x${combinedImg.height}
                `;

                combinedContainer.appendChild(verificationInfo);
                combinedContainer.appendChild(combinedWrapper);

                const container = document.getElementById('imageContainer');
                container.appendChild(combinedContainer);

                console.log(`Images at indexes ${index1} and ${index2} combined successfully`);
            })
            .catch(error => {
                console.error(`Error combining images at indexes ${index1} and ${index2}:`, error);

                const errorContainer = document.createElement('div');
                errorContainer.className = 'verification-info';
                errorContainer.textContent = `Error combining images: ${error.message}`;

                const container = document.getElementById('imageContainer');
                container.appendChild(errorContainer);
            });
      
        }
}
const canvasImages = [];
function convertImageDataToUrl(imageData, width, height) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    
    const ctx = canvas.getContext('2d');
    const imgDataObj = new ImageData(imageData, width, height);
    
    ctx.putImageData(imgDataObj, 0, 0);
    return canvas.toDataURL('image/png');
}
function displayImagesOnCanvases() {
    // Clear previous canvas images
    canvasImages.length = 0;

    // Get the container for canvases
    const canvasContainer = document.getElementById('canvasContainer');
    canvasContainer.innerHTML = ''; // Clear previous canvases

    generatedImages.forEach((imageData, index) => {
        // Create a wrapper div for each canvas
        const wrapper = document.createElement('div');
        wrapper.className = 'canvas-wrapper';

        // Create canvas element
        const canvas = document.createElement('canvas');
        canvas.width = imageData.width;
        canvas.height = imageData.height;
        canvas.id = `canvas-${index}`;

        // Get canvas context
        const ctx = canvas.getContext('2d');

        // Create an image element to draw on canvas
        const img = new Image();
        img.onload = function() {
            ctx.drawImage(img, 0, 0, imageData.width, imageData.height);
            
            // Store canvas reference
            canvasImages.push({
                canvas: canvas,
                imageData: imageData
            });
        };
        img.src = imageData.imageUrl;

        // Create info paragraph
        const infoP = document.createElement('p');
        infoP.textContent = `Canvas ${index + 1} - Type: ${imageData.type}, Width: ${imageData.width}, Height: ${imageData.height}`;

        // Append canvas and info to wrapper
        wrapper.appendChild(canvas);
        wrapper.appendChild(infoP);

        // Add wrapper to container
        canvasContainer.appendChild(wrapper);
    });
}

    // Worker processing
    Promise.allSettled(
        workersToUse.map(({ worker }, index) =>
            new Promise((resolve, reject) => {
                worker.onmessage = (e) => {
                    if (e.data.type === 'processedVariations') {
                        if (!e.data.variations || e.data.variations.length === 0) {
                            console.warn(`No variations for ${workersToUse[index].type} worker`);
                            return;
                        }

                        workersToUse[index].variations = e.data.variations;

                        e.data.variations.forEach((variation, variationIndex) => {
                            const img = document.createElement('img');
                            img.src = variation.imageUrl || createImageFromData(
                                variation.imageData,
                                variation.width,
                                variation.height
                            );


                            
                            const wrapper = document.createElement('div');
                            wrapper.className = 'image-wrapper1';

                            wrapper.appendChild(img);

                            const container = document.getElementById('imageContainer');
                            container.appendChild(wrapper);
                            console.log(workersToUse[index].type);
                            
                                //              
            // 
            //    function displayGeneratedImages() {
            //     const imageDisplay = document.getElementById('imageDisplay');
                
            //     // Clear previous images
            //     imageDisplay.innerHTML = '';

            //     // Loop through generated images and create display elements
            //     generatedImages.forEach((imageData, index) => {
            //         // Create a container for each image
            //         const imageContainer = document.createElement('div');
            //         imageContainer.classList.add('image-item');

            //         // Create image element
            //         const imgElement = document.createElement('img');
                    
            //         // Determine how to set the image source
            //         if (imageData.imageUrl) {
            //             imgElement.src = imageData.imageUrl;
            //         } else if (imageData.imageData) {
            //             // If imageData is a base64 string or Blob
            //             imgElement.src = imageData.imageData;
            //         } else if (imageData.image instanceof HTMLImageElement) {
            //             // If it's an already created image element
            //             imgElement.src = imageData.image.src;
            //         }

            //         // Create additional info paragraph
            //         const infoP = document.createElement('p');
            //         infoP.textContent = `Image ${index + 1} - Type: ${imageData.type}, Width: ${imageData.width}, Height: ${imageData.height}`;

            //         // Append elements
            //         imageContainer.appendChild(imgElement);
            //         imageContainer.appendChild(infoP);
            //         imageDisplay.appendChild(imageContainer);
            //     });

            //     // Log for debugging
            //     console.log('Generated Images:', generatedImages);
            // }

            // // Call the display function after generating images
            // displayGeneratedImages();
            
        
            function convertImageDataToUrl(imageData, width, height) {
        // Create a canvas to draw the image data
        const canvas = document.createElement('canvas');
        canvas.width = width;
        canvas.height = height;
        
        // Get the canvas context
        const ctx = canvas.getContext('2d');
        
        // Create an ImageData object from the Uint8ClampedArray
        const imgData = new ImageData(imageData, width, height);
        
        // Draw the image data on the canvas
        ctx.putImageData(imgData, 0, 0);
        
        // Convert the canvas to a data URL
        return canvas.toDataURL('image/png');
    }
    const imageUrl = convertImageDataToUrl(variation.imageData, variation.width, variation.height);

// Store image data in generatedImages array
generatedImages.push({ 
    index: variationIndex, 
    type: workersToUse[index].type, 
    image: img, 
    imageUrl: imageUrl, 
    imageData: variation.imageData, 
    width: variation.width, 
    height: variation.height 
});
displayImagesOnCanvases();



    function displayGeneratedImages() {
        const imageDisplay = document.getElementById('imageDisplay');
        imageDisplay.innerHTML = ''; // Clear previous images

        generatedImages.forEach((imageData, index) => {
            const imageContainer = document.createElement('div');
            imageContainer.classList.add('image-item');

            const imgElement = document.createElement('img');
            
            // Use the created imageUrl
            imgElement.src = imageData.imageUrl;

            const infoP = document.createElement('p');
            infoP.textContent = `Image ${index + 1} - Type: ${imageData.type}, Width: ${imageData.width}, Height: ${imageData.height}`;

            imageContainer.appendChild(imgElement);
            imageContainer.appendChild(infoP);
            imageDisplay.appendChild(imageContainer);
        });

        console.log('Generated Images:', generatedImages);
    }
        });

                        resolve({
                            type: workersToUse[index].type,
                            variations: e.data.variations
                        });
                
                    }
                };

                worker.onerror = (error) => {
                    console.error('Worker error:', error);
                    reject(error);
                };

                worker.postMessage(workerMessages[index]);
            })
        )
    ).catch(error => {
        console.error('Overall worker processing error:', error);
    });

// Helper function to create an image from ImageData
function createImageFromData(imageData, width, height) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    ctx.putImageData(new ImageData(imageData, width, height), 0, 0);
    return canvas.toDataURL();
}


    }
}
}


async function legfootcode() {
console.log("button clicked");

const img = document.getElementById('sourceImage');
const mainContainer = document.getElementById('mainContainer');
mainContainer.innerHTML = '';

const imageGrid = document.createElement('div');
imageGrid.className = 'image-grid';

const segmentation = segmentationResult;
const bodyPartImages = {};
// collectedPoints.clear();

for (let partId = 0; partId < 24; partId++) {
    const partName = Object.keys(BODY_PARTS)[partId];
    if (!partName) continue;
    if (!segmentation.data.includes(partId)) {

        continue; 
    
    }

    const segmentCanvas = document.createElement('canvas');
    segmentCanvas.width = img.width;
    segmentCanvas.height = img.height;
    // const segmentCtx = segmentCanvas.getContext('2d');
    const segmentCtx = segmentCanvas.getContext('2d', { willReadFrequently: true });
    segmentCtx.drawImage(img, 0, 0);

    const imageData = segmentCtx.getImageData(0, 0, img.width, img.height);
  
    for (let i = 0; i < segmentation.data.length; i++) {
        if (segmentation.data[i] !== partId) imageData.data[i * 4 + 3] = 0;
    }

    const variations = await processSegmentVariations(imageData, partName);
    bodyPartImages[partName] = variations.map(v => ({
        imageData: v.data,
        width: img.width,
        height: img.height,
        extremePoints: v.extremePoints
    }));
}

const pointsToProcess = {
    leftFace: collectedPoints.get('left_face'),
    rightFace: collectedPoints.get('right_face'),
    leftUpperArmFront: collectedPoints.get('left_upper_arm_front'),
    leftUpperArmBack: collectedPoints.get('left_upper_arm_back'),
    leftLowerArmFront: collectedPoints.get('left_lower_arm_front'),
    leftLowerArmBack: collectedPoints.get('left_lower_arm_back'),
    leftHand: collectedPoints.get('left_hand'),
    rightUpperArmFront: collectedPoints.get('right_upper_arm_front'),
    rightUpperArmBack: collectedPoints.get('right_upper_arm_back'),
    rightLowerArmFront: collectedPoints.get('right_lower_arm_front'),
    rightLowerArmBack: collectedPoints.get('right_lower_arm_back'),
    rightHand: collectedPoints.get('right_hand'),
    torsoFront: collectedPoints.get('torso_front'),
    torsoBack: collectedPoints.get('torso_back'),
    leftUpperLegFront: collectedPoints.get('left_upper_leg_front'),
    leftUpperLegBack: collectedPoints.get('left_upper_leg_back'),
    leftLowerLegFront: collectedPoints.get('left_lower_leg_front'),
    leftLowerLegBack: collectedPoints.get('left_lower_leg_back'),
    rightUpperLegFront: collectedPoints.get('right_upper_leg_front'),
    rightUpperLegBack: collectedPoints.get('right_upper_leg_back'),
    rightLowerLegFront: collectedPoints.get('right_lower_leg_front'),
    rightLowerLegBack: collectedPoints.get('right_lower_leg_back'),
    leftFoot: collectedPoints.get('left_foot'),
    rightFoot: collectedPoints.get('right_foot')
};

segmentationWorker.postMessage({
    type: 'calculateAverage',
    points: pointsToProcess,
    bodyPartImages,
    partNames: {
        leftUpperArmFront: 'left_upper_arm_front',
        leftUpperArmBack: 'left_upper_arm_back',
        leftLowerArmFront: 'left_lower_arm_front',
        leftLowerArmBack: 'left_lower_arm_back',
        leftHand: 'left_hand',
        rightUpperArmFront: 'right_upper_arm_front',
        rightUpperArmBack: 'right_upper_arm_back',
        rightLowerArmFront: 'right_lower_arm_front',
        rightLowerArmBack: 'right_lower_arm_back',
        rightHand: 'right_hand',
        leftFoot: 'left_foot',
        rightFoot: 'right_foot',
        leftUpperFoot: 'left_upper_foot',
        leftLowerFoot: 'left_lower_foot',
        rightUpperFoot: 'right_upper_foot',
        rightLowerFoot: 'right_lower_foot',
        leftUpperLegFront: 'left_upper_leg_front',
        leftUpperLegBack: 'left_upper_leg_back',
        leftLowerLegFront: 'left_lower_leg_front',
        leftLowerLegBack: 'left_lower_leg_back',
        rightUpperLegFront: 'right_upper_leg_front',
        rightUpperLegBack: 'right_upper_leg_back',
        rightLowerLegFront: 'right_lower_leg_front',
        rightLowerLegBack: 'right_lower_leg_back'
    },
    offset: { x: 100, y: 50 },
    imageArray
}); 

segmentationWorker.onmessage = e => {
    const { type, averages, extremePoints, partNames } = e.data;
    console.log(e.data);
    if (type === 'combinedResults' && (averages || extremePoints)) {
        
        processedData = {
            averages,
            extremePoints,
            partNames,
            timestamp: Date.now()
        };

        const canvas = document.createElement('canvas');
        canvas.width = img.width;
        canvas.height = img.height;
        // const ctx = canvas.getContext('2d');
        const ctx = canvas.getContext('2d', { willReadFrequently: true });
        ctx.drawImage(img, 0, 0);

        const wrapper = document.createElement('div');
        wrapper.className = 'image-wrapper';
        wrapper.id = "wrapperid";
        wrapper.appendChild(canvas);
        canvas.id = "canvasid1";
        wrapper.appendChild(document.createElement('div')).className = 'keypoints-label';
        mainContainer.appendChild(wrapper);

        // Determine which workers to use based on extreme points

// Determine which workers to use based on extreme points
const legParts = [
    'leftFoot', 'rightFoot',
    'leftUpperLegFront', 'leftUpperLegBack', 
    'leftLowerLegFront', 'leftLowerLegBack',
    'rightUpperLegFront', 'rightUpperLegBack', 
    'rightLowerLegFront', 'rightLowerLegBack'
];

// Check if any leg parts are present
const hasLegParts = legParts.some(part => 
    extremePoints[part] && 
    Object.keys(extremePoints[part]).length > 0
);

// Create workers conditionally
const workersToUse = [];
if (hasLegParts) {
    const legWorker = new Worker('legfootworker.js');
    workersToUse.push({
        worker: legWorker,
        type: 'leg'
    });
}

numberOfVariations = document.getElementById('imageCount').value;

const workerMessages = workersToUse.map(({type}) => ({
    type: e.data.type,
    imageData: ctx.getImageData(0, 0, canvas.width, canvas.height),
    width: canvas.width,
    height: canvas.height,
    extremePoints,
    averages,
    timestamp: Date.now(),
    partNames,
    numberOfVariations, 
    bodyPartImages,
    rotationAngles: [0, 45, 90, 135, 180],
    imageArray
}));

let accumulatedVariations = [];

let totalVariationsExpected = numberOfVariations;

let totalworkerimages = 2 * numberOfVariations;

const generatedImages = [];
let combinedImagesProcessed = 0;

// Track already processed pairs
const processedPairs = new Set();

function attemptCombination() {
    for (let i = 0; i < totalworkerimages; i += 2) {
        // Stop if we already have the required number of combined images
        if (combinedImagesProcessed >= numberOfVariations) {
            break;
        }

        // Check if the pair has already been processed
        const pairKey = `${i}-${i + 1}`;
        if (generatedImages.length >= i + 2 && !processedPairs.has(pairKey)) {
            combineImagesByIndexes(i, i + 1); // Combine images in pairs
            processedPairs.add(pairKey); // Mark the pair as processed
            combinedImagesProcessed++;
        }
    }
}

// Overwrite the `push` method of the generatedImages array to trigger combination checks dynamically
// const originalPush = generatedImages.push;
// generatedImages.push = function (...args) {
//     originalPush.apply(this, args);
//     attemptCombination(); // Check if combinations can be processed
// };

function safelyCombineImages(img1, img2) {
    return new Promise((resolve, reject) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');

        canvas.width = img1.width;
        canvas.height = img1.height;

        ctx.drawImage(img1, 0, 0);

        ctx.globalAlpha = 1;

        const x = (canvas.width - img2.width) / 2;
        const y = (canvas.height - img2.height) / 2;
        ctx.drawImage(img2, x, y);

        ctx.globalAlpha = 1;

        const combinedImg = new Image();
        combinedImg.onload = () => resolve(combinedImg);
        combinedImg.onerror = reject;
        combinedImg.src = canvas.toDataURL();
    });
}

function combineImagesByIndexes(index1, index2) {
    const firstImageType = generatedImages[index1].type;
    const secondImageType = generatedImages[index2].type;

    console.log(`Attempting to combine - First Type: ${firstImageType}, Second Type: ${secondImageType}`);

    // Prevent combination if types are the same
    if (firstImageType === secondImageType) {
        console.error(`Cannot combine images of the same type: ${firstImageType}`);
        return; // Exit the function immediately
    }

    if (generatedImages.length > Math.max(index1, index2)) {
        const imagePromises = [index1, index2].map(idx => {
            return new Promise((resolve, reject) => {
                const imgInfo = generatedImages[idx];
                const img = new Image();

                if (imgInfo.imageUrl) {
                    img.src = imgInfo.imageUrl;
                } else {
                    try {
                        img.src = createImageFromData(
                            imgInfo.imageData,
                            imgInfo.width,
                            imgInfo.height
                        );
                    } catch (error) {
                        console.error('Error creating image:', error);
                        reject(error);
                        return;
                    }
                }

                img.onload = () => resolve(img);
                img.onerror = reject;
            });
        });

        Promise.all(imagePromises)
            .then(([img1, img2]) => safelyCombineImages(img1, img2))
            .then(combinedImg => {
                const combinedContainer = document.createElement('div');
                combinedContainer.className = 'combined-images-container';

                const combinedWrapper = document.createElement('div');
                combinedWrapper.className = 'image-wrapper combined';
                combinedWrapper.appendChild(combinedImg);

                const verificationInfo = document.createElement('div');
                verificationInfo.className = 'verification-info';
                verificationInfo.innerHTML = `
                    Combination Verification:
                    <br>First Image Type: ${generatedImages[index1].type}
                    <br>Second Image Type: ${generatedImages[index2].type}
                    <br>First Image Dimensions: ${generatedImages[index1].width}x${generatedImages[index1].height}
                    <br>Second Image Dimensions: ${generatedImages[index2].width}x${generatedImages[index2].height}
                    <br>Combined Image Dimensions: ${combinedImg.width}x${combinedImg.height}
                `;

                combinedContainer.appendChild(verificationInfo);
                combinedContainer.appendChild(combinedWrapper);

                const container = document.getElementById('imageContainer');
                container.appendChild(combinedContainer);

                console.log(`Images at indexes ${index1} and ${index2} combined successfully`);
            })
            .catch(error => {
                console.error(`Error combining images at indexes ${index1} and ${index2}:`, error);

                const errorContainer = document.createElement('div');
                errorContainer.className = 'verification-info';
                errorContainer.textContent = `Error combining images: ${error.message}`;

                const container = document.getElementById('imageContainer');
                container.appendChild(errorContainer);
            });
      
        }
}

// Worker processing
Promise.allSettled(
    workersToUse.map(({ worker }, index) =>
        new Promise((resolve, reject) => {
            worker.onmessage = (e) => {
                if (e.data.type === 'processedVariations') {
                    if (!e.data.variations || e.data.variations.length === 0) {
                        console.warn(`No variations for ${workersToUse[index].type} worker`);
                        return;
                    }

                    workersToUse[index].variations = e.data.variations;

                    e.data.variations.forEach((variation, variationIndex) => {
                        const img = document.createElement('img');
                        img.src = variation.imageUrl || createImageFromData(
                            variation.imageData,
                            variation.width,
                            variation.height
                        );

                        const wrapper = document.createElement('div');
                        wrapper.className = 'image-wrapper';

                        wrapper.appendChild(img);

                        const container = document.getElementById('imageContainer');
                        container.appendChild(wrapper);
                        console.log(workersToUse[index].type);
                        generatedImages.push({
                            index: variationIndex,
                            type: workersToUse[index].type,
                            image: img,
                            imageUrl: variation.imageUrl,
                            imageData: variation.imageData,
                            width: variation.width,
                            height: variation.height
                        });
                        console.log('generatedImages :>> ', generatedImages);
                        displayGeneratedImages(generatedImages);



                    });

                    resolve({
                        type: workersToUse[index].type,
                        variations: e.data.variations
                    });
                }
            };

            worker.onerror = (error) => {
                console.error('Worker error:', error);
                reject(error);
            };

            worker.postMessage(workerMessages[index]);
        })
    )
).catch(error => {
    console.error('Overall worker processing error:', error);
});


// Helper function to create an image from ImageData
function createImageFromData(imageData, width, height) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    ctx.putImageData(new ImageData(imageData, width, height), 0, 0);
    return canvas.toDataURL();
}


    }
}
}


async function rightfootcode() {
console.log("button clicked");

const img = document.getElementById('sourceImage');
const mainContainer = document.getElementById('mainContainer');
mainContainer.innerHTML = '';

const imageGrid = document.createElement('div');
imageGrid.className = 'image-grid';

const segmentation = segmentationResult;
const bodyPartImages = {};
// collectedPoints.clear();

for (let partId = 0; partId < 24; partId++) {
    const partName = Object.keys(BODY_PARTS)[partId];
    if (!partName) continue;
    if (!segmentation.data.includes(partId)) {

        continue; 
    
    }

    const segmentCanvas = document.createElement('canvas');
    segmentCanvas.width = img.width;
    segmentCanvas.height = img.height;
    // const segmentCtx = segmentCanvas.getContext('2d');
    const segmentCtx = segmentCanvas.getContext('2d', { willReadFrequently: true });
    segmentCtx.drawImage(img, 0, 0);

    const imageData = segmentCtx.getImageData(0, 0, img.width, img.height);
  
    for (let i = 0; i < segmentation.data.length; i++) {
        if (segmentation.data[i] !== partId) imageData.data[i * 4 + 3] = 0;
    }

    const variations = await processSegmentVariations(imageData, partName);
    bodyPartImages[partName] = variations.map(v => ({
        imageData: v.data,
        width: img.width,
        height: img.height,
        extremePoints: v.extremePoints
    }));
}

const pointsToProcess = {
    leftFace: collectedPoints.get('left_face'),
    rightFace: collectedPoints.get('right_face'),
    leftUpperArmFront: collectedPoints.get('left_upper_arm_front'),
    leftUpperArmBack: collectedPoints.get('left_upper_arm_back'),
    leftLowerArmFront: collectedPoints.get('left_lower_arm_front'),
    leftLowerArmBack: collectedPoints.get('left_lower_arm_back'),
    leftHand: collectedPoints.get('left_hand'),
    rightUpperArmFront: collectedPoints.get('right_upper_arm_front'),
    rightUpperArmBack: collectedPoints.get('right_upper_arm_back'),
    rightLowerArmFront: collectedPoints.get('right_lower_arm_front'),
    rightLowerArmBack: collectedPoints.get('right_lower_arm_back'),
    rightHand: collectedPoints.get('right_hand'),
    torsoFront: collectedPoints.get('torso_front'),
    torsoBack: collectedPoints.get('torso_back'),
    leftUpperLegFront: collectedPoints.get('left_upper_leg_front'),
    leftUpperLegBack: collectedPoints.get('left_upper_leg_back'),
    leftLowerLegFront: collectedPoints.get('left_lower_leg_front'),
    leftLowerLegBack: collectedPoints.get('left_lower_leg_back'),
    rightUpperLegFront: collectedPoints.get('right_upper_leg_front'),
    rightUpperLegBack: collectedPoints.get('right_upper_leg_back'),
    rightLowerLegFront: collectedPoints.get('right_lower_leg_front'),
    rightLowerLegBack: collectedPoints.get('right_lower_leg_back'),
    leftFoot: collectedPoints.get('left_foot'),
    rightFoot: collectedPoints.get('right_foot')
};

segmentationWorker.postMessage({
    type: 'calculateAverage',
    points: pointsToProcess,
    bodyPartImages,
    partNames: {
        leftUpperArmFront: 'left_upper_arm_front',
        leftUpperArmBack: 'left_upper_arm_back',
        leftLowerArmFront: 'left_lower_arm_front',
        leftLowerArmBack: 'left_lower_arm_back',
        leftHand: 'left_hand',
        rightUpperArmFront: 'right_upper_arm_front',
        rightUpperArmBack: 'right_upper_arm_back',
        rightLowerArmFront: 'right_lower_arm_front',
        rightLowerArmBack: 'right_lower_arm_back',
        rightHand: 'right_hand',
        leftFoot: 'left_foot',
        rightFoot: 'right_foot',
        leftUpperFoot: 'left_upper_foot',
        leftLowerFoot: 'left_lower_foot',
        rightUpperFoot: 'right_upper_foot',
        rightLowerFoot: 'right_lower_foot',
        leftUpperLegFront: 'left_upper_leg_front',
        leftUpperLegBack: 'left_upper_leg_back',
        leftLowerLegFront: 'left_lower_leg_front',
        leftLowerLegBack: 'left_lower_leg_back',
        rightUpperLegFront: 'right_upper_leg_front',
        rightUpperLegBack: 'right_upper_leg_back',
        rightLowerLegFront: 'right_lower_leg_front',
        rightLowerLegBack: 'right_lower_leg_back'
    },
    offset: { x: 100, y: 50 },
    imageArray
}); 

segmentationWorker.onmessage = e => {
    const { type, averages, extremePoints, partNames } = e.data;
    console.log(e.data);
    if (type === 'combinedResults' && (averages || extremePoints)) {
        
        processedData = {
            averages,   
            extremePoints,
            partNames,
            timestamp: Date.now()
        };

        const canvas = document.createElement('canvas');
        canvas.width = img.width;
        canvas.height = img.height;
        // const ctx = canvas.getContext('2d');
        const ctx = canvas.getContext('2d', { willReadFrequently: true });
        ctx.drawImage(img, 0, 0);

        const wrapper = document.createElement('div');
        wrapper.className = 'image-wrapper';
        wrapper.id = "wrapperid";
        wrapper.appendChild(canvas);
        canvas.id = "canvasid1";
        wrapper.appendChild(document.createElement('div')).className = 'keypoints-label';
        mainContainer.appendChild(wrapper);

        // Determine which workers to use based on extreme points

// Determine which workers to use based on extreme points
const legParts = [
    'leftFoot', 'rightFoot',
    'leftUpperLegFront', 'leftUpperLegBack', 
    'leftLowerLegFront', 'leftLowerLegBack',
    'rightUpperLegFront', 'rightUpperLegBack', 
    'rightLowerLegFront', 'rightLowerLegBack'
];

// Check if any leg parts are present
const hasLegParts = legParts.some(part => 
    extremePoints[part] && 
    Object.keys(extremePoints[part]).length > 0
);

// Create workers conditionally
const workersToUse = [];
if (hasLegParts) {
    console.log("right foot ");
    const legWorker = new Worker('rightfootworker.js');
    workersToUse.push({
        worker: legWorker,
        type: 'leg'
    });
}

numberOfVariations = document.getElementById('imageCount').value;

const workerMessages = workersToUse.map(({type}) => ({
    type: e.data.type,
    imageData: ctx.getImageData(0, 0, canvas.width, canvas.height),
    width: canvas.width,
    height: canvas.height,
    extremePoints,
    averages,
    timestamp: Date.now(),
    partNames,
    numberOfVariations, 
    bodyPartImages,
    rotationAngles: [0, 45, 90, 135, 180],
    imageArray
}));

let accumulatedVariations = [];

let totalVariationsExpected = numberOfVariations;

let totalworkerimages = 2 * numberOfVariations;

const generatedImages = [];
let combinedImagesProcessed = 0;

// Track already processed pairs
const processedPairs = new Set();

function attemptCombination() {
    for (let i = 0; i < totalworkerimages; i += 2) {
        // Stop if we already have the required number of combined images
        if (combinedImagesProcessed >= numberOfVariations) {
            break;
        }

        // Check if the pair has already been processed
        const pairKey = `${i}-${i + 1}`;
        if (generatedImages.length >= i + 2 && !processedPairs.has(pairKey)) {
            combineImagesByIndexes(i, i + 1); // Combine images in pairs
            processedPairs.add(pairKey); // Mark the pair as processed
            combinedImagesProcessed++;
        }
    }
}

const originalPush = generatedImages.push;
generatedImages.push = function (...args) {
    originalPush.apply(this, args);
    attemptCombination(); // Check if combinations can be processed
};

function safelyCombineImages(img1, img2) {
    return new Promise((resolve, reject) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');

        canvas.width = img1.width;
        canvas.height = img1.height;

        ctx.drawImage(img1, 0, 0);

        ctx.globalAlpha = 1;

        const x = (canvas.width - img2.width) / 2;
        const y = (canvas.height - img2.height) / 2;
        ctx.drawImage(img2, x, y);

        ctx.globalAlpha = 1;

        const combinedImg = new Image();
        combinedImg.onload = () => resolve(combinedImg);
        combinedImg.onerror = reject;
        combinedImg.src = canvas.toDataURL();
    });
}

function combineImagesByIndexes(index1, index2) {
    const firstImageType = generatedImages[index1].type;
    const secondImageType = generatedImages[index2].type;

    console.log(`Attempting to combine - First Type: ${firstImageType}, Second Type: ${secondImageType}`);

    // Prevent combination if types are the same
    if (firstImageType === secondImageType) {
        console.error(`Cannot combine images of the same type: ${firstImageType}`);
        return; // Exit the function immediately
    }

    if (generatedImages.length > Math.max(index1, index2)) {
        const imagePromises = [index1, index2].map(idx => {
            return new Promise((resolve, reject) => {
                const imgInfo = generatedImages[idx];
                const img = new Image();

                if (imgInfo.imageUrl) {
                    img.src = imgInfo.imageUrl;
                } else {
                    try {
                        img.src = createImageFromData(
                            imgInfo.imageData,
                            imgInfo.width,
                            imgInfo.height
                        );
                    } catch (error) {
                        console.error('Error creating image:', error);
                        reject(error);
                        return;
                    }
                }

                img.onload = () => resolve(img);
                img.onerror = reject;
            });
        });

        Promise.all(imagePromises)
            .then(([img1, img2]) => safelyCombineImages(img1, img2))
            .then(combinedImg => {
                const combinedContainer = document.createElement('div');
                combinedContainer.className = 'combined-images-container';

                const combinedWrapper = document.createElement('div');
                combinedWrapper.className = 'image-wrapper combined';
                combinedWrapper.appendChild(combinedImg);

                const verificationInfo = document.createElement('div');
                verificationInfo.className = 'verification-info';
                verificationInfo.innerHTML = `
                    Combination Verification:
                    <br>First Image Type: ${generatedImages[index1].type}
                    <br>Second Image Type: ${generatedImages[index2].type}
                    <br>First Image Dimensions: ${generatedImages[index1].width}x${generatedImages[index1].height}
                    <br>Second Image Dimensions: ${generatedImages[index2].width}x${generatedImages[index2].height}
                    <br>Combined Image Dimensions: ${combinedImg.width}x${combinedImg.height}
                `;

                combinedContainer.appendChild(verificationInfo);
                combinedContainer.appendChild(combinedWrapper);

                const container = document.getElementById('imageContainer');
                container.appendChild(combinedContainer);

                console.log(`Images at indexes ${index1} and ${index2} combined successfully`);
            })
            .catch(error => {
                console.error(`Error combining images at indexes ${index1} and ${index2}:`, error);

                const errorContainer = document.createElement('div');
                errorContainer.className = 'verification-info';
                errorContainer.textContent = `Error combining images: ${error.message}`;

                const container = document.getElementById('imageContainer');
                container.appendChild(errorContainer);
            });
      
        }
}

// Worker processing
Promise.allSettled(
    workersToUse.map(({ worker }, index) =>
        new Promise((resolve, reject) => {
            worker.onmessage = (e) => {
                if (e.data.type === 'processedVariations') {
                    if (!e.data.variations || e.data.variations.length === 0) {
                        console.warn(`No variations for ${workersToUse[index].type} worker`);
                        return;
                    }

                    workersToUse[index].variations = e.data.variations;

                    e.data.variations.forEach((variation, variationIndex) => {
                        const img = document.createElement('img');
                        img.src = variation.imageUrl || createImageFromData(
                            variation.imageData,
                            variation.width,
                            variation.height
                        );

                        const wrapper = document.createElement('div');
                        wrapper.className = 'image-wrapper';

                        wrapper.appendChild(img);

                        const container = document.getElementById('imageContainer');
                        container.appendChild(wrapper);
                        console.log(workersToUse[index].type);
                        generatedImages.push({
                            index: variationIndex,
                            type: workersToUse[index].type,
                            image: img,
                            imageUrl: variation.imageUrl,
                            imageData: variation.imageData,
                            width: variation.width,
                            height: variation.height
                        });
                        console.log('generatedImages :>> ', generatedImages);
                    });

                    resolve({
                        type: workersToUse[index].type,
                        variations: e.data.variations
                    });
                }
            };

            worker.onerror = (error) => {
                console.error('Worker error:', error);
                reject(error);
            };

            worker.postMessage(workerMessages[index]);
        })
    )
).catch(error => {
    console.error('Overall worker processing error:', error);
});

// Helper function to create an image from ImageData
function createImageFromData(imageData, width, height) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    ctx.putImageData(new ImageData(imageData, width, height), 0, 0);
    return canvas.toDataURL();
}


    }
}
}


async function leftlegonly() {


const img = document.getElementById('sourceImage');
const mainContainer = document.getElementById('mainContainer');
mainContainer.innerHTML = '';

const imageGrid = document.createElement('div');
imageGrid.className = 'image-grid';

const segmentation = segmentationResult;
const bodyPartImages = {};
// collectedPoints.clear();

for (let partId = 0; partId < 24; partId++) {
    const partName = Object.keys(BODY_PARTS)[partId];
    if (!partName) continue;
    if (!segmentation.data.includes(partId)) {

        continue; 
    
    }

    const segmentCanvas = document.createElement('canvas');
    segmentCanvas.width = img.width;
    segmentCanvas.height = img.height;
    // const segmentCtx = segmentCanvas.getContext('2d');
    const segmentCtx = segmentCanvas.getContext('2d', { willReadFrequently: true });
    segmentCtx.drawImage(img, 0, 0);

    const imageData = segmentCtx.getImageData(0, 0, img.width, img.height);
  
    for (let i = 0; i < segmentation.data.length; i++) {
        if (segmentation.data[i] !== partId) imageData.data[i * 4 + 3] = 0;
    }

    const variations = await processSegmentVariations(imageData, partName);
    bodyPartImages[partName] = variations.map(v => ({
        imageData: v.data,
        width: img.width,
        height: img.height,
        extremePoints: v.extremePoints
    }));
}

const pointsToProcess = {
    leftFace: collectedPoints.get('left_face'),
    rightFace: collectedPoints.get('right_face'),
    leftUpperArmFront: collectedPoints.get('left_upper_arm_front'),
    leftUpperArmBack: collectedPoints.get('left_upper_arm_back'),
    leftLowerArmFront: collectedPoints.get('left_lower_arm_front'),
    leftLowerArmBack: collectedPoints.get('left_lower_arm_back'),
    leftHand: collectedPoints.get('left_hand'),
    rightUpperArmFront: collectedPoints.get('right_upper_arm_front'),
    rightUpperArmBack: collectedPoints.get('right_upper_arm_back'),
    rightLowerArmFront: collectedPoints.get('right_lower_arm_front'),
    rightLowerArmBack: collectedPoints.get('right_lower_arm_back'),
    rightHand: collectedPoints.get('right_hand'),
    torsoFront: collectedPoints.get('torso_front'),
    torsoBack: collectedPoints.get('torso_back'),
    leftUpperLegFront: collectedPoints.get('left_upper_leg_front'),
    leftUpperLegBack: collectedPoints.get('left_upper_leg_back'),
    leftLowerLegFront: collectedPoints.get('left_lower_leg_front'),
    leftLowerLegBack: collectedPoints.get('left_lower_leg_back'),
    rightUpperLegFront: collectedPoints.get('right_upper_leg_front'),
    rightUpperLegBack: collectedPoints.get('right_upper_leg_back'),
    rightLowerLegFront: collectedPoints.get('right_lower_leg_front'),
    rightLowerLegBack: collectedPoints.get('right_lower_leg_back'),
    leftFoot: collectedPoints.get('left_foot'),
    rightFoot: collectedPoints.get('right_foot')
};

segmentationWorker.postMessage({
    type: 'calculateAverage',
    points: pointsToProcess,
    bodyPartImages,
    partNames: {
        leftUpperArmFront: 'left_upper_arm_front',
        leftUpperArmBack: 'left_upper_arm_back',
        leftLowerArmFront: 'left_lower_arm_front',
        leftLowerArmBack: 'left_lower_arm_back',
        leftHand: 'left_hand',
        rightUpperArmFront: 'right_upper_arm_front',
        rightUpperArmBack: 'right_upper_arm_back',
        rightLowerArmFront: 'right_lower_arm_front',
        rightLowerArmBack: 'right_lower_arm_back',
        rightHand: 'right_hand',
        leftFoot: 'left_foot',
        rightFoot: 'right_foot',
        leftUpperFoot: 'left_upper_foot',
        leftLowerFoot: 'left_lower_foot',
        rightUpperFoot: 'right_upper_foot',
        rightLowerFoot: 'right_lower_foot',
        leftUpperLegFront: 'left_upper_leg_front',
        leftUpperLegBack: 'left_upper_leg_back',
        leftLowerLegFront: 'left_lower_leg_front',
        leftLowerLegBack: 'left_lower_leg_back',
        rightUpperLegFront: 'right_upper_leg_front',
        rightUpperLegBack: 'right_upper_leg_back',
        rightLowerLegFront: 'right_lower_leg_front',
        rightLowerLegBack: 'right_lower_leg_back'
    },
    offset: { x: 100, y: 50 },
    imageArray
}); 

segmentationWorker.onmessage = e => {
    const { type, averages, extremePoints, partNames } = e.data;
    console.log(e.data);
    if (type === 'combinedResults' && (averages || extremePoints)) {
        
        processedData = {
            averages,
            extremePoints,
            partNames,
            timestamp: Date.now()
        };

        const canvas = document.createElement('canvas');
        canvas.width = img.width;
        canvas.height = img.height;
        // const ctx = canvas.getContext('2d');
        const ctx = canvas.getContext('2d', { willReadFrequently: true });
        ctx.drawImage(img, 0, 0);

        const wrapper = document.createElement('div');
        wrapper.className = 'image-wrapper';
        wrapper.id = "wrapperid";
        wrapper.appendChild(canvas);
        canvas.id = "canvasid1";
        wrapper.appendChild(document.createElement('div')).className = 'keypoints-label';
        mainContainer.appendChild(wrapper);

        // Determine which workers to use based on extreme points

// Determine which workers to use based on extreme points
const legParts = [
    'leftFoot', 'rightFoot',
    'leftUpperLegFront', 'leftUpperLegBack', 
    'leftLowerLegFront', 'leftLowerLegBack',
    'rightUpperLegFront', 'rightUpperLegBack', 
    'rightLowerLegFront', 'rightLowerLegBack'
];

// Check if any leg parts are present
const hasLegParts = legParts.some(part => 
    extremePoints[part] && 
    Object.keys(extremePoints[part]).length > 0
);

// Create workers conditionally
const workersToUse = [];
if (hasLegParts) {
    const legWorker = new Worker('leftlegonlyworker.js');
    workersToUse.push({
        worker: legWorker,
        type: 'leg'
    });
}

numberOfVariations = document.getElementById('imageCount').value;

const workerMessages = workersToUse.map(({type}) => ({
    type: e.data.type,
    imageData: ctx.getImageData(0, 0, canvas.width, canvas.height),
    width: canvas.width,
    height: canvas.height,
    extremePoints,
    averages,
    timestamp: Date.now(),
    partNames,
    numberOfVariations, 
    bodyPartImages,
    rotationAngles: [0, 45, 90, 135, 180],
    imageArray
}));

let accumulatedVariations = [];

let totalVariationsExpected = numberOfVariations;

let totalworkerimages = 2 * numberOfVariations;

const generatedImages = [];
let combinedImagesProcessed = 0;

// Track already processed pairs
const processedPairs = new Set();

function attemptCombination() {
    for (let i = 0; i < totalworkerimages; i += 2) {
        // Stop if we already have the required number of combined images
        if (combinedImagesProcessed >= numberOfVariations) {
            break;
        }

        // Check if the pair has already been processed
        const pairKey = `${i}-${i + 1}`;
        if (generatedImages.length >= i + 2 && !processedPairs.has(pairKey)) {
            combineImagesByIndexes(i, i + 1); // Combine images in pairs
            processedPairs.add(pairKey); // Mark the pair as processed
            combinedImagesProcessed++;
        }
    }
}

// Overwrite the `push` method of the generatedImages array to trigger combination checks dynamically
const originalPush = generatedImages.push;
generatedImages.push = function (...args) {
    originalPush.apply(this, args);
    attemptCombination(); // Check if combinations can be processed
};

function safelyCombineImages(img1, img2) {
    return new Promise((resolve, reject) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');

        canvas.width = img1.width;
        canvas.height = img1.height;

        ctx.drawImage(img1, 0, 0);

        ctx.globalAlpha = 1;

        const x = (canvas.width - img2.width) / 2;
        const y = (canvas.height - img2.height) / 2;
        ctx.drawImage(img2, x, y);

        ctx.globalAlpha = 1;

        const combinedImg = new Image();
        combinedImg.onload = () => resolve(combinedImg);
        combinedImg.onerror = reject;
        combinedImg.src = canvas.toDataURL();
    });
}

function combineImagesByIndexes(index1, index2) {
    const firstImageType = generatedImages[index1].type;
    const secondImageType = generatedImages[index2].type;

    console.log(`Attempting to combine - First Type: ${firstImageType}, Second Type: ${secondImageType}`);

    // Prevent combination if types are the same
    if (firstImageType === secondImageType) {
        console.error(`Cannot combine images of the same type: ${firstImageType}`);
        return; // Exit the function immediately
    }

    if (generatedImages.length > Math.max(index1, index2)) {
        const imagePromises = [index1, index2].map(idx => {
            return new Promise((resolve, reject) => {
                const imgInfo = generatedImages[idx];
                const img = new Image();

                if (imgInfo.imageUrl) {
                    img.src = imgInfo.imageUrl;
                } else {
                    try {
                        img.src = createImageFromData(
                            imgInfo.imageData,
                            imgInfo.width,
                            imgInfo.height
                        );
                    } catch (error) {
                        console.error('Error creating image:', error);
                        reject(error);
                        return;
                    }
                }

                img.onload = () => resolve(img);
                img.onerror = reject;
            });
        });

        Promise.all(imagePromises)
            .then(([img1, img2]) => safelyCombineImages(img1, img2))
            .then(combinedImg => {
                const combinedContainer = document.createElement('div');
                combinedContainer.className = 'combined-images-container';

                const combinedWrapper = document.createElement('div');
                combinedWrapper.className = 'image-wrapper combined';
                combinedWrapper.appendChild(combinedImg);

                const verificationInfo = document.createElement('div');
                verificationInfo.className = 'verification-info';
                verificationInfo.innerHTML = `
                    Combination Verification:
                    <br>First Image Type: ${generatedImages[index1].type}
                    <br>Second Image Type: ${generatedImages[index2].type}
                    <br>First Image Dimensions: ${generatedImages[index1].width}x${generatedImages[index1].height}
                    <br>Second Image Dimensions: ${generatedImages[index2].width}x${generatedImages[index2].height}
                    <br>Combined Image Dimensions: ${combinedImg.width}x${combinedImg.height}
                `;

                combinedContainer.appendChild(verificationInfo);
                combinedContainer.appendChild(combinedWrapper);

                const container = document.getElementById('imageContainer');
                container.appendChild(combinedContainer);

                console.log(`Images at indexes ${index1} and ${index2} combined successfully`);
            })
            .catch(error => {
                console.error(`Error combining images at indexes ${index1} and ${index2}:`, error);

                const errorContainer = document.createElement('div');
                errorContainer.className = 'verification-info';
                errorContainer.textContent = `Error combining images: ${error.message}`;

                const container = document.getElementById('imageContainer');
                container.appendChild(errorContainer);
            });
      
        }
}

// Worker processing
Promise.allSettled(
    workersToUse.map(({ worker }, index) =>
        new Promise((resolve, reject) => {
            worker.onmessage = (e) => {
                if (e.data.type === 'processedVariations') {
                    if (!e.data.variations || e.data.variations.length === 0) {
                        console.warn(`No variations for ${workersToUse[index].type} worker`);
                        return;
                    }

                    workersToUse[index].variations = e.data.variations;

                    e.data.variations.forEach((variation, variationIndex) => {
                        const img = document.createElement('img');
                        img.src = variation.imageUrl || createImageFromData(
                            variation.imageData,
                            variation.width,
                            variation.height
                        );

                        const wrapper = document.createElement('div');
                        wrapper.className = 'image-wrapper';

                        wrapper.appendChild(img);

                        const container = document.getElementById('imageContainer');
                        container.appendChild(wrapper);
                        console.log(workersToUse[index].type);
                        generatedImages.push({
                            index: variationIndex,
                            type: workersToUse[index].type,
                            image: img,
                            imageUrl: variation.imageUrl,
                            imageData: variation.imageData,
                            width: variation.width,
                            height: variation.height
                        });
                        console.log('generatedImages :>> ', generatedImages);
                    });

                    resolve({
                        type: workersToUse[index].type,
                        variations: e.data.variations
                    });
                }
            };

            worker.onerror = (error) => {
                console.error('Worker error:', error);
                reject(error);
            };

            worker.postMessage(workerMessages[index]);
        })
    )
).catch(error => {
    console.error('Overall worker processing error:', error);
});

// Helper function to create an image from ImageData
function createImageFromData(imageData, width, height) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    ctx.putImageData(new ImageData(imageData, width, height), 0, 0);
    return canvas.toDataURL();
}


    }
}
}



async function rightlegonly() {

console.log("1111111111111111");
const img = document.getElementById('sourceImage');
const mainContainer = document.getElementById('mainContainer');
mainContainer.innerHTML = '';

const imageGrid = document.createElement('div');
imageGrid.className = 'image-grid';

const segmentation = segmentationResult;
const bodyPartImages = {};
// collectedPoints.clear();

for (let partId = 0; partId < 24; partId++) {
    const partName = Object.keys(BODY_PARTS)[partId];
    if (!partName) continue;
    if (!segmentation.data.includes(partId)) {

        continue; 
    
    }

    const segmentCanvas = document.createElement('canvas');
    segmentCanvas.width = img.width;
    segmentCanvas.height = img.height;
    // const segmentCtx = segmentCanvas.getContext('2d');
    const segmentCtx = segmentCanvas.getContext('2d', { willReadFrequently: true });
    segmentCtx.drawImage(img, 0, 0);

    const imageData = segmentCtx.getImageData(0, 0, img.width, img.height);
  
    for (let i = 0; i < segmentation.data.length; i++) {
        if (segmentation.data[i] !== partId) imageData.data[i * 4 + 3] = 0;
    }

    const variations = await processSegmentVariations(imageData, partName);
    bodyPartImages[partName] = variations.map(v => ({
        imageData: v.data,
        width: img.width,
        height: img.height,
        extremePoints: v.extremePoints
    }));
}

const pointsToProcess = {
    leftFace: collectedPoints.get('left_face'),
    rightFace: collectedPoints.get('right_face'),
    leftUpperArmFront: collectedPoints.get('left_upper_arm_front'),
    leftUpperArmBack: collectedPoints.get('left_upper_arm_back'),
    leftLowerArmFront: collectedPoints.get('left_lower_arm_front'),
    leftLowerArmBack: collectedPoints.get('left_lower_arm_back'),
    leftHand: collectedPoints.get('left_hand'),
    rightUpperArmFront: collectedPoints.get('right_upper_arm_front'),
    rightUpperArmBack: collectedPoints.get('right_upper_arm_back'),
    rightLowerArmFront: collectedPoints.get('right_lower_arm_front'),
    rightLowerArmBack: collectedPoints.get('right_lower_arm_back'),
    rightHand: collectedPoints.get('right_hand'),
    torsoFront: collectedPoints.get('torso_front'),
    torsoBack: collectedPoints.get('torso_back'),
    leftUpperLegFront: collectedPoints.get('left_upper_leg_front'),
    leftUpperLegBack: collectedPoints.get('left_upper_leg_back'),
    leftLowerLegFront: collectedPoints.get('left_lower_leg_front'),
    leftLowerLegBack: collectedPoints.get('left_lower_leg_back'),
    rightUpperLegFront: collectedPoints.get('right_upper_leg_front'),
    rightUpperLegBack: collectedPoints.get('right_upper_leg_back'),
    rightLowerLegFront: collectedPoints.get('right_lower_leg_front'),
    rightLowerLegBack: collectedPoints.get('right_lower_leg_back'),
    leftFoot: collectedPoints.get('left_foot'),
    rightFoot: collectedPoints.get('right_foot')
};

segmentationWorker.postMessage({
    type: 'calculateAverage',
    points: pointsToProcess,
    bodyPartImages,
    partNames: {
        leftUpperArmFront: 'left_upper_arm_front',
        leftUpperArmBack: 'left_upper_arm_back',
        leftLowerArmFront: 'left_lower_arm_front',
        leftLowerArmBack: 'left_lower_arm_back',
        leftHand: 'left_hand',
        rightUpperArmFront: 'right_upper_arm_front',
        rightUpperArmBack: 'right_upper_arm_back',
        rightLowerArmFront: 'right_lower_arm_front',
        rightLowerArmBack: 'right_lower_arm_back',
        rightHand: 'right_hand',
        leftFoot: 'left_foot',
        rightFoot: 'right_foot',
        leftUpperFoot: 'left_upper_foot',
        leftLowerFoot: 'left_lower_foot',
        rightUpperFoot: 'right_upper_foot',
        rightLowerFoot: 'right_lower_foot',
        leftUpperLegFront: 'left_upper_leg_front',
        leftUpperLegBack: 'left_upper_leg_back',
        leftLowerLegFront: 'left_lower_leg_front',
        leftLowerLegBack: 'left_lower_leg_back',
        rightUpperLegFront: 'right_upper_leg_front',
        rightUpperLegBack: 'right_upper_leg_back',
        rightLowerLegFront: 'right_lower_leg_front',
        rightLowerLegBack: 'right_lower_leg_back'
    },
    offset: { x: 100, y: 50 },
    imageArray
}); 

segmentationWorker.onmessage = e => {
    const { type, averages, extremePoints, partNames } = e.data;
    console.log(e.data);
    if (type === 'combinedResults' && (averages || extremePoints)) {
        
        processedData = {
            averages,
            extremePoints,
            partNames,
            timestamp: Date.now()
        };

        const canvas = document.createElement('canvas');
        canvas.width = img.width;
        canvas.height = img.height;
        // const ctx = canvas.getContext('2d');
        const ctx = canvas.getContext('2d', { willReadFrequently: true });
        ctx.drawImage(img, 0, 0);

        const wrapper = document.createElement('div');
        wrapper.className = 'image-wrapper';
        wrapper.id = "wrapperid";
        wrapper.appendChild(canvas);
        canvas.id = "canvasid1";
        wrapper.appendChild(document.createElement('div')).className = 'keypoints-label';
        mainContainer.appendChild(wrapper);

        // Determine which workers to use based on extreme points

// Determine which workers to use based on extreme points
const legParts = [
    'leftFoot', 'rightFoot',
    'leftUpperLegFront', 'leftUpperLegBack', 
    'leftLowerLegFront', 'leftLowerLegBack',
    'rightUpperLegFront', 'rightUpperLegBack', 
    'rightLowerLegFront', 'rightLowerLegBack'
];

// Check if any leg parts are present
const hasLegParts = legParts.some(part => 
    extremePoints[part] && 
    Object.keys(extremePoints[part]).length > 0
);

// Create workers conditionally
const workersToUse = [];
if (hasLegParts) {
    const legWorker = new Worker('rightlegonlyworker.js');
    workersToUse.push({
        worker: legWorker,
        type: 'leg'
    });
}

numberOfVariations = document.getElementById('imageCount').value;

const workerMessages = workersToUse.map(({type}) => ({
    type: e.data.type,
    imageData: ctx.getImageData(0, 0, canvas.width, canvas.height),
    width: canvas.width,
    height: canvas.height,
    extremePoints,
    averages,
    timestamp: Date.now(),
    partNames,
    numberOfVariations, 
    bodyPartImages,
    rotationAngles: [0, 45, 90, 135, 180],
    imageArray
}));

let accumulatedVariations = [];

let totalVariationsExpected = numberOfVariations;

let totalworkerimages = 2 * numberOfVariations;

const generatedImages = [];
let combinedImagesProcessed = 0;

// Track already processed pairs
const processedPairs = new Set();

function attemptCombination() {
    for (let i = 0; i < totalworkerimages; i += 2) {
        // Stop if we already have the required number of combined images
        if (combinedImagesProcessed >= numberOfVariations) {
            break;
        }

        // Check if the pair has already been processed
        const pairKey = `${i}-${i + 1}`;
        if (generatedImages.length >= i + 2 && !processedPairs.has(pairKey)) {
            combineImagesByIndexes(i, i + 1); // Combine images in pairs
            processedPairs.add(pairKey); // Mark the pair as processed
            combinedImagesProcessed++;
        }
    }
}

// Overwrite the `push` method of the generatedImages array to trigger combination checks dynamically
const originalPush = generatedImages.push;
generatedImages.push = function (...args) {
    originalPush.apply(this, args);
    attemptCombination(); // Check if combinations can be processed
};

function safelyCombineImages(img1, img2) {
    return new Promise((resolve, reject) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');

        canvas.width = img1.width;
        canvas.height = img1.height;

        ctx.drawImage(img1, 0, 0);

        ctx.globalAlpha = 1;

        const x = (canvas.width - img2.width) / 2;
        const y = (canvas.height - img2.height) / 2;
        ctx.drawImage(img2, x, y);

        ctx.globalAlpha = 1;

        const combinedImg = new Image();
        combinedImg.onload = () => resolve(combinedImg);
        combinedImg.onerror = reject;
        combinedImg.src = canvas.toDataURL();
    });
}

function combineImagesByIndexes(index1, index2) {
    const firstImageType = generatedImages[index1].type;
    const secondImageType = generatedImages[index2].type;

    console.log(`Attempting to combine - First Type: ${firstImageType}, Second Type: ${secondImageType}`);

    // Prevent combination if types are the same
    if (firstImageType === secondImageType) {
        console.error(`Cannot combine images of the same type: ${firstImageType}`);
        return; // Exit the function immediately
    }

    if (generatedImages.length > Math.max(index1, index2)) {
        const imagePromises = [index1, index2].map(idx => {
            return new Promise((resolve, reject) => {
                const imgInfo = generatedImages[idx];
                const img = new Image();

                if (imgInfo.imageUrl) {
                    img.src = imgInfo.imageUrl;
                } else {
                    try {
                        img.src = createImageFromData(
                            imgInfo.imageData,
                            imgInfo.width,
                            imgInfo.height
                        );
                    } catch (error) {
                        console.error('Error creating image:', error);
                        reject(error);
                        return;
                    }
                }

                img.onload = () => resolve(img);
                img.onerror = reject;
            });
        });

        Promise.all(imagePromises)
            .then(([img1, img2]) => safelyCombineImages(img1, img2))
            .then(combinedImg => {
                const combinedContainer = document.createElement('div');
                combinedContainer.className = 'combined-images-container';

                const combinedWrapper = document.createElement('div');
                combinedWrapper.className = 'image-wrapper combined';
                combinedWrapper.appendChild(combinedImg);

                const verificationInfo = document.createElement('div');
                verificationInfo.className = 'verification-info';
                verificationInfo.innerHTML = `
                    Combination Verification:
                    <br>First Image Type: ${generatedImages[index1].type}
                    <br>Second Image Type: ${generatedImages[index2].type}
                    <br>First Image Dimensions: ${generatedImages[index1].width}x${generatedImages[index1].height}
                    <br>Second Image Dimensions: ${generatedImages[index2].width}x${generatedImages[index2].height}
                    <br>Combined Image Dimensions: ${combinedImg.width}x${combinedImg.height}
                `;

                combinedContainer.appendChild(verificationInfo);
                combinedContainer.appendChild(combinedWrapper);

                const container = document.getElementById('imageContainer');
                container.appendChild(combinedContainer);

                console.log(`Images at indexes ${index1} and ${index2} combined successfully`);
            })
            .catch(error => {
                console.error(`Error combining images at indexes ${index1} and ${index2}:`, error);

                const errorContainer = document.createElement('div');
                errorContainer.className = 'verification-info';
                errorContainer.textContent = `Error combining images: ${error.message}`;

                const container = document.getElementById('imageContainer');
                container.appendChild(errorContainer);
            });
      
        }
}

// Worker processing
Promise.allSettled(
    workersToUse.map(({ worker }, index) =>
        new Promise((resolve, reject) => {
            worker.onmessage = (e) => {
                if (e.data.type === 'processedVariations') {
                    if (!e.data.variations || e.data.variations.length === 0) {
                        console.warn(`No variations for ${workersToUse[index].type} worker`);
                        return;
                    }

                    workersToUse[index].variations = e.data.variations;

                    e.data.variations.forEach((variation, variationIndex) => {
                        const img = document.createElement('img');
                        img.src = variation.imageUrl || createImageFromData(
                            variation.imageData,
                            variation.width,
                            variation.height
                        );

                        const wrapper = document.createElement('div');
                        wrapper.className = 'image-wrapper';

                        wrapper.appendChild(img);

                        const container = document.getElementById('imageContainer');
                        container.appendChild(wrapper);
                        console.log(workersToUse[index].type);
                        generatedImages.push({
                            index: variationIndex,
                            type: workersToUse[index].type,
                            image: img,
                            imageUrl: variation.imageUrl,
                            imageData: variation.imageData,
                            width: variation.width,
                            height: variation.height
                        });
                        console.log('generatedImages :>> ', generatedImages);
                    });

                    resolve({
                        type: workersToUse[index].type,
                        variations: e.data.variations
                    });
                }
            };

            worker.onerror = (error) => {
                console.error('Worker error:', error);
                reject(error);
            };

            worker.postMessage(workerMessages[index]);
        })
    )
).catch(error => {
    console.error('Overall worker processing error:', error);
});

// Helper function to create an image from ImageData
function createImageFromData(imageData, width, height) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    ctx.putImageData(new ImageData(imageData, width, height), 0, 0);
    return canvas.toDataURL();
}


    }
}
}


async function rightarmonly() {

console.log("1111111111111111");
const img = document.getElementById('sourceImage');
const mainContainer = document.getElementById('mainContainer');
mainContainer.innerHTML = '';

const imageGrid = document.createElement('div');
imageGrid.className = 'image-grid';

const segmentation = segmentationResult;
const bodyPartImages = {};
// collectedPoints.clear();

for (let partId = 0; partId < 24; partId++) {
    const partName = Object.keys(BODY_PARTS)[partId];
    if (!partName) continue;
    if (!segmentation.data.includes(partId)) {

        continue; 
    
    }

    const segmentCanvas = document.createElement('canvas');
    segmentCanvas.width = img.width;
    segmentCanvas.height = img.height;
    // const segmentCtx = segmentCanvas.getContext('2d');
    const segmentCtx = segmentCanvas.getContext('2d', { willReadFrequently: true });
    segmentCtx.drawImage(img, 0, 0);

    const imageData = segmentCtx.getImageData(0, 0, img.width, img.height);
  
    for (let i = 0; i < segmentation.data.length; i++) {
        if (segmentation.data[i] !== partId) imageData.data[i * 4 + 3] = 0;
    }

    const variations = await processSegmentVariations(imageData, partName);
    bodyPartImages[partName] = variations.map(v => ({
        imageData: v.data,
        width: img.width,
        height: img.height,
        extremePoints: v.extremePoints
    }));
}

const pointsToProcess = {
    leftFace: collectedPoints.get('left_face'),
    rightFace: collectedPoints.get('right_face'),
    leftUpperArmFront: collectedPoints.get('left_upper_arm_front'),
    leftUpperArmBack: collectedPoints.get('left_upper_arm_back'),
    leftLowerArmFront: collectedPoints.get('left_lower_arm_front'),
    leftLowerArmBack: collectedPoints.get('left_lower_arm_back'),
    leftHand: collectedPoints.get('left_hand'),
    rightUpperArmFront: collectedPoints.get('right_upper_arm_front'),
    rightUpperArmBack: collectedPoints.get('right_upper_arm_back'),
    rightLowerArmFront: collectedPoints.get('right_lower_arm_front'),
    rightLowerArmBack: collectedPoints.get('right_lower_arm_back'),
    rightHand: collectedPoints.get('right_hand'),
    torsoFront: collectedPoints.get('torso_front'),
    torsoBack: collectedPoints.get('torso_back'),
    leftUpperLegFront: collectedPoints.get('left_upper_leg_front'),
    leftUpperLegBack: collectedPoints.get('left_upper_leg_back'),
    leftLowerLegFront: collectedPoints.get('left_lower_leg_front'),
    leftLowerLegBack: collectedPoints.get('left_lower_leg_back'),
    rightUpperLegFront: collectedPoints.get('right_upper_leg_front'),
    rightUpperLegBack: collectedPoints.get('right_upper_leg_back'),
    rightLowerLegFront: collectedPoints.get('right_lower_leg_front'),
    rightLowerLegBack: collectedPoints.get('right_lower_leg_back'),
    leftFoot: collectedPoints.get('left_foot'),
    rightFoot: collectedPoints.get('right_foot')
};

segmentationWorker.postMessage({
    type: 'calculateAverage',
    points: pointsToProcess,
    bodyPartImages,
    partNames: {
        leftUpperArmFront: 'left_upper_arm_front',
        leftUpperArmBack: 'left_upper_arm_back',
        leftLowerArmFront: 'left_lower_arm_front',
        leftLowerArmBack: 'left_lower_arm_back',
        leftHand: 'left_hand',
        rightUpperArmFront: 'right_upper_arm_front',
        rightUpperArmBack: 'right_upper_arm_back',
        rightLowerArmFront: 'right_lower_arm_front',
        rightLowerArmBack: 'right_lower_arm_back',
        rightHand: 'right_hand',
        leftFoot: 'left_foot',
        rightFoot: 'right_foot',
        leftUpperFoot: 'left_upper_foot',
        leftLowerFoot: 'left_lower_foot',
        rightUpperFoot: 'right_upper_foot',
        rightLowerFoot: 'right_lower_foot',
        leftUpperLegFront: 'left_upper_leg_front',
        leftUpperLegBack: 'left_upper_leg_back',
        leftLowerLegFront: 'left_lower_leg_front',
        leftLowerLegBack: 'left_lower_leg_back',
        rightUpperLegFront: 'right_upper_leg_front',
        rightUpperLegBack: 'right_upper_leg_back',
        rightLowerLegFront: 'right_lower_leg_front',
        rightLowerLegBack: 'right_lower_leg_back'
    },
    offset: { x: 100, y: 50 },
    imageArray
}); 

segmentationWorker.onmessage = e => {
    const { type, averages, extremePoints, partNames } = e.data;
    console.log(e.data);
    if (type === 'combinedResults' && (averages || extremePoints)) {
        
        processedData = {
            averages,
            extremePoints,
            partNames,
            timestamp: Date.now()
        };

        const canvas = document.createElement('canvas');
        canvas.width = img.width;
        canvas.height = img.height;
        // const ctx = canvas.getContext('2d');
        const ctx = canvas.getContext('2d', { willReadFrequently: true });
        ctx.drawImage(img, 0, 0);

        const wrapper = document.createElement('div');
        wrapper.className = 'image-wrapper';
        wrapper.id = "wrapperid";
        wrapper.appendChild(canvas);
        canvas.id = "canvasid1";
        wrapper.appendChild(document.createElement('div')).className = 'keypoints-label';
        mainContainer.appendChild(wrapper);

        // Determine which workers to use based on extreme points

// Determine which workers to use based on extreme points
const legParts = [
    'leftFoot', 'rightFoot',
    'leftUpperLegFront', 'leftUpperLegBack', 
    'leftLowerLegFront', 'leftLowerLegBack',
    'rightUpperLegFront', 'rightUpperLegBack', 
    'rightLowerLegFront', 'rightLowerLegBack'
];

// Check if any leg parts are present
const hasLegParts = legParts.some(part => 
    extremePoints[part] && 
    Object.keys(extremePoints[part]).length > 0
);

// Create workers conditionally
const workersToUse = [];
if (hasLegParts) {
    const legWorker = new Worker('rightarmonlyworker.js');
    workersToUse.push({
        worker: legWorker,
        type: 'leg'
    });
}

numberOfVariations = document.getElementById('imageCount').value;

const workerMessages = workersToUse.map(({type}) => ({
    type: e.data.type,
    imageData: ctx.getImageData(0, 0, canvas.width, canvas.height),
    width: canvas.width,
    height: canvas.height,
    extremePoints,
    averages,
    timestamp: Date.now(),
    partNames,
    numberOfVariations, 
    bodyPartImages,
    rotationAngles: [0, 45, 90, 135, 180],
    imageArray
}));

let accumulatedVariations = [];

let totalVariationsExpected = numberOfVariations;

let totalworkerimages = 2 * numberOfVariations;

const generatedImages = [];
let combinedImagesProcessed = 0;

// Track already processed pairs
const processedPairs = new Set();

function attemptCombination() {
    for (let i = 0; i < totalworkerimages; i += 2) {
        // Stop if we already have the required number of combined images
        if (combinedImagesProcessed >= numberOfVariations) {
            break;
        }

        // Check if the pair has already been processed
        const pairKey = `${i}-${i + 1}`;
        if (generatedImages.length >= i + 2 && !processedPairs.has(pairKey)) {
            combineImagesByIndexes(i, i + 1); // Combine images in pairs
            processedPairs.add(pairKey); // Mark the pair as processed
            combinedImagesProcessed++;
        }
    }
}

// Overwrite the `push` method of the generatedImages array to trigger combination checks dynamically
const originalPush = generatedImages.push;
generatedImages.push = function (...args) {
    originalPush.apply(this, args);
    attemptCombination(); // Check if combinations can be processed
};

function safelyCombineImages(img1, img2) {
    return new Promise((resolve, reject) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');

        canvas.width = img1.width;
        canvas.height = img1.height;

        ctx.drawImage(img1, 0, 0);

        ctx.globalAlpha = 1;

        const x = (canvas.width - img2.width) / 2;
        const y = (canvas.height - img2.height) / 2;
        ctx.drawImage(img2, x, y);

        ctx.globalAlpha = 1;

        const combinedImg = new Image();
        combinedImg.onload = () => resolve(combinedImg);
        combinedImg.onerror = reject;
        combinedImg.src = canvas.toDataURL();
    });
}

function combineImagesByIndexes(index1, index2) {
    const firstImageType = generatedImages[index1].type;
    const secondImageType = generatedImages[index2].type;

    console.log(`Attempting to combine - First Type: ${firstImageType}, Second Type: ${secondImageType}`);

    // Prevent combination if types are the same
    if (firstImageType === secondImageType) {
        console.error(`Cannot combine images of the same type: ${firstImageType}`);
        return; // Exit the function immediately
    }

    if (generatedImages.length > Math.max(index1, index2)) {
        const imagePromises = [index1, index2].map(idx => {
            return new Promise((resolve, reject) => {
                const imgInfo = generatedImages[idx];
                const img = new Image();

                if (imgInfo.imageUrl) {
                    img.src = imgInfo.imageUrl;
                } else {
                    try {
                        img.src = createImageFromData(
                            imgInfo.imageData,
                            imgInfo.width,
                            imgInfo.height
                        );
                    } catch (error) {
                        console.error('Error creating image:', error);
                        reject(error);
                        return;
                    }
                }

                img.onload = () => resolve(img);
                img.onerror = reject;
            });
        });

        Promise.all(imagePromises)
            .then(([img1, img2]) => safelyCombineImages(img1, img2))
            .then(combinedImg => {
                const combinedContainer = document.createElement('div');
                combinedContainer.className = 'combined-images-container';

                const combinedWrapper = document.createElement('div');
                combinedWrapper.className = 'image-wrapper combined';
                combinedWrapper.appendChild(combinedImg);

                const verificationInfo = document.createElement('div');
                verificationInfo.className = 'verification-info';
                verificationInfo.innerHTML = `
                    Combination Verification:
                    <br>First Image Type: ${generatedImages[index1].type}
                    <br>Second Image Type: ${generatedImages[index2].type}
                    <br>First Image Dimensions: ${generatedImages[index1].width}x${generatedImages[index1].height}
                    <br>Second Image Dimensions: ${generatedImages[index2].width}x${generatedImages[index2].height}
                    <br>Combined Image Dimensions: ${combinedImg.width}x${combinedImg.height}
                `;

                combinedContainer.appendChild(verificationInfo);
                combinedContainer.appendChild(combinedWrapper);

                const container = document.getElementById('imageContainer');
                container.appendChild(combinedContainer);

                console.log(`Images at indexes ${index1} and ${index2} combined successfully`);
            })
            .catch(error => {
                console.error(`Error combining images at indexes ${index1} and ${index2}:`, error);

                const errorContainer = document.createElement('div');
                errorContainer.className = 'verification-info';
                errorContainer.textContent = `Error combining images: ${error.message}`;

                const container = document.getElementById('imageContainer');
                container.appendChild(errorContainer);
            });
      
        }
}

// Worker processing
Promise.allSettled(
    workersToUse.map(({ worker }, index) =>
        new Promise((resolve, reject) => {
            worker.onmessage = (e) => {
                if (e.data.type === 'processedVariations') {
                    if (!e.data.variations || e.data.variations.length === 0) {
                        console.warn(`No variations for ${workersToUse[index].type} worker`);
                        return;
                    }

                    workersToUse[index].variations = e.data.variations;

                    e.data.variations.forEach((variation, variationIndex) => {
                        const img = document.createElement('img');
                        img.src = variation.imageUrl || createImageFromData(
                            variation.imageData,
                            variation.width,
                            variation.height
                        );

                        const wrapper = document.createElement('div');
                        wrapper.className = 'image-wrapper';

                        wrapper.appendChild(img);

                        const container = document.getElementById('imageContainer');
                        container.appendChild(wrapper);
                        console.log(workersToUse[index].type);
                        generatedImages.push({
                            index: variationIndex,
                            type: workersToUse[index].type,
                            image: img,
                            imageUrl: variation.imageUrl,
                            imageData: variation.imageData,
                            width: variation.width,
                            height: variation.height
                        });
                        console.log('generatedImages :>> ', generatedImages);

                        function displayGeneratedImages() {
            const imageDisplay = document.getElementById('imageDisplay');
            
            // Clear previous images
            imageDisplay.innerHTML = '';

            // Loop through generated images and create display elements
            generatedImages.forEach((imageData, index) => {
                // Create a container for each image
                const imageContainer = document.createElement('div');
                imageContainer.classList.add('image-item');

                // Create image element
                const imgElement = document.createElement('img');
                
                // Determine how to set the image source
                if (imageData.imageUrl) {
                    imgElement.src = imageData.imageUrl;
                } else if (imageData.imageData) {
                    // If imageData is a base64 string or Blob
                    imgElement.src = imageData.imageData;
                } else if (imageData.image instanceof HTMLImageElement) {
                    // If it's an already created image element
                    imgElement.src = imageData.image.src;
                }

                // Create additional info paragraph
                const infoP = document.createElement('p');
                infoP.textContent = `Image ${index + 1} - Type: ${imageData.type}, Width: ${imageData.width}, Height: ${imageData.height}`;

                // Append elements
                imageContainer.appendChild(imgElement);
                imageContainer.appendChild(infoP);
                imageDisplay.appendChild(imageContainer);
            });

            // Log for debugging
            console.log('Generated Images:', generatedImages);
        }

        // Call the display function after generating images
        displayGeneratedImages();
                    });

                    resolve({
                        type: workersToUse[index].type,
                        variations: e.data.variations
                    });
                }
            };

            worker.onerror = (error) => {
                console.error('Worker error:', error);
                reject(error);
            };

            worker.postMessage(workerMessages[index]);
        })
    )
).catch(error => {
    console.error('Overall worker processing error:', error);
});

// Helper function to create an image from ImageData
function createImageFromData(imageData, width, height) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    ctx.putImageData(new ImageData(imageData, width, height), 0, 0);
    return canvas.toDataURL();
}


    }
}
}


async function leftarmonly() {

console.log("1111111111111111");
const img = document.getElementById('sourceImage');
const mainContainer = document.getElementById('mainContainer');
mainContainer.innerHTML = '';

const imageGrid = document.createElement('div');
imageGrid.className = 'image-grid';

const segmentation = segmentationResult;
const bodyPartImages = {};
// collectedPoints.clear();

for (let partId = 0; partId < 24; partId++) {
    const partName = Object.keys(BODY_PARTS)[partId];
    if (!partName) continue;
    if (!segmentation.data.includes(partId)) {

        continue; 
    
    }

    const segmentCanvas = document.createElement('canvas');
    segmentCanvas.width = img.width;
    segmentCanvas.height = img.height;
    // const segmentCtx = segmentCanvas.getContext('2d');
    const segmentCtx = segmentCanvas.getContext('2d', { willReadFrequently: true });
    segmentCtx.drawImage(img, 0, 0);

    const imageData = segmentCtx.getImageData(0, 0, img.width, img.height);
  
    for (let i = 0; i < segmentation.data.length; i++) {
        if (segmentation.data[i] !== partId) imageData.data[i * 4 + 3] = 0;
    }

    const variations = await processSegmentVariations(imageData, partName);
    bodyPartImages[partName] = variations.map(v => ({
        imageData: v.data,
        width: img.width,
        height: img.height,
        extremePoints: v.extremePoints
    }));
}

const pointsToProcess = {
    leftFace: collectedPoints.get('left_face'),
    rightFace: collectedPoints.get('right_face'),
    leftUpperArmFront: collectedPoints.get('left_upper_arm_front'),
    leftUpperArmBack: collectedPoints.get('left_upper_arm_back'),
    leftLowerArmFront: collectedPoints.get('left_lower_arm_front'),
    leftLowerArmBack: collectedPoints.get('left_lower_arm_back'),
    leftHand: collectedPoints.get('left_hand'),
    rightUpperArmFront: collectedPoints.get('right_upper_arm_front'),
    rightUpperArmBack: collectedPoints.get('right_upper_arm_back'),
    rightLowerArmFront: collectedPoints.get('right_lower_arm_front'),
    rightLowerArmBack: collectedPoints.get('right_lower_arm_back'),
    rightHand: collectedPoints.get('right_hand'),
    torsoFront: collectedPoints.get('torso_front'),
    torsoBack: collectedPoints.get('torso_back'),
    leftUpperLegFront: collectedPoints.get('left_upper_leg_front'),
    leftUpperLegBack: collectedPoints.get('left_upper_leg_back'),
    leftLowerLegFront: collectedPoints.get('left_lower_leg_front'),
    leftLowerLegBack: collectedPoints.get('left_lower_leg_back'),
    rightUpperLegFront: collectedPoints.get('right_upper_leg_front'),
    rightUpperLegBack: collectedPoints.get('right_upper_leg_back'),
    rightLowerLegFront: collectedPoints.get('right_lower_leg_front'),
    rightLowerLegBack: collectedPoints.get('right_lower_leg_back'),
    leftFoot: collectedPoints.get('left_foot'),
    rightFoot: collectedPoints.get('right_foot')
};

segmentationWorker.postMessage({
    type: 'calculateAverage',
    points: pointsToProcess,
    bodyPartImages,
    partNames: {
        leftUpperArmFront: 'left_upper_arm_front',
        leftUpperArmBack: 'left_upper_arm_back',
        leftLowerArmFront: 'left_lower_arm_front',
        leftLowerArmBack: 'left_lower_arm_back',
        leftHand: 'left_hand',
        rightUpperArmFront: 'right_upper_arm_front',
        rightUpperArmBack: 'right_upper_arm_back',
        rightLowerArmFront: 'right_lower_arm_front',
        rightLowerArmBack: 'right_lower_arm_back',
        rightHand: 'right_hand',
        leftFoot: 'left_foot',
        rightFoot: 'right_foot',
        leftUpperFoot: 'left_upper_foot',
        leftLowerFoot: 'left_lower_foot',
        rightUpperFoot: 'right_upper_foot',
        rightLowerFoot: 'right_lower_foot',
        leftUpperLegFront: 'left_upper_leg_front',
        leftUpperLegBack: 'left_upper_leg_back',
        leftLowerLegFront: 'left_lower_leg_front',
        leftLowerLegBack: 'left_lower_leg_back',
        rightUpperLegFront: 'right_upper_leg_front',
        rightUpperLegBack: 'right_upper_leg_back',
        rightLowerLegFront: 'right_lower_leg_front',
        rightLowerLegBack: 'right_lower_leg_back'
    },
    offset: { x: 100, y: 50 },
    imageArray
}); 

segmentationWorker.onmessage = e => {
    const { type, averages, extremePoints, partNames } = e.data;
    console.log(e.data);
    if (type === 'combinedResults' && (averages || extremePoints)) {
        
        processedData = {
            averages,
            extremePoints,
            partNames,
            timestamp: Date.now()
        };

        const canvas = document.createElement('canvas');
        canvas.width = img.width;
        canvas.height = img.height;
        // const ctx = canvas.getContext('2d');
        const ctx = canvas.getContext('2d', { willReadFrequently: true });
        ctx.drawImage(img, 0, 0);

        const wrapper = document.createElement('div');
        wrapper.className = 'image-wrapper';
        wrapper.id = "wrapperid";
        wrapper.appendChild(canvas);
        canvas.id = "canvasid1";
        wrapper.appendChild(document.createElement('div')).className = 'keypoints-label';
        mainContainer.appendChild(wrapper);

        // Determine which workers to use based on extreme points

// Determine which workers to use based on extreme points
const legParts = [
    'leftFoot', 'rightFoot',
    'leftUpperLegFront', 'leftUpperLegBack', 
    'leftLowerLegFront', 'leftLowerLegBack',
    'rightUpperLegFront', 'rightUpperLegBack', 
    'rightLowerLegFront', 'rightLowerLegBack'
];

// Check if any leg parts are present
const hasLegParts = legParts.some(part => 
    extremePoints[part] && 
    Object.keys(extremePoints[part]).length > 0
);

// Create workers conditionally
const workersToUse = [];
if (hasLegParts) {
    const legWorker = new Worker('leftarmonlyworker.js');
    workersToUse.push({
        worker: legWorker,
        type: 'leg'
    });
}

numberOfVariations = document.getElementById('imageCount').value;

const workerMessages = workersToUse.map(({type}) => ({
    type: e.data.type,
    imageData: ctx.getImageData(0, 0, canvas.width, canvas.height),
    width: canvas.width,
    height: canvas.height,
    extremePoints,
    averages,
    timestamp: Date.now(),
    partNames,
    numberOfVariations, 
    bodyPartImages,
    rotationAngles: [0, 45, 90, 135, 180],
    imageArray
}));

let accumulatedVariations = [];

let totalVariationsExpected = numberOfVariations;

let totalworkerimages = 2 * numberOfVariations;

const generatedImages = [];
let combinedImagesProcessed = 0;

// Track already processed pairs
const processedPairs = new Set();

function attemptCombination() {
    for (let i = 0; i < totalworkerimages; i += 2) {
        // Stop if we already have the required number of combined images
        if (combinedImagesProcessed >= numberOfVariations) {
            break;
        }

        // Check if the pair has already been processed
        const pairKey = `${i}-${i + 1}`;
        if (generatedImages.length >= i + 2 && !processedPairs.has(pairKey)) {
            combineImagesByIndexes(i, i + 1); // Combine images in pairs
            processedPairs.add(pairKey); // Mark the pair as processed
            combinedImagesProcessed++;
        }
    }
}

// Overwrite the `push` method of the generatedImages array to trigger combination checks dynamically
const originalPush = generatedImages.push;
generatedImages.push = function (...args) {
    originalPush.apply(this, args);
    attemptCombination(); // Check if combinations can be processed
};

function safelyCombineImages(img1, img2) {
    return new Promise((resolve, reject) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');

        canvas.width = img1.width;
        canvas.height = img1.height;

        ctx.drawImage(img1, 0, 0);

        ctx.globalAlpha = 1;

        const x = (canvas.width - img2.width) / 2;
        const y = (canvas.height - img2.height) / 2;
        ctx.drawImage(img2, x, y);

        ctx.globalAlpha = 1;

        const combinedImg = new Image();
        combinedImg.onload = () => resolve(combinedImg);
        combinedImg.onerror = reject;
        combinedImg.src = canvas.toDataURL();
    });
}

function combineImagesByIndexes(index1, index2) {
    const firstImageType = generatedImages[index1].type;
    const secondImageType = generatedImages[index2].type;

    console.log(`Attempting to combine - First Type: ${firstImageType}, Second Type: ${secondImageType}`);

    // Prevent combination if types are the same
    if (firstImageType === secondImageType) {
        console.error(`Cannot combine images of the same type: ${firstImageType}`);
        return; // Exit the function immediately
    }

    if (generatedImages.length > Math.max(index1, index2)) {
        const imagePromises = [index1, index2].map(idx => {
            return new Promise((resolve, reject) => {
                const imgInfo = generatedImages[idx];
                const img = new Image();

                if (imgInfo.imageUrl) {
                    img.src = imgInfo.imageUrl;
                } else {
                    try {
                        img.src = createImageFromData(
                            imgInfo.imageData,
                            imgInfo.width,
                            imgInfo.height
                        );
                    } catch (error) {
                        console.error('Error creating image:', error);
                        reject(error);
                        return;
                    }
                }

                img.onload = () => resolve(img);
                img.onerror = reject;
            });
        });

        Promise.all(imagePromises)
            .then(([img1, img2]) => safelyCombineImages(img1, img2))
            .then(combinedImg => {
                const combinedContainer = document.createElement('div');
                combinedContainer.className = 'combined-images-container';

                const combinedWrapper = document.createElement('div');
                combinedWrapper.className = 'image-wrapper combined';
                combinedWrapper.appendChild(combinedImg);

                const verificationInfo = document.createElement('div');
                verificationInfo.className = 'verification-info';
                verificationInfo.innerHTML = `
                    Combination Verification:
                    <br>First Image Type: ${generatedImages[index1].type}
                    <br>Second Image Type: ${generatedImages[index2].type}
                    <br>First Image Dimensions: ${generatedImages[index1].width}x${generatedImages[index1].height}
                    <br>Second Image Dimensions: ${generatedImages[index2].width}x${generatedImages[index2].height}
                    <br>Combined Image Dimensions: ${combinedImg.width}x${combinedImg.height}
                `;

                combinedContainer.appendChild(verificationInfo);
                combinedContainer.appendChild(combinedWrapper);

                const container = document.getElementById('imageContainer');
                container.appendChild(combinedContainer);

                console.log(`Images at indexes ${index1} and ${index2} combined successfully`);
            })
            .catch(error => {
                console.error(`Error combining images at indexes ${index1} and ${index2}:`, error);

                const errorContainer = document.createElement('div');
                errorContainer.className = 'verification-info';
                errorContainer.textContent = `Error combining images: ${error.message}`;

                const container = document.getElementById('imageContainer');
                container.appendChild(errorContainer);
            });
      
        }
}

// Worker processing
Promise.allSettled(
    workersToUse.map(({ worker }, index) =>
        new Promise((resolve, reject) => {
            worker.onmessage = (e) => {
                if (e.data.type === 'processedVariations') {
                    if (!e.data.variations || e.data.variations.length === 0) {
                        console.warn(`No variations for ${workersToUse[index].type} worker`);
                        return;
                    }

                    workersToUse[index].variations = e.data.variations;

                    e.data.variations.forEach((variation, variationIndex) => {
                        const img = document.createElement('img');
                        img.src = variation.imageUrl || createImageFromData(
                            variation.imageData,
                            variation.width,
                            variation.height
                        );

                        const wrapper = document.createElement('div');
                        wrapper.className = 'image-wrapper';

                        wrapper.appendChild(img);

                        const container = document.getElementById('imageContainer');
                        container.appendChild(wrapper);
                        console.log(workersToUse[index].type);
                        generatedImages.push({
                            index: variationIndex,
                            type: workersToUse[index].type,
                            image: img,
                            imageUrl: variation.imageUrl,
                            imageData: variation.imageData,
                            width: variation.width,
                            height: variation.height
                        });
                        console.log('generatedImages :>> ', generatedImages);
                    });

                    resolve({
                        type: workersToUse[index].type,
                        variations: e.data.variations
                    });
                }
            };

            worker.onerror = (error) => {
                console.error('Worker error:', error);
                reject(error);
            };

            worker.postMessage(workerMessages[index]);
        })
    )
).catch(error => {
    console.error('Overall worker processing error:', error);
});

// Helper function to create an image from ImageData
function createImageFromData(imageData, width, height) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    ctx.putImageData(new ImageData(imageData, width, height), 0, 0);
    return canvas.toDataURL();
}


    }
}
}


function startProcessing(processType) {
    if (!segmentationResult) {
        alert('Segmentation not ready. Please wait.');
        return;
    }
    count = 1    
    // Clear imageArray before starting new processing
    imageArray = [];
    if (!window.imageArrayCollection) {
        window.imageArrayCollection = [];
    }

    // for (let i = 0; i < count; i++) {
        processImageWithOverlay(processType);
    // }

    // Capture the source image data and add it to the imageArray
    const gifImage = document.getElementById('sourceImage');
    const canvas = document.createElement('canvas');
    canvas.width = gifImage.width;
    canvas.height = gifImage.height;
    // const ctx = canvas.getContext('2d');
    const ctx = canvas.getContext('2d', { willReadFrequently: true });
    ctx.drawImage(gifImage, 0, 0);
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const sourceImageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    
    // Create a new array for this processing instance
    const currentProcessImageArray = [sourceImageData.data];

    // Add the image data to imageArray if not already added
    if (!imageArray.some(arr => arr.length === imageData.data.length && 
        arr.every((val, index) => val === imageData.data[index]))) {
    }
}


async function loadModels() {
    net = await bodyPix.load({
        architecture: 'MobileNetV1',
        outputStride: 16,
        multiplier: 0.75,
        quantBytes: 2
    });
}


document.getElementById('visualizeBtn').addEventListener('click', () => {
    startProcessing('visualize');
});

async function processImageWithOverlay(processType) {
    const img = document.getElementById('sourceImage');
    const mainContainer = document.getElementById('mainContainer');
    mainContainer.innerHTML = '';

    const imageGrid = document.createElement('div');
    imageGrid.className = 'image-grid';

    const segmentation = segmentationResult;
    const bodyPartImages = {};
    // collectedPoints.clear();

    for (let partId = 0; partId < 24; partId++) {
        const partName = Object.keys(BODY_PARTS)[partId];
        if (!partName) continue;
        if (!segmentation.data.includes(partId)) {

            continue; // Skip processing if part is not present
        }

        const segmentCanvas = document.createElement('canvas');
        segmentCanvas.width = img.width;
        segmentCanvas.height = img.height;
        // const segmentCtx = segmentCanvas.getContext('2d');
        const segmentCtx = segmentCanvas.getContext('2d', { willReadFrequently: true });
        segmentCtx.drawImage(img, 0, 0);

        const imageData = segmentCtx.getImageData(0, 0, img.width, img.height);
      
        for (let i = 0; i < segmentation.data.length; i++) {
            if (segmentation.data[i] !== partId) imageData.data[i * 4 + 3] = 0;
        }

        const variations = await processSegmentVariations(imageData, partName);
        bodyPartImages[partName] = variations.map(v => ({
            imageData: v.data,
            width: img.width,
            height: img.height,
            extremePoints: v.extremePoints
        }));
    }
    
    const pointsToProcess = {
        leftFace: collectedPoints.get('left_face'),
        rightFace: collectedPoints.get('right_face'),
        leftUpperArmFront: collectedPoints.get('left_upper_arm_front'),
        leftUpperArmBack: collectedPoints.get('left_upper_arm_back'),
        leftLowerArmFront: collectedPoints.get('left_lower_arm_front'),
        leftLowerArmBack: collectedPoints.get('left_lower_arm_back'),
        leftHand: collectedPoints.get('left_hand'),
        rightUpperArmFront: collectedPoints.get('right_upper_arm_front'),
        rightUpperArmBack: collectedPoints.get('right_upper_arm_back'),
        rightLowerArmFront: collectedPoints.get('right_lower_arm_front'),
        rightLowerArmBack: collectedPoints.get('right_lower_arm_back'),
        rightHand: collectedPoints.get('right_hand'),
        torsoFront: collectedPoints.get('torso_front'),
        torsoBack: collectedPoints.get('torso_back'),
        leftUpperLegFront: collectedPoints.get('left_upper_leg_front'),
        leftUpperLegBack: collectedPoints.get('left_upper_leg_back'),
        leftLowerLegFront: collectedPoints.get('left_lower_leg_front'),
        leftLowerLegBack: collectedPoints.get('left_lower_leg_back'),
        rightUpperLegFront: collectedPoints.get('right_upper_leg_front'),
        rightUpperLegBack: collectedPoints.get('right_upper_leg_back'),
        rightLowerLegFront: collectedPoints.get('right_lower_leg_front'),
        rightLowerLegBack: collectedPoints.get('right_lower_leg_back'),
        leftFoot: collectedPoints.get('left_foot'),
        rightFoot: collectedPoints.get('right_foot')
    };

    segmentationWorker.postMessage({
        type: 'calculateAverage',
        points: pointsToProcess,
        bodyPartImages,
        partNames: {
            leftUpperArmFront: 'left_upper_arm_front',
            leftUpperArmBack: 'left_upper_arm_back',
            leftLowerArmFront: 'left_lower_arm_front',
            leftLowerArmBack: 'left_lower_arm_back',
            leftHand: 'left_hand',
            rightUpperArmFront: 'right_upper_arm_front',
            rightUpperArmBack: 'right_upper_arm_back',
            rightLowerArmFront: 'right_lower_arm_front',
            rightLowerArmBack: 'right_lower_arm_back',
            rightHand: 'right_hand',
            leftFoot: 'left_foot',
            rightFoot: 'right_foot',
            leftUpperFoot: 'left_upper_foot',
            leftLowerFoot: 'left_lower_foot',
            rightUpperFoot: 'right_upper_foot',
            rightLowerFoot: 'right_lower_foot',
            leftUpperLegFront: 'left_upper_leg_front',
            leftUpperLegBack: 'left_upper_leg_back',
            leftLowerLegFront: 'left_lower_leg_front',
            leftLowerLegBack: 'left_lower_leg_back',
            rightUpperLegFront: 'right_upper_leg_front',
            rightUpperLegBack: 'right_upper_leg_back',
            rightLowerLegFront: 'right_lower_leg_front',
            rightLowerLegBack: 'right_lower_leg_back'
        },
        offset: { x: 100, y: 50 },
        imageArray
    }); 

    segmentationWorker.onmessage = e => {
        const { type, averages, extremePoints, partNames } = e.data;
        
        if (type === 'combinedResults' && (averages || extremePoints)) {
            
            processedData = {
                averages,
                extremePoints,
                partNames,
                timestamp: Date.now()
            };

            // // //console.log('processedData :>> ', processedData);

            const canvas = document.createElement('canvas');
            canvas.width = img.width;
            canvas.height = img.height;
            // const ctx = canvas.getContext('2d');
            const ctx = canvas.getContext('2d', { willReadFrequently: true });
            ctx.drawImage(img, 0, 0);

        const wrapper = document.createElement('div');
        wrapper.className = 'image-wrapper';
        wrapper.id = "wrapperid";
        wrapper.appendChild(canvas);
        canvas.id = "canvasid1";
        wrapper.appendChild(document.createElement('div')).className = 'keypoints-label';
        mainContainer.appendChild(wrapper);

// Determine which workers to use based on extreme points
const handParts = [
    'leftHand', 'rightHand', 
    'leftUpperArmFront', 'leftUpperArmBack', 
    'leftLowerArmFront', 'leftLowerArmBack',
    'rightUpperArmFront', 'rightUpperArmBack', 
    'rightLowerArmFront', 'rightLowerArmBack'
];

const legParts = [
    'leftFoot', 'rightFoot',
    'leftUpperLegFront', 'leftUpperLegBack', 
    'leftLowerLegFront', 'leftLowerLegBack',
    'rightUpperLegFront', 'rightUpperLegBack', 
    'rightLowerLegFront', 'rightLowerLegBack'
];

// Check if any hand or leg parts are present
const hasHandParts = handParts.some(part => 
    extremePoints[part] && 
    Object.keys(extremePoints[part]).length > 0
);

const hasLegParts = legParts.some(part => 
    extremePoints[part] && 
    Object.keys(extremePoints[part]).length > 0
);

// Create workers conditionally
const workersToUse = [];
if (hasHandParts) {
    const postprocessingWorker = new Worker('handworker.js');
    workersToUse.push({
        worker: postprocessingWorker,
        type: 'hand'
    });
}

if (hasLegParts) {
    const legWorker = new Worker('legworker.js');
    workersToUse.push({
        worker: legWorker,
        type: 'leg'
    });
}

numberOfVariations = document.getElementById('imageCount').value;

const workerMessages = workersToUse.map(({type}) => ({
    type: e.data.type,
    imageData: ctx.getImageData(0, 0, canvas.width, canvas.height),
    width: canvas.width,
    height: canvas.height,
    extremePoints,
    averages,
    timestamp: Date.now(),
    partNames,
    numberOfVariations, 
    bodyPartImages,
    rotationAngles: [0, 45, 90, 135, 180],
    imageArray
}));

let accumulatedVariations = [];

let totalVariationsExpected = numberOfVariations; // Assuming this is correctly set before processing starts

// let totalworkerimages = 2 * numberOfVariations;

// const generatedImages = [];
// let combinedImagesProcessed = 0;


// function attemptCombination() {
//     // Combine images dynamically based on generatedImages length
//     if (generatedImages.length >= 2 && combinedImagesProcessed === 0) {
//         combineImagesByIndexes(0, 1); // Combine first two images
//         combinedImagesProcessed++;
//     }
//     if (generatedImages.length >= 4 && combinedImagesProcessed === 1) {
//         combineImagesByIndexes(2, 3); // Combine 3rd and 4th images
//         combinedImagesProcessed++;
//     }
//     if (generatedImages.length >= 6 && combinedImagesProcessed === 2) {
//         combineImagesByIndexes(4, 5); // Combine 5th and 6th images
//         combinedImagesProcessed++;
//     }
//     if (generatedImages.length >= 8 && combinedImagesProcessed === 3) {
//         combineImagesByIndexes(6, 7); // Combine 3rd and 4th images
//         combinedImagesProcessed++;
//     }
//     if (generatedImages.length >= 10 && combinedImagesProcessed === 4) {
//         combineImagesByIndexes(8, 9); // Combine 3rd and 4th images
//         combinedImagesProcessed++;
//     }
// }

let totalworkerimages = 2 * numberOfVariations;

const generatedImages = [];
let combinedImagesProcessed = 0;

// Track already processed pairs
const processedPairs = new Set();

function attemptCombination() {
    for (let i = 0; i < totalworkerimages; i += 2) {
        // Stop if we already have the required number of combined images
        if (combinedImagesProcessed >= numberOfVariations) {
            break;
        }

        // Check if the pair has already been processed
        const pairKey = `${i}-${i + 1}`;
        if (generatedImages.length >= i + 2 && !processedPairs.has(pairKey)) {
            combineImagesByIndexes(i, i + 1); // Combine images in pairs
            processedPairs.add(pairKey); // Mark the pair as processed
            combinedImagesProcessed++;
        }
    }
}

// Overwrite the `push` method of the generatedImages array to trigger combination checks dynamically
const originalPush = generatedImages.push;
generatedImages.push = function (...args) {
    originalPush.apply(this, args);
    attemptCombination(); // Check if combinations can be processed
};

function safelyCombineImages(img1, img2) {
    return new Promise((resolve, reject) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');

        canvas.width = img1.width;
        canvas.height = img1.height;

        ctx.drawImage(img1, 0, 0);

        ctx.globalAlpha = 1;

        const x = (canvas.width - img2.width) / 2;
        const y = (canvas.height - img2.height) / 2;
        ctx.drawImage(img2, x, y);

        ctx.globalAlpha = 1;

        const combinedImg = new Image();
        combinedImg.onload = () => resolve(combinedImg);
        combinedImg.onerror = reject;
        combinedImg.src = canvas.toDataURL();
    });
}

function combineImagesByIndexes(index1, index2) {
    
    const firstImageType = generatedImages[index1].type;
    const secondImageType = generatedImages[index2].type;

    console.log(`Attempting to combine - First Type: ${firstImageType}, Second Type: ${secondImageType}`);

    // Prevent combination if types are the same
    if (firstImageType === secondImageType) {
        console.error(`Cannot combine images of the same type: ${firstImageType}`);
        return; // Exit the function immediately
    }

    // console.log('index2 :>> ', index2);
    // console.log('index1 :>> ', index1);
    // console.log(generatedImages);
    // console.log(' generatedImages[0] :>> ',  generatedImages[0]);
    // console.log(' generatedImages[1] :>> ',  generatedImages[1]);

    // console.log(' generatedImages[index1] :>> ',  generatedImages[index1]);
    // console.log(' generatedImages[index2] :>> ',  generatedImages[index2]);

    if (generatedImages.length > Math.max(index1, index2)) {
        const imagePromises = [index1, index2].map(idx => {
            return new Promise((resolve, reject) => {
                const imgInfo = generatedImages[idx];
                const img = new Image();
                // console.log('img :>> ', img);

                if (imgInfo.imageUrl) {
                    img.src = imgInfo.imageUrl;
                } else {
                    try {
                        img.src = createImageFromData(
                            imgInfo.imageData,
                            imgInfo.width,
                            imgInfo.height
                        );
                    } catch (error) {
                        console.error('Error creating image:', error);
                        reject(error);
                        return;
                    }
                }

                img.onload = () => resolve(img);
                img.onerror = reject;
            });
        });

        Promise.all(imagePromises)
            .then(([img1, img2]) => safelyCombineImages(img1, img2))
            .then(combinedImg => {
                const combinedContainer = document.createElement('div');
                combinedContainer.className = 'combined-images-container';

                const combinedWrapper = document.createElement('div');
                combinedWrapper.className = 'image-wrapper combined';
                combinedWrapper.appendChild(combinedImg);

                const verificationInfo = document.createElement('div');
                verificationInfo.className = 'verification-info';
                verificationInfo.innerHTML = `
                    Combination Verification:
                    <br>First Image Type: ${generatedImages[index1].type}
                    <br>Second Image Type: ${generatedImages[index2].type}
                    <br>First Image Dimensions: ${generatedImages[index1].width}x${generatedImages[index1].height}
                    <br>Second Image Dimensions: ${generatedImages[index2].width}x${generatedImages[index2].height}
                    <br>Combined Image Dimensions: ${combinedImg.width}x${combinedImg.height}
                `;

                combinedContainer.appendChild(verificationInfo);
                combinedContainer.appendChild(combinedWrapper);

                const container = document.getElementById('imageContainer');
                container.appendChild(combinedContainer);

                console.log(`Images at indexes ${index1} and ${index2} combined successfully`);
            })
            .catch(error => {
                console.error(`Error combining images at indexes ${index1} and ${index2}:`, error);

                const errorContainer = document.createElement('div');
                errorContainer.className = 'verification-info';
                errorContainer.textContent = `Error combining images: ${error.message}`;

                const container = document.getElementById('imageContainer');
                container.appendChild(errorContainer);
            });
      
        }
}

// Worker processing remains unchanged
Promise.allSettled(
    workersToUse.map(({ worker }, index) =>
        new Promise((resolve, reject) => {
            worker.onmessage = (e) => {
                if (e.data.type === 'processedVariations') {
                    if (!e.data.variations || e.data.variations.length === 0) {
                        console.warn(`No variations for ${workersToUse[index].type} worker`);
                        return;
                    }

                    workersToUse[index].variations = e.data.variations;

                    e.data.variations.forEach((variation, variationIndex) => {
                        const img = document.createElement('img');
                        img.src = variation.imageUrl || createImageFromData(
                            variation.imageData,
                            variation.width,
                            variation.height
                        );

                        const wrapper = document.createElement('div');
                        wrapper.className = 'image-wrapper';

                        wrapper.appendChild(img);

                        const container = document.getElementById('imageContainer');
                        container.appendChild(wrapper);
                        console.log(workersToUse[index].type);
                        generatedImages.push({
                            index: variationIndex,
                            type: workersToUse[index].type,
                            image: img,
                            imageUrl: variation.imageUrl,
                            imageData: variation.imageData,
                            width: variation.width,
                            height: variation.height
                        });
                        console.log('generatedImages :>> ', generatedImages);
                    });

                    resolve({
                        type: workersToUse[index].type,
                        variations: e.data.variations
                    });
                }
            };

            worker.onerror = (error) => {
                console.error('Worker error:', error);
                reject(error);
            };

            worker.postMessage(workerMessages[index]);
        })
    )
).catch(error => {
    console.error('Overall worker processing error:', error);
});



// Helper function to create an image from ImageData
function createImageFromData(imageData, width, height) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    ctx.putImageData(new ImageData(imageData, width, height), 0, 0);
    return canvas.toDataURL();
}

}
 
    };


}



        let originalImageData;
        let maskData; 
        let size;
        let objectMask;
        let lines = [];
        effects.forEach(effect => {
            const controlDiv = document.createElement('div');
            controlDiv.className = 'effect-control';
            const checkbox = document.createElement('input');
            checkbox.type = 'checkbox';
            checkbox.id = `${effect}Checkbox`;
            checkbox.checked = true;
            checkbox.addEventListener('change', updateMasterCheckbox);
            const label = document.createElement('label');
            label.htmlFor = `${effect}Checkbox`;
            label.textContent = effect;
            controlDiv.appendChild(checkbox);
            controlDiv.appendChild(label);
            effectControls.appendChild(controlDiv);
        }); 

        imageUpload.addEventListener('change', (event) => {
            const file = event.target.files[0];
            if (file) {
                const reader = new FileReader();
                reader.onload = (e) => {
                    originalImage = new Image();
                    originalImage.onload = () => {
                        imageCanvas.width = originalImage.width;
                        imageCanvas.height = originalImage.height;
                        imageCanvas.id = "icid"
                        ctx.drawImage(originalImage, 0, 0);
                        originalImageData = ctx.getImageData(0, 0, imageCanvas.width, imageCanvas.height);
                        maskData = ctx.getImageData(0, 0, imageCanvas.width, imageCanvas.height); // Initialize maskData
                        size = { w: imageCanvas.width, h: imageCanvas.height }; // Initialize size
                        objectMask = matrix(imageCanvas.width, imageCanvas.height, false); // Initialize objectMask
                        // // //console.log('Image loaded and originalImageData set.');
                    };
                    originalImage.onerror = (error) => {
                        console.error('Error loading image:', error);
                        alert('Failed to load the image. Please try again.');
                    };
                    originalImage.src = e.target.result; // Load the image into the canvas
                };
                reader.onerror = (error) => {
                    console.error('Error reading file:', error);
                    alert('Failed to read the file. Please try again.');
                };
                reader.readAsDataURL(file); // Read the file as a data URL
            }
        });

        function loadImage() {
            originalImage = new Image();
            originalImage.onload = function() {
                imageCanvas.width = originalImage.width;
                imageCanvas.height = originalImage.height;
                ctx.drawImage(originalImage, 0, 0);
                originalImageData = ctx.getImageData(0, 0, imageCanvas.width, imageCanvas.height);
                maskData = ctx.getImageData(0, 0, imageCanvas.width, imageCanvas.height); // Initialize maskData
                size = { w: imageCanvas.width, h: imageCanvas.height }; // Initialize size
                objectMask = matrix(imageCanvas.width, imageCanvas.height, false); // Initialize objectMask
                window.uploadedImageData = originalImageData;
                imageCanvas.id = "icid1"

                displaySelectedRegionsBorders();
            }
            // originalImage.src = 'face.jpg'; // Ensure this path is correct

        }

        </script>
</body>
</html>
