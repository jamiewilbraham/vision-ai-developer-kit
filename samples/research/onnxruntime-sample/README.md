# Develop and deploy an IoT Edge module OnnxRuntimeModule

onnxruntime-sample IoT Edge solution sample is used to demo how to build and deploy an ONNX Runtime module.  This sample will input camera preview RTSP stream, detect objects by **Tiny YOLO V2** or other ONNX models under CPU mode, and save the detection result result.jpg to the /data/misc/qmmf folder.

## Setup Build Environment

1. Refer to [**Setup Visual Studio Code Development Environment**] section in [VisionSample README.md](../VisionSample/README.md) to setup build environment.

1. Install [**Docker Community Edition (CE)**](https://docs.docker.com/install/#supported-platforms).  Don't sign in **Docker Desktop** after Docker CE installed.

1. Install [**Docker Extension**](https://marketplace.visualstudio.com/items?itemName=PeterJausovec.vscode-docker) to **Visual Studio Code**.

## Build a Local Container Image with a Single Module: OnnxRuntimeModule

modules\\**OnnxRuntimeModule** folder includes:
   * **app** folder: source code used to detect objects.
       * **main.py**: main program used to detect objects by some ONNX models from [ONNX Model Zoo](https://github.com/onnx/models).
       * **getmodel.py**: used to preprocess input data and parse output data for the following ONNX models:
           * [Tiny YOLOv2](https://github.com/onnx/models/tree/master/vision/object_detection_segmentation/tiny_yolov2)
           * [YOLO V3](https://github.com/onnx/models/tree/master/vision/object_detection_segmentation/yolov3)
           * [Faster R-CNN](https://github.com/onnx/models/tree/master/vision/object_detection_segmentation/faster-rcnn)
           * [Emotion FERPlus](https://github.com/onnx/models/tree/master/vision/body_analysis/emotion_ferplus) 
   * **Dockerfile.arm32v7** file: include instructions used to install ONNX Runtime and its related Python packages, download ONNX models, and copy sample code to build the container image.
   * **module.json** file: config file for this module.

1. Launch **Visual Studio Code**, open folder to **onnxruntime-sample** located folder, and set Default Platform to be **arm32v7**.

2. Update the **.env** file with the values for your container registry.  Refer to [**Create a container registry**](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-portal#create-a-container-registry) for more detail about ACR.
     ```<language>
     CONTAINER_REGISTRY_NAME="Your ACR address"
     CONTAINER_REGISTRY_USERNAME="Your ACR username"
     CONTAINER_REGISTRY_PASSWORD="Your ACR password"
     ```

3. Sign in **Azure Container Registry** by entering the following command in the **Visual Studio Code** integrated terminal (replace <CONTAINER_REGISTRY_NAME>, <CONTAINER_REGISTRY_USERNAME>, and <CONTAINER_REGISTRY_PASSWORD> to your container registry values specified in the **.env** file):
    ```<language>
    docker login <CONTAINER_REGISTRY_NAME> -u <CONTAINER_REGISTRY_USERNAME> -p <CONTAINER_REGISTRY_PASSWORD> 
    ```

4. Open **modules\OnnxRuntimeModule\module.json** file and change **version** setting in **tag** property to create a new version of the module image.

5. Right-clicking on **deployment.template.json** file and select **[Build and Push IoT Edge Solution]** command to generate a new **deployment.arm32v7.json** file in **config** folder, build a module image, and push the image to the specified ACR repository.

6. Right-clicking on **config/deployment.arm32v7.json** file, select **[Create Deployment for Single Device]**, and choose the targeted IoT Edge device to deploy the container Image.

7. Modify **deployment.arm32v7.json** by changing the model value for **CMD** option and deploy it again to run other ONNX models .  Available ONNX model values:
    * **tinyyolov2**: Tiny YOLO V2 (it's default model)
    * **yolov3**: YOLO V3
    * **fasterrcnn**: Faster R-CNN
        > **Note:** if OnnxRuntimeModule fails to restart after changing to Faster R-CNN model, please check its log by using `adb shell docker logs -f OnnxRuntimeModule` commannd.  If it prompts "Exception in detect_image: Method run failed due to: [ONNXRuntimeError] : 1 : GENERAL ERROR : std::bad_alloc" exception, it means not enough memory and you'd better use `adb reboot` to restart the device.
    * **emotion**: Emotion FERPlus

8. Check the detection result:
    * Set your local machine to connect to the same Wi-Fi as Vision AI Dev Kit connecting.
    * Use `adb shell ifconfig wlan0` command to get the camera's wireless IP address or find the IP address from **rtst_addr** property shown in OnnxRuntimeModule's **Module Identity Twin** page.
    * Open a browser to browse `http://CAMERA_IP:1080/media/result.jpg` or `http://CAMERA_IP:1080/media/result.html` (it will auto refresh result.jpg by using Edge) where CAMERA_IP is the camera's IP address you found above.
    * Or execute `python show_result.py "http://CAMERA_IP:1080/media/result.jpg"` command from **onnxruntime-sample\modules\OnnxRuntimeModule\test** folder to display the detection result in an OpenCV window.
        * Type any key in the OpenCV window to quit show_result.py.
        * Set **is_recording** flag to be **True** to record detection results to a video file.
    * Detection results:
        * Tiny YOLO V2: [result.jpg](./modules/OnnxRuntimeModule/test/result.jpg), [result.mp4](./modules/OnnxRuntimeModule/test/result.mp4)
        * YOLO V3: [result_yolov3.jpg](./modules/OnnxRuntimeModule/test/result_yolov3.jpg)
        * Faster R-CNN: [result_fasterrcnn.jpg](./modules/OnnxRuntimeModule/test/result_fasterrcnn.jpg) 
        * Emotion FERPlus: [result_emotion.jpg](./modules/OnnxRuntimeModule/test/result_emotion.jpg)

> **Note:**
> * This POC sample is used to validate ONNX Runtime package can be installed on Vision AI DevKit, and it can use ONNX Runtime APIs to load ONNX models from ONNX Model Zoo and detect objects by the images captured from Vision AI DevKit.  So we will focus on the detection results and ignore the following unstable/lag/delay issues:
>     * There is no API can be used to captures frames from camera directly in the current Camera SDK, so this sample capture frames from camera's preview RTSP stream.  And it needs to use a faster WiFi connection, otherwise cv2.VideoCapture() will read RTSP stream fail and need to be re-initialized frequently caused by the RTSP sream is not stable.
>     * There is no API can be used to draw labels and bounding boxes to HDMI display directly, so this sample saves the detection result to a disk file and the result content update will be delayed about 5 to 10 sec slower than the preview  update.
>     * The detection accruacy for YOLO V3 and Faster RCNN ONNX models are better than YOLO V2 model, but YOLO V3's inference rate 0.16 fps and Faster RCNN's 0.04 fps are much slower than YOLO V2's 1.42 fps under CPU mode.  So it's better to run YOLO V3 and Faster RCNN models under DSP mode in the future. Check [result_yolov3.jpg](./modules/OnnxRuntimeModule/test/result_yolov3.jpg) and [result_fasterrcnn.jpg](./modules/OnnxRuntimeModule/test/result_fasterrcnn.jpg) vs [result_tinyyolov2.jpg](./modules/OnnxRuntimeModule/test/result_tinyyolov2.jpg).
