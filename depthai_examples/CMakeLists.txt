cmake_minimum_required(VERSION 3.0.2)
# set(CMAKE_BUILD_TYPE Debug)

# get_filename_component (DEPTHAI_CORE "${CMAKE_CURRENT_LIST_DIR}/../external/depthai-core" REALPATH)
# include ("${DEPTHAI_CORE}/cmake/HunterGate.cmake")

set (CMAKE_POSITION_INDEPENDENT_CODE ON)
# set (HUNTER_CONFIGURATION_TYPES "Release" CACHE STRING "Hunter dependencies list of build configurations")


# HunterGate (
#     URL "https://github.com/cpp-pm/hunter/archive/v0.23.258.tar.gz"
#     SHA1 "062a19ab13ce8dffa9a882b6ce3e43bdabdf75d3"
#     FILEPATH ${DEPTHAI_CORE}/cmake/Hunter/config.cmake # Add depthai-core config  (hunter limitation)
# )
# add_compile_options(-std=c++11)
project(depthai_examples)
# add_compile_options(-std=c++14)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
## Compile as C++11, supported in ROS Kinetic and newer
# add_compile_options(-std=c++11)

## Find catkin macros and libraries
## if COMPONENTS list like find_package(catkin REQUIRED COMPONENTS xyz)
## is used, also find other catkin packages
find_package(catkin REQUIRED COMPONENTS
  camera_info_manager
  depthai_ros_msgs
  roscpp
  sensor_msgs
  std_msgs
  cv_bridge
  vision_msgs
  depthai_bridge
  message_filters
  nodelet
)

set(_opencv_version 4)
find_package(OpenCV 4 QUIET COMPONENTS imgproc highgui)
if(NOT OpenCV_FOUND)
  message(STATUS "----------------Did not find OpenCV 4, trying OpenCV 3--------------")
  set(_opencv_version 3)
  find_package(OpenCV 3 REQUIRED COMPONENTS imgproc highgui)
endif()
find_package(depthai REQUIRED)
# find_package(depthai REQUIRED PATHS "/home/sachin/Desktop/luxonis/depthai-core/build/install/lib/cmake")

# add_subdirectory(${DEPTHAI_CORE} depthai-core)

catkin_package(
 INCLUDE_DIRS include
 LIBRARIES ${PROJECT_NAME}
 CATKIN_DEPENDS depthai_ros_msgs roscpp sensor_msgs std_msgs camera_info_manager depthai_bridge vision_msgs cv_bridge message_filters
 DEPENDS
 OpenCV
 )

###########
## Build ##
###########

## Specify additional locations of header files
## Your package locations should be listed before other locations
include_directories(
  include
  ${catkin_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
)

FILE(GLOB EXAMPLE_LIB
   "lib/*.cpp"
)

add_library(${PROJECT_NAME} ${EXAMPLE_LIB})

add_dependencies(${PROJECT_NAME} ${${PROJECT_NAME}_EXPORTED_TARGETS}
  ${catkin_EXPORTED_TARGETS} 
)

target_link_libraries(${PROJECT_NAME}
  ${catkin_LIBRARIES}
  depthai::core
  opencv_imgproc
  opencv_highgui
)


add_library(nodelet_stereo src/stereo_nodelet.cpp)

add_dependencies(nodelet_stereo ${${PROJECT_NAME}_EXPORTED_TARGETS}
  ${catkin_EXPORTED_TARGETS} 
  ${PROJECT_NAME}
)

target_link_libraries(nodelet_stereo
  ${catkin_LIBRARIES}
  depthai::core
  ${PROJECT_NAME}
  opencv_imgproc
  opencv_highgui
)

# hunter_private_data(
#     URL "https://artifacts.luxonis.com/artifactory/luxonis-depthai-data-local/network/mobilenet-ssd_openvino_2021.2_6shave.blob"
#     SHA1 "f0e14978b3f77a4f93b9f969cd39e58bb7aef490"
#     FILE "mobilenet-ssd_openvino_2021.2_6shave.blob"
#     LOCATION mobilenet_blob
# )

set(mobilenet_blob "${PROJECT_SOURCE_DIR}/resources/mobilenet-ssd_openvino_2021.2_6shave.blob")
file(DOWNLOAD "https://artifacts.luxonis.com/artifactory/luxonis-depthai-data-local/network/mobilenet-ssd_openvino_2021.2_6shave.blob" 
${mobilenet_blob}
TIMEOUT 60  # seconds
EXPECTED_HASH SHA1=f0e14978b3f77a4f93b9f969cd39e58bb7aef490
TLS_VERIFY ON)

set(tiny_yolo_v4_blob "${PROJECT_SOURCE_DIR}/resources/tiny-yolo-v4_openvino_2021.2_6shave.blob")
file(DOWNLOAD "https://artifacts.luxonis.com/artifactory/luxonis-depthai-data-local/network/tiny-yolo-v4_openvino_2021.2_6shave.blob"
${tiny_yolo_v4_blob}
TIMEOUT 60  # seconds
EXPECTED_HASH SHA1=219d949610a5760e62a8458941e1300b81c3fe4a
TLS_VERIFY ON)


macro(dai_add_node node_name node_src)
    add_executable("${node_name}" "${node_src}")
    
    add_dependencies("${node_name}"
      ${catkin_EXPORTED_TARGETS} 
      ${${PROJECT_NAME}_EXPORTED_TARGETS}
      ${PROJECT_NAME} 
    )

    target_link_libraries("${node_name}" 
      ${catkin_LIBRARIES}
      depthai::core
      ${PROJECT_NAME}
      opencv_imgproc
      opencv_highgui
    )
    
endmacro() 


dai_add_node(crop_control_service src/crop_control_service.cpp)
dai_add_node(stereo_node src/stereo_publisher.cpp)
dai_add_node(rgb_node src/rgb_publisher.cpp)
dai_add_node(mobilenet_node src/mobilenet_publisher.cpp)
dai_add_node(rgb_stereo_node src/rgb_stereo_node.cpp)
dai_add_node(stereo_mimic_node src/stereo_mimic_node.cpp)
dai_add_node(rgb_subscriber_node src/rgb_video_subscriber.cpp)
dai_add_node(yolov4_spatial_node src/yolov4_spatial_publisher.cpp)


target_compile_definitions(mobilenet_node PRIVATE BLOB_PATH="${mobilenet_blob}")
target_compile_definitions(yolov4_spatial_node PRIVATE BLOB_PATH="${tiny_yolo_v4_blob}")
