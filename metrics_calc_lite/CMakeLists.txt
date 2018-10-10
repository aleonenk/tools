# Copyright (c) 2017 Intel Corporation
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 2.8)

project(metrics_calc_lite)

OPTION(USE_IPP "Enable Intel(R) IPP optimizations" OFF)
OPTION(USE_MSDK "Enable Intel(R) MediaSDK decoders" OFF)

add_executable(metrics_calc_lite src/metrics_calc_lite_utils.cpp src/metrics_calc_lite.cpp)
target_include_directories(metrics_calc_lite PRIVATE include)

if (NOT USE_IPP)
  add_definitions(-DNO_IPP)
else()
  if (NOT DEFINED ENV{IPP_ROOT})
    message(FATAL_ERROR "Env var IPP_ROOT should be set")
  endif()

  set(IPP_ROOT $ENV{IPP_ROOT})
  string(REPLACE "\"" "" IPP_ROOT ${IPP_ROOT})

  if (WIN32)
    if (CMAKE_GENERATOR MATCHES "(Win64|IA64)")
      file(TO_CMAKE_PATH ${IPP_ROOT}/include IPP8_INC)
      file(TO_CMAKE_PATH ${IPP_ROOT}/lib/intel64 IPP8_LIB)
      file(TO_CMAKE_PATH ${IPP_ROOT}/em64t/include IPP_INC)
      file(TO_CMAKE_PATH ${IPP_ROOT}/em64t/lib IPP_LIB)
    else()
      file(TO_CMAKE_PATH ${IPP_ROOT}/include IPP8_INC)
      file(TO_CMAKE_PATH ${IPP_ROOT}/lib/ia32 IPP8_LIB)
      file(TO_CMAKE_PATH ${IPP_ROOT}/ia32/include IPP_INC)
      file(TO_CMAKE_PATH ${IPP_ROOT}/ia32/lib IPP_LIB)
    endif()
    set(IPP_LIBS ippcoremt ippsmt ippimt ippcvmt ippccmt)
  else()
    file(TO_CMAKE_PATH ${IPP_ROOT}/include IPP8_INC)
    file(TO_CMAKE_PATH ${IPP_ROOT}/lib/intel64 IPP8_LIB)
    file(TO_CMAKE_PATH ${IPP_ROOT}/em64t/include IPP_INC)
    file(TO_CMAKE_PATH ${IPP_ROOT}/em64t/lib IPP_LIB)
    set(IPP_LIBS libippi.a libipps.a libippcc.a libippcv.a libippcore.a)
  endif()

  find_path(IPP_INCLUDE NAMES ippcore.h PATHS ${IPP8_INC} ${IPP_INC})
  if (NOT IPP_INCLUDE MATCHES NOTFOUND)
    target_include_directories(metrics_calc_lite BEFORE PRIVATE "${IPP_INCLUDE}")
    message(STATUS "IPP headers found in ${IPP_INCLUDE}")
    file(STRINGS ${IPP_INCLUDE}/ippversion.h IPPVERSION)
    foreach(_LINE ${IPPVERSION})
      string(REGEX MATCH "IPP_VERSION_MAJOR.* ([0-9]+)" VERSION ${_LINE})
      if(CMAKE_MATCH_1)
        message("IPP library version major is ${CMAKE_MATCH_1}")
        if(CMAKE_MATCH_1 LESS 9)
          message("Legacy IPP detected, limited set of metrics will be supported")
          add_definitions(-DLEGACY_IPP)
        else()
          message("Modern IPP detected, all metrics will be supported")
        endif()
      endif()
    endforeach()
  else()
    message(FATAL_ERROR "Can't find IPP headers!")
  endif()

  foreach(LIB ${IPP_LIBS})
    find_library(_LIB NAMES ${LIB} ${LIB}_l PATHS ${IPP8_LIB} ${IPP_LIB})
    if (NOT _LIB MATCHES NOTFOUND)
      target_link_libraries(metrics_calc_lite ${_LIB})
      message("IPP ${LIB} library found at ${_LIB}")
    else ()
      message(FATAL_ERROR "Can't find ${LIB} library!")
    endif()
    unset(_LIB CACHE )
  endforeach()
endif()

if (NOT USE_MSDK)
  add_definitions(-DNO_MSDK)
else()
  if( UNIX )
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(PKG_LIBVA     libva>=0.33)
    pkg_check_modules(PKG_LIBDRM    libdrm)
    pkg_check_modules(PKG_LIBVA_DRM libva-drm>=0.33)
    pkg_check_modules(PKG_MFX       mfx)
    if(PKG_MFX_FOUND)
      target_include_directories(metrics_calc_lite BEFORE PRIVATE ${PKG_MFX_INCLUDE_DIRS})
      foreach(LIB ${PKG_LIBVA_LIBRARIES} ${PKG_LIBDRM_LIBRARIES} ${PKG_LIBVA_DRM_LIBRARIES} ${PKG_MFX_LIBRARIES})
        find_library(FOUND_LIB_${LIB} ${LIB} PATHS ${PKG_LIBVA_LIBDIR} ${PKG_LIBDRM_LIBDIR} ${PKG_LIBVA_DRM_LIBDIR} ${PKG_MFX_LIBDIR})
        if(NOT ${FOUND_LIB_${LIB}} MATCHES NOTFOUND)
          list(APPEND MCL_LIBS ${FOUND_LIB_${LIB}})
        else()
          list(APPEND MCL_LIBS ${LIB})
        endif()
      endforeach()
      list(REMOVE_DUPLICATES MCL_LIBS)
      target_link_libraries(metrics_calc_lite ${MCL_LIBS})
    else()
      message(STATUS "Cannot locate MSDK library package_config files, check if PKG_CONFIG_PATH contains /opt/intel/mediasdk/lib64/pkgconfig/")
      add_definitions(-DNO_MSDK)
    endif()
  else()
    if (NOT DEFINED ENV{INTELMEDIASDKROOT})
      message(STATUS "Cannot locate MSDK library, INTELMEDIASDKROOT is not defined")
      add_definitions(-DNO_MSDK)
    else()
      set(MSDK_ROOT $ENV{INTELMEDIASDKROOT})
      find_path(MSDK_INCLUDE NAMES mfxdefs.h PATHS ${MSDK_ROOT}/include  ${MSDK_ROOT}/include/mfx)
      if (NOT MSDK_INCLUDE MATCHES NOTFOUND)
        include_directories(${MSDK_ROOT}/include ${MSDK_ROOT}/include/mfx)
      else()
       message(FATAL_ERROR "Can't find MediaSDK headers!")
      endif()
      set(MSDK_LIB libmfx_vs2015 mfx)
      if (CMAKE_BUILD_TYPE EQUAL "DEBUG")
        set(MSDK_LIB libmfx_vs2015_d mfx)
      endif()
      find_library(_LIB NAMES ${MSDK_LIB} PATHS ${MSDK_ROOT}/lib/x64 ${MSDK_ROOT}/lib64)
      if (NOT _LIB MATCHES NOTFOUND)
        target_link_libraries(metrics_calc_lite ${_LIB})
        message("Found MSDK ${_LIB} library in ${MSDK_ROOT}/lib/x64!")
      else ()
        message(FATAL_ERROR "Can't find ${MSDK_LIB} library in ${MSDK_ROOT}/lib/x64!")
      endif()
    endif()
  endif()

endif()

foreach(flag_var
    CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
    CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
  if(${flag_var} MATCHES "/MD")
    string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
  endif(${flag_var} MATCHES "/MD")
  if(${flag_var} MATCHES "/MTd")
    string(REGEX REPLACE "/MTd" "/MT" ${flag_var} "${${flag_var}}")
  endif(${flag_var} MATCHES "/MTd")
endforeach(flag_var)

if (UNIX)
  install(TARGETS metrics_calc_lite RUNTIME DESTINATION bin)
endif()
