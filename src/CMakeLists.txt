FIND_PACKAGE(OpenGL)

find_package(ASSIMP QUIET)
if(NOT ASSIMP_FOUND)
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(ASSIMP assimp)
endif()
if(NOT ASSIMP_FOUND)
  message(WARNING "Assimp not found, not building synthetic views")
endif()
if(ASSIMP_FOUND AND NOT ${ASSIMP_VERSION} VERSION_LESS "3.0.0")
  message(STATUS "Found assimp v3 ${ASSIMP_LIBRARY} ${ASSIMP_INCLUDE_DIRS}")
  set(EXTRA_SOURCES)
  link_directories(${ASSIMP_LIBRARY_DIRS})
  set(EXTRA_LIBRARIES ${ASSIMP_LIBRARIES})
else()
  message(STATUS "Building assimp v3")
  set(ASSIMP_INCLUDE_DIRS ./assimp/include)
  aux_source_directory(./assimp/contrib/clipper EXTRA_SOURCES_clipper)
  aux_source_directory(./assimp/contrib/ConvertUTF EXTRA_SOURCES_ConvertUTF)
  aux_source_directory(./assimp/contrib/irrXML EXTRA_SOURCES_irrXML)
  aux_source_directory(./assimp/contrib/poly2tri/poly2tri/common EXTRA_SOURCES_poly2tri)
  aux_source_directory(./assimp/contrib/poly2tri/poly2tri/sweep EXTRA_SOURCES_poly2tri_sweep)
  aux_source_directory(./assimp/contrib/unzip EXTRA_SOURCES_unzip)
  aux_source_directory(./assimp/contrib/zlib EXTRA_SOURCES_zlib)
  aux_source_directory(./assimp/code EXTRA_SOURCES)
  set(EXTRA_SOURCES ${EXTRA_SOURCES} ${EXTRA_SOURCES_clipper} ${EXTRA_SOURCES_ConvertUTF} ${EXTRA_SOURCES_irrXML} ${EXTRA_SOURCES_poly2tri} ${EXTRA_SOURCES_poly2tri_sweep} ${EXTRA_SOURCES_unzip} ${EXTRA_SOURCES_zlib})
  set(EXTRA_LIBRARIES)
endif()

# create the 2d rendering library
add_library(${PROJECT_NAME}_2d renderer2d.cpp)
target_link_libraries(${PROJECT_NAME}_2d ${OpenCV_LIBRARIES})
if(catkin_FOUND)
install(TARGETS ${PROJECT_NAME}_2d
          DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
)
endif()

# create the 3d rendering library
set(SOURCES model.cpp renderer3d.cpp utils.cpp)

include_directories(BEFORE ${ASSIMP_INCLUDE_DIRS})

find_package(OpenCV REQUIRED)

INCLUDE_DIRECTORIES(SYSTEM ${CMAKE_CURRENT_SOURCE_DIR}
                           ${OpenCV_INCLUDE_DIR}
                           ${OPENGL_INCLUDE_DIR}
)

if (${USE_GLUT})
  add_definitions(-DUSE_RENDERER_GLUT=1)

  find_package(GLUT)
  include_directories(SYSTEM ${GLUT_INCLUDE_DIR})

  set(EXTRA_SOURCES ${EXTRA_SOURCES} renderer3d_impl_glut.cpp)
  set(EXTRA_LIBRARIES ${EXTRA_LIBRARIES} ${OPENGL_LIBRARIES} ${GLUT_LIBRARIES})
else()
  set(EXTRA_SOURCES ${EXTRA_SOURCES} renderer3d_impl_osmesa.cpp)
  set(EXTRA_LIBRARIES ${EXTRA_LIBRARIES} OSMesa GLU)
endif()

add_library(${PROJECT_NAME}_3d ${SOURCES} ${EXTRA_SOURCES})

target_link_libraries(${PROJECT_NAME}_3d ${EXTRA_LIBRARIES}
                                         ${OpenCV_LIBRARIES}
                                         freeimage
)

if(catkin_FOUND)
install(TARGETS ${PROJECT_NAME}_3d
        DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
)
endif()

# add an executable to render views to a file
ADD_EXECUTABLE(view_generator main.cpp)

SET_PROPERTY(TARGET view_generator PROPERTY DEBUG_POSTFIX ${ASSIMP_DEBUG_POSTFIX})


target_link_libraries(view_generator ${EXTRA_LIBRARIES} ${PROJECT_NAME}_3d ${PROJECT_NAME}_2d )

SET_TARGET_PROPERTIES(view_generator PROPERTIES
                                     OUTPUT_NAME view_generator
)

if(catkin_FOUND)
install(TARGETS view_generator
        DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
endif()

# add some executables
add_subdirectory(apps)
