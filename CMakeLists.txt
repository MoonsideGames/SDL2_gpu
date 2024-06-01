# CMake Project for SDL2_gpu
# Written by @thatcosmonaut
cmake_minimum_required(VERSION 2.8.12)
project(SDL_gpu C)

# Options
option(BUILD_SHARED_LIBS "Build shared library" ON)

# Version
SET(LIB_MAJOR_VERSION "2")
SET(LIB_MINOR_VERSION "0")
SET(LIB_REVISION "0")
SET(LIB_VERSION "${LIB_MAJOR_VERSION}.${LIB_MINOR_VERSION}.${LIB_REVISION}")

# Build Type
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	# By default, we use Release
	message(STATUS "Setting build type to 'Release' as none was specified.")
	set(CMAKE_BUILD_TYPE "Release" CACHE
		STRING "Choose the type of build." FORCE
	)

	# Set the possible values of build type for cmake-gui
	set_property(CACHE CMAKE_BUILD_TYPE PROPERTY
		STRINGS "Debug" "Release" "RelWithDebInfo"
	)
endif()

# Platform Flags
if(APPLE)
	set(CMAKE_MACOSX_RPATH ON)
	set(CMAKE_OSX_DEPLOYMENT_TARGET 10.9)
	set(LOBJC "objc")
elseif(WIN32)
	# "SDL_gpu.dll", not "libSDL_gpu.dll"
	set(CMAKE_SHARED_LIBRARY_PREFIX "")
endif()

# Defines
add_definitions(
	-DSDL_GPU_ENABLE_VULKAN
)

if (WIN32)
	add_definitions(
		-DSDL_GPU_ENABLE_D3D11
	)
endif()

# Source lists
add_library(SDL_gpu
	# Public Headers
	include/SDL_gpu.h
	# Internal Headers
	src/SDL_gpu_driver.h
    src/SDL_gpu_spirv_c.h
    src/spirv_cross_c.h
	src/vulkan/SDL_gpu_vulkan_vkfuncs.h
	# Source Files
	src/SDL_gpu.c
	src/d3d11/SDL_gpu_d3d11.c
	src/vulkan/SDL_gpu_vulkan.c
)

# Build flags
if(NOT MSVC)
	set_property(TARGET SDL_gpu PROPERTY COMPILE_FLAGS "-std=gnu99 -Wall -Wno-strict-aliasing -pedantic")
endif()

# Windows is silly and we need to manually include the Vulkan SDK
if(MSVC)
	target_include_directories(SDL_gpu PUBLIC $ENV{VULKAN_SDK}/include)
endif()

# SDL_gpu folders as includes, for other targets to consume
target_include_directories(SDL_gpu PUBLIC
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/Vulkan-Headers/include>
)

# MinGW builds should statically link libgcc
if(MINGW)
	target_link_libraries(SDL_gpu PRIVATE -static-libgcc)
endif()

# Soname
set_target_properties(SDL_gpu PROPERTIES OUTPUT_NAME "SDL_gpu"
	VERSION ${LIB_VERSION}
	SOVERSION ${LIB_MAJOR_VERSION}
)

# SDL2 Dependency
if (DEFINED SDL2_INCLUDE_DIRS AND DEFINED SDL2_LIBRARIES)
	message(STATUS "using pre-defined SDL2 variables SDL2_INCLUDE_DIRS and SDL2_LIBRARIES")
	target_include_directories(SDL_gpu PUBLIC "$<BUILD_INTERFACE:${SDL2_INCLUDE_DIRS}>")
	target_link_libraries(SDL_gpu PUBLIC ${SDL2_LIBRARIES})
else()
	# Only try to autodetect if both SDL2 variables aren't explicitly set
	find_package(SDL2 CONFIG)
	if (TARGET SDL2::SDL2)
		message(STATUS "using TARGET SDL2::SDL2")
		target_link_libraries(SDL_gpu PUBLIC SDL2::SDL2)
	elseif (TARGET SDL2)
		message(STATUS "using TARGET SDL2")
		target_link_libraries(SDL_gpu PUBLIC SDL2)
	else()
		message(STATUS "no TARGET SDL2::SDL2, or SDL2, using variables")
		target_include_directories(SDL_gpu PUBLIC "$<BUILD_INTERFACE:${SDL2_INCLUDE_DIRS}>")
		target_link_libraries(SDL_gpu PUBLIC ${SDL2_LIBRARIES})
	endif()
endif()
