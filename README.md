#include <jni.h>
#include <opencv2/opencv.hpp>
#include <android/log.h>

#define TAG "NativeProcessor"

// JNI method implementation signature (Java_com_example_app_MainActivity_processFrame)
extern "C" JNIEXPORT void JNICALL
Java_com_example_edgedetector_NativeProcessor_processFrame(JNIEnv *env, jobject instance,
                                                           jlong matAddrInput,
                                                           jlong matAddrOutput) {
    // Convert jlong addresses back to cv::Mat pointers
    cv::Mat& inputMat = *(cv::Mat*)matAddrInput;
    cv::Mat& outputMat = *(cv::Mat*)matAddrOutput;

    if (inputMat.empty()) {
        __android_log_print(ANDROID_LOG_ERROR, TAG, "Input Mat is empty!");
        return;
    }

    try {
        // 1. Convert to Grayscale (OpenCV C++ side)
        cv::cvtColor(inputMat, outputMat, cv::COLOR_RGBA2GRAY);

        // 2. Apply Gaussian Blur for noise reduction (improves Canny edges)
        cv::GaussianBlur(outputMat, outputMat, cv::Size(5, 5), 0, 0);

        // 3. Canny Edge Detection
        // Note: The thresholds (100, 200) are example values.
        cv::Canny(outputMat, outputMat, 100, 200, 3);

        // 4. Convert output back to RGBA for OpenGL Texture if needed (or stick to grayscale)
        // For simplicity with OpenGL texture updates, we can convert back to 4-channel.
        cv::cvtColor(outputMat, outputMat, cv::COLOR_GRAY2RGBA);

    } catch (const cv::Exception& e) {
        __android_log_print(ANDROID_LOG_ERROR, TAG, "OpenCV Exception: %s", e.what());
    }
}



<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Edge Detection Web Viewer</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
        }
        #viewer-container {
            max-width: 90%;
            margin: 2rem auto;
            box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
        }
        .stat-card {
            min-width: 120px;
        }
    </style>
</head>
<body class="p-4">

    <div id="viewer-container" class="bg-white rounded-xl overflow-hidden p-6 md:p-8">
        <h1 class="text-3xl font-bold text-gray-800 mb-2">Processed Frame Viewer</h1>
        <p class="text-gray-500 mb-6">Displaying real-time (mock) frame output from the Android NDK pipeline.</p>

        <!-- Stats Panel -->
        <div class="flex flex-wrap gap-4 mb-8">
            <div class="stat-card bg-indigo-50 p-3 rounded-lg flex flex-col justify-center">
                <span class="text-xs font-medium text-indigo-600 uppercase">Status</span>
                <span id="status" class="text-lg font-semibold text-indigo-800">MOCK STREAMING</span>
            </div>
            <div class="stat-card bg-emerald-50 p-3 rounded-lg flex flex-col justify-center">
                <span class="text-xs font-medium text-emerald-600 uppercase">FPS</span>
                <span id="fps" class="text-lg font-semibold text-emerald-800">18.5</span>
            </div>
            <div class="stat-card bg-red-50 p-3 rounded-lg flex flex-col justify-center">
                <span class="text-xs font-medium text-red-600 uppercase">Resolution</span>
                <span id="resolution" class="text-lg font-semibold text-red-800">1280x720</span>
            </div>
        </div>

        <!-- Canvas for Processed Image -->
        <div class="relative w-full aspect-video bg-gray-200 rounded-lg overflow-hidden border-4 border-gray-100">
            <canvas id="frameCanvas" class="w-full h-full"></canvas>
            <div id="loading" class="absolute inset-0 flex items-center justify-center bg-gray-900 bg-opacity-70 text-white font-medium transition-opacity duration-300">
                Loading Frame Data...
            </div>
        </div>
    </div>

    <script>
        // --- START TypeScript/JavaScript Logic (Simulating WebClient.ts) ---

        /**
         * WebClient.ts: Handles fetching mock data and rendering the image onto the canvas.
         */
        class WebClient {
            private canvas: HTMLCanvasElement;
            private ctx: CanvasRenderingContext2D;
            private loadingElement: HTMLElement;
            private fpsElement: HTMLElement;
            private resolutionElement: HTMLElement;
            
            // Mock Base64 of a simple black square with a white Canny-like edge pattern
            // This simulates receiving a processed frame (e.g., from a C++ backend)
            private MOCK_PROCESSED_FRAME_BASE64: string = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAASwAAACWAQMAAACw1+H6AAAABlBMVEX///8AAABVwt/MAAAACXBIWXMAAA7DAAAOwwHHb6hkAAAASElEQVRoge3OsQ0AMAwEUL+yE/p/xW0C9L6pA6S7g3/vS1FSpUqVKlWqVKlSpUqVKlSpUqVKlSpVqlSr9H/pWAAAAABJRU5ErkJggg==';

            constructor() {
                this.canvas = document.getElementById('frameCanvas');
                this.ctx = this.canvas.getContext('2d');
                this.loadingElement = document.getElementById('loading');
                this.fpsElement = document.getElementById('fps');
                this.resolutionElement = document.getElementById('resolution');

                if (!this.canvas || !this.ctx || !this.loadingElement) {
                    console.error("Critical DOM elements not found.");
                    return;
                }
                
                // Ensure canvas is sized correctly
                this.canvas.width = 512;
                this.canvas.height = 512;

                this.initializeViewer();
            }

            private initializeViewer(): void {
                this.renderMockFrame(this.MOCK_PROCESSED_FRAME_BASE64);
                this.updateStats({ fps: 22.1, resolution: '1024x768' });

                // In a real application, this would be replaced by a WebSocket listener or HTTP polling.
                // setInterval(() => this.fetchLiveFrame(), 1000); 
            }

            private updateStats(stats: { fps: number; resolution: string }): void {
                this.fpsElement.textContent = stats.fps.toFixed(1);
                this.resolutionElement.textContent = stats.resolution;
            }

            private renderMockFrame(base64Image: string): void {
                const img = new Image();
                
                img.onload = () => {
                    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
                    
                    // Fit image to canvas while maintaining aspect ratio
                    const aspectRatio = img.width / img.height;
                    let drawWidth = this.canvas.width;
                    let drawHeight = this.canvas.height;
                    let offsetX = 0;
                    let offsetY = 0;

                    if (this.canvas.width / this.canvas.height > aspectRatio) {
                        drawWidth = this.canvas.height * aspectRatio;
                        offsetX = (this.canvas.width - drawWidth) / 2;
                    } else {
                        drawHeight = this.canvas.width / aspectRatio;
                        offsetY = (this.canvas.height - drawHeight) / 2;
                    }

                    this.ctx.drawImage(img, offsetX, offsetY, drawWidth, drawHeight);
                    this.loadingElement.style.opacity = '0';
                    this.loadingElement.style.pointerEvents = 'none';
                };

                img.onerror = () => {
                    this.loadingElement.textContent = 'Error loading frame data.';
                };

                img.src = base64Image;
            }

            // Mock function to simulate a fetch call
            private fetchLiveFrame(): void {
                // ... fetch logic here ...
                // On success: renderMockFrame(newBase64Data);
                // On success: updateStats({ ... });
                console.log("Mock: Attempting to fetch live frame data...");
            }
        }

        // Initialize the Web Viewer
        window.onload = () => {
            new WebClient();
        };

        // --- END TypeScript/JavaScript Logic ---
    </script>
</body>
</html>
