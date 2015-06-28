# Script to ease the use of WiringPi and an ARM cross compiler
#
# The goal of this script is to make it easier to work with a cross compiler and
# WiringPi.
#
# This script needs some varaibles to be defined:
#	- CMAKE_CXX_COMPILER and CMAKE_C_COMPILER should point to a valid ARM
#		cross compiler (even though your project is only one of the two
#		languages).
#	- RASPBERRY_PI_HOST the address and username to use when the code is
#		uploaded. Must be in the form : [<username>@]<ip_adress>.
#	- PKG_CONFIG_PATH (environment variable) must point to where wiringpi.pc
#		can be found. Note: this must be defined as an environment variable
#
# We suggest you to create a small file containing all those variables and to
# include it in your project before this file.
#
#
# This script takes care of including WiringPi in your project, the include
# directories are automatically updated. Though you must link to the libraries.
# In order to make it easy a variable is available: LIBS. Use it as follow :
#
#	- target_link_libraries(<your_target> ${LIBS})
#
# Finally a new target is available: upload !
# To allow a target to be uploaded to the raspberry pi, simply add the following
# to you project:
#
#	- allow_target_upload(<your_target>)
#
# Author: Raphaël Lallement - raphael [dot] lallement [at] laposte [dot] net


cmake_minimum_required(VERSION 2.8)

#Detect the config file (or create it if not existing)
set(CONFIG_FILE "${CMAKE_CURRENT_LIST_DIR}/raspberry.config")
if(NOT EXISTS "${CONFIG_FILE}")
#The file has not been found, create it with default values
	message(AUTHOR_WARNING "\nCould not find the configuration file, create one with default value, please edit ${CONFIG_FILE}")
	file(WRITE "${CONFIG_FILE}" "set(CMAKE_INSTALL_PREFIX \"<path/to/raspberry/install>\")\n")
	file(APPEND "${CONFIG_FILE}" "set(CMAKE_C_COMPILER \"unknown\")\n")
	file(APPEND "${CONFIG_FILE}" "set(CMAKE_CXX_COMPILER \"unknown\")\n")
	file(APPEND "${CONFIG_FILE}" "set(ENV{PKG_CONFIG_PATH} \"\${CMAKE_INSTALL_PREFIX}/lib/pkgconfig/:\$ENV{PKG_CONFIG_PATH}\")\n")
	file(APPEND "${CONFIG_FILE}" "set(CMAKE_MODULE_PATH \"\${CMAKE_INSTALL_PREFIX}/share/cmake/Modules\")\n")
	file(APPEND "${CONFIG_FILE}" "set(RASPBERRY_PI_HOST root@<ip_address>)\n")
	file(APPEND "${CONFIG_FILE}" "set(RASPBERRY_PI_INSTALL_PREFIX \"/root\")\n")
endif(NOT EXISTS "${CONFIG_FILE}")
include("${CONFIG_FILE}")

#Check that the project name is already declared
if(NOT PROJECT_NAME)
	message(FATAL_ERROR "\nInclude the CMakeLists.txt of the raspberry workspace AFTER you decalre your project (it needs a project name)")
endif(NOT PROJECT_NAME)

#VERIFY that both C and C++ compilers are ARM cross compilers
execute_process(COMMAND ${CMAKE_CXX_COMPILER} -dumpspecs
	COMMAND fgrep -A 1 "*cross_compile" 
	COMMAND fgrep -q "1"
	RESULT_VARIABLE IS_CXX_CROSS_COMPILER
)
execute_process(COMMAND ${CMAKE_C_COMPILER} -dumpspecs
	COMMAND fgrep -A 1 "*cross_compile" 
	COMMAND fgrep -q "1"
	RESULT_VARIABLE IS_C_CROSS_COMPILER
)

if(IS_CXX_CROSS_COMPILER EQUAL "0" AND IS_C_CROSS_COMPILER EQUAL "0")
	execute_process(COMMAND ${CMAKE_CXX_COMPILER} -dumpspecs
		COMMAND fgrep -A 1 "*multilib_defaults" 
		COMMAND grep -q "arm"
		RESULT_VARIABLE CXX_ARM_CROSS_COMPILER_FOUND
	)

	execute_process(COMMAND ${CMAKE_CXX_COMPILER} -dumpspecs
		COMMAND fgrep -A 1 "*multilib_defaults" 
		COMMAND grep -q "arm"
		RESULT_VARIABLE C_ARM_CROSS_COMPILER_FOUND
	)
	
	if(NOT CXX_ARM_CROSS_COMPILER_FOUND EQUAL "0" OR NOT C_ARM_CROSS_COMPILER_FOUND EQUAL "0")
		message(FATAL_ERROR "\nThis CMake file should not be used with something that is not a C/C++ ARM cross compiler")
	endif(NOT CXX_ARM_CROSS_COMPILER_FOUND EQUAL "0" OR NOT C_ARM_CROSS_COMPILER_FOUND EQUAL "0")
else(IS_CXX_CROSS_COMPILER EQUAL "0" AND IS_C_CROSS_COMPILER EQUAL "0")
	message(FATAL_ERROR "\nThis CMake file should not be used with something that is not a C/C++ cross compiler")
endif(IS_CXX_CROSS_COMPILER EQUAL "0" AND IS_C_CROSS_COMPILER EQUAL "0")


#Include PkgConfig to find wiringPi, and PThread to link with it correctly
find_package(PkgConfig REQUIRED)
find_package(Threads REQUIRED)

#Import the wiringPi library
pkg_check_modules(wiringPi REQUIRED wiringPi)
include_directories(${wiringPi_INCLUDE_DIRS})
set(LIBS ${wiringPi_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
link_directories(${wiringPi_LIBRARY_DIRS})

#Handle the RPATH (takes care of finding the libraries)
set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
set(CMAKE_INSTALL_RPATH "${RASPBERRY_PI_INSTALL_PREFIX}/${PROJECT_NAME}")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

#Define a macro to offer the possibility to upload a target to the raspberry
add_custom_target(upload COMMENT "Upload all the targets to the raspberry" VERBATIM)

macro(allow_target_upload TARGET)
	if(NOT RASPBERRY_PI_HOST)
		message(AUTHOR_WARNING "\nThe raspberry pi host has not be defined for this project, please use \"set(RASPBERRY_PI_HOST <user@ip_address>)\" to fix this issue\n")
	else(NOT RASPBERRY_PI_HOST)
		message(STATUS "The target \"${TARGET}\" can be upload directly to the Raspberry (use make upload)")
		
		get_target_property(version ${TARGET} VERSION)
		if(version)
			message(AUTHOR_WARNING "\nThe target \"${TARGET}\" should not have a version number\n")
		endif(version)
		
		add_custom_target(upload_${TARGET} scp $<TARGET_FILE:${TARGET}> ${RASPBERRY_PI_HOST}:${RASPBERRY_PI_INSTALL_PREFIX}/${PROJECT_NAME}/. COMMENT "Upload the target \"${TARGET}\" to the raspberry" VERBATIM)
		add_dependencies(upload upload_${TARGET} ${TARGET})
	endif(NOT RASPBERRY_PI_HOST)
endmacro(allow_target_upload)
