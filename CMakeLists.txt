cmake_minimum_required(VERSION 2.8.10.2)
# Requirements from Cygwin:
# patch, dos2unix, make, binutils
#
# This project allows to configure, build, archive an ICU install tree.
#

project(ICU)
include(ExternalProject)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

if(${MSVC_VERSION} LESS "1900") # Before VS 2015, so VS 2013 and earlier
  set(version "58.2")
  set(MD5 f4fca37508fc5d14390501cf17aef084)
else()  # VS2015 && VS2017
  set(version "59.1")
  set(MD5 29a41f9bb576b06c7eef0487a84a7674)
endif()
string(REPLACE "." "_" version_underscore ${version})
set(root_name "icu4c-${version_underscore}")
set(source_archive_name "${root_name}-src.zip")
math(EXPR bitness "${CMAKE_SIZEOF_VOID_P} * 8")
set(binary_archive_name "${root_name}-install-msvc${MSVC_VERSION}-${bitness}")
set(binary_archive_name_with_config "${binary_archive_name}-${CMAKE_BUILD_TYPE}")
set(binary_archive_name_with_config_tmp "${binary_archive_name}-${CMAKE_BUILD_TYPE}-tmp")

option(SUPERBUILD "Build ${proj} Debug and Release tree" ON)

set(CYGWIN_PATH "c:\\cygwin" CACHE PATH "Cygwin install path")
find_program(dos2unix_COMMAND dos2unix.exe DOC "Path to dos2unix executable" HINTS ${CYGWIN_PATH}/bin)
if(NOT dos2unix_COMMAND)
  message(FATAL_ERROR "dos2unix_COMMAND is not set !")
endif()
get_filename_component(dos2unix_BIN_DIR ${dos2unix_COMMAND} PATH)

set(proj ${PROJECT_NAME})

if(NOT SUPERBUILD)

  if(CMAKE_SIZEOF_VOID_P EQUAL 4)
  elseif(CMAKE_SIZEOF_VOID_P EQUAL 8)
  endif()
  if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(config "--enable-debug --disable-release")
	set(suffix d)
  endif()
   
  # Tools have to be enabled. They are necessary for installation/packaging (using --disable-tools fails)
  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${proj}-configure.bat"
  "set PATH=%PATH%;${dos2unix_BIN_DIR}
cd ${CMAKE_CURRENT_BINARY_DIR}/${proj}/source
dos2unix  *
dos2unix -f configure
bash runConfigureICU ${config} Cygwin/MSVC -prefix=${CMAKE_CURRENT_BINARY_DIR}/${binary_archive_name_with_config_tmp} -enable-static -disable-shared --disable-tests --disable-extras
")

  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${proj}-build.bat"
  "set PATH=%PATH%;${dos2unix_BIN_DIR}
cd ${CMAKE_CURRENT_BINARY_DIR}/${proj}/source
make")

  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${proj}-install.bat"
  "set PATH=${dos2unix_BIN_DIR}
cd ${CMAKE_CURRENT_BINARY_DIR}/${proj}/source
make install")

  ExternalProject_Add(${proj}
    URL "http://download.icu-project.org/files/icu4c/${version}/${source_archive_name}"
    URL_MD5 "${MD5}"
    DOWNLOAD_DIR ${CMAKE_CURRENT_BINARY_DIR}
    SOURCE_DIR ${CMAKE_BINARY_DIR}/${proj}
    BUILD_IN_SOURCE 1
    CONFIGURE_COMMAND "${CMAKE_CURRENT_BINARY_DIR}/${proj}-configure.bat"
    BUILD_COMMAND "${CMAKE_CURRENT_BINARY_DIR}/${proj}-build.bat"
    INSTALL_COMMAND "${CMAKE_CURRENT_BINARY_DIR}/${proj}-install.bat"
    )

  # Package only what is necessary for DCMTK
  ExternalProject_Add_Step(${proj} mk_install_dir
    COMMAND ${CMAKE_COMMAND} -E
      make_directory ${binary_archive_name_with_config}
    DEPENDEES install
    )
  ExternalProject_Add_StepTargets(${proj} mk_install_dir)

  foreach(lib dt uc)
  ExternalProject_Add_Step(${proj} package_${lib}
    COMMAND
	${CMAKE_COMMAND} -E copy ${binary_archive_name_with_config_tmp}/lib/sicu${lib}${suffix}.lib ${binary_archive_name_with_config}/lib/icu${lib}${suffix}.lib
    DEPENDEES mk_install_dir
    )
  ExternalProject_Add_StepTargets(${proj} package_${lib})
  list(APPEND lib_dependencies ${proj} package_${lib})
  endforeach()

  ExternalProject_Add_Step(${proj} install_include
    COMMAND ${CMAKE_COMMAND} -E
      copy_directory ${binary_archive_name_with_config_tmp}/include ${binary_archive_name_with_config}/include
    DEPENDEES ${lib_dependencies}
    )
  ExternalProject_Add_StepTargets(${proj} install_include)
  
  ExternalProject_Add_Step(${proj} create-archive
    COMMAND ${CMAKE_COMMAND} -E
      tar czvf ${binary_archive_name_with_config}.tar.gz ${binary_archive_name_with_config}
    DEPENDEES install_include
    )
  ExternalProject_Add_StepTargets(${proj} create-archive)
    
  if(DEFINED MULTICONFIG_DIR)
    ExternalProject_Add_Step(${proj} copy-into-multiconfig-dir
      COMMAND ${CMAKE_COMMAND} -E copy_directory
        ${CMAKE_BINARY_DIR}/${binary_archive_name_with_config} 
        ${MULTICONFIG_DIR}/${CMAKE_BUILD_TYPE}
      DEPENDEES create-archive
      )
    ExternalProject_Add_StepTargets(${proj} copy-into-multiconfig-dir)
  endif()

else()
  
  set(multiconfig_dir ${CMAKE_BINARY_DIR}/${binary_archive_name})
  file(MAKE_DIRECTORY ${multiconfig_dir})
  
  set(ep_cmake_cache_args
    -DCYGWIN_PATH:PATH=${CYGWIN_PATH}
  -Ddos2unix_COMMAND:FILEPATH=${dos2unix_COMMAND}
    -DSUPERBUILD:BOOL=OFF
    -DMULTICONFIG_DIR:PATH=${multiconfig_dir}
    )

  ExternalProject_Add(${proj}-Release
    DOWNLOAD_COMMAND ""
    SOURCE_DIR "${CMAKE_SOURCE_DIR}"
    BINARY_DIR "${CMAKE_BINARY_DIR}/${proj}-Release"
    CMAKE_CACHE_ARGS
	  ${ep_cmake_cache_args}
	  -DCMAKE_BUILD_TYPE:STRING=Release
	  #-DCMAKE_CONFIGURATION_TYPES:STRING=Release
    )
    
  ExternalProject_Add(${proj}-Debug
    DOWNLOAD_COMMAND ""
    SOURCE_DIR "${CMAKE_SOURCE_DIR}"
    BINARY_DIR "${CMAKE_BINARY_DIR}/${proj}-Debug"
    CMAKE_CACHE_ARGS ${ep_cmake_cache_args}
	  -DCMAKE_BUILD_TYPE:STRING=Debug
	  #-DCMAKE_CONFIGURATION_TYPES:STRING=Debug
    )

  ExternalProject_Add(${proj}-MultiConfig
    DOWNLOAD_COMMAND ""
    CONFIGURE_COMMAND ""
    BUILD_COMMAND
      ${CMAKE_COMMAND} -E chdir ${CMAKE_BINARY_DIR}
      ${CMAKE_COMMAND} -E tar czvf ${binary_archive_name}.tar.gz ${binary_archive_name}
    INSTALL_COMMAND ""
    DEPENDS
      ${proj}-Debug
      ${proj}-Release
    STEP_TARGETS build
    )
endif()
