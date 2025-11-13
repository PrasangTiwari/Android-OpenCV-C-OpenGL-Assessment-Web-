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
