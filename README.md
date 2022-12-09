# CMakeLists.txt
Quick note for commands and related links for preparing CMakeLists.txt


#### set the CMake version as follows: https://cmake.org/cmake/help/latest/command/cmake_minimum_required.html#command:cmake_minimum_required
````
cmake_minimum_required(VERSION 3.24.3)
````
#### set the project name & version: https://cmake.org/cmake/help/latest/command/project.html#command:project
````
project(Tutorial VERSION 1.0)
````
#### enable support for a specific C++ standard as follows: 
#### Make sure to add CMAKE_CXX_STANDARD declarations above the call to add_executable().
````
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)
````
#### executable of the main cpp file: https://cmake.org/cmake/help/latest/command/add_executable.html#command:add_executable
````
add_executable(Tutorial tutorial.cxx)
````
#### to copy the input file with the specified CMake variables replaced: https://cmake.org/cmake/help/latest/command/configure_file.html#command:configure_file
````
configure_file(TutorialConfig.h.in TutorialConfig.h)
````
####  The source file for the library is passed as an argument: https://cmake.org/cmake/help/latest/command/add_library.html#command:add_library
````
add_library(MathFunctions mysqrt.cxx)
````
_______________________________ CASE 1 _________________________________
# To make use of the new library we will add an add_subdirectory() call in the top-level CMakeLists.txt file so that the library will get built. 
# (https://cmake.org/cmake/help/latest/command/add_subdirectory.html#command:add_subdirectory)
add_subdirectory(MathFunctions)

# the new library target is linked to the executable target  (https://cmake.org/cmake/help/latest/command/target_link_libraries.html#command:target_link_libraries)
target_link_libraries(Tutorial PUBLIC MathFunctions)

# to specify where the executable target should look for include files.
# to specify the library's header file location
# (https://cmake.org/cmake/help/latest/command/target_include_directories.html#command:target_include_directories)
target_include_directories(Tutorial PUBLIC 
                           "${PROJECT_BINARY_DIR}"
                           "${PROJECT_SOURCE_DIR}/MathFunctions"
                            )
##################################################################### CASE 1 END ########################################################################################################

##################################################################### CASE 2 ############################################################################################################
#  make the "MathFunctions" library optional, first step is to add an option to the top-level CMakeLists.txt file. (https://cmake.org/cmake/help/latest/command/option.html#command:option)
# "USE_MYMATH" controller need to be defined in TutorialConfig.h.in and can be used in tutorial.cxx script
option(USE_MYMATH "Use tutorial provided math implementation" ON)

##################################################################### SUB CASE 2.1 ######################################################################################################
# When USE_MYMATH is ON, the lists will be generated and will be added to our project. When USE_MYMATH is OFF, the lists stay empty. 
# With this strategy, we allow users to toggle USE_MYMATH to manipulate what library is used in the build.
if(USE_MYMATH)
    add_subdirectory(MathFunctions)
    list(APPEND EXTRA_LIBS MathFunctions)
    list(APPEND EXTRA_INCLUDES "${PROJECT_SOURCE_DIR}/MathFunctions")
endif()

# replace the written out library names with EXTRA_LIBS
target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS})

# replace the library's header file location with EXTRA_INCLUDES
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           ${EXTRA_INCLUDES}
                           ) 
#################################################################### SUB CASE 2.1 END ##################################################################################################

#################################################################### SUB CASE 2.2 ######################################################################################################
#  modern CMake approach, "MathFunctions" will specify any needed include directories itself
# "Tutorial" simply needs to link to "MathFunctions" and not worry about any additional include directories.
# At the end of MathFunctions/CMakeLists.txt, use target_include_directories() with the INTERFACE keyword, 
# Remember "INTERFACE" means things that consumers require but the producer doesn't.
target_include_directories(MathFunctions 
                           INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
                           )                   

# we've specified usage requirements for "MathFunctions" we can safely remove our uses of the "EXTRA_INCLUDES" variable from the top-level CMakeLists.txt, here:
if(USE_MYMATH)
    add_subdirectory(MathFunctions)
    list(APPEND EXTRA_LIBS MathFunctions)
endif()

# Now only need to add the name of the library target
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )
################################################################### SUB CASE 2.2 END ################################################################################################### 

############################################################################# CASE 2 END ################################################################################################

################################################################## USE CASES: ADDING GENERATOR EXPRESSIONS #######################################################################
# Generator expressions may be used to enable conditional linking, conditional definitions used when compiling, conditional include directories and more.
# (https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html#manual:cmake-generator-expressions(7))
# first remove below two lines from CMakeLists.txt
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# create an interface library, tutorial_compiler_flags. And then use target_compile_features() to add the compiler feature cxx_std_11.
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_STD_11)

# Finally, with our interface library set up, we need to link our executable Target and our MathFunctions library to our new tutorial_compiler_flags library
target_link_libraries(Tutorial PUBLIC
                      ${EXTRA_LIBS}
                      tutorial_compiler_flags
                      )

# Also modify MathFunctions/CMakeLists.txt
target_link_libraries(MathFunctions tutorial_compiler_flags)

#With above modifications(line 84 - line 102), all of our code still requires C++ 11 to build. Notice though that with this method, it gives us the ability to be specific about which targets get specific requirements.
# In addition, we create a single source of truth in our interface library.

################################################## USE CASES: Adding Compiler Warning Flags with Generator Expressions ###############################################################

# Update as to require at least CMake version 3.25:
cmake_minimum_required(VERSION 3.25)

# Next determine which compiler our system is using to build since warning flags vary based on the compiler used
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")

# add the desired compiler warning flags that we want for our project
target_compile_options(tutorial_compiler_flags INTERFACE
                       "$<${gcc_like_cxx}:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>"
                       "$<${msvc_cxx}:-W3>"
                       )

# Lastly, we only want these warning flags to be used during builds. 
# Consumers of our installed project should not inherit our warning flags. 
# To specify this, we wrap our flags in a generator expression using the BUILD_INTERFACE condition.
# change Line 118 - Line 121 as follows:
target_compile_options(tutorial_compiler_flags INTERFACE
                       "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
                       "$<${msvcc_cxx}:$<BUILD_INTERFACE:-W3>>"
                       )
################################################################################# USE CASES END #######################################################################################

################################################################################ INSTALLING ##########################################################################################
# This step will install the appropriate header files, libraries, and executables.
$ cmake --install .

# For multi-configuration tools, don't forget to use the --config argument to specify the configuration
$ cmake --install . --config Release

# build the same install target from the command line like below:
$ cmake --build . --target install --config Debug

# The CMake variable CMAKE_INSTALL_PREFIX is used to determine the root of where the files will be installed 
# (https://cmake.org/cmake/help/latest/variable/CMAKE_INSTALL_PREFIX.html#variable:CMAKE_INSTALL_PREFIX)
$ cmake --install . --prefix "/home/myuser/installdir"

# For MathFunctions, we want to install the libraries and header file to the lib and include directories respectively
# so the end of MathFunctions/CMakeLists.txt we add:
set(installable_libs MathFunctions tutorial_compiler_flags)
install(TARGETS ${installable_libs} DESTINATION lib)
install(FILES MathFunctions.h DESTINATION include)

# For the Tutorial executable, we want to install the executable and configured header file to the bin and include directories respectively.
# The install rules for the Tutorial executable and configured header file are similar. To the end of the top-level CMakeLists.txt we add:
install(TARGETS Tutorial DESTINATION bin)
install(FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h"
        DESTINATION include
        )
################################################################################# INSTALLING END ########################################################################################

################################################################################## TESTING ############################################################################################
# At the end of the top-level CMakeLists.txt file we first need to enable testing with the enable_testing() command.
enable_testing()

# With testing enabled, we will add a number of basic tests to verify that the application is working correctly CMakeLists.txt. (assuming this library return squareroot of the input value)
add_test(NAME Runs COMMAND Tutorial 25)

# use the PASS_REGULAR_EXPRESSION test property to verify that the output of the test contains certain strings CMakeLists.txt
add_test(NAME Usage COMMAND Tutorial)
set_tests_properties(Usage
                     PROPERTIES PASS_REGULAR_EXPRESSION "Usage:.*number"
                     )
    
# next test we will add verifies the computed value is truly the square root
add_test(NAME StandardUse COMMAND Tutorial 4)
set_tests_properties(StandardUse
                     PROPERTIES PASS_REGULAR_EXPRESSION "4 is 2"
                     )

# We should add more tests to verify this. To easily add more tests, we make a function called do_test that runs the application and verifies that 
# the computed square root is correct for given input. 
# For each invocation of do_test, another test is added to the project with a name, input, and expected results based on the passed arguments. 
function(do_test target arg result)
    add_test(NAME Comp${arg} COMMAND ${target} ${arg})
    set_tests_properties(Comp${arg}
                         PROPERTIES PASS_REGULAR_EXPRESSION ${result}
                         )
endfunction()

# da a bunch of result based tests
do_test(Tutorial 4 "4 is 2")
do_test(Tutorial 9 "9 is 3")
do_test(Tutorial 5 "5 is 2.236")
do_test(Tutorial 7 "7 is 2.645")
do_test(Tutorial 25 "25 is 5")
do_test(Tutorial -25 "-25 is (-nan|nan|0)")
do_test(Tutorial 0.0001 "0.0001 is 0.01")

########################################################## Adding Support for a Testing Dashboard #####################################################################################
#  To include support for dashboards we include the CTest module in our top-level CMakeLists.txt. 
# (https://cmake.org/cmake/help/latest/guide/tutorial/Adding%20Support%20for%20a%20Testing%20Dashboard.html)
# Replace CMakeLists.txt
enable_testing()
# with CMakeLists.txt (The CTest module will automatically call enable_testing(), so we can remove it from our CMake files.)
include(CTest)

# We will also need to acquire a CTestConfig.cmake file to be placed in the top-level directory where we can specify information to CTest about the project
set(CTEST_PROJECT_NAME "CMakeTutorial")
set(CTEST_NIGHTLY_START_TIME "00:00:00 EST")

set(CTEST_DROP_METHOD "http")
set(CTEST_DROP_SITE "my.cdash.org")
set(CTEST_DROP_LOCATION "/submit.php?project=CMakeTutorial")
set(CTEST_DROP_SITE_CDASH TRUE)

# The ctest executable will read in this file when it runs. 
# To create a simple dashboard you can run the cmake executable or the cmake-gui to configure the project, but do not build it yet.
# Instead, change directory to the binary tree, and then run:
$ ctest [-VV] -D Experimental

# Remember, for multi-config generators (e.g. Visual Studio), the configuration type must be specified:
$ ctest [-VV] -C Debug -D Experimental
################################################################################# TESTING END ##########################################################################################

# Adding System Introspection (https://cmake.org/cmake/help/latest/guide/tutorial/Adding%20System%20Introspection.html)
# Adding a Custom Command and Generated File (https://cmake.org/cmake/help/latest/guide/tutorial/Adding%20a%20Custom%20Command%20and%20Generated%20File.html)

############################################################################## Packaging an Installer ##################################################################################
# to provide both binary and source distributions on a variety of platforms.
# use CPack to create platform specific installers. Specifically we need to add a few lines to the bottom of our top-level CMakeLists.txt file.
# (https://cmake.org/cmake/help/latest/guide/tutorial/Packaging%20an%20Installer.html)
include(InstallRequiredSystemLibraries)
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")
set(CPACK_PACKAGE_VERSION_MAJOR "${Tutorial_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${Tutorial_VERSION_MINOR}")
set(CPACK_SOURCE_GENERATOR "TGZ")
include(CPack)

#The next step is to build the project in the usual manner and then run the cpack executable. To build a binary distribution, from the binary directory run:
$ cpack 

# To specify the generator, use the -G option. For multi-config builds, use -C to specify the configuration. For example:
$ cpack -G ZIP -C Debug 

# To create an archive of the full source tree you would type:
$ cpack --config CPackSourceConfig.cmake 
################################################################## Packaging an Installer END ##########################################################################################

################################################################ Selecting Static or Shared Libraries ################################################################################
# (https://cmake.org/cmake/help/latest/guide/tutorial/Selecting%20Static%20or%20Shared%20Libraries.html)
# The first step is to update the starting section of the top-level CMakeLists.txt to look like:
---------------------------------------------------------------------------------------------------
cmake_minimum_required(VERSION 3.15)

# set the project name and version
project(Tutorial VERSION 1.0)

# specify the C++ standard
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)

# add compiler warning flags just when building this project via
# the BUILD_INTERFACE genex
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")
target_compile_options(tutorial_compiler_flags INTERFACE
  "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
  "$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
)

# control where the static and shared libraries are built so that on windows
# we don't need to tinker with the path to run the executable
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")

option(BUILD_SHARED_LIBS "Build using shared libraries" ON)

# configure a header file to pass the version number only
configure_file(TutorialConfig.h.in TutorialConfig.h)

# add the MathFunctions library
add_subdirectory(MathFunctions)

# add the executable
add_executable(Tutorial tutorial.cxx)
target_link_libraries(Tutorial PUBLIC MathFunctions tutorial_compiler_flags)
----------------------------------------------------------------------------------------------------

# The end result is that MathFunctions/CMakeLists.txt should look like:
-----------------------------------------------------------------------------------------------------
# add the library that runs
add_library(MathFunctions MathFunctions.cxx)

# state that anybody linking to us needs to include the current source dir
# to find MathFunctions.h, while we don't.
target_include_directories(MathFunctions
                           INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
                           )

# should we use our own math functions
option(USE_MYMATH "Use tutorial provided math implementation" ON)
if(USE_MYMATH)

  target_compile_definitions(MathFunctions PRIVATE "USE_MYMATH")

  # first we add the executable that generates the table
  add_executable(MakeTable MakeTable.cxx)
  target_link_libraries(MakeTable PRIVATE tutorial_compiler_flags)

  # add the command to generate the source code
  add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h
    COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h
    DEPENDS MakeTable
    )

  # library that just does sqrt
  add_library(SqrtLibrary STATIC
              mysqrt.cxx
              ${CMAKE_CURRENT_BINARY_DIR}/Table.h
              )

  # state that we depend on our binary dir to find Table.h
  target_include_directories(SqrtLibrary PRIVATE
                             ${CMAKE_CURRENT_BINARY_DIR}
                             )

  target_link_libraries(SqrtLibrary PUBLIC tutorial_compiler_flags)
  target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
endif()

target_link_libraries(MathFunctions PUBLIC tutorial_compiler_flags)

# define the symbol stating we are using the declspec(dllexport) when
# building on windows
target_compile_definitions(MathFunctions PRIVATE "EXPORTING_MYMATH")

# install libs
set(installable_libs MathFunctions tutorial_compiler_flags)
if(TARGET SqrtLibrary)
  list(APPEND installable_libs SqrtLibrary)
endif()
install(TARGETS ${installable_libs} DESTINATION lib)
# install include headers
install(FILES MathFunctions.h DESTINATION include)
----------------------------------------------------------------------------------------------------------------
################################################################ Selecting Static or Shared Libraries END ################################################################################

# Adding Export Configuration (https://cmake.org/cmake/help/latest/guide/tutorial/Adding%20Export%20Configuration.html)
# Packaging Debug and Release (https://cmake.org/cmake/help/latest/guide/tutorial/Packaging%20Debug%20and%20Release.html)


################################################################# User Interaction Guide #############################################################################################
# (https://cmake.org/cmake/help/latest/guide/user-interaction/index.html#)

# A simple but typical use of cmake(1) with a fresh copy of software source code is to create a build directory and invoke cmake there:
$ cd some_software-1.4.2
$ mkdir build
$ cd build
$ cmake .. -DCMAKE_INSTALL_PREFIX=/opt/the/prefix
$ cmake --build .
$ cmake --build . --target install

/////////////////////////////////////////////////////////////////

Alternately:
$ cd projectdir
$ mkdir build
   
# use the following command to build if you are in project root
$ cmake -Bbuild -DCMAKE_BUILD_TYPE=Release -DBUILD_DEPS=ON

# or use the following command if you are in build directory
# cmake ../ -DCMAKE_BUILD_TYPE=Release

# Finally build the executable if you are in the project root 
$ cmake --build build

# If you are in the build directory build
$ cmake --build .
