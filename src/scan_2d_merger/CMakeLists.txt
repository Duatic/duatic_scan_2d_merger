cmake_minimum_required(VERSION 3.22)
project(scan_2d_merger)

if(NOT CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# C++20 is required for std::barrier. Enforced unconditionally so an external
# -DCMAKE_CXX_STANDARD flag cannot silently downgrade and break the build.
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(ament_cmake_auto REQUIRED)
ament_auto_find_build_dependencies()

# Explicit Threads linkage rather than relying on rclcpp's transitive pull-in.
find_package(Threads REQUIRED)

set(COMPONENT_NAME component_${PROJECT_NAME})
ament_auto_add_library(${COMPONENT_NAME} SHARED ./src/scan_2d_merger.cpp)
target_include_directories(${COMPONENT_NAME} PUBLIC include)
target_link_libraries(${COMPONENT_NAME} Threads::Threads)
rclcpp_components_register_nodes(${COMPONENT_NAME} "util::LaserScanMerger")

install(TARGETS ${COMPONENT_NAME}
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

install(DIRECTORY launch config rviz test
  DESTINATION share/${PROJECT_NAME}
)

install(DIRECTORY include
  DESTINATION include
)

if(BUILD_TESTING)
  # Linting is handled by pre-commit (see .pre-commit-config.yaml), not the
  # ament_lint test suite, so it is intentionally not wired up here.
  find_package(ament_cmake_gtest REQUIRED)
  find_package(tf2_ros REQUIRED)

  ament_add_gtest(test_scan_merger test/test_scan_merger.cpp)
  target_include_directories(test_scan_merger PRIVATE include)
  # Link against modern namespaced targets rather than ament_target_dependencies,
  # which was removed in ROS 2 Lyrical/Rolling.
  target_link_libraries(test_scan_merger
    ${COMPONENT_NAME}
    Threads::Threads
    rclcpp::rclcpp
    ${sensor_msgs_TARGETS}
    ${geometry_msgs_TARGETS}
    tf2_ros::tf2_ros)
endif()

# Opts in to the scoped header install layout that becomes the default in Kilted Kaiju.
ament_auto_package(USE_SCOPED_HEADER_INSTALL_DIR)
