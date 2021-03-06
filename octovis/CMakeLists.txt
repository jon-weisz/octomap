CMAKE_MINIMUM_REQUIRED(VERSION 2.6)
PROJECT( octovis )

# # version (e.g. for packaging)
set(OCTOVIS_MAJOR_VERSION 1)
set(OCTOVIS_MINOR_VERSION 5)
set(OCTOVIS_PATCH_VERSION 3)
set(OCTOVIS_VERSION ${OCTOVIS_MAJOR_VERSION}.${OCTOVIS_MINOR_VERSION}.${OCTOVIS_PATCH_VERSION})
# get rid of a useless warning:
if(COMMAND cmake_policy)
  cmake_policy(SET CMP0003 NEW)
endif(COMMAND cmake_policy)

SET (CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/CMakeModules")

# COMPILER SETTINGS (default: Release) and flags
INCLUDE(CompilerSettings)

# Set output directories for libraries and executables
SET( BASE_DIR ${CMAKE_SOURCE_DIR} )
SET( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${BASE_DIR}/lib )
SET( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${BASE_DIR}/lib )
SET( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${BASE_DIR}/bin )

# We need the main octomap library to link againt.
# Look at parent directory by default, in case the complete distribution 
# (octomap & octovis together) is built. 
# Otherwise you need to export octomap_DIR to find the CMakeConfig.
find_package(octomap REQUIRED
  HINTS ${CMAKE_SOURCE_DIR}/lib/cmake/octomap
   ${CMAKE_SOURCE_DIR}/../octomap/lib/cmake/octomap
)
INCLUDE_DIRECTORIES(${OCTOMAP_INCLUDE_DIRS})
LINK_DIRECTORIES(${OCTOMAP_LIBRARY_DIRS})

# Export the package for use from the build-tree
# (this registers the build-tree with a global CMake-registry)
export(PACKAGE octovis)

set(INSTALL_TARGETS_DEFAULT_ARGS
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)

# Builds the "octovis" viewer based on OpenGL and 
# libQGLViewer, if dependencies available
SET( BUILD_VIEWER 0)

# Look for required libraries:
FIND_PACKAGE(OpenGL)
FIND_PACKAGE(Qt4)
IF (OpenGL-NOTFOUND OR Qt4-NOTFOUND) 
	MESSAGE ( "OpenGL and QT4 are required for octovis but could not be found.")	
ELSE()
	FIND_PACKAGE(QGLViewer)
	IF(QGLViewer_FOUND)
	  SET( BUILD_VIEWER 1)
	ELSE()
	  MESSAGE ( "\n")
  	MESSAGE ( "libQGLViewer could not be found or generated.")
	ENDIF()
ENDIF()


IF(BUILD_VIEWER)
  MESSAGE(STATUS "\n")
  MESSAGE(STATUS "viewer octovis will be built")
  
  set(INCLUDE_DIRS "${PROJECT_SOURCE_DIR}/include")
  INCLUDE_DIRECTORIES(${INCLUDE_DIRS})
   
  INCLUDE( CMakeLists_src.txt )
    
  # Create an octovis-config.cmake file for the use from the build tree
  set(OCTOVIS_INCLUDE_DIR "${INCLUDE_DIRS}")
  set(OCTOVIS_LIB_DIR     "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")
  # not used right now (export depends?)
  #set(OCTOMAP_CMAKE_DIR "${PROJECT_BINARY_DIR}")
  configure_file(octovis-config.cmake.in
    "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/cmake/octovis/octovis-config.cmake" @ONLY)
  configure_file(octovis-config-version.cmake.in
    "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/cmake/octovis/octovis-config-version.cmake" @ONLY)  
    
  # Create a octovis-config.cmake file for the use from the install tree
  # and install it
  set(OCTOVIS_INCLUDE_DIRS "${CMAKE_INSTALL_PREFIX}/include")
  set(OCTOVIS_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib")
  #set(OCTOMAP_CMAKE_DIR "${INSTALL_DATA_DIR}/FooBar/CMake")
  configure_file(octovis-config.cmake.in
    "${PROJECT_BINARY_DIR}/InstallFiles/octovis-config.cmake" @ONLY)
  configure_file(octovis-config-version.cmake.in
    "${PROJECT_BINARY_DIR}/InstallFiles/octovis-config-version.cmake" @ONLY)
  install(FILES
    "${PROJECT_BINARY_DIR}/InstallFiles/octovis-config.cmake"
    "${PROJECT_BINARY_DIR}/InstallFiles/octovis-config-version.cmake" 
    DESTINATION share/octovis/)
    
  # make package release from source
	IF (NOT WIN32)
		SET(OCTOVIS_PKG_NAME "${PROJECT_NAME}-${OCTOVIS_VERSION}.tar.gz")
		SET(DIST_DIR "${CMAKE_BINARY_DIR}/dist-${PROJECT_NAME}")
		ADD_CUSTOM_TARGET("dist-${PROJECT_NAME}" 
		  rm -rf "${DIST_DIR}" "${CMAKE_BINARY_DIR}/${OCTOVIS_PKG_NAME}"
		  COMMAND mkdir "${DIST_DIR}"
		  COMMAND svn export --force -q "${PROJECT_SOURCE_DIR}" "${DIST_DIR}/${PROJECT_NAME}" 
		  COMMAND tar -czf "${CMAKE_BINARY_DIR}/${OCTOVIS_PKG_NAME}" -C "${DIST_DIR}" --exclude=".hidden" "${PROJECT_NAME}"
		    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
		  # cleanup so that there is no copy in the source dir
	      COMMAND rm -rf "${DIST_DIR}"
		)
	ENDIF ()
    
  # #installation:
  # # store all header files to install:
  file(GLOB octovis_HDRS  *.h *.hxx *.hpp)
  install(FILES ${octovis_HDRS} DESTINATION include/octovis)
  
ELSE()
    MESSAGE ( "Unfortunately, the viewer (octovis) can not be built because some requirements are missing.")
	MESSAGE ( "This will not affect the compilation of the stand-alone library and tools (octomap)")
    MESSAGE ( "See README.txt or http://octomap.sf.net for further information.\n")
ENDIF()

