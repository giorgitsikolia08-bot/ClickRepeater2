cmake_minimum_required(VERSION 3.22)
project(ClickRepeater LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(NOT ANDROID)
    message(FATAL_ERROR "ClickRepeater builds only with the Android NDK toolchain (arm64-v8a).")
endif()
if(NOT (ANDROID_ABI STREQUAL "arm64-v8a" OR CMAKE_ANDROID_ARCH_ABI STREQUAL "arm64-v8a"))
    message(FATAL_ERROR "ClickRepeater supports only arm64-v8a (got: ${ANDROID_ABI}${CMAKE_ANDROID_ARCH_ABI})")
endif()

# ---- package / manifest settings ----
set(MOD_ID "click_repeater" CACHE STRING "Mod id (package directory name)")
set(MOD_NAME "Click Repeater" CACHE STRING "Display name")
set(MOD_AUTHOR "ClickRepeater" CACHE STRING "Author")
set(MOD_VERSION "0.1.0" CACHE STRING "Version")
set(MOD_LIBRARY_NAME "click_repeater" CACHE STRING "Native library name without lib prefix / .so suffix")
set(MOD_MINECRAFT_VERSIONS "[\"1.26.52.*\"]" CACHE STRING "Raw JSON array for manifest minecraft_versions")
set(LEVI_PACKAGE_FILE_NAME "ClickRepeater.levipack" CACHE STRING "Output package file name")
# ON = files inside a <MOD_ID>/ folder in the zip (same layout as the working Armor HUD package).
option(LEVI_PACKAGE_WRAP_IN_DIR "Put files inside a <MOD_ID>/ folder in the zip" ON)

# ---- SDK headers only (the launcher provides the preloader at runtime) ----
include(FetchContent)
FetchContent_Declare(preloader_android
    GIT_REPOSITORY https://github.com/LiteLDev/preloader-android.git
    GIT_TAG 0.2.2)
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 11.2.0)
set(FMT_DOC OFF CACHE BOOL "" FORCE)
set(FMT_TEST OFF CACHE BOOL "" FORCE)
set(FMT_INSTALL OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(fmt)

# preloader-android 0.2.2 is used for its include/ folder only: populate without add_subdirectory,
# otherwise a second copy of the preloader would be linked into the mod.
FetchContent_GetProperties(preloader_android)
if(NOT preloader_android_POPULATED)
    FetchContent_Populate(preloader_android)
endif()
set(LEVI_PRELOADER_INCLUDE_DIR "${preloader_android_SOURCE_DIR}/include")

add_library(${MOD_LIBRARY_NAME} SHARED src/ModEntry.cpp)
set_target_properties(${MOD_LIBRARY_NAME} PROPERTIES OUTPUT_NAME "${MOD_LIBRARY_NAME}")
target_include_directories(${MOD_LIBRARY_NAME} PRIVATE "${LEVI_PRELOADER_INCLUDE_DIR}")
target_compile_options(${MOD_LIBRARY_NAME} PRIVATE
    $<$<COMPILE_LANG_AND_ID:CXX,Clang,GNU>:-fvisibility=hidden -fvisibility-inlines-hidden>)
# 16 KiB page alignment; SDK symbols (pl::input, pl::modmenu, ...) are resolved by the launcher at load time.
target_link_options(${MOD_LIBRARY_NAME} PRIVATE "-Wl,-z,max-page-size=16384" "-Wl,--unresolved-symbols=ignore-all")

find_library(log-lib log REQUIRED)
target_link_libraries(${MOD_LIBRARY_NAME} PRIVATE ${log-lib} fmt::fmt-header-only)

if(NOT CMAKE_STRIP)
    message(FATAL_ERROR "CMAKE_STRIP is required to strip the generated library.")
endif()
add_custom_command(TARGET ${MOD_LIBRARY_NAME} POST_BUILD
    COMMAND "${CMAKE_STRIP}" --strip-unneeded "$<TARGET_FILE:${MOD_LIBRARY_NAME}>"
    COMMENT "Stripping symbols" VERBATIM)

# ---- .levipack packaging (a zip: manifest.json + lib<name>.so) ----
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/manifest.json.in" "${CMAKE_CURRENT_BINARY_DIR}/manifest.json" @ONLY)

set(PKG_ROOT  "${CMAKE_CURRENT_BINARY_DIR}/package")
set(PKG_STAGE "${PKG_ROOT}/${MOD_ID}")
set(PKG_FILE  "${CMAKE_CURRENT_BINARY_DIR}/${LEVI_PACKAGE_FILE_NAME}")

if(LEVI_PACKAGE_WRAP_IN_DIR)
    set(ZIP_CWD "${PKG_ROOT}")
    set(ZIP_ITEMS "${MOD_ID}")
else()
    set(ZIP_CWD "${PKG_STAGE}")
    set(ZIP_ITEMS manifest.json lib${MOD_LIBRARY_NAME}.so)
endif()

add_custom_target(levi_package
    COMMAND "${CMAKE_COMMAND}" -E rm -rf "${PKG_ROOT}"
    COMMAND "${CMAKE_COMMAND}" -E make_directory "${PKG_STAGE}"
    COMMAND "${CMAKE_COMMAND}" -E copy "$<TARGET_FILE:${MOD_LIBRARY_NAME}>" "${PKG_STAGE}/lib${MOD_LIBRARY_NAME}.so"
    COMMAND "${CMAKE_COMMAND}" -E copy "${CMAKE_CURRENT_BINARY_DIR}/manifest.json" "${PKG_STAGE}/manifest.json"
    COMMAND "${CMAKE_COMMAND}" -E rm -f "${PKG_FILE}"
    COMMAND "${CMAKE_COMMAND}" -E chdir "${ZIP_CWD}" "${CMAKE_COMMAND}" -E tar cf "${PKG_FILE}" --format=zip ${ZIP_ITEMS}
    DEPENDS ${MOD_LIBRARY_NAME}
    COMMENT "Packaging ${LEVI_PACKAGE_FILE_NAME}"
    VERBATIM)
