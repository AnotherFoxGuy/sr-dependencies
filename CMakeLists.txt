# The MIT License (MIT)
#
# Copyright (c) 2017 Edgar (Edgar@AnotherFoxGuy.com)
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

cmake_minimum_required(VERSION 3.2)


# ------------------------------------------------------------------------------------------------#
# The intention of this script is to provide a simple solution for building all dependencies
# required for stuntrally. It is in fact a super- or meta-build project which makes use of
# the ExternalProject module of CMake to build the individual dependencies.
# ------------------------------------------------------------------------------------------------#

project(sr-dependencies VERSION 0.1)


# ------------------------------------------------------------------------------------------------#
# Options to enable/disable building specific dependencies
# ------------------------------------------------------------------------------------------------#

# Options with same default value on all platforms
option(BUILD_OGRE "Build Ogre3D graphics library" ON)
option(BUILD_MYGUI "Build MyGUI graphical UI library" ON)
option(BUILD_OPENAL "Build OpenAL audio library" ON)
option(BUILD_SDL2 "Build SDL2 library" ON)
option(BUILD_OGG_VORBIS "Build OGG and VorbisFile library" ON)
option(BUILD_BULLET "Build Bullet Physics library" ON)
option(BUILD_ENET "Build ENet UDP networking library" ON)


 set(BOOST_ROOT "C:/Libraries/boost_1_64_0" CACHE PATH "Boost installation prefix")


# ------------------------------------------------------------------------------------------------#
# General setup and initialization
# ------------------------------------------------------------------------------------------------#

include(ExternalProject)

# Set the EP_BASE directory property to setup the build directory structure (see the
# ExternalProject documentation for more information)
set_property(DIRECTORY PROPERTY EP_BASE ${CMAKE_BINARY_DIR})


# Determine the name for the output directory where dependencies are going to be installed
if (WIN32)
    set(DEPENDENCIES_OUTPUT_DIR ${CMAKE_BINARY_DIR}/Dependencies_${CMAKE_SYSTEM_NAME}_${CMAKE_GENERATOR})
    string(REPLACE " " "-" DEPENDENCIES_OUTPUT_DIR ${DEPENDENCIES_OUTPUT_DIR})
else ()
    set(DEPENDENCIES_OUTPUT_DIR ${CMAKE_BINARY_DIR}/Dependencies_${CMAKE_SYSTEM_NAME})
endif ()

if (MSVC)
    set(DEPENDENCIES_INCLUDE_DIR ${DEPENDENCIES_OUTPUT_DIR}/include)
    set(DEPENDENCIES_LIB_DIR ${DEPENDENCIES_OUTPUT_DIR}/lib)
    set(DEPENDENCIES_BIN_DIR ${DEPENDENCIES_OUTPUT_DIR}/bin)
else ()
    set(DEPENDENCIES_INCLUDE_DIR ${DEPENDENCIES_OUTPUT_DIR}/include)
    set(DEPENDENCIES_LIB_DIR ${DEPENDENCIES_OUTPUT_DIR}/lib/${CMAKE_BUILD_TYPE})
    set(DEPENDENCIES_BIN_DIR ${DEPENDENCIES_OUTPUT_DIR}/bin/${CMAKE_BUILD_TYPE})
endif ()

file(MAKE_DIRECTORY ${DEPENDENCIES_INCLUDE_DIR})
file(MAKE_DIRECTORY ${DEPENDENCIES_LIB_DIR})
file(MAKE_DIRECTORY ${DEPENDENCIES_BIN_DIR})


# Compile flags
if (MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /bigobj /w")
endif ()

if (UNIX)
    #set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native" )
endif ()


# On Linux some packages rely on pkgconfig to be found correctly. Therefore the environment variable
# needs to be set up accordingly. In addition when building depedencies which need to find other
# dependencies (e.g. MyGUI searching for OGRE) PKG_CONFIG_USE_CMAKE_PREFIX_PATH needs to be set to TRUE.
if (UNIX)
    set(ENV{PKG_CONFIG_PATH} "$ENV{PKG_CONFIG_PATH}:${DEPENDENCIES_LIB_DIR}/pkgconfig")
endif ()



# ------------------------------------------------------------------------------------------------#
#  Ogre3D
# ------------------------------------------------------------------------------------------------#
# Patching is needed for the D3D9 plugin when compiling with VS2015.
# See http://www.ogre3d.org/forums/viewtopic.php?f=2&t=84008

if (BUILD_OGRE)

    # First build the required dependencies for Ogre itself. They are provided in a dedicated repository.
    ExternalProject_Add(
            ogredeps
            URL https://bitbucket.org/cabalistic/ogredeps/get/7168c50f9d04.zip
            CMAKE_ARGS
            -DOGREDEPS_BUILD_SDL2=OFF
            -DOGREDEPS_BUILD_AMD_QBS=OFF
            -DOGREDEPS_BUILD_NVIDIA_NVAPI=OFF
            -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/OgreDependencies
            -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
    )

    if (WIN32)
        # By default the cg.dll is only installed in the build directory. Therefore manually copy it to the dependency output directory.
        ExternalProject_Add_Step(
                ogredeps copy_cg_dll
                COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/Build/ogredeps/bin/Release/cg.dll ${DEPENDENCIES_BIN_DIR}/cg.dll
                DEPENDEES build
        )
    endif ()

    ExternalProject_Add(
            ogre
            DEPENDS ogredeps
            URL https://bitbucket.org/sinbad/ogre/get/v1-9-0.tar.gz
            UPDATE_COMMAND ""
            PATCH_COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/patches/OgreTerrain.cpp ${CMAKE_BINARY_DIR}/Source/ogre/Components/Terrain/src/OgreTerrain.cpp  # Fix stackoverflow on terrain load (PR #336)
            COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/patches/OgreD3D9Prerequisites.h ${CMAKE_BINARY_DIR}/Source/ogre/RenderSystems/Direct3D9/include/OgreD3D9Prerequisites.h # workaround for VS2015
            COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/patches/Ogre_RenderSystem_Direct3D9_CMakeLists.txt ${CMAKE_BINARY_DIR}/Source/ogre/RenderSystems/Direct3D9/CMakeLists.txt # To include bundled DirectX-SDK files
            CMAKE_ARGS
            -DBOOST_ROOT=${BOOST_ROOT}
            -DCMAKE_INSTALL_PREFIX=${DEPENDENCIES_OUTPUT_DIR}
            -DOGRE_DEPENDENCIES_DIR=${CMAKE_BINARY_DIR}/OgreDependencies
            -DOGRE_BUILD_SAMPLES=OFF
            -DOGRE_BUILD_TOOLS=OFF
            -DOGRE_BUILD_RENDERSYSTEM_D3D9=ON
            -DOGRE_BUILD_RENDERSYSTEM_D3D11=OFF  # TODO
            -DOGRE_BUILD_RENDERSYSTEM_GL3PLUS=OFF
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
            -DDirectX9_D3DX9_LIBRARY=${CMAKE_SOURCE_DIR}/win32-directx/lib/x86/d3dx9.lib  # Add bundled DirectX libs
            -DDirectX9_DXERR_LIBRARY=${CMAKE_SOURCE_DIR}/win32-directx/lib/x86/DxErr.lib  # Add bundled DirectX libs
            -DDirectX9_BUNDLED_INCLUDE_DIR=${CMAKE_SOURCE_DIR}/win32-directx/include      # Add bundled DirectX headers; requires modified CMakeLists.txt
    )

endif ()


# ------------------------------------------------------------------------------------------------#
#  MyGUI
# ------------------------------------------------------------------------------------------------#

if (BUILD_MYGUI)

    ExternalProject_Add(
            mygui
            DEPENDS ogre
            URL https://github.com/MyGUI/mygui/archive/master.tar.gz
            UPDATE_COMMAND ""
            CMAKE_ARGS
            -DBOOST_ROOT=${BOOST_ROOT}
            -DPKG_CONFIG_USE_CMAKE_PREFIX_PATH=ON
            -DCMAKE_PREFIX_PATH=${DEPENDENCIES_OUTPUT_DIR}
            -DCMAKE_INSTALL_PREFIX=${DEPENDENCIES_OUTPUT_DIR}
            -DMYGUI_DEPENDENCIES_DIR=${CMAKE_BINARY_DIR}/OgreDependencies
            -DOGRE_SOURCE_DIR=${CMAKE_BINARY_DIR}/Source/ogre
            -DOGRE_BUILD=${DEPENDENCIES_OUTPUT_DIR}
            -DCMAKE_MODULE_PATH=${CMAKE_MODULE_PATH}
            -DMYGUI_BUILD_DEMOS=OFF
            -DMYGUI_BUILD_DOCS=OFF
            -DMYGUI_BUILD_TEST_APP=OFF
            -DMYGUI_BUILD_TOOLS=OFF
            -DMYGUI_BUILD_PLUGINS=NO
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
    )

endif ()


# ------------------------------------------------------------------------------------------------#
#  OpenAL
# ------------------------------------------------------------------------------------------------#

if (BUILD_OPENAL)

    ExternalProject_Add(
            openal
            GIT_REPOSITORY https://github.com/kcat/openal-soft
            GIT_TAG openal-soft-1.17.1
            UPDATE_COMMAND ""
            CMAKE_ARGS
            -DCMAKE_INSTALL_PREFIX=${DEPENDENCIES_OUTPUT_DIR}
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
    )

endif ()

# ------------------------------------------------------------------------------------------------#
#  SDL2
# ------------------------------------------------------------------------------------------------#

if (BUILD_SDL2)

    ExternalProject_Add(
            SDL2
            URL https://libsdl.org/release/SDL2-2.0.7.zip
            UPDATE_COMMAND ""
            CMAKE_ARGS
            -DCMAKE_INSTALL_PREFIX=${DEPENDENCIES_OUTPUT_DIR}
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
    )

endif ()


# ------------------------------------------------------------------------------------------------#
#  vorbis
# ------------------------------------------------------------------------------------------------#


if (BUILD_OGG_VORBIS)

	ExternalProject_Add(
			ogg
			GIT_REPOSITORY https://github.com/xiph/ogg.git
			# UPDATE_COMMAND ""
			CMAKE_ARGS
			-DCMAKE_INSTALL_PREFIX=${DEPENDENCIES_OUTPUT_DIR}
			-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
			-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
    )
	
	ExternalProject_Add(
            vorbis
			DEPENDS ogg
            GIT_REPOSITORY https://github.com/xiph/vorbis.git
            # UPDATE_COMMAND ""
            CMAKE_ARGS
            -DCMAKE_INSTALL_PREFIX=${DEPENDENCIES_OUTPUT_DIR}
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
    )

endif ()

# ------------------------------------------------------------------------------------------------#
#  Bullet Physics
# ------------------------------------------------------------------------------------------------#

if (BUILD_BULLET)
	
	ExternalProject_Add(
            bulletphysics
            URL https://github.com/bulletphysics/bullet3/archive/master.zip
            UPDATE_COMMAND ""
            CMAKE_ARGS
			-DINSTALL_LIBS=ON
			-DBUILD_BULLET2_DEMOS=OFF
			-DBUILD_CPU_DEMOS=OFF
			-DBUILD_EXTRAS=ON
			-DBUILD_OPENGL3_DEMOS=OFF
			-DBUILD_UNIT_TESTS=OFF
            -DCMAKE_INSTALL_PREFIX=${DEPENDENCIES_OUTPUT_DIR}
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
    )

endif ()

# ------------------------------------------------------------------------------------------------#
#  ENet 
# ------------------------------------------------------------------------------------------------#

if (BUILD_ENET)

	set( ENET_BUILD_DIR ${CMAKE_BINARY_DIR}/Build/enet )
	set( ENET_SOURCE_DIR ${CMAKE_BINARY_DIR}/Source/enet )

    ExternalProject_Add(
            enet
            GIT_REPOSITORY https://github.com/lsalzman/enet.git
            #UPDATE_COMMAND ""
            CMAKE_ARGS
            -DCMAKE_INSTALL_PREFIX=${DEPENDENCIES_OUTPUT_DIR}
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
            INSTALL_COMMAND ${CMAKE_COMMAND} -E copy ${ENET_BUILD_DIR}/Release/enet.lib ${DEPENDENCIES_LIB_DIR}/enet.lib
            COMMAND	${CMAKE_COMMAND} -E copy_directory ${ENET_SOURCE_DIR}/include/ ${DEPENDENCIES_INCLUDE_DIR}/
    )

endif ()
