# CMAKE project for openrct2
cmake_minimum_required(VERSION 3.1)
if (CMAKE_BINARY_DIR STREQUAL CMAKE_SOURCE_DIR)
    message(FATAL_ERROR "Building in-source is not supported! Create a build dir and remove ${CMAKE_SOURCE_DIR}/CMakeCache.txt")
endif()

project(openrct2)

include(FindPkgConfig)
include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)
include(GNUInstallDirs)

set(ROOT_DIR "${CMAKE_CURRENT_LIST_DIR}")
set(CMAKE_MACOSX_RPATH 1)

set(TITLE_SEQUENCE_URL  "https://github.com/OpenRCT2/title-sequences/releases/download/v0.1.0/title-sequence-v0.1.0.zip")
set(TITLE_SEQUENCE_SHA1 "b587d83de508d0b104d14c599b76f8565900fce0")

option(FORCE32 "Force 32-bit build. It will add `-m32` to compiler flags.")
option(WITH_TESTS "Build tests")
option(PORTABLE "Create a portable build (-rpath=$ORIGIN)" OFF)
option(DOWNLOAD_TITLE_SEQUENCES "Download title sequences during installation." ON)

if (FORCE32)
    set(TARGET_M "-m32")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${TARGET_M}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TARGET_M}")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${TARGET_M}")
endif ()

if (PORTABLE OR WIN32)
    set(CMAKE_INSTALL_LIBDIR "${CMAKE_INSTALL_BINDIR}")
    set(CMAKE_INSTALL_RPATH "$ORIGIN")
endif ()

# Define current git branch
execute_process(
    COMMAND git rev-parse --abbrev-ref HEAD
    WORKING_DIRECTORY ${ROOT_DIR}
    OUTPUT_VARIABLE OPENRCT2_BRANCH
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
)

# Define short commit hash
execute_process(
    COMMAND git rev-parse --short HEAD
    WORKING_DIRECTORY ${ROOT_DIR}
    OUTPUT_VARIABLE OPENRCT2_COMMIT_SHA1_SHORT
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
)

function (ADD_CHECK_C_COMPILER_FLAG _CFLAGS _CACHE_VAR _FLAG)
    CHECK_C_COMPILER_FLAG("${_FLAG}" "${_CACHE_VAR}")
    if (${_CACHE_VAR})
        set(${_CFLAGS} "${${_CFLAGS}} ${_FLAG}" PARENT_SCOPE)
    else ()
        message(STATUS "Unsupported CFLAG: ${_FLAG}")
    endif ()
endfunction ()

function (ADD_CHECK_CXX_COMPILER_FLAG _CXXFLAGS _CACHE_VAR _FLAG)
    CHECK_CXX_COMPILER_FLAG("${_FLAG}" "${_CACHE_VAR}")
    if (${_CACHE_VAR})
        set(${_CXXFLAGS} "${${_CXXFLAGS}} ${_FLAG}" PARENT_SCOPE)
    else ()
        message(STATUS "Unsupported CXXFLAG: ${_FLAG}")
    endif ()
endfunction ()

ADD_CHECK_C_COMPILER_FLAG(CMAKE_C_FLAGS C_WARN_MISSING_VARIABLE_DECLARATIONS -Wmissing-variable-declarations)
ADD_CHECK_C_COMPILER_FLAG(CMAKE_C_FLAGS C_WARN_DUPLICATED_COND -Wduplicated-cond)
ADD_CHECK_C_COMPILER_FLAG(CMAKE_C_FLAGS C_WARN_DUPLICATED_BRANCHES -Wduplicated-branches)
ADD_CHECK_C_COMPILER_FLAG(CMAKE_C_FLAGS C_WARN_LOGICAL_OP -Wlogical-op)
ADD_CHECK_C_COMPILER_FLAG(CMAKE_C_FLAGS C_WARN_RESTRICT -Wrestrict)
ADD_CHECK_C_COMPILER_FLAG(CMAKE_C_FLAGS C_WARN_UNREACHABLE_CODE_BREAK -Wunreachable-code-break)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_NULL_DEREFERENCE -Wnull-dereference)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_SUGGEST_FINAL_TYPES -Wsuggest-final-types)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_SUGGEST_FINAL_METHODS -Wsuggest-final-methods)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_SUGGEST_OVERRIDE -Wsuggest-override)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_DUPLICATED_COND -Wduplicated-cond)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_NON_VIRTUAL_DTOR -Wnon-virtual-dtor)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_MISSING_VARIABLE_DECLARATIONS -Wmissing-variable-declarations)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_DUPLICATED_BRANCHES -Wduplicated-branches)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_RESTRICT -Wrestrict)

if (MINGW)
    ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_SUGGEST_OVERRIDE -Wno-error=redundant-decls)
endif ()

# Items below are not supported by ICC
ADD_CHECK_C_COMPILER_FLAG(CMAKE_C_FLAGS C_WARN_REDUNDANT_DECLS -Wredundant-decls)
ADD_CHECK_C_COMPILER_FLAG(CMAKE_C_FLAGS C_WARN_IGNORED_QUALIFIERS -Wignored-qualifiers)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_REDUNDANT_DECLS -Wredundant-decls)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_IGNORED_QUALIFIERS -Wignored-qualifiers)

# -Wstrict-overflow is only active when -fstrict-overflow is enabled, but -fstrict-overflow
# is enabled on -O2, -O3, -Os. This should help catch bugs locally before they reach Travis
# As of 2a435bf -Wstrict-overflow=1 passes, but higher values do not.
set(COMMON_COMPILE_OPTIONS "${COMMON_COMPILE_OPTIONS} -fstrict-overflow")
ADD_CHECK_C_COMPILER_FLAG(CMAKE_C_FLAGS C_WARN_STRICT_OVERFLOW -Wstrict-overflow=1)
ADD_CHECK_CXX_COMPILER_FLAG(CMAKE_CXX_FLAGS CXX_WARN_STRICT_OVERFLOW -Wstrict-overflow=1)

if(EXISTS "${ROOT_DIR}/discord-rpc")
    add_subdirectory("${ROOT_DIR}/discord-rpc")
    add_definitions(-D__ENABLE_DISCORD__)
    include_directories("${ROOT_DIR}/discord-rpc/include")
    set(HAVE_DISCORD_RPC TRUE)
    message("Building with discord-rpc support")
else()
	message("No discord-rpc detected, to enable clone discord-rpc to root directory: ${ROOT_DIR}")
endif()

# Copied from https://github.com/opencv/opencv/blob/dcdd6af5a856826fe62c95322145731e702e54c5/cmake/OpenCVDetectCXXCompiler.cmake#L63-L70
if(MSVC64 OR MINGW64)
    set(X86_64 1)
elseif(MINGW OR (MSVC AND NOT CMAKE_CROSSCOMPILING))
    set(X86 1)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "amd64.*|x86_64.*|AMD64.*")
    set(X86_64 1)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "i686.*|i386.*|x86.*|amd64.*|AMD64.*")
    set(X86 1)
endif()

# Include sub-projects
include("${ROOT_DIR}/src/openrct2/CMakeLists.txt" NO_POLICY_SCOPE)
include("${ROOT_DIR}/src/openrct2-cli/CMakeLists.txt" NO_POLICY_SCOPE)
include("${ROOT_DIR}/src/openrct2-ui/CMakeLists.txt" NO_POLICY_SCOPE)

# g2
add_custom_command(
    OUTPUT g2.dat
    COMMAND ./openrct2 sprite build ${CMAKE_BINARY_DIR}/g2.dat ${ROOT_DIR}/resources/g2/sprites.json
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)
add_custom_target(g2 DEPENDS ${PROJECT} g2.dat)

# Include tests
if (WITH_TESTS)
    enable_testing()
    if (UNIX AND (NOT USE_MMAP) AND FORCE32)
        include("${ROOT_DIR}/test/testpaint/CMakeLists.txt" NO_POLICY_SCOPE)
    endif ()
    include("${ROOT_DIR}/test/tests/CMakeLists.txt" NO_POLICY_SCOPE)
endif ()

# Install
# Don't recurse, grab all *.txt and *.md files
file(GLOB DOC_FILES "${ROOT_DIR}/distribution/*.txt")
list(APPEND DOC_FILES "${ROOT_DIR}/contributors.md"
                      "${ROOT_DIR}/licence.txt")

# CMake does not allow specifying a dependency chain which includes built-in
# targets, like `install`, so we have to trick it and execute dependency ourselves.
install(CODE "execute_process(COMMAND \"${CMAKE_COMMAND}\" --build \"${CMAKE_CURRENT_BINARY_DIR}\" --target g2)")
IF (DOWNLOAD_TITLE_SEQUENCES)
    install(CODE "file(DOWNLOAD ${TITLE_SEQUENCE_URL} \$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/title/title-sequences.zip EXPECTED_HASH SHA1=${TITLE_SEQUENCE_SHA1} SHOW_PROGRESS)")
    install(CODE "execute_process(COMMAND \"${CMAKE_COMMAND}\" -E chdir \$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/title/ \"${CMAKE_COMMAND}\" -E tar xvf title-sequences.zip)")
    install(CODE "file(REMOVE \$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT}/title/title-sequences.zip)")
endif ()
install(TARGETS "libopenrct2" LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
                              ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}")
install(TARGETS "openrct2" RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
install(TARGETS "openrct2-cli" OPTIONAL RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/g2.dat" DESTINATION "${CMAKE_INSTALL_DATADIR}/openrct2")
install(DIRECTORY "data/" DESTINATION "${CMAKE_INSTALL_DATADIR}/openrct2")
install(FILES ${DOC_FILES} DESTINATION "${CMAKE_INSTALL_DOCDIR}")
install(FILES "distribution/linux/openrct2.appdata.xml" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/metainfo")
install(FILES "resources/logo/icon_x16.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/16x16/apps" RENAME "openrct2.png")
install(FILES "resources/logo/icon_x24.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/24x24/apps" RENAME "openrct2.png")
install(FILES "resources/logo/icon_x32.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/32x32/apps" RENAME "openrct2.png")
install(FILES "resources/logo/icon_x48.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/48x48/apps" RENAME "openrct2.png")
install(FILES "resources/logo/icon_x64.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/64x64/apps" RENAME "openrct2.png")
install(FILES "resources/logo/icon_x96.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/96x96/apps" RENAME "openrct2.png")
install(FILES "resources/logo/icon_x128.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/128x128/apps" RENAME "openrct2.png")
install(FILES "resources/logo/icon_x256.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/256x256/apps" RENAME "openrct2.png")
install(FILES "resources/logo/icon_flag.svg" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/scalable/apps" RENAME "openrct2.svg")
install(FILES "distribution/linux/openrct2.desktop" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/applications")
install(FILES "distribution/linux/openrct2-savegame.desktop" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/applications")
install(FILES "distribution/linux/openrct2-scenario.desktop" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/applications")
install(FILES "distribution/linux/openrct2-uri.desktop" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/applications")
install(FILES "distribution/linux/openrct2-mimeinfo.xml" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/mime/packages/" RENAME "openrct2.xml")
install(DIRECTORY "distribution/man/" DESTINATION "${CMAKE_INSTALL_MANDIR}/man6" FILES_MATCHING PATTERN "*.6")
