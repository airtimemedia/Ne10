# Copyright Airtime Media 2016

# Allow reading location of libraries
cmake_policy(SET CMP0026 OLD)

set(at_ne10_include_dir ${CMAKE_CURRENT_LIST_DIR}/../inc CACHE INTERNAL "")

if (IOS)
  set(IOS_PLATFORM YES)
  set(NE10_IOS_TARGET_ARCH ${at_target_arch})
  set(MIN_IOS_VER ${at_ios_deployment_target})
  set(NE10_BUILD_UNIT_TEST OFF CACHE INTERNAL "")
elseif (TVOS)
  set(IOS_PLATFORM YES)
  set(NE10_IOS_TARGET_ARCH ${at_target_arch})
  set(MIN_IOS_VER ${at_tvos_deployment_target})
  set(NE10_BUILD_UNIT_TEST OFF CACHE INTERNAL "")
elseif (ANDROID)
  if (${at_target_arch} STREQUAL "armv7")
    #
    # HACK HACK HACK HACK HACK HACK HACK
    #
    # In order to build Ne10 on Android (armv7 ONLY) with Clang,
    # we trick the Ne10 CMakeLists to think that we are actually building for ios so that:
    #
    # - -mthumb-interwork will not be included in build flags as that is unsupported by Clang.
    # - NE10_ASM_OPTIMIZATION is disabled as it doesn't build on Clang.
    # - convert_gas will be triggered for Clang build.
    #
    # The side effect is that there will be other compile flags added:
    # -miphoneos-version-min   : Does not seem to have any effect
    # -arch armv7 -arch armv7s : Does not seem to have any effect
    # -mfpu=vfp3               : Overwritten by build-wide -mfpu=neon
    set(ANDROID_PLATFORM NO)
    set(IOS_PLATFORM YES)
    set(NE10_IOS_TARGET_ARCH ${at_target_arch})
  else()
    # For arm64 and everything else, proceed as usual.
    set(ANDROID_PLATFORM YES)
    set(ANDROID_API_LEVEL ${CMAKE_SYSTEM_VERSION})
    set(NE10_TARGET_ARCH ${at_target_arch})
  endif()
endif()

set(NE10_BUILD_EXAMPLES OFF CACHE INTERNAL "")

add_subdirectory(.. ${CMAKE_CURRENT_BINARY_DIR}/Ne10)

# CAR-896: NE10 explicitly links against stdc++ and we don't want that
get_target_property(ne10_link_libs NE10 INTERFACE_LINK_LIBRARIES)
list(FIND ne10_link_libs stdc++ stdcplusplus_index)
if (stdcplusplus_index GREATER -1)
  list(REMOVE_AT ne10_link_libs ${stdcplusplus_index})
  set_target_properties(NE10 PROPERTIES
    LINK_LIBRARIES "${ne10_link_libs}"
  )
endif()

# This is a workaround for the unused var warning that seems to happen in some computers (CAR-657)
set_target_properties(NE10 PROPERTIES COMPILE_OPTIONS "-Wno-unused-variable")

# FIXME(tom): This is gross, but the Ne10 cmake files specifically reference
# this directory and don't ensure that it is created so we have to do it here
file(MAKE_DIRECTORY "${NE10_BINARY_DIR}/modules/CMakeFiles/NE10.dir")

get_property(at_ne10_lib TARGET NE10 PROPERTY LOCATION)
get_filename_component(at_ne10_lib_dir ${at_ne10_lib} DIRECTORY)
set(at_ne10_lib_dir ${at_ne10_lib_dir} CACHE INTERNAL "")
