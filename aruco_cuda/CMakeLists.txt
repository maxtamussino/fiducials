cmake_minimum_required(VERSION 3.0.2)
project(aruco_cuda)

## Compile as C++11, supported in ROS Kinetic and newer
add_compile_options(-std=c++11)

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

find_package(OpenCV REQUIRED)
find_package(CUDA REQUIRED)
include_directories("${CUDA_INCLUDE_DIRS}")

generate_dynamic_reconfigure_options(cfg/DetectorParams.cfg)

catkin_package(
#  INCLUDE_DIRS include
#  LIBRARIES mav_identify
#  CATKIN_DEPENDS message_runtime
#  DEPENDS system_lib
)

include_directories(include/opencv2 ${catkin_INCLUDE_DIRS})

add_executable(aruco_detect_cuda src/aruco_detect_cuda.cpp src/aruco_cuda.cpp src/apriltag_quad_thresh.cpp src/zmaxheap.cpp)

add_dependencies(aruco_detect_cuda ${${PROJECT_NAME}_EXPORTED_TARGETS}
                 ${catkin_EXPORTED_TARGETS})

target_link_libraries(aruco_detect_cuda ${catkin_LIBRARIES} ${OpenCV_LIBS} ${CUDA_LIBS})