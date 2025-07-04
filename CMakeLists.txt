# XPMP2 - Set up to be used in the provided docker environment to build lin and mac
#         Set up to be used in a Visual Studio environment to build win (File > Open > Folder, then VS recognized the CMAKE configuration)
#
# Targets XPMP2-Remote and XPMP2-Sample are EXCLUDE_FROM_ALL.
# If you want to build them you have to explicitely state them on the command line, like
#   ninja XPMP2-Sample XPMP2-Remote
#
# If you want to build with FMOD sound support,
# 1. Make sure to understand the FMOD license and attribution requirements:
#    https://www.fmod.com/licensing
#    https://www.fmod.com/attribution
# 2. Define INCLUDE_FMOD_SOUND cache entry, e.g. using `cmake -G Ninja -D INCLUDE_FMOD_SOUND=1 ..`

cmake_minimum_required(VERSION 3.16)

# Mac: Need to tell early on that we want a cross platform build
if(DEFINED ENV{platform})
    message ("-- Platform is $ENV{platform}")
    if($ENV{platform} STREQUAL "mac-x86")
        message ("   Building cross-platform for mac/x86_64")
        set(CMAKE_OSX_ARCHITECTURES "x86_64" CACHE STRING "Archs to build")
    elseif($ENV{platform} STREQUAL "mac-arm")
        message ("   Building cross-platform for mac/arm64")
        set(CMAKE_OSX_ARCHITECTURES "arm64" CACHE STRING "Archs to build")
    endif()
else()
    # No 'platform' defined could mean running from command line, assume we build universal image in one go via XCode
    set(CMAKE_OSX_ARCHITECTURES "x86_64;arm64" CACHE STRING "Archs to build")
endif()

project(XPMP2
        VERSION 3.1.1
        DESCRIPTION "Multiplayer library for X-Plane 11 and 12")

# Provide compile macros from the above project version definition
add_compile_definitions(
    XPMP2_VERSION="${PROJECT_VERSION}"
    XPMP2_VER_MAJOR=${PROJECT_VERSION_MAJOR}
    XPMP2_VER_MINOR=${PROJECT_VERSION_MINOR}
    XPMP2_VER_PATCH=${PROJECT_VERSION_PATCH}
)

message ("== Building: ${PROJECT_NAME} ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH} ==")
message ("Compiler Info:")
message ("CMAKE_CXX_COMPILER_ID         = ${CMAKE_CXX_COMPILER_ID}")
message ("CMAKE_CXX_COMPILER_VERSION    = ${CMAKE_CXX_COMPILER_VERSION}")
message ("CMAKE_CXX_COMPILER            = ${CMAKE_CXX_COMPILER}")
message ("WIN32 / MSVC / MINGW          = ${WIN32} / ${MSVC} / ${MINGW}")
message ("UNIX / APPLE                  = ${UNIX} / ${APPLE}")
if (APPLE)
    message ("OSX_SDK_PATH                  = $ENV{OSX_SDK_PATH}")
    message ("CMAKE_OSX_ARCHITECTURES       = ${CMAKE_OSX_ARCHITECTURES}")
endif()

################################################################################
# Target Systems
################################################################################

# Windows: Target Windows 7.0 and later
if (WIN32)
    add_compile_definitions(_WIN32_WINNT=0x0601)
    if (NOT DEFINED ENV{platform})
        set(ENV{platform} "win")
    endif()
elseif(APPLE)
    set(CMAKE_OSX_DEPLOYMENT_TARGET 11.0)
    add_compile_options(-mmacosx-version-min=11.0)
    add_link_options(-mmacosx-version-min=11.0)
endif()

################################################################################
# C++ Standard required
################################################################################

set(CMAKE_CXX_STANDARD 17)
set_property(GLOBAL PROPERTY CXX_STANDARD_REQUIRED 17)
set_property(GLOBAL PROPERTY CXX_STANDARD 17)
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

################################################################################
# Compile Options
################################################################################

# Enable all X-Plane SDK APIs up to the newest version.
add_compile_definitions(XPLM200=1 XPLM210=1 XPLM300=1 XPLM301=1 XPLM303=1)

# Define platform macros.
add_compile_definitions(APL=$<BOOL:${APPLE}> IBM=$<BOOL:${WIN32}> LIN=$<AND:$<BOOL:${UNIX}>,$<NOT:$<BOOL:${APPLE}>>>)

# Enable stricter warnings and then disable some we are not interested in.
# For XPMP2 compile, we don't need to be warned about our self-defined depreciations
if (MSVC)
    add_compile_options(/wd4996 /wd4068)
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
else()
    add_compile_options(-Wall -Wshadow -Wextra -Wno-deprecated-declarations)
    if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 9.0 AND NOT APPLE)
        add_compile_options(-Wno-stringop-truncation)
    endif()

    # Force-enable exception support. This is most likely redundant, although for C
    # code the default is the opposite. Since we are mixing C++ and C libraries,
    # safer to set it on?
    add_compile_options(-fexceptions)

    # Makes symbols non-exported by default.
    add_compile_options(-fvisibility=hidden)
endif()

# Mingw Threads (if not already defined by an outer target)
if (MINGW AND NOT TARGET mingw_stdthreads)
    option(MINGW_STDTHREADS_GENERATE_STDHEADERS "" ON)
    add_subdirectory(lib/mingw-std-threads)
endif()

################################################################################
# XPMP2 Library
################################################################################

# Source list
add_library(XPMP2 STATIC
    inc/XPCAircraft.h
    inc/XPMPAircraft.h
    inc/XPMPMultiplayer.h
    inc/XPMPRemote.h
    inc/XPMPPlaneRenderer.h
    src/2D.h
    src/2D.cpp
    src/AIMultiplayer.h
    src/AIMultiplayer.cpp
    src/Aircraft.h
    src/Aircraft.cpp
    src/Contrail.cpp
    src/CSLCopy.cpp
    src/CSLModels.h
    src/CSLModels.cpp
    src/Map.h
    src/Map.cpp
    src/Network.h
    src/Network.cpp
    src/RelatedDoc8643.h
    src/RelatedDoc8643.cpp
    src/Remote.h
    src/Remote.cpp
    src/Utilities.h
    src/Utilities.cpp
    src/XPMP2.h
    src/XPMPMultiplayer.cpp
)

# Define pre-compiled header
target_precompile_headers(XPMP2 PRIVATE src/XPMP2.h)

# Header include directories
target_include_directories(XPMP2
	PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/inc
    PRIVATE
		${CMAKE_CURRENT_SOURCE_DIR}/lib/SDK/CHeaders/XPLM
		${CMAKE_CURRENT_SOURCE_DIR}/src
)

# Some files are known to do float comparison, but are good with that
if (NOT MSVC)
    set_source_files_properties(
        src/Remote.cpp
    PROPERTIES
        COMPILE_FLAGS -Wno-float-equal)
endif()

# Include MingW threads
if (MINGW)
    target_link_libraries(XPMP2 mingw_stdthreads)
endif()

# FMOD Library support if requested
# (The actual FMOD lib is not included here, that needs to be done by the plugin target)
if(INCLUDE_FMOD_SOUND)
    # Compile options
    add_compile_definitions(INCLUDE_FMOD_SOUND)
    target_include_directories(XPMP2 PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/lib/fmod/inc)
    target_sources(XPMP2 PRIVATE
        src/Sound.h
        src/Sound.cpp
    )
endif()

# make it a framework
if(APPLE)
    list(APPEND XPMP2_PUB_HEADERS
        ../inc/XPCAircraft.h
        ../inc/XPMPAircraft.h
        ../inc/XPMPMultiplayer.h
        ../inc/XPMPPlaneRenderer.h
        ../inc/XPMPRemote.h
    )

    set_target_properties(XPMP2 PROPERTIES
        FRAMEWORK TRUE
        FRAMEWORK_VERSION ${PROJECT_VERSION}
        MACOSX_FRAMEWORK_IDENTIFIER com.twinfan.XPMP2
        MACOSX_FRAMEWORK_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/Info.plist
        VERSION ${PROJECT_VERSION}.0
        SOVERSION ${PROJECT_VERSION}.0
        PUBLIC_HEADER "${XPMP2_PUB_HEADERS}"
    )
endif()

if(MSVC)
    target_compile_options(XPMP2 PRIVATE
        $<$<CONFIG:Debug>:/Zi>
        $<$<CONFIG:Release>:/Zi /O2 /GL->
    )
else()
    target_compile_options(XPMP2 PRIVATE
        $<$<CONFIG:Debug>:-O0 -g -fPIC>
        $<$<CONFIG:Release>:-O3 -fPIC>
    )
endif()