cmake_minimum_required(VERSION 3.2)
project(PS4EYECam VERSION 0.0.1)
include(ExternalProject)

set(ROOT_DIR ${CMAKE_CURRENT_LIST_DIR})

LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")
IF (NOT MSVC)
    #set(CMAKE_CXX_FLAGS "-std=c++11 -stdlib=libc++")
    set(CMAKE_CXX_FLAGS "-std=c++11")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-switch")
ENDIF()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# Shared architecture label used for install folder locations
if (${CMAKE_C_SIZEOF_DATA_PTR} EQUAL 8)
    if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
        set(ARCH_LABEL "Win64")
    elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
        set(ARCH_LABEL "OSX64")
    else()
        set(ARCH_LABEL "Linux64")
    endif()
else()
    if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
        set(ARCH_LABEL "Win32")
    elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
        set(ARCH_LABEL "OSX32")
    else()
        set(ARCH_LABEL "Linux32")
    endif()
endif()

# When not using MSVC, we recommend using system-wide libraries
# (installed via homebrew on Mac or apt-get in Linux/Ubuntu)
# In MSVC, we recommend downloading pre-compiled binaries where
# possible, and building Protobuf yourself (see README.md)

IF(MSVC)
#I cannot remember which one, but one of our dependencies
#links against static runtime, so we need our apps to link
#against the static runtime too.
#https://cmake.org/Wiki/CMake_FAQ#How_can_I_build_my_MSVC_application_with_a_static_runtime.3F

    SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
    SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")

    set(CompilerFlags
        CMAKE_CXX_FLAGS
        CMAKE_CXX_FLAGS_DEBUG
        CMAKE_CXX_FLAGS_RELEASE
        CMAKE_C_FLAGS
        CMAKE_C_FLAGS_DEBUG
        CMAKE_C_FLAGS_RELEASE
        )
    foreach(CompilerFlag ${CompilerFlags})
      string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
    endforeach()
ENDIF(MSVC)

# OpenCV
# Override by adding "-DOpenCV_DIR=C:\path\to\opencv\build" to your cmake command
IF(NOT OpenCV_DIR)
    IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
        ExternalProject_Add(opencv
          PREFIX ${ROOT_DIR}/deps/opencv
          GIT_REPOSITORY https://github.com/Armada651/opencv.git
          GIT_SHALLOW 1
          GIT_TAG msmf-media-types
          CMAKE_GENERATOR ${gen}
          CMAKE_ARGS
            -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
            -DCMAKE_INCLUDE_PATH=${ROOT_DIR}/deps/local/include
            -DCMAKE_LIBRARY_PATH=${ROOT_DIR}/deps/local/lib
            -DBUILD_WITH_STATIC_CRT:BOOL=ON
            -DBUILD_SHARED_LIBS:BOOL=OFF
            -DBUILD_DOCS:BOOL=OFF
            -DBUILD_EXAMPLES:BOOL=OFF
            -DBUILD_TESTS:BOOL=OFF
            -DBUILD_PERF_TESTS:BOOL=OFF
            #-DCMAKE_BUILD_TYPE:STRING=Release
            -DWITH_FFMPEG:BOOL=OFF
            -DWITH_OPENEXR:BOOL=OFF
            -DWITH_JASPER:BOOL=OFF
            -DWITH_TIFF:BOOL=OFF
            -DWITH_IPP:BOOL=OFF
            -DWITH_CUDA:BOOL=OFF
            -DWITH_MSMF:BOOL=ON
            -DBUILD_opencv_apps:BOOL=OFF
            -DBUILD_opencv_calib3d:BOOL=ON
            -DBUILD_opencv_flann:BOOL=ON
            -DBUILD_opencv_features2d:BOOL=ON
            -DBUILD_opencv_objdetect:BOOL=ON
            -DBUILD_opencv_photo:BOOL=ON
            -DBUILD_opencv_ts:BOOL=OFF
            -DBUILD_opencv_ml:BOOL=ON
            -DBUILD_opencv_video:BOOL=ON
            -DBUILD_opencv_java:BOOL=OFF
            -DBUILD_opencv_python2:BOOL=OFF
            -DBUILD_opencv_python3:BOOL=OFF
            -DPYTHON2_LIBRARY:STRING=C:/Python27/libs/python27.lib
            -DPYTHON3_LIBRARY:STRING=C:/Python35/libs/python35.lib
          INSTALL_DIR ${ROOT_DIR}/deps/local/
        )

        add_definitions(-DHAS_OPENCV)

        set(OpenCV_DIR ${ROOT_DIR}/deps/local)
        set(OpenCV_INCLUDE_DIRS ${ROOT_DIR}/deps/local/include )
        if (${CMAKE_C_SIZEOF_DATA_PTR} EQUAL 8)
            set(OPENCV_LIBS_DIR ${ROOT_DIR}/deps/local/x64/vc14/staticlib)
        else()
            set(OPENCV_LIBS_DIR ${ROOT_DIR}/deps/local/x86/vc14/staticlib)
        endif()

        foreach(__CVLIB core calib3d features2d flann imgproc imgcodecs ml highgui objdetect video videoio)
            set(OpenCV_${__CVLIB}_LIBRARY debug ${OPENCV_LIBS_DIR}/opencv_${__CVLIB}330d.lib optimized ${OPENCV_LIBS_DIR}/opencv_${__CVLIB}330.lib CACHE STRING "" FORCE)
            set(OpenCV_LIBS ${OpenCV_LIBS} ${OpenCV_${__CVLIB}_LIBRARY})
        endforeach(__CVLIB)    

        foreach(__CVLIB libjpeg libpng libwebp zlib ittnotify)
            set(OpenCV_${__CVLIB}_LIBRARY debug ${OPENCV_LIBS_DIR}/${__CVLIB}d.lib optimized ${OPENCV_LIBS_DIR}/${__CVLIB}.lib CACHE STRING "" FORCE)
            set(OpenCV_LIBS ${OpenCV_LIBS} ${OpenCV_${__CVLIB}_LIBRARY})
        endforeach(__CVLIB)     

        LIST(APPEND OpenCV_LIBS vfw32.lib)

    ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
        # Location of homebrew opencv3's OpenCVConfig.cmake
        # Alternatively, can do `brew ln opencv3 --force`
        MESSAGE(STATUS "Using homebrew opencv3")
        set(OpenCV_DIR "/usr/local/opt/opencv3/share/OpenCV")
    ELSE()
        set(OpenCV_DIR "usr/local/share/OpenCV")
    ENDIF()#Windows or Darwin
ENDIF(NOT OpenCV_DIR)
LIST(APPEND CMAKE_MODULE_PATH ${OpenCV_DIR})
set(OpenCV_STATIC ON)
IF(NOT(${CMAKE_SYSTEM_NAME} MATCHES "Windows"))
    FIND_PACKAGE(OpenCV REQUIRED)
ENDIF()

# Step into the subdirectories
add_subdirectory(driver)
add_subdirectory(ewc_test)
add_subdirectory(opencv_test)