cmake_minimum_required(VERSION 3.0.2)
project(aruco_cuda)

## Compile as C++11, supported in ROS Kinetic and newer
add_compile_options(-std=c++11)

set(cv_bridge_DIR /home/jetson/catkin_ws/devel/share/cv_bridge/cmake)

## Find catkin macros and libraries
find_package(catkin REQUIRED COMPONENTS
  roscpp
  tf2_geometry_msgs
  tf2_ros
  tf2
  visualization_msgs
  vision_msgs
  image_transport
  cv_bridge
  sensor_msgs
  std_msgs
  fiducial_msgs
  dynamic_reconfigure
)

find_package(CUDA REQUIRED)
if(CUDA_FOUND)
  message("CUDA Version: ${CUDA_VERSION}")
  message("CUDA include dirs: ${CUDA_INCLUDE_DIRS}")
  include_directories(${CUDA_INCLUDE_DIRS})
else()
  message(FATAL_ERROR "CUDA not found")
endif() 

find_package(OpenCV 3.4 REQUIRED PATHS /usr/local/)
if(OpenCV_FOUND)
  message("OpenCV version: ${OpenCV_VERSION}")
  message("OpenCV include dirs: ${OpenCV_INCLUDE_DIRS}")
  include_directories(${OpenCV_INCLUDE_DIRS})
else()
  message(FATAL_ERROR "OpenCV 3.4.x not found")
endif()

include_directories(include ${catkin_INCLUDE_DIRS})
generate_dynamic_reconfigure_options(cfg/DetectorParams.cfg)

catkin_package(
  INCLUDE_DIRS include
  DEPENDS OpenCV CUDA
)

add_executable(aruco_detect_cuda src/aruco_detect_cuda.cpp src/aruco_cuda.cpp src/apriltag_quad_thresh.cpp src/zmaxheap.cpp)

add_dependencies(aruco_detect_cuda ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(aruco_detect_cuda ${catkin_LIBRARIES} ${OpenCV_LIBS} ${CUDA_LIBRARIES})