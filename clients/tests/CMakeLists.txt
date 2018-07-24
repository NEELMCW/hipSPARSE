# ########################################################################
# Copyright 2018 Advanced Micro Devices, Inc.
# ########################################################################

# Download some test matrices
set(TEST_MATRICES
  Bova/rma10
  JGD_BIBD/bibd_22_8
  Williams/mac_econ_fwd500
  Williams/mc2depi
  Hamm/scircuit
  Sandia/ASIC_320k
  GHS_psdef/bmwcra_1
  HB/nos1
  HB/nos2
  HB/nos3
  HB/nos4
  HB/nos5
  HB/nos6
  HB/nos7
)

foreach(m ${TEST_MATRICES})
  string(REPLACE "/" ";" sep_m ${m})
  list(GET sep_m 0 dir)
  list(GET sep_m 1 mat)

  # Download test matrices if not already downloaded
  if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/matrices/${mat}.bin")
    message("  Downloading and extracting test matrix ${m}.tar.gz")
    file(DOWNLOAD http://www.cise.ufl.edu/research/sparse/MM/${m}.tar.gz
         ${CMAKE_CURRENT_BINARY_DIR}/matrices/${mat}.tar.gz)
    execute_process(COMMAND tar xf ${mat}.tar.gz
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/matrices)
    execute_process(COMMAND mv ${mat}/${mat}.mtx .
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/matrices)
    execute_process(COMMAND ${CMAKE_SOURCE_DIR}/deps/convert ${mat}.mtx ${mat}.bin
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/matrices)
    execute_process(COMMAND rm ${mat}.tar.gz ${mat} ${mat}.mtx -rf
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/matrices)
  endif()
endforeach()


find_package(GTest REQUIRED)

set(HIPSPARSE_TEST_SOURCES
  hipsparse_gtest_main.cpp
  test_axpyi.cpp
  test_doti.cpp
  test_gthr.cpp
  test_gthrz.cpp
  test_roti.cpp
  test_sctr.cpp
  test_coomv.cpp
  test_csrmv.cpp
  test_ellmv.cpp
  test_hybmv.cpp
  test_csr2coo.cpp
  test_csr2csc.cpp
  test_csr2ell.cpp
  test_csr2hyb.cpp
  test_coo2csr.cpp
  test_identity.cpp
  test_csrsort.cpp
  test_coosort.cpp
)

set(HIPSPARSE_CLIENTS_COMMON
  ../common/arg_check.cpp
  ../common/unit.cpp
  ../common/utility.cpp
  ../common/hipsparse_template_specialization.cpp
)

add_executable(hipsparse-test ${HIPSPARSE_TEST_SOURCES} ${HIPSPARSE_CLIENTS_COMMON})

target_include_directories(hipsparse-test
  PRIVATE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>
)

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
target_link_libraries(hipsparse-test PRIVATE Threads::Threads)
target_link_libraries(hipsparse-test PRIVATE hipsparse ${GTEST_BOTH_LIBRARIES})

set_target_properties(hipsparse-test PROPERTIES CXX_STANDARD 11 CXX_STANDARD_REQUIRED ON)

target_compile_definitions(hipsparse-test PRIVATE GOOGLE_TEST)

target_include_directories(hipsparse-test
  SYSTEM
    PRIVATE
      $<BUILD_INTERFACE:${GTEST_INCLUDE_DIRS}>
      $<BUILD_INTERFACE:${HIP_INCLUDE_DIRS}>
)

if(NOT BUILD_CUDA)
  target_compile_definitions(hipsparse-test PRIVATE __HIP_PLATFORM_HCC__)
  get_target_property(HIP_HCC_LOCATION hip::hip_hcc IMPORTED_LOCATION_RELEASE)
  get_target_property(HCC_AM_LOCATION hcc::hc_am IMPORTED_LOCATION_RELEASE)
  target_link_libraries(hipsparse-test PRIVATE ${HIP_HCC_LOCATION} ${HCC_AM_LOCATION})
else()
  target_compile_definitions(hipsparse-test PRIVATE __HIP_PLATFORM_NVCC__)
  target_link_libraries(hipsparse-test PRIVATE ${CUDA_LIBRARIES})
endif()

add_test(hipsparse-test hipsparse-test)