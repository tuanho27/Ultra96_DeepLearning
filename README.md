# **README**

### step 1: Setup
 - Download DNNDK tool from Xilinx and setup on host machine (follow DNNDK userguide)
 [Link]: https://www.xilinx.com/member/forms/download/dnndk-eula-xef.html?filename=xlnx_dnndk_v2.08_190201.tar.gz
 ```Note```: version 2.08 is plausible with board's image version 2018-12-10, all DNNDK late version is not supported by the board's image yet.
- Download image and prepare to set up the evaluation board (Ultra96)
 [Link]: https://www.xilinx.com/member/forms/download/ultra96-image-license-xef.html?filename=xilinx-ultra96-desktop-stretch-2018-12-10.img.zip
 Then flash the downloaded OS image to the SDCard by using [etcher](https://etcher.io/) as below:
 Install etcher, open it and select the image OS, insert the SD card into the slot on your computer. Click the Flash button and wait till the process is done.
 Copy the sample from DNNDK tool folder (above install): (*path*/xilinx_dnndk_v2.08/Ultra96/)  to SDcard root folder (/media/user/ROOTFS/root/), be noted that this step also could be done by using wifi and ```bash scp``` command after the board is boot (incase your wifi connection is fast enough)
- Now we are going to boot the Ultra96 board with this SDCard
  Insert the SDCard into board and connect the UART interface to the host PC and other peripherals as required. Turn on the power, and wait for the system to boot.
 
  Configure the UART with following settings :
    * *baud rate*: 115200 bps
    * *data bit*: 8
    * *stop bit*: 1
    * *no parity*
  Or you can also access the board as Standalone by connecting keyboard, mouse and monitor (user: root, pwd: root)

### step 2: Test examples
   - Asume that you access the board as Standalone mode (this will make the board works as it's performance and not effected by the wifi speed)
   Fisrtly, we will test with the available example the we copy to the board before.
     ```bash
     cd /root/Ultra96/sample
     cd video_analysis
     make
     ./video_analysis video/*.mp4 
        ``` 
        If everything work well, you will see the demo example made by Deephi
        
  - Now, we download the ssd-mobilenet-v2 from model zoo and test this model perform on our board
  [link]: https://github.com/Xilinx/Edge-AI-Platform-Tutorials/tree/master/docs/AI-Model-Zoo
  Download all and choose cf_ssdmobilenetv2_bdd_360_480_6.57G, this will contain all type of models (floating point and quanitize model already done by Xilinx), for more information about how to make it, refer (https://github.com/Xilinx/Edge-AI-Platform-Tutorials/tree/master/docs/ML-SSD-PASCAL)
  Then create a script to compile the compress model prototxt and weight into the FPGA execute file (DPU kernel):
  ( the deploy.caffemodel and deploy.prototxt are copied to ssd_mobilenet_v2.prototxt and ssd_mobilenet_v2.caffemodel just for remember )
  Create the dnnc.sh script as below:
    ```bash
        net="ssd_mobilenet_v2"
        dnndk_board="Ultra96"
        # Set DPU Arch
        DPU_ARCH="1152FA"
        info=$(ls /usr/local/bin/dnnc -lh |grep dpu1.3.0)
        if [ "$info" != "" ]; then
            DPU_ARCH="4096FA"
        fi
        # Set CPU Arch
        CPU_ARCH="arm64"
        # Set DNNC mode
        DNNC_MODE="debug"
        model_dir="deploy"
        output_dir="dnnc_output"
        
        if [ ! -d "$model_dir" ]; then
            echo "Can not found directory of $model_dir"
            exit 1
        fi
        [ -d "$output_dir" ] || mkdir "$output_dir"
        echo "Compiling network: ${net}"
        
        dnnc   --prototxt=${model_dir}/ssd_mobilenet_v2.prototxt     \
               --caffemodel=${model_dir}/ssd_mobilenet_v2.caffemodel \
               --output_dir=${output_dir}                  \
               --dpu=${DPU_ARCH}                           \
               --net_name=${net}                           \
               --mode=${DNNC_MODE}                         \
               --cpu_arch=${CPU_ARCH}
    ```
    Execute this script, we will receive the ssd_mobilenet_v2.elf file
    - After we has the *elf file, we need to create the C++ wrapper to call and exec this DPU kernel. Since ssd model is the example from Deephi (the example that we test before), we can reuse it and change the setting of Prior box to make it suitable for MobilenetV2 backbone.
    Copy  /root/Ultra96/sample/video_analysis/ to  /root/Ultra96/sample/ssd_mobilenet_v2/
    Copy ssd_mobilenet_v2.elf /root/Ultra96/sample/ssd_mobilenet_v2/model
    - Below is the change in file /root/Ultra96/sample/ssd_mobilenet_v2/src/main.cc
    ```c++
    // DPU Kernel name for SSD Convolution layers
    #define KERNEL_CONV "ssd_mobilenet_v2"
    // DPU node name for input and output
    #define CONV_INPUT_NODE "conv1"
    int num_classes = 11;
    ...
    // vehicle detect
    prior_boxes.emplace_back(PriorBoxes{
          480, 360, 60, 45, variances, {15.0, 30}, {33.0, 66}, {2}, 0.5, 8.0, 8.0});
    prior_boxes.emplace_back(PriorBoxes{
          480, 360, 30, 23, variances, {66.0}, {127.0}, {2, 3}, 0.5, 16, 16});
    // prior_boxes.emplace_back(PriorBoxes{
    //      480, 360, 15, 12, variances, {127.0}, {188.0}, {2, 3}, 0.5, 32, 32});
    prior_boxes.emplace_back(PriorBoxes{
          480, 360, 7, 5, variances, {188.0}, {249.0}, {2, 3}, 0.5, 64, 64});
    prior_boxes.emplace_back(PriorBoxes{
          480, 360, 3, 2, variances, {249.0}, {310.0}, {2}, 0.5, 100, 100});
    prior_boxes.emplace_back(PriorBoxes{
          480, 360, 2, 1, variances, {310.0}, {372.0}, {2}, 0.5, 300, 300});
     ```

-  execute make command and wait to finish
-  check the process by execute: ```./ssd_mobilenet_v2 video/*mp4 ``` 

### Step 3: Handle custome network: cybercore ssd_denet (T.B.D)
- Since this model was not compressed yet, we need to handle the compression part using DCent tool
- Prepare the script dcent.sh, calibaration dataset and model.prototxt file

