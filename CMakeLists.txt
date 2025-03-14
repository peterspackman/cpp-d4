cmake_minimum_required(VERSION 3.14) # Using a more recent version for better modern features

project(dftd4
  VERSION 2.2.0
  LANGUAGES CXX
  DESCRIPTION "cpp-d4 project"
)

# Options
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(DFTD4_USE_EIGEN "Use Eigen instead of BLAS/LAPACK" OFF)
option(DFTD4_USE_EIGEN "Use Eigen instead of BLAS/LAPACK" OFF)
option(SANITIZER "Sanitizer type" "none")

# Set C++ standard using target properties instead of global variables
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Create source file list
set(SOURCES
  src/dftd_cutoff.cpp
  src/dftd_damping.cpp
  src/dftd_dispersion.cpp
  src/dftd_eeq.cpp
  src/dftd_model.cpp
  src/dftd_ncoord.cpp
  src/damping/dftd_atm.cpp
  src/damping/dftd_rational.cpp
)

# Create header file list
set(HEADERS
  include/dftd_cutoff.h
  include/dftd_damping.h
  include/dftd_dispersion.h
  include/dftd_eeq.h
  include/dftd_geometry.h
  include/dftd_matrix.h
  include/dftd_model.h
  include/dftd_ncoord.h
  include/dftd_parameters.h
  include/damping/dftd_atm.h
  include/damping/dftd_rational.h
)

if(DFTD4_USE_EIGEN)
  if(NOT TARGET Eigen3::Eigen)
    find_package(Eigen3 3.3 REQUIRED)
    find_package(Eigen3 REQUIRED NO_MODULE)
    if(NOT EIGEN3_FOUND)
      message(FATAL_ERROR "Eigen3 library not found")
    endif()
  endif()
  message(STATUS "Using Eigen ${EIGEN3_VERSION}")
else()
  # Find BLAS and LAPACK
  find_package(BLAS REQUIRED)
  find_package(LAPACK REQUIRED)
  message(STATUS "Using BLAS/LAPACK")
endif()

# Create shared library
add_library(${PROJECT_NAME} ${SOURCES})
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

# Create static library if shared libs are enabled
if(BUILD_SHARED_LIBS)
  add_library(${PROJECT_NAME}_static STATIC ${SOURCES})
  set_target_properties(${PROJECT_NAME}_static PROPERTIES 
    OUTPUT_NAME ${PROJECT_NAME}
    POSITION_INDEPENDENT_CODE ON
  )
  add_library(${PROJECT_NAME}::static ALIAS ${PROJECT_NAME}_static)
endif()

# Set library version properties
set_target_properties(${PROJECT_NAME} PROPERTIES
  VERSION ${PROJECT_VERSION}
  SOVERSION ${PROJECT_VERSION_MAJOR}
  POSITION_INDEPENDENT_CODE ON
)

# Use target_include_directories instead of include_directories
target_include_directories(${PROJECT_NAME}
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

if(BUILD_SHARED_LIBS)
  target_include_directories(${PROJECT_NAME}_static
    PUBLIC
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
      $<INSTALL_INTERFACE:include>
  )
endif()

# Set compiler flags using target_compile_options
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  target_compile_options(${PROJECT_NAME} PRIVATE 
    -Wall -Wextra -Wpedantic
  )
  
  if(BUILD_SHARED_LIBS)
    target_compile_options(${PROJECT_NAME}_static PRIVATE 
      -Wall -Wextra -Wpedantic
    )
  endif()
endif()

# Debug build specifics using target_compile_options
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(${PROJECT_NAME} PRIVATE
      -Wzero-as-null-pointer-constant
      -Wlogical-not-parentheses
      -fno-omit-frame-pointer
    )
    
    if(BUILD_SHARED_LIBS)
      target_compile_options(${PROJECT_NAME}_static PRIVATE
        -Wzero-as-null-pointer-constant
        -Wlogical-not-parentheses
        -fno-omit-frame-pointer
      )
    endif()
    
    # Sanitizer options
    if(SANITIZER STREQUAL "asan")
      message(STATUS "Enabling ASan + UBSan + LSan")
      target_compile_options(${PROJECT_NAME} PRIVATE 
        -fsanitize=address,undefined -fsanitize-address-use-after-scope
      )
      target_link_options(${PROJECT_NAME} PRIVATE 
        -fsanitize=address,undefined -fsanitize-address-use-after-scope -fno-omit-frame-pointer
      )
      
      if(BUILD_SHARED_LIBS)
        target_compile_options(${PROJECT_NAME}_static PRIVATE 
          -fsanitize=address,undefined -fsanitize-address-use-after-scope
        )
        target_link_options(${PROJECT_NAME}_static PRIVATE 
          -fsanitize=address,undefined -fsanitize-address-use-after-scope -fno-omit-frame-pointer
        )
      endif()
      
    elseif(SANITIZER STREQUAL "msan")
      message(STATUS "Enabling Memory Sanitizer (MSan)")
      target_compile_options(${PROJECT_NAME} PRIVATE -fsanitize=memory)
      target_link_options(${PROJECT_NAME} PRIVATE -fsanitize=memory)
      
      if(BUILD_SHARED_LIBS)
        target_compile_options(${PROJECT_NAME}_static PRIVATE -fsanitize=memory)
        target_link_options(${PROJECT_NAME}_static PRIVATE -fsanitize=memory)
      endif()
      
    elseif(SANITIZER STREQUAL "tsan")
      message(STATUS "Enabling Thread Sanitizer (TSan)")
      target_compile_options(${PROJECT_NAME} PRIVATE -fsanitize=thread)
      target_link_options(${PROJECT_NAME} PRIVATE -fsanitize=thread)
      
      if(BUILD_SHARED_LIBS)
        target_compile_options(${PROJECT_NAME}_static PRIVATE -fsanitize=thread)
        target_link_options(${PROJECT_NAME}_static PRIVATE -fsanitize=thread)
      endif()
      
    else()
      message(STATUS "No sanitizers enabled")
    endif()
  endif()
endif()

# Link dependencies using target_link_libraries
if(DFTD4_USE_EIGEN)
  target_compile_definitions(${PROJECT_NAME} PUBLIC DFTD4_USE_EIGEN)
  target_link_libraries(${PROJECT_NAME} PUBLIC Eigen3::Eigen)
  
  if(BUILD_SHARED_LIBS)
    target_compile_definitions(${PROJECT_NAME}_static PUBLIC DFTD4_USE_EIGEN)
    target_link_libraries(${PROJECT_NAME}_static PUBLIC Eigen3::Eigen)
  endif()
else()
  target_link_libraries(${PROJECT_NAME} PUBLIC ${BLAS_LIBRARIES} ${LAPACK_LIBRARIES})
  
  if(BUILD_SHARED_LIBS)
    target_link_libraries(${PROJECT_NAME}_static PUBLIC ${BLAS_LIBRARIES} ${LAPACK_LIBRARIES})
  endif()
endif()

# Create executable
add_executable(${PROJECT_NAME}_app 
  app/main.cpp
  app/readxyz.cpp
)

set_target_properties(${PROJECT_NAME}_app PROPERTIES 
  OUTPUT_NAME ${PROJECT_NAME}
)

target_link_libraries(${PROJECT_NAME}_app PRIVATE ${PROJECT_NAME})

# Install targets using modern install commands
include(GNUInstallDirs)

install(TARGETS ${PROJECT_NAME} ${PROJECT_NAME}_app
  EXPORT ${PROJECT_NAME}Targets
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

if(BUILD_SHARED_LIBS)
  install(TARGETS ${PROJECT_NAME}_static
    EXPORT ${PROJECT_NAME}Targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )
endif()

# Install headers
install(FILES ${HEADERS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME})

# Install export target
install(EXPORT ${PROJECT_NAME}Targets
  FILE ${PROJECT_NAME}Targets.cmake
  NAMESPACE ${PROJECT_NAME}::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)
