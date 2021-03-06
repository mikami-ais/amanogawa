cmake_minimum_required(VERSION 3.0)
project(amanogawa)

execute_process(COMMAND
    sh -c "which clang++ | tr -d '\\n'"
    OUTPUT_VARIABLE CMAKE_CXX_COMPILER)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
set(CMAKE_CXX_FLAGS_DEBUG "-O0 -g -Wall -Wno-return-type-c-linkage -DSPDLOG_DEBUG_ON -DDEBUG_ON")
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -Wno-return-type-c-linkage")

execute_process(COMMAND
    sh -c "python3 -V 2>&1 | grep -oP '\\d\\.\\d(?=\\.\\d)' | tr -d '\\n'"
    OUTPUT_VARIABLE PYTHON_VERSION)
set(PYBIND11_PYTHON_VERSION ${PYTHON_VERSION})

if (UNIX AND NOT APPLE)
    # linux
    set(LIBS_EXT so)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
else ()
    # macos
    set(LIBS_EXT dylib)
#    include_directories(SYSTEM src/pcm/MacMSRDriver)
endif ()
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DLIBS_EXT=${LIBS_EXT}")

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/src/arrow/cpp/cmake_modules)

#if (${CMAKE_BUILD_TYPE} MATCHES Debug)
#    ## pcm
#    set(PCM_SOURCES
#            src/pcm/cpucounters.cpp
#            src/pcm/msr.cpp
#            src/pcm/pci.cpp
#            src/pcm/client_bw.cpp)
#    set(LIB_PCM pcm)
#    add_library(${LIB_PCM} STATIC ${PCM_SOURCES})
#endif ()

# dependencies
## pybind11
add_subdirectory(src/pybind11)

## numpy
#find_package(NumPy REQUIRED)

# python
find_package(PythonLibsNew REQUIRED)

## arrow
set(ARROW_BUILD_SHARED ON)
set(ARROW_BUILD_STATIC ON)
set(ARROW_BUILD_COMPUTE ON)
set(ARROW_JEMALLOC_USE_SHARED ON)
set(ARROW_HDFS ON)
set(ARROW_BOOST_USE_SHARED ON)
set(ARROW_SSE3 ON)
set(ARROW_ALTIVEC ON)
set(ARROW_WITH_BROTLI ON)
set(ARROW_WITH_SNAPPY ON)
set(ARROW_WITH_ZLIB ON)
set(ARROW_WITH_ZSTD ON)
set(ARROW_PYTHON ON)
set(ARROW_VERBOSE_THIRDPARTY_BUILD ON)
add_subdirectory(src/arrow/cpp)
set(LIB_ARROW arrow_static)
set(LIB_ARROW_PYTHON arrow_python_static)


file(GLOB SOURCES src/amanogawa/**/*.h)
include_directories(src)
include_directories(SYSTEM
#        ${NUMPY_INCLUDE_DIRS}
        src/arrow/cpp/src
        src/cpptoml/include
        src/spdlog/include
        src/text-csv/include
        src/pybind11/include
        src/json/src
        src/pugixml/src)


# plugins
## source
### file
set(AMANOGAWA_PLUGIN_SOURCE_FILE source_file)
add_library(${AMANOGAWA_PLUGIN_SOURCE_FILE} SHARED src/amanogawa/plugin/source/file/file.cpp)
target_link_libraries(${AMANOGAWA_PLUGIN_SOURCE_FILE} ${LIB_ARROW})

## format
### csv
set(AMANOGAWA_PLUGIN_FORMAT_CSV format_csv)
add_library(${AMANOGAWA_PLUGIN_FORMAT_CSV} SHARED src/amanogawa/plugin/format/csv/csv.cpp)
target_link_libraries(${AMANOGAWA_PLUGIN_FORMAT_CSV} ${LIB_ARROW})

### json
set(AMANOGAWA_PLUGIN_FORMAT_JSON format_json)
add_library(${AMANOGAWA_PLUGIN_FORMAT_JSON} SHARED src/amanogawa/plugin/format/json/json.cpp)
target_link_libraries(${AMANOGAWA_PLUGIN_FORMAT_JSON} ${LIB_ARROW})

### xml
set(AMANOGAWA_PLUGIN_FORMAT_XML format_xml)
add_library(${AMANOGAWA_PLUGIN_FORMAT_XML} SHARED src/amanogawa/plugin/format/xml/xml.cpp)
add_subdirectory(src/pugixml)
set_property(TARGET pugixml PROPERTY POSITION_INDEPENDENT_CODE ON)
target_link_libraries(${AMANOGAWA_PLUGIN_FORMAT_XML} ${LIB_ARROW} pugixml)

## flow
### filter
set(AMANOGAWA_PLUGIN_FLOW_FILTER flow_filter)
add_library(${AMANOGAWA_PLUGIN_FLOW_FILTER} SHARED src/amanogawa/plugin/flow/filter/filter.cpp)
target_link_libraries(${AMANOGAWA_PLUGIN_FLOW_FILTER} ${LIB_ARROW})

### to_graph
set(AMANOGAWA_PLUGIN_FLOW_TO_GRAPH flow_to_graph)
find_library(LIB_MECAB NAMES mecab)
file(GLOB FLOW_TO_GRAPH_SOURCES src/amanogawa/plugin/flow/to_graph/*.h)
add_library(${AMANOGAWA_PLUGIN_FLOW_TO_GRAPH} SHARED
        ${FLOW_TO_GRAPH_SOURCES} src/amanogawa/plugin/flow/to_graph/to_graph.cpp)
target_link_libraries(${AMANOGAWA_PLUGIN_FLOW_TO_GRAPH} ${LIB_ARROW} ${LIB_MECAB})

## branch
set(AMANOGAWA_PLUGIN_BRANCH_COLUMN branch_column)
add_library(${AMANOGAWA_PLUGIN_BRANCH_COLUMN} SHARED src/amanogawa/plugin/branch/column/column.cpp)
target_link_libraries(${AMANOGAWA_PLUGIN_BRANCH_COLUMN} ${LIB_ARROW})

## confluence
### key
set(AMANOGAWA_PLUGIN_CONFLUENCE_KEY confluence_key)
add_library(${AMANOGAWA_PLUGIN_CONFLUENCE_KEY} SHARED src/amanogawa/plugin/confluence/key/key.cpp)
target_link_libraries(${AMANOGAWA_PLUGIN_CONFLUENCE_KEY} ${LIB_ARROW})

## sink
### file
set(AMANOGAWA_PLUGIN_SINK_FILE sink_file)
add_library(${AMANOGAWA_PLUGIN_SINK_FILE} SHARED src/amanogawa/plugin/sink/file/file.cpp)
target_link_libraries(${AMANOGAWA_PLUGIN_SINK_FILE} ${LIB_ARROW})

### numpy
#set(AMANOGAWA_PLUGIN_SINK_NUMPY sink_numpy)
#pybind11_add_module(${AMANOGAWA_PLUGIN_SINK_NUMPY} SHARED src/amanogawa/plugin/sink/numpy/numpy.cpp)
#target_link_libraries(${AMANOGAWA_PLUGIN_SINK_NUMPY} ${LIB_ARROW} ${LIB_ARROW_PYTHON} ${PYTHON_LIBRARIES})

set_target_properties(
        ${AMANOGAWA_PLUGIN_SOURCE_FILE}
        ${AMANOGAWA_PLUGIN_FORMAT_CSV}
        ${AMANOGAWA_PLUGIN_FORMAT_JSON}
        ${AMANOGAWA_PLUGIN_FORMAT_XML}
        ${AMANOGAWA_PLUGIN_FLOW_FILTER}
        ${AMANOGAWA_PLUGIN_FLOW_TO_GRAPH}
        ${AMANOGAWA_PLUGIN_BRANCH_COLUMN}
        ${AMANOGAWA_PLUGIN_CONFLUENCE_KEY}
        ${AMANOGAWA_PLUGIN_SINK_FILE}
#        ${AMANOGAWA_PLUGIN_SINK_NUMPY}
        PROPERTIES PREFIX lib SUFFIX .${LIBS_EXT})


# cli
set(AMANOGAWA_CLI amanogawa-cli)
add_executable(${AMANOGAWA_CLI} ${SOURCES} src/amanogawa/cli.cpp)
target_link_libraries(${AMANOGAWA_CLI} dl ${LIB_ARROW} ${LIB_ARROW_PYTHON})


# pybind
set(AMANOGAWA amanogawa)
pybind11_add_module(${AMANOGAWA} SHARED ${SOURCES} src/amanogawa/pybind.cpp)
target_link_libraries(${AMANOGAWA} dl ${LIB_ARROW} ${LIB_ARROW_PYTHON})


#if (${CMAKE_BUILD_TYPE} MATCHES Debug)
#    target_link_libraries(${AMANOGAWA_CLI} ${LIB_PCM})
#    target_link_libraries(${AMANOGAWA} ${LIB_PCM})
#endif ()
