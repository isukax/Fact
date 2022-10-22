cmake_minimum_required(VERSION 3.24)

set(project_name "Fact")
project(${project_name})

find_package(Vulkan REQUIRED)

# Executable
# TODO remove file(GLOB), use target_sources
file(GLOB_RECURSE src src/*.cpp src/*.h shaders/*.vert shaders/*.frag)
add_executable(${project_name} ${src})
target_compile_options(${project_name} PRIVATE -W4)
set_property(TARGET ${project_name} PROPERTY CXX_STANDARD 20)
set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT ${project_name})
target_include_directories(${project_name} PUBLIC ${Vulkan_INCLUDE_DIRS})

add_subdirectory(external/glfw EXCLUDE_FROM_ALL)
add_subdirectory(external/glm EXCLUDE_FROM_ALL)
add_subdirectory(external/assimp EXCLUDE_FROM_ALL)

target_link_libraries(${project_name}
  PRIVATE ${Vulkan_LIBRARIES}
  PRIVATE glfw
  PRIVATE glm
  PRIVATE assimp
)
target_include_directories(glfw PUBLIC ${Vulkan_INCLUDE_DIRS})
target_include_directories(${project_name} PUBLIC external/stb)
target_include_directories(${project_name} PUBLIC external/assimp)

if(WIN32)
	set(MSVC_DISABLED_WARNINGS_LIST
		"C4100" # unreferenced formal parameter
		"C4201" # nonstandard extension used: nameless struct/union
		"C4820" # '<x>' : '<y>' bytes padding added after data member '<z>'
		)
	string(REPLACE "C" " -wd" MSVC_DISABLED_WARNINGS_STR ${MSVC_DISABLED_WARNINGS_LIST})
	set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS} -W4 -WX /MP /arch:AVX2 ${MSVC_DISABLED_WARNINGS_STR}")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -W4 -WX /MP /arch:AVX2 ${MSVC_DISABLED_WARNINGS_STR}")
endif()

###############
# shaders
###############

# https://github.com/StableCoder/cmake-scripts/blob/main/glsl-shaders.cmake
find_program(GLSLANGVALIDATOR_EXE "glslangValidator")
mark_as_advanced(FORCE GLSLANGVALIDATOR_EXE)
if(GLSLANGVALIDATOR_EXE)
  message(STATUS "glslangValidator found: ${GLSLANGVALIDATOR_EXE}")
else()
  message(STATUS "glslangValidator not found!")
endif()

function(target_glsl_shaders TARGET_NAME)
  if(NOT GLSLANGVALIDATOR_EXE)
    message(
      FATAL_ERROR "Cannot compile GLSL to SPIR-V is glslangValidator not found!"
    )
  endif()

  set(OPTIONS)
  set(SINGLE_VALUE_KEYWORDS)
  set(MULTI_VALUE_KEYWORDS INTERFACE PUBLIC PRIVATE COMPILE_OPTIONS)
  cmake_parse_arguments(
    target_glsl_shaders "${OPTIONS}" "${SINGLE_VALUE_KEYWORDS}"
    "${MULTI_VALUE_KEYWORDS}" ${ARGN})

  foreach(GLSL_FILE IN LISTS target_glsl_shaders_INTERFACE)
    add_custom_command(
      OUTPUT ${GLSL_FILE}.spv
      COMMAND ${GLSLANGVALIDATOR_EXE} ${target_glsl_shaders_COMPILE_OPTIONS} -V
              "${GLSL_FILE}" -o "${GLSL_FILE}.spv"
      MAIN_DEPENDENCY ${GLSL_FILE})

    target_sources(${TARGET_NAME} INTERFACE ${GLSL_FILE}.spv)
  endforeach()

  foreach(GLSL_FILE IN LISTS target_glsl_shaders_PUBLIC)
    add_custom_command(
      OUTPUT ${GLSL_FILE}.spv
      COMMAND ${GLSLANGVALIDATOR_EXE} ${target_glsl_shaders_COMPILE_OPTIONS} -V
              "${GLSL_FILE}" -o "${GLSL_FILE}.spv"
      MAIN_DEPENDENCY ${GLSL_FILE})

    target_sources(${TARGET_NAME} PUBLIC ${GLSL_FILE}.spv)
  endforeach()

  foreach(GLSL_FILE IN LISTS target_glsl_shaders_PRIVATE)
    add_custom_command(
      OUTPUT ${GLSL_FILE}.spv
      COMMAND ${GLSLANGVALIDATOR_EXE} ${target_glsl_shaders_COMPILE_OPTIONS} -V
              "${GLSL_FILE}" -o "${GLSL_FILE}.spv"
      MAIN_DEPENDENCY ${GLSL_FILE})

    target_sources(${TARGET_NAME} PRIVATE ${GLSL_FILE}.spv)
  endforeach()
endfunction()

file(GLOB_RECURSE shaders 
	shaders/*.vert
	shaders/*.frag)

target_glsl_shaders(${project_name}
    PRIVATE ${shaders}
    COMPILE_OPTIONS --target-env vulkan1.1)