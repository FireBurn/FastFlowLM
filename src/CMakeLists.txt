cmake_minimum_required(VERSION 3.22)
project(flm LANGUAGES CXX VERSION 0.1.0)

# Set build type to Release by default
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

# Set C++ standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Configure CMake to stop at first error
set(CMAKE_ERROR_ON_ABSOLUTE_INSTALL_DESTINATION ON)
set(CMAKE_ERROR_DEPRECATED ON)

# Set output directories
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/out/)

# ———————————————————————————————————————————————
# User-tweakable settings (override on cmake command-line)
# !!! IMPORTANT !!! Please update these paths for your Linux environment
# ———————————————————————————————————————————————
if(WIN32)
  set(XRT_INCLUDE_DIR C:/dev/XRT/src/runtime_src/core/include CACHE PATH "Where XRT headers live")
  set(XRT_LIB_DIR     C:/dev/xrtNPUfromDLL     CACHE PATH "Where XRT libs live")
  set(BOOST_ROOT      C:/dev/boost_1_88_0      CACHE PATH "Path to Boost root")
  set(VCPKG_ROOT      C:/dev/vcpkg/installed/x64-windows CACHE PATH "Path to vcpkg installed root")
else()
  # --- EDIT THESE PATHS FOR YOUR LINUX SYSTEM ---
  set(XRT_INCLUDE_DIR /usr/include CACHE PATH "Where XRT headers live")
  set(XRT_LIB_DIR     /usr/lib64     CACHE PATH "Where XRT libs live")
  set(BOOST_ROOT      /usr/include/boost             CACHE PATH "Path to Boost root")
  set(VCPKG_ROOT      /path/to/vcpkg/installed/x64-linux CACHE PATH "Path to vcpkg installed root")
endif()


# ———————————————————————————————————————————————
# Gather your sources
# ———————————————————————————————————————————————
file(GLOB SOURCES "src/*.cpp" "runner/*.cpp" "common/*.cpp" "common/*/*.cpp" "server/*.cpp" "pull/*.cpp" )
file(GLOB HEADERS "include/*.hpp" "runner/*.hpp" "common/*.hpp" "common/*/*.hpp" "server/*.hpp" "pull/*.hpp")

add_executable(flm ${SOURCES} ${HEADERS})

# ———————————————————————————————————————————————
# Library specific settings
# ———————————————————————————————————————————————
target_include_directories(flm PUBLIC
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/runner
    ${CMAKE_SOURCE_DIR}/server
    ${CMAKE_SOURCE_DIR}/pull
    ${XRT_INCLUDE_DIR}
    ${BOOST_ROOT}
    ${VCPKG_ROOT}/include
)

target_link_directories(flm PUBLIC
    ${BOOST_ROOT}/stage/lib
    ${XRT_LIB_DIR}
    ${CMAKE_SOURCE_DIR}/lib
    ${VCPKG_ROOT}/lib
)

# Common compile definitions
target_compile_definitions(flm PUBLIC
    DISABLE_ABI_CHECK=1
    CURL_STATICLIB
    BOOST_ALL_NO_LIB
    BOOST_ALL_STATIC_LINK
)

# Platform-specific compile definitions
if(WIN32)
    target_compile_definitions(flm PUBLIC
        WIN32_LEAN_AND_MEAN
        NOMINMAX
        _CRT_SECURE_NO_WARNINGS
        _CRT_NONSTDC_NO_DEPRECATE
        _IS_WINDOWS_
    )
else() # UNIX
    target_compile_definitions(flm PUBLIC _IS_LINUX_)
endif()


# ———————————————————————————————————————————————
# Link Libraries
# ———————————————————————————————————————————————
# Common libraries (names might differ below)
set(COMMON_LIBS
    xrt_coreutil q4_npu_eXpress llama_npu qwen_npu gemma_npu gemma_text_npu
    gpt_oss_npu whisper_npu qwen3vl_npu gemma_embedding dequant gemm lm_head npu_utils
    avformat avcodec avutil swscale swresample
    tokenizers_cpp tokenizers_c sentencepiece
)

target_link_libraries(flm PRIVATE ${COMMON_LIBS})

# Platform-specific libraries
if(WIN32)
    target_link_libraries(flm PRIVATE
        libcurl
        libboost_program_options-vc143-mt-x64-1_88 # Example for MSVC 14.3, Boost 1.88
        libfftw3-3
        libfftw3f-3
        libfftw3l-3
        ntdll wsock32 ws2_32 Bcrypt iphlpapi userenv psapi
    )
else() # UNIX
    target_link_libraries(flm PRIVATE
        curl
        boost_program_options
        fftw3
        fftw3f
        fftw3l
        pthread dl # Common Linux libraries
    )
endif()

# ———————————————————————————————————————————————
# Add compiler-specific flags for performance / warning fixes
# ———————————————————————————————————————————————
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(flm PRIVATE -mavx -mavx2 -mavx512f -Wno-psabi)
endif()

# ———————————————————————————————————————————————
# Post-Build Steps
# ———————————————————————————————————————————————
if(WIN32)
    set(OUTPUT_EXE_NAME flm.exe)
else()
    set(OUTPUT_EXE_NAME flm)
endif()

# Copy the built executable into the out directory
add_custom_command(TARGET flm POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy
        $<TARGET_FILE:flm>
        ${CMAKE_SOURCE_DIR}/out/${OUTPUT_EXE_NAME}
)

# Add a custom target to check for remaining library dependencies
add_custom_target(check_dependencies ALL DEPENDS flm)
if(WIN32)
    add_custom_command(TARGET check_dependencies POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E echo "Checking for DLL dependencies..."
        COMMAND dumpbin /dependents $<TARGET_FILE:flm> | findstr /i ".dll"
        COMMENT "--> Checking for remaining DLL dependencies"
    )
else() # UNIX
    add_custom_command(TARGET check_dependencies POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E echo "Checking for shared library dependencies..."
        COMMAND ldd $<TARGET_FILE:flm>
        COMMENT "--> Checking for remaining shared library dependencies"
    )
endif()
