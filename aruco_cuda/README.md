# aruco_cuda package

**NOT WORKING YET**

As mentioned in this fork's [README](../README.md), this package provides a GPU aruco detection node for NVIDIA Jetson platforms.

Unfortunately, to the date this is written, ArUco detection is not implemented with CUDA support even in the manually built OpenCV version. This is what the `aruco_cuda` package does. It includes files from [opencv_contrib/tree/3.4.16/modules/aruco](https://github.com/opencv/opencv_contrib/tree/3.4.16/modules/aruco), which are modified to utilise underlying OpenCV CUDA features.


## Prerequisites

### Building OpenCV 3.4.16 with CUDA

To enable CUDA features for OpenCV, it must be built from source using the appropriate cmake flags. The standard ROS Melodic OpenCV version is `3.2`. As many ROS packages also rely on OpenCV `3.x` (and do not build with `4.x`), version `3.4.16` is chosen to be built from source with the appropriate CUDA support flags. It is compatible with `3.2` and distinguishable from it by specifying the exact version in any `CMakeLists.txt` files. Removing ROS OpenCV `3.2` with `apt` (in order to replace it by manual installation) would cause many other ROS packages to be removed by `apt`.

Detailed instructions on how to build OpenCV are available on [qengineering.eu](https://qengineering.eu/install-opencv-4.5-on-Jetson-nano.html). Do **not** use the automated script provided at the beginning of these instructions. Follow the *manual* installation so the following aspects of the instructions may be customised:

- Install *without* QT
- Instead of installing `4.5.x`, choose version `3.4.16` (compatible with ROS OpenCV `3.2`)
- Instead of installing to `/usr`, install to `/usr/local`

Be aware that the build process may take several hours. The bottleneck is the low RAM of only 2GB. The described swapfile extends RAM but is much slower. Therefore, only using `make -j2` is recommended.


### Replace fiducials package

The default `apt` package [ros-melodic-fiducials](http://wiki.ros.org/fiducials) is replaced by a [forked version](https://github.com/maxtamussino/fiducials). If this feature is persued, remove the default `apt` package

```
$ sudo apt remove ros-melodic-fiducials
```

and clone the appropriate forked version

```
$ git clone https://github.com/maxtamussino/fiducials.git
```


### Modify cv_bridge 

The `fiducials` package depends on `cv_bridge` and the default ROS package `cv_bridge` is built with OpenCV `3.2`. Mixing OpenCV versions like this causes issues. The default ROS `cv_bridge` package cannot be removed by `apt`, because this would cause many other ROS packages to be removed because of this missing dependency. To circumvent this issue, the source code of `cv_bridge` is built within the catkin workspace and its `CmakeLists.txt` is changed so that it builds with the manually installed OpenCV `3.4.16`.

```
$ mkdir -p ~/Git/ros-perception
$ cd ~/Git/ros-perception
$ git clone https://github.com/ros-perception/vision_opencv/tree/melodic
$ cd ~/catkin_ws/src
$ ln -s ~/Git/ros-perception/vision_opencv
```

In its `CmakeLists.txt`, change the line

```
find_package(OpenCV 3 REQUIRED
```

which will find the default ROS OpenCV `3.2`, to

```
find_package(OpenCV 3.4 REQUIRED PATHS /usr/local/
```

to look for the manually installed version `3.4.16`. All other packages will still find the ROS default version of `cv_bridge` in their `CmakeLists.txt`. Only `aruco_cuda` will find the version built with OpenCV `3.4.16`.

## What has been done

### Starting point

As a starting point, the `aruco_detect` package from the original `fiducials` was copied and renamed to `aruco_cuda` because it provides the necessary functionality using the CPU. Additionally, the CPU implementation of ArUco detection of OpenCV `3.4.16` ([opencv_contrib/modules/aruco/src](https://github.com/opencv/opencv_contrib/tree/3.4.16/modules/aruco/src)) is also copied into the `src` directory of this package. This is done in order to replace CPU library OpenCV calls to local GPU implementations.


### Simple changes

For the manually built OpenCV `3.4.16`, CUDA functionality is enabled. To use this, the necessary header files need to be imported.

```c++
#include <opencv2/cudafilters.hpp>
#include <opencv2/cudaarithm.hpp>
#include <opencv2/cudawarping.hpp>
```

Additional small changes:

- Replaced all functions with `cv::cuda` equivalents where possible
- Replaced CPU data structures `cv::Mat` with GPU data structures `cv::cuda::GpuMat` to ensure GPU use


### Adaptive thresholding

The function `_detectInitialCandidates` calls `_threshold` in a loop. The function `_threshold` was re-implemented to use GPU functionality:

```c++
Ptr<cuda::Filter> filter = cuda::createBoxFilter(CV_8UC1, CV_8UC1, Size(winSize,winSize));
filter->apply(_in, _out);                // _out holds mean of neighbouring pixels
cuda::add(_out, -constant, _out);        // _out holds mean - C
cuda::compare(_in, _out, _out, CMP_LT);  // compare input to mean - C and store 255 if in<out and 0 if in>=out
```

This lead to a detection time of around 650ms (for a frame rate of 30Hz, the absolute maximum is 33ms) for the standard settings. The bottleneck is the application of the box filter. Although this is a very simple and highly parallelisable operation, it takes an awfully long time. Please refer to [this line](https://github.com/opencv/opencv_contrib/blob/a26f71313009c93d105151094436eecd4a0990ed/modules/cudafilters/src/filtering.cpp#L164) in `opencv_contrib` to see that OpenCV uses the correct function `nppiFilterBox_8u_C1R` which is provided by *NVIDIA 2D Image And Signal Performance Primitives* (NPP).


### Memory transfer

The function `_findMarkerContours` could not be implemented using GPU functionality. Initially, `cv::cuda::GpuMat::download(...)` was used to simply transfer the data to the CPU and perform the operation there. However, the function `_detectInitialCandidates` calls `_findMarkerContours` in a loop within a loop. 
In [this](https://forums.developer.nvidia.com/t/eliminate-upload-download-for-opencv-cuda-gpumat-using-shared-memory/83090/6) very helpful thread, a possible solution is discussed. Using unified memory, both CPU datastructure `cv::Mat` and the GPU datastructure `cv::aruco::GpuMat` can point to the same data. It works as follows:

```c++
const unsigned int imageByteSize = img_width * img_height * img_channels;
void *unified_ptr;
cudaMallocManaged(&unified_ptr, imageByteSize);
cv::Mat imgCPU(img_width, img_height, CV_8UC1, unified_ptr);
cv::cuda::GpuMat imgGPU(img_width, img_height, CV_8UC1, unified_ptr);
...
cudaFree(unified_ptr);
```