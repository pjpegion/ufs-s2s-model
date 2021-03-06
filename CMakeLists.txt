cmake_minimum_required(VERSION 3.15)

foreach(env_var IN ITEMS
         CMAKE_C_COMPILER CMAKE_CXX_COMPILER CMAKE_Fortran_COMPILER
         CMAKE_Platform)
  if ( NOT DEFINED ENV{${env_var}})
    message(FATAL_ERROR "${env_var} is not defined")
  endif()
endforeach()

set(CMAKE_C_COMPILER $ENV{CMAKE_C_COMPILER})
set(CMAKE_CXX_COMPILER $ENV{CMAKE_CXX_COMPILER})
set(CMAKE_Fortran_COMPILER $ENV{CMAKE_Fortran_COMPILER})
set(CMAKE_Platform $ENV{CMAKE_Platform})

project(ufs-s2s-model
        VERSION 1.0
        LANGUAGES C CXX Fortran)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

include(GNUInstallDirs)

message(STATUS      "UFS-S2S-MODEL cmake build capability is a work in progress")
message(STATUS      "Please use the existing GNUMake capabiltiy")
message(FATAL_ERROR "Once ready, this message will be removed, and configuration will proceed.")

get_filename_component (C_COMPILER_NAME ${CMAKE_C_COMPILER} NAME)
get_filename_component (CXX_COMPILER_NAME ${CMAKE_CXX_COMPILER} NAME)
get_filename_component (Fortran_COMPILER_NAME ${CMAKE_Fortran_COMPILER} NAME)
message("C       compiler: ${CMAKE_C_COMPILER_ID} ${CMAKE_C_COMPILER_VERSION} (${C_COMPILER_NAME})")
message("CXX     compiler: ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION} (${CXX_COMPILER_NAME})")
message("Fortran compiler: ${CMAKE_Fortran_COMPILER_ID} ${CMAKE_Fortran_COMPILER_VERSION} (${Fortran_COMPILER_NAME})")
message("")

if (CMAKE_Fortran_COMPILER_ID MATCHES "GNU" AND CMAKE_Fortran_COMPILER_VERSION VERSION_LESS 9.0.0)
  message(FATAL_ERROR "GNU Compiler >= 9 is required")
endif()

include(cmake/${CMAKE_Fortran_COMPILER_ID}.cmake)
message("")

# Find dependency packages
find_package(NetCDF REQUIRED C Fortran)
find_package(MPI REQUIRED)
find_package(ESMF MODULE REQUIRED)
find_package(FMS REQUIRED)

find_package(nemsio REQUIRED)
find_package(sp REQUIRED)
find_package(w3emc REQUIRED)
find_package(w3nco REQUIRED)

if(INLINE_POST)
  find_package(nceppost REQUIRED)
endif()

if(CCPP)
    find_package(Python 3 QUIET COMPONENTS Interpreter)
    if (NOT Python_Interpreter_FOUND)
      find_package(Python 2.7 QUIET REQUIRED COMPONENTS Interpreter)
    endif()
    message("Found Python: ${Python_EXECUTABLE}")
endif()

# This after FindESMF.cmake,
# because ESMF_MOD depends on variables defined in the find.
message("")
message("Setting configuration for $ENV{CMAKE_Platform}")
message("")
include(${PROJECT_SOURCE_DIR}/cmake/configure_${CMAKE_Platform}.cmake)

add_definitions(-Duse_WRTCOMP)
#add_definitions(-Duse_LARGEFILE)
add_definitions(-DUSE_GFSL63)
add_definitions(-DGFS_PHYS)
add_definitions(-DMOIST_CAPPA)
add_definitions(-DUSE_COND)
add_definitions(-DNEW_TAUCTMAX)

if(CCPP)

    if(DEFINED CCPP_SUITES)
        message("Calling CCPP code generator (ccpp_prebuild.py) for SUITES = ${CCPP_SUITES}")
        execute_process(COMMAND FV3/ccpp/framework/scripts/ccpp_prebuild.py
                                "--config=FV3/ccpp/config/ccpp_prebuild_config.py"
                                "--suites=${CCPP_SUITES}"
                                "--builddir=${PROJECT_BINARY_DIR}/FV3"
                        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
                        OUTPUT_FILE ${PROJECT_BINARY_DIR}/ccpp_prebuild.out
                        ERROR_FILE ${PROJECT_BINARY_DIR}/ccpp_prebuild.err
                        RESULT_VARIABLE RC
                       )
    else()
        message("Calling CCPP code generator (ccpp_prebuild.py) ...")
        execute_process(COMMAND FV3/ccpp/framework/scripts/ccpp_prebuild.py
                                "--config=FV3/ccpp/config/ccpp_prebuild_config.py"
                                "--builddir=${PROJECT_BINARY_DIR}/FV3"
                        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
                        OUTPUT_FILE ${PROJECT_BINARY_DIR}/ccpp_prebuild.out
                        ERROR_FILE ${PROJECT_BINARY_DIR}/ccpp_prebuild.err
                        RESULT_VARIABLE RC
                       )
    endif()

    # Check return code from ccpp_prebuild.py
    if(RC EQUAL 0)
        message("")
    else()
        message(FATAL_ERROR "An error occured while running ccpp_prebuild.py, check ${PROJECT_BINARY_DIR}/ccpp_prebuild.{out,err}")
    endif()

    # this should not be necessary; including CCPP_*.cmake here and passing
    # SCHEMES, CAPS and TYPEDEFS via environment variables to CCPP build.
    # CCPP should be able to directly include those three .cmake files.

    include(${PROJECT_BINARY_DIR}/FV3/ccpp/physics/CCPP_SCHEMES.cmake)
    include(${PROJECT_BINARY_DIR}/FV3/ccpp/physics/CCPP_CAPS.cmake)
    include(${PROJECT_BINARY_DIR}/FV3/ccpp/physics/CCPP_TYPEDEFS.cmake)

    set(ENV{CCPP_SCHEMES} "${SCHEMES}")
    set(ENV{CCPP_CAPS} "${CAPS}")
    set(ENV{CCPP_TYPEDEFS} "${TYPEDEFS}")

endif(CCPP)

###############################################################################
### FV3
###############################################################################
add_subdirectory(FV3)

###############################################################################
### MOM6
###############################################################################
add_subdirectory(MOM6-interface)

###############################################################################
### CICE
###############################################################################
add_subdirectory(CICE-interface)

###############################################################################
### WW3
###############################################################################
if(WW3)
  set(WW3_COMP ${CMAKE_Platform})
  if(${CMAKE_Platform} STREQUAL "hera.intel")
    set(WW3_COMP "hera")
  endif()
  if(${CMAKE_Platform} STREQUAL "orion.intel")
    set(WW3_COMP "orion")
  endif()
  message("Build WW3:")
  message("   run: ${CMAKE_BUILD_TOOL} WW3_PARCOMPN=4 WW3_COMP=${WW3_COMP} ww3_nemslibonly")
  message("   in:  ${PROJECT_SOURCE_DIR}/WW3/model/esmf")
  message("")

  add_custom_target(ww3_nems
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/WW3/model/esmf
    COMMAND ${CMAKE_BUILD_TOOL} WW3_PARCOMPN=4 WW3_COMP=${WW3_COMP} ww3_nemslibonly > make.log 2>&1
  )

    set(WW3_LIBS ${PROJECT_SOURCE_DIR}/WW3/model/obj/libww3_multi_esmf.a)
endif(WW3)

###############################################################################
### NEMS
###############################################################################
add_library(
    ufs-s2s-model-lib

    NEMS/src/module_NEMS_UTILS.F90
    NEMS/src/module_MEDIATOR_methods.F90
    NEMS/src/module_MEDIATOR.F90
    NEMS/src/module_MEDIATOR_SpaceWeather.F90
    NEMS/src/module_EARTH_INTERNAL_STATE.F90
    NEMS/src/module_EARTH_GRID_COMP.F90
    NEMS/src/module_NEMS_INTERNAL_STATE.F90
    NEMS/src/module_NEMS_GRID_COMP.F90
    NEMS/src/module_NEMS_Rusage.F90
    NEMS/src/nems_c_rusage.c
    NEMS/src/ENS_Cpl/ENS_CplComp_ESMFMod_STUB.F90
)
add_dependencies(ufs-s2s-model-lib fv3cap)
target_link_libraries(ufs-s2s-model-lib PRIVATE esmf)

target_compile_definitions(ufs-s2s-model-lib PRIVATE -DESMF_VERSION_MAJOR=${ESMF_VERSION_MAJOR})
target_compile_definitions(ufs-s2s-model-lib PRIVATE -DFRONT_FMS)
target_compile_definitions(ufs-s2s-model-lib PRIVATE -DFRONT_FV3=fv3gfs_cap_mod)

target_include_directories(ufs-s2s-model-lib PRIVATE ${PROJECT_BINARY_DIR}/NEMS/src)
target_include_directories(ufs-s2s-model-lib PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/FV3/mod)

if(WW3)
    target_compile_definitions(ufs-s2s-model-lib PRIVATE -DFRONT_WW3=WMESMFMD)
    target_include_directories(ufs-s2s-model-lib PRIVATE ${PROJECT_SOURCE_DIR}/WW3/model/mod)
    set_target_properties(ufs-s2s-model-lib PROPERTIES Fortran_MODULE_DIRECTORY ${PROJECT_SOURCE_DIR}/WW3/model/mod)
    add_dependencies(ufs-s2s-model-lib ww3_nems)
endif()

set_target_properties(ufs-s2s-model-lib PROPERTIES Fortran_MODULE_DIRECTORY ${PROJECT_BINARY_DIR}/mod)
target_include_directories(ufs-s2s-model-lib PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/mod>
                                       $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/FV3/mod>
                                       $<INSTALL_INTERFACE:${CMAKE_INSTALL_PREFIX}/mod>)

configure_file(${PROJECT_SOURCE_DIR}/NEMS/src/ESMFVersionDefine_ESMF_NUOPC.h
               ${PROJECT_BINARY_DIR}/NEMS/src/ESMFVersionDefine.h
               COPYONLY)
# until we remove "../ESMFVersionDefine.h" from ENS_Cpl/ENS_CplComp_ESMFMod_STUB.F90
configure_file(${PROJECT_SOURCE_DIR}/NEMS/src/ESMFVersionDefine_ESMF_NUOPC.h
               ${PROJECT_BINARY_DIR}/NEMS/ESMFVersionDefine.h
               COPYONLY)
configure_file(${PROJECT_SOURCE_DIR}/NEMS/src/ESMFConvenienceMacros.h
               ${PROJECT_BINARY_DIR}/NEMS/src/ESMFConvenienceMacros.h
               COPYONLY)

###############################################################################
### ufs-s2s-model
###############################################################################
add_executable(
    NEMS.exe

    NEMS/src/MAIN_NEMS.F90
)
add_dependencies(NEMS.exe ufs-s2s-model-lib)

target_compile_definitions(NEMS.exe PRIVATE -DESMF_VERSION_MAJOR=${ESMF_VERSION_MAJOR})
target_include_directories(NEMS.exe PRIVATE ${PROJECT_BINARY_DIR}/NEMS/src)

set_target_properties(NEMS.exe PROPERTIES Fortran_MODULE_DIRECTORY ${PROJECT_BINARY_DIR}/mod)

if(WW3)
    target_compile_definitions(NEMS.exe PRIVATE -DFRONT_WW3=WMESMFMD)
    set_target_properties(NEMS.exe PROPERTIES Fortran_MODULE_DIRECTORY ${PROJECT_SOURCE_DIR}/WW3/model/mod)
    add_dependencies(NEMS.exe ww3_nems)
endif()

if(CCPP)
    set(CCPP_LIBRARIES ccppdriver ccppphys ccpp)
else()
    set(IPD_LIBRARIES ipd)
endif()

target_link_libraries(NEMS.exe
                      ufs-s2s-model-lib
                      fv3cap
                      fv3core
                      io
                      ipd
                      gfsphysics
                      ${CCPP_LIBRARIES}
                      cpl
                      stochastic_physics
                      FMS::fms_r4
                      ${WW3_LIBS}
                      nemsio::nemsio w3emc::w3emc_d w3nco::w3nco_d sp::sp_d
                      esmf NetCDF::NetCDF_Fortran ${MKL_LIB})

target_link_libraries(ufs-s2s-model-lib INTERFACE
                      fv3cap
                      fv3core
                      io
                      ipd
                      gfsphysics
                      ${CCPP_LIBRARIES}
                      cpl
                      stochastic_physics
                      FMS::fms_r4
                      ${WW3_LIBS}
                      nemsio::nemsio w3emc::w3emc_d w3nco::w3nco_d sp::sp_d
                      esmf NetCDF::NetCDF_Fortran ${MKL_LIB})

###############################################################################
### Install
###############################################################################

install(
  TARGETS ufs-s2s-model-lib fv3cap fv3core io ipd gfsphysics ${CCPP_LIBRARIES} fv3cpl stochastic_physics
  EXPORT ufs-s2s-model-lib-config
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/mod DESTINATION ${CMAKE_INSTALL_PREFIX})

install(EXPORT ufs-s2s-model-lib-config
  DESTINATION lib/cmake
)

###############################################################################
### done
###############################################################################
