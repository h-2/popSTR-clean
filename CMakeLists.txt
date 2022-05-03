cmake_minimum_required (VERSION 3.4)
project (popSTR CXX)
include(ExternalProject)

#####################################################
# check submodules
#####################################################

if (NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/submodules/popSTR-data/defaultModel OR
    NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/submodules/seqan/README.md OR
    NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/submodules/liblinear/README)

    message (FATAL_ERROR "One or more submodules were not checked out correctly.")
    message (FATAL_ERROR "When using git, run 'git submodule update --init' .")

endif()

#####################################################
# data setup
#####################################################

set(DATA_DIR ${CMAKE_CURRENT_SOURCE_DIR}/submodules/popSTR-data/)
add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/data_created
    COMMAND mkdir -p ${CMAKE_BINARY_DIR}/markerInfo
    COMMAND for i in `/usr/bin/seq 1 22` \; do gunzip < ${DATA_DIR}/markerInfo/chr$\${i}markerInfo.gz > ${CMAKE_BINARY_DIR}/markerInfo/chr$\${i}markerInfo \; done \;
    COMMAND cp -a ${DATA_DIR}/longRepeats ${CMAKE_BINARY_DIR}/markerInfo/
    COMMAND cp -a ${DATA_DIR}/defaultModel ${CMAKE_BINARY_DIR}/markerInfo/
    COMMAND tar -C ${CMAKE_BINARY_DIR} -xvf ${DATA_DIR}/kernel.tar.gz > /dev/null
    COMMAND tar -C ${CMAKE_BINARY_DIR} -xvf ${DATA_DIR}/panelMarkerInfo.tar.gz > /dev/null
    COMMAND touch ${CMAKE_BINARY_DIR}/data_created
    COMMENT "Creating data files (this may take a moment).")

add_custom_target(create_data DEPENDS ${CMAKE_BINARY_DIR}/data_created)

#####################################################
# find requirements
#####################################################

find_package(ZLIB REQUIRED)
find_package(Boost REQUIRED COMPONENTS iostreams)
find_package(Threads)

# SeqAn
list(APPEND MY_INCLUDES "${CMAKE_CURRENT_SOURCE_DIR}/submodules/seqan/include")
list(APPEND MY_CXXFLAGS -DSEQAN_ENABLE_TESTING=0 -DSEQAN_ENABLE_DEBUG=0 -DSEQAN_HAS_ZLIB=1)

# Liblinear
ExternalProject_Add(
    liblinear
    BUILD_IN_SOURCE 1
    SOURCE_DIR ${CMAKE_CURRENT_BINARY_DIR}/liblinear
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/liblinear
    CONFIGURE_COMMAND cp -a ${CMAKE_CURRENT_SOURCE_DIR}/submodules/liblinear ${CMAKE_CURRENT_BINARY_DIR}/
    BUILD_COMMAND $(MAKE) -C ${CMAKE_CURRENT_BINARY_DIR}/liblinear lib
    INSTALL_COMMAND ""
)
list(APPEND MY_INCLUDES "${CMAKE_CURRENT_SOURCE_DIR}/submodules/")

#####################################################
# misc flags
#####################################################

#TODO -Wall -Wextra
list(APPEND MY_CXXFLAGS "-Wmaybe-uninitialized")

#####################################################
# define build targets
#####################################################

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

add_executable(popSTR src/popSTR.cpp src/msGenotyper.cpp src/computePnSlippage.cpp src/computeReadAttributes.cpp src/computePnSlippageDefault.cpp src/msGenotyperDefault.cpp)

target_include_directories(popSTR PRIVATE ${MY_INCLUDES})
target_compile_options(popSTR PRIVATE ${MY_CXXFLAGS})
target_link_libraries (popSTR PRIVATE Threads::Threads ZLIB::ZLIB Boost::iostreams)
target_link_libraries (popSTR PRIVATE ${CMAKE_BINARY_DIR}/liblinear/liblinear.so.3)

#TODO raise standard
set_property(TARGET popSTR PROPERTY CXX_STANDARD 14)

#####################################################
# copy scripts
#####################################################

configure_file(scripts/run.sh bin/run.sh COPYONLY)
configure_file(scripts/runPerChrom.sh bin/runPerChrom.sh COPYONLY)
configure_file(scripts/runSmall.sh bin/runSmall.sh COPYONLY)
