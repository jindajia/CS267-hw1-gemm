cmake_minimum_required(VERSION 3.14)
project(hw1 LANGUAGES C CXX)

find_package(BLAS REQUIRED)

# Run the full set of tests
option(ALL_SIZES "Test an extended set of sizes; will take more CPU time." OFF)

# On Bridges2 -- 3.4 GHz * 4 vector width * 2 vector pipelines * 2 flops for FMA = 54.4 GF/s
set(MAX_SPEED 54 CACHE STRING "The max speed of the CPU in GF/s")

# List of BLOCK_SIZE values
set(BLOCK_SIZE_LIST 5 7 10 15 20 25 27 30 33 35 40 45 50)

# Group number
set(GROUP_NO 00 CACHE STRING "Your group number (two digits)")

# Packaging system
if (NOT ${GROUP_NO} STREQUAL 00)
    set(CPACK_GENERATOR TGZ)
    set(CPACK_PACKAGE_FILE_NAME "cs267Group${GROUP_NO}_hw1")
    install(FILES dgemm-blocked.c DESTINATION .)
    install(FILES ${CPACK_PACKAGE_FILE_NAME}.pdf DESTINATION .)
    include(CPack)
endif ()

# We require the GNU compiler for this assignment.
if (NOT "${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")
    set(Prg-Intel "PrgEnv-intel")
    set(Prg-Clang "PrgEnv-cray")
    message(WARNING
            "Must use GNU Compiler for submission. Make sure you ran:\n"
            "module swap ${Prg-${CMAKE_C_COMPILER_ID}} PrgEnv-gnu")
endif ()

# Common library target for benchmarking.
add_library(benchmark OBJECT benchmark.cpp)
target_compile_definitions(benchmark PRIVATE MAX_SPEED=${MAX_SPEED} $<$<BOOL:${ALL_SIZES}>:ALL_SIZES>)
target_compile_features(benchmark PRIVATE cxx_std_11)

foreach (BENCHMARK IN ITEMS naive blas)
    # Create benchmarking executable
    add_executable(benchmark-${BENCHMARK} dgemm-${BENCHMARK}.c)
    target_link_libraries(benchmark-${BENCHMARK} PRIVATE benchmark ${BLAS_LIBRARIES})
    target_compile_features(benchmark-${BENCHMARK} PRIVATE c_std_11 c_restrict)
    target_compile_options(benchmark-${BENCHMARK} PRIVATE -Wall -pedantic -march=znver2)

    # Generate job script
    configure_file(job.in job-${BENCHMARK})
endforeach ()

foreach (BLOCK_SIZE IN LISTS BLOCK_SIZE_LIST)
    # Create a new dgemm-blocked.c file for each BLOCK_SIZE value
    configure_file(dgemm-blocked.c.in dgemm-blocked-${BLOCK_SIZE}.c @ONLY)

    # Create benchmarking executable
    add_executable(benchmark-blocked-${BLOCK_SIZE} dgemm-blocked-${BLOCK_SIZE}.c)
    target_link_libraries(benchmark-blocked-${BLOCK_SIZE} PRIVATE benchmark ${BLAS_LIBRARIES})
    target_compile_features(benchmark-blocked-${BLOCK_SIZE} PRIVATE c_std_11 c_restrict)
    target_compile_options(benchmark-blocked-${BLOCK_SIZE} PRIVATE -Wall -pedantic -march=znver2)
endforeach ()

# Generate blocked job script
configure_file(job-blocked.in job-blocked)

foreach(item ${BLOCK_SIZE_LIST})
    file(APPEND build/job-blocked "singularity run --bind \$HOME/CS267-hw1-gemm/build:/mnt gcc_cmake_openblas_1.0.sif /mnt/benchmark-blocked-${item}\n")
endforeach()

file(APPEND build/job-blocked "\n\n#sbatch -o ../output/output-blocked.txt -p RM-shared -t 00:25:00 --ntasks-per-node=1 job-blocked
")
