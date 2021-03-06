project(TuriCreate)

# Fix the current source directory of the framework stuff so the macro below
# references everything correctly.
set(__CAPI_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR} CACHE INTERNAL "")

set(_turi_capi_source_files
  "${__CAPI_SOURCE_DIR}/impl/capi_error_handling.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_flexible_type.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_datetime.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_flex_dict.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_flex_list.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_flex_image.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_ndarray.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_functions.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_sarray.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_parameters.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_sframe.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_models.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_functions.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_initialization.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_variant.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_flex_enum_list.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_wrapper_structs.cpp"
  "${__CAPI_SOURCE_DIR}/impl/capi_memory_management.cpp")

# Create a static version of the library that also bundles in the server initializer
make_library(TuriCreate_static
  SOURCES
    ${_turi_capi_source_files}
    default_server_initializer.cpp
  REQUIRES
    unity unity_core numerics)

# A macro that allows building a custom C-API library.
# 
#  The main variable here is the initializer source file and whether or not to 
#  build it as a framework. 
#
#  To build it with a custom server initialization function, which allows for 
#  specifying which models are availalable for calling through the C-API, call 
#
#  make_capi_library(MyTuriExecutor INITIALIZER_SOURCE_FILE my_turi_model_init.cpp REQUIRES unity)
#
#  See default_server_initializer.cpp as a template for this. 
#
macro(make_capi_library NAME)

  set(options FRAMEWORK)
  set(one_value_args INITIALIZER_SOURCE_FILE FRAMEWORK_IDENTIFIER REQUIRES)

  CMAKE_PARSE_ARGUMENTS(__capi "${options}" "${one_value_args}" "" ${ARGN})

  if(NOT __capi_INITIALIZER_SOURCE_FILE)
    set(__capi_INITIALIZER_SOURCE_FILE "${__CAPI_SOURCE_DIR}/default_server_initializer.cpp")
  endif()

  if(${__capi_FRAMEWORK})

    make_library(${NAME}
      SOURCES
      ${_turi_capi_source_files}
      ${__capi_INITIALIZER_SOURCE_FILE}
       REQUIRES
       unity unity_core numerics
       ${__capi_REQUIRES}
      SHARED_ALL_DEFINED
      SHARED
      EXPORT_LINUX_MAP_FILE "${__CAPI_SOURCE_DIR}/framework/tc_capi_exports_linux.ver"
      EXPORT_OSX_MAP_FILE "${__CAPI_SOURCE_DIR}/framework/tc_capi_exports_osx.ver"
      DEAD_STRIP
    )

    set_target_properties(TuriCreate PROPERTIES
      FRAMEWORK TRUE
      FRAMEWORK_VERSION A
      MACOSX_FRAMEWORK_IDENTIFIER "${__capi_FRAMEWORK_IDENTIFIER}"
      MACOSX_FRAMEWORK_INFO_PLIST "${__CAPI_SOURCE_DIR}/framework/Info.plist"
      PUBLIC_HEADER TuriCreate.h
      MACOSX_RPATH TRUE
    )

    add_custom_command(TARGET TuriCreate
    POST_BUILD
      COMMAND "${__CAPI_SOURCE_DIR}/framework/process_framework.sh"
      --name ${NAME}
      --src-headers "${__CAPI_SOURCE_DIR}"
      --framework "${CMAKE_CURRENT_BINARY_DIR}/${NAME}.framework"
      --install-location "@rpath"
      --create-tbd-file
    BYPRODUCTS
      "${CMAKE_CURRENT_BINARY_DIR}/${NAME}.framework/Versions/Current/Header/TuriCreate.h"
      "${CMAKE_CURRENT_BINARY_DIR}/${NAME}.framework/Versions/Current/TuriCreate.tbd"
    )

    add_dependencies(${NAME} "${__CAPI_SOURCE_DIR}/framework/process_framework.sh")
    add_dependencies(${NAME} "${__CAPI_SOURCE_DIR}/framework/Info.plist")
    add_dependencies(${NAME} "${__CAPI_SOURCE_DIR}/framework/tc_capi_exports_osx.ver")
    add_dependencies(${NAME} "${__CAPI_SOURCE_DIR}/framework/tc_capi_exports_linux.ver")

else()
  #build out as a regular library
    make_library(${NAME}
  SOURCES 
        ${__capi_INITIALIZER_SOURCE_FILE}
        ${_turi_capi_source_files}
   REQUIRES
         ${__capi_REQUIRES}
         unity unity_core numerics
  SHARED_ALL_DEFINED
  SHARED
      EXPORT_LINUX_MAP_FILE "${__CAPI_SOURCE_DIR}/framework/tc_capi_exports_linux.ver"
      EXPORT_OSX_MAP_FILE "${__CAPI_SOURCE_DIR}/framework/tc_capi_exports_osx.ver"
  DEAD_STRIP
)

endif()

endmacro()

if(${TC_BUILD_CAPI_FRAMEWORK})
  make_capi_library(TuriCreate FRAMEWORK)
else()
  make_capi_library(TuriCreate)
endif()

