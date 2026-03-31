cmake_minimum_required(VERSION 3.8)
project(autoware_universe_trajectory_generator_launch)

find_package(ament_cmake REQUIRED)

install(DIRECTORY launch/ DESTINATION share/${PROJECT_NAME}/launch)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  set(ament_cmake_copyright_FOUND TRUE)
  set(ament_cmake_cpplint_FOUND TRUE)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
