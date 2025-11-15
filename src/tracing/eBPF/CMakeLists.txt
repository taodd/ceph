# eBPF tracing tools (cephtrace)
# Requires: clang, libbpf, libelf, libdw

if(NOT WITH_CEPHTRACE)
  return()
endif()

set(ARCH ${CMAKE_SYSTEM_PROCESSOR})
if(ARCH STREQUAL "x86_64")
  set(ARCH "x86")
elseif(ARCH STREQUAL "aarch64")
  set(ARCH "arm64")
elseif(ARCH STREQUAL "ppc64le")
  set(ARCH "powerpc")
elseif(ARCH MATCHES "mips.*")
  set(ARCH "mips")
endif()

# Paths
set(EBPF_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(EBPF_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(LIBBPF_SRC ${EBPF_SOURCE_DIR}/libbpf/src)
set(LIBBPF_TOP ${EBPF_SOURCE_DIR}/libbpf)
set(BPFTOOL_SRC ${EBPF_SOURCE_DIR}/bpftool/src)
set(BPFTOOL_OUTPUT ${EBPF_BINARY_DIR}/bpftool)
set(BPFTOOL ${BPFTOOL_OUTPUT}/bootstrap/bpftool)
set(LIBBPF_OBJ ${EBPF_BINARY_DIR}/libbpf/libbpf.a)

# Include paths
set(EBPF_INCLUDES
  -I${EBPF_SOURCE_DIR}
  -I${EBPF_BINARY_DIR}
  -I${LIBBPF_TOP}/include/uapi
  -I${LIBBPF_SRC})

# Get clang BPF system includes
execute_process(
  COMMAND ${CLANG_EXECUTABLE} -v -E -
  INPUT_FILE /dev/null
  ERROR_VARIABLE CLANG_STDERR
  OUTPUT_QUIET
)
string(REGEX MATCHALL "[ \t]([^\n]+)" CLANG_INCLUDE_DIRS "${CLANG_STDERR}")
set(CLANG_BPF_SYS_INCLUDES "")
set(IN_SEARCH_LIST FALSE)
foreach(LINE ${CLANG_INCLUDE_DIRS})
  string(STRIP "${LINE}" LINE)
  if(LINE MATCHES "^#include <...> search starts here:")
    set(IN_SEARCH_LIST TRUE)
  elseif(LINE MATCHES "^End of search list")
    set(IN_SEARCH_LIST FALSE)
  elseif(IN_SEARCH_LIST AND LINE MATCHES "^/")
    list(APPEND CLANG_BPF_SYS_INCLUDES "-idirafter${LINE}")
  endif()
endforeach()

# Build libbpf
add_custom_command(
  OUTPUT ${LIBBPF_OBJ}
  COMMAND ${CMAKE_COMMAND} -E make_directory ${EBPF_BINARY_DIR}/libbpf/build
  COMMAND ${CMAKE_COMMAND} -E make_directory ${EBPF_BINARY_DIR}/libbpf/install
  COMMAND make -C ${LIBBPF_SRC}
    BUILD_STATIC_ONLY=1
    OBJDIR=${EBPF_BINARY_DIR}/libbpf/build
    DESTDIR=${EBPF_BINARY_DIR}/libbpf/install
    INCLUDEDIR= LIBDIR= UAPIDIR=
    install
  COMMAND ${CMAKE_COMMAND} -E copy
    ${EBPF_BINARY_DIR}/libbpf/install/libbpf.a
    ${LIBBPF_OBJ}
  DEPENDS ${LIBBPF_SRC}/Makefile
  COMMENT "Building libbpf"
)

add_custom_target(libbpf_target DEPENDS ${LIBBPF_OBJ})

# Build bpftool
add_custom_command(
  OUTPUT ${BPFTOOL}
  COMMAND ${CMAKE_COMMAND} -E make_directory ${BPFTOOL_OUTPUT}
  COMMAND make -C ${BPFTOOL_SRC}
    OUTPUT=${BPFTOOL_OUTPUT}/
    bootstrap
  DEPENDS ${BPFTOOL_SRC}/Makefile
  COMMENT "Building bpftool"
)

add_custom_target(bpftool_target DEPENDS ${BPFTOOL})

# Generate bpf_ceph_exported.h from source headers
add_custom_command(
  OUTPUT ${EBPF_BINARY_DIR}/bpf_ceph_exported.h
  COMMAND ${EBPF_SOURCE_DIR}/gen_bpf_types.sh ${EBPF_BINARY_DIR}/bpf_ceph_exported.h
  DEPENDS
    ${CMAKE_SOURCE_DIR}/src/include/ceph_fs.h
    ${CMAKE_SOURCE_DIR}/src/msg/Message.h
    ${CMAKE_SOURCE_DIR}/src/include/rados.h
    ${CMAKE_SOURCE_DIR}/src/os/bluestore/BlueStore.h
    ${EBPF_SOURCE_DIR}/gen_bpf_types.sh
    ${EBPF_SOURCE_DIR}/bpf_ceph_types.h
  COMMENT "Generating bpf_ceph_exported.h from Ceph headers"
  VERBATIM
)

add_custom_target(bpf_ceph_export_target
  DEPENDS ${EBPF_BINARY_DIR}/bpf_ceph_exported.h
)

# Build BPF object file
add_custom_command(
  OUTPUT ${EBPF_BINARY_DIR}/radostrace.bpf.o
  COMMAND ${CLANG_EXECUTABLE} -g -O2 -target bpf
    -D__TARGET_ARCH_${ARCH}
    ${EBPF_INCLUDES}
    ${CLANG_BPF_SYS_INCLUDES}
    -c ${EBPF_SOURCE_DIR}/radostrace.bpf.c
    -o ${EBPF_BINARY_DIR}/radostrace.tmp.bpf.o
  COMMAND ${BPFTOOL} gen object
    ${EBPF_BINARY_DIR}/radostrace.bpf.o
    ${EBPF_BINARY_DIR}/radostrace.tmp.bpf.o
  DEPENDS
    ${EBPF_SOURCE_DIR}/radostrace.bpf.c
    ${EBPF_BINARY_DIR}/bpf_ceph_exported.h
    ${EBPF_SOURCE_DIR}/bpf_ceph_types.h
    ${EBPF_SOURCE_DIR}/bpf_utils.h
    bpftool_target
    libbpf_target
    bpf_ceph_export_target
  COMMENT "Compiling BPF program radostrace.bpf.c"
  VERBATIM
)

# Generate BPF skeleton
add_custom_command(
  OUTPUT ${EBPF_BINARY_DIR}/radostrace.skel.h
  COMMAND ${BPFTOOL} gen skeleton
    ${EBPF_BINARY_DIR}/radostrace.bpf.o
    > ${EBPF_BINARY_DIR}/radostrace.skel.h
  DEPENDS ${EBPF_BINARY_DIR}/radostrace.bpf.o
  COMMENT "Generating BPF skeleton radostrace.skel.h"
  VERBATIM
)

# Build dwarf_parser object
add_library(dwarf_parser OBJECT
  ${EBPF_SOURCE_DIR}/dwarf_parser.cc
  ${EBPF_SOURCE_DIR}/dwarf_parser.h
)

target_include_directories(dwarf_parser PRIVATE
  ${EBPF_SOURCE_DIR}
  ${EBPF_BINARY_DIR}
  ${EBPF_BINARY_DIR}/libbpf/install
  ${LIBBPF_TOP}/include/uapi
  ${LIBBPF_SRC}
)


add_dependencies(dwarf_parser libbpf_target)

# Build radostrace executable
add_executable(radostrace
  ${EBPF_SOURCE_DIR}/radostrace.cc
  ${EBPF_SOURCE_DIR}/dwarf_parser.h
  ${EBPF_BINARY_DIR}/bpf_ceph_exported.h
  ${EBPF_SOURCE_DIR}/bpf_ceph_types.h
  ${EBPF_SOURCE_DIR}/bpf_utils.h
  ${EBPF_BINARY_DIR}/radostrace.skel.h
  $<TARGET_OBJECTS:dwarf_parser>
)

target_include_directories(radostrace PRIVATE
  ${EBPF_SOURCE_DIR}
  ${EBPF_BINARY_DIR}
  ${EBPF_BINARY_DIR}/libbpf/install
  ${LIBBPF_TOP}/include/uapi
  ${LIBBPF_SRC}
  ${CMAKE_SOURCE_DIR}/src
)

target_link_libraries(radostrace
  ${LIBBPF_OBJ}
  LibElf::LibElf
  LibDw::LibDw
  z
  ${CMAKE_DL_LIBS}
  common
)

add_dependencies(radostrace
  libbpf_target
  bpftool_target
  bpf_ceph_export_target
)

# Install radostrace
install(TARGETS radostrace DESTINATION ${CMAKE_INSTALL_BINDIR})

message(STATUS "eBPF tracing (radostrace) will be built")
