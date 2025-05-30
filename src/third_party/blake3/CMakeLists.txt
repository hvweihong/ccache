register_dependency(Blake3 BUNDLED 1.8.2)

add_library(
  dep_blake3 STATIC
  "${CMAKE_CURRENT_SOURCE_DIR}/blake3/blake3.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/blake3/blake3_dispatch.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/blake3/blake3_portable.c"
)

target_include_directories(dep_blake3 INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/blake3")
target_link_libraries(dep_blake3 PRIVATE standard_settings)

if(MSVC)
  # No object file is created if masm is passed the compile options from
  # standard_settings, so don't pass any flags at all to assembler (as no
  # flags are needed anyway).
  string(REPLACE "<FLAGS> " "" CMAKE_ASM_MASM_COMPILE_OBJECT "${CMAKE_ASM_MASM_COMPILE_OBJECT}")
endif()

include(CheckCSourceCompiles)

function(_add_blake3_source_if_enabled feature msvc_flags others_flags intrinsic)
  if(MSVC)
    set(compile_flags "${msvc_flags}")
  else()
    set(compile_flags "${others_flags}")
  endif()

  if(MSVC)
    set(suffix "_x86-64_windows_msvc.asm")
  elseif(WIN32 OR CYGWIN)
    set(suffix "_x86-64_windows_gnu.S")
  else()
    set(suffix "_x86-64_unix.S")
  endif()

  # First check if it's possible to use the assembler variant for the feature.
  string(TOUPPER "have_asm_${feature}" have_feature)
  if(NOT DEFINED "${have_feature}" AND CMAKE_SIZEOF_VOID_P EQUAL 8
      # Force intrinsic version for msbuild because of a bug in the CMake
      # generator or msbuild itself with masm flags.
      AND NOT CMAKE_GENERATOR MATCHES "Visual Studio")

    if(NOT CMAKE_REQUIRED_QUIET)
      message(STATUS "Performing Test ${have_feature}")
    endif()

    set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

    # Must set CMAKE_ASM_MASM_CREATE_STATIC_LIBRARY explicitly otherwise
    # try_compile fails, see
    # <https://discourse.cmake.org/t/building-lib-file-from-asm-cmake-bug/1959>.
    try_compile(
      ${have_feature}
      ${CMAKE_CURRENT_BINARY_DIR}
      "${CMAKE_CURRENT_SOURCE_DIR}/blake3/blake3_${feature}${suffix}"
      CMAKE_FLAGS -DCMAKE_ASM_MASM_CREATE_STATIC_LIBRARY=${CMAKE_C_CREATE_STATIC_LIBRARY}
      COMPILE_DEFINITIONS ${compile_flags}
    )

    unset(CMAKE_TRY_COMPILE_TARGET_TYPE)

    if(NOT CMAKE_REQUIRED_QUIET)
      if (${${have_feature}})
        message(STATUS "Performing Test ${have_feature} - Success")
      else()
        message(STATUS "Performing Test ${have_feature} - Failed")
      endif()
    endif()
  endif()

  # If the assembler variant didn't work, try the C variant.
  if(NOT ${have_feature})
    string(TOUPPER "have_c_${feature}" have_feature)
    set(suffix ".c")

    set(CMAKE_REQUIRED_FLAGS ${compile_flags})
    check_c_source_compiles(
      [=[
        #include <immintrin.h>
        int main() { ${intrinsic}; return 0; }
      ]=]
      ${have_feature})
    unset(CMAKE_REQUIRED_FLAGS)
  endif()

  if(${have_feature})
    target_sources(
      dep_blake3
      PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/blake3/blake3_${feature}${suffix}"
    )
    if(suffix STREQUAL ".c")
      if(MINGW AND feature STREQUAL "avx512")
        # Workaround for <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=65782>.
        # Taken from blake3's build.rs.
        set(compile_flags "${compile_flags} -fno-asynchronous-unwind-tables")
      endif()
      set_property(
        SOURCE "${CMAKE_CURRENT_SOURCE_DIR}/blake3/blake3_${feature}${suffix}"
        APPEND PROPERTY COMPILE_FLAGS ${compile_flags})
    elseif(NOT MSVC)
      set_property(
        SOURCE "${CMAKE_CURRENT_SOURCE_DIR}/blake3/blake3_${feature}${suffix}"
        PROPERTY COMPILE_FLAGS ${compile_flags})
    endif()
  else()
    string(TOUPPER "blake3_no_${feature}" no_feature)
    target_compile_definitions(dep_blake3 PRIVATE ${no_feature})
  endif()
endfunction()

# https://software.intel.com/sites/landingpage/IntrinsicsGuide/
_add_blake3_source_if_enabled(sse2 "" "-msse2" "_mm_set1_epi32(42)")
_add_blake3_source_if_enabled(sse41 "" "-msse4.1" "_mm_test_all_ones(_mm_set1_epi32(42))")
_add_blake3_source_if_enabled(avx2 "/arch:AVX2" "-mavx2" "_mm256_abs_epi8(_mm256_set1_epi32(42))")
_add_blake3_source_if_enabled(avx512 "/arch:AVX512" "-mavx512f -mavx512vl" "_mm256_abs_epi64(_mm256_set1_epi32(42))")

# Neon is always available on AArch64
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  # https://developer.arm.com/architectures/instruction-sets/simd-isas/neon/intrinsics
  check_c_source_compiles(
    [=[
      #include <arm_neon.h>
      int main() { vdupq_n_s32(42); return 0; }
    ]=]
    HAVE_NEON)
  if(HAVE_NEON)
    target_sources(dep_blake3 PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/blake3/blake3_neon.c")
    target_compile_definitions(dep_blake3 PRIVATE BLAKE3_USE_NEON)
  endif()
endif()

file(GLOB headers *.h)
target_sources(dep_blake3 PRIVATE ${headers})
