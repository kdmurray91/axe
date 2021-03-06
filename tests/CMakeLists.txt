INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/tinytest)

ADD_EXECUTABLE(test_axe test.c ${CMAKE_CURRENT_SOURCE_DIR}/tinytest/tinytest.c test_libaxe.c)
TARGET_LINK_LIBRARIES(test_axe ${AXE_DEPENDS_LIBRARIES} axelib qes_static)

# Copy test files over to bin dir & make output
ADD_CUSTOM_TARGET(setup_tests ALL
	COMMAND ${CMAKE_COMMAND} -E copy_directory
		   ${CMAKE_CURRENT_SOURCE_DIR}/data
                   ${CMAKE_BINARY_DIR}/data
	COMMAND ${CMAKE_COMMAND} -E copy
                   ${CMAKE_CURRENT_SOURCE_DIR}/axe_cli_tests.py
                   ${CMAKE_BINARY_DIR}/bin
	COMMAND ${CMAKE_COMMAND} -E make_directory
                   ${CMAKE_BINARY_DIR}/out)
ADD_DEPENDENCIES(test_axe setup_tests)

ADD_TEST(NAME "UnitTests" COMMAND test_axe)
SET(COVERAGE_CMD test_axe)
SET(COVERAGE_OUT "${CMAKE_BINARY_DIR}/coverage_html")

ADD_TEST(NAME "IntegrationTests" COMMAND python
         ${CMAKE_BINARY_DIR}/bin/axe_cli_tests.py
         ${CMAKE_BINARY_DIR})

IF (CMAKE_BUILD_TYPE STREQUAL "Coverage")
	FIND_PROGRAM( GCOV_PATH gcov )
	FIND_PROGRAM( LCOV_PATH lcov )
	FIND_PROGRAM( GENHTML_PATH genhtml )
	FIND_PROGRAM( GCOVR_PATH gcovr PATHS ${CMAKE_SOURCE_DIR}/tests)

	SET(CMAKE_C_FLAGS_COVERAGE
	    "${CMAKE_C_FLAGS_DEBUG} -g -O0 --coverage -fprofile-arcs -ftest-coverage"
	    CACHE STRING "Flags used by the C compiler during coverage builds."
	    FORCE)
	SET(CMAKE_EXE_LINKER_FLAGS_COVERAGE
	    ""
	    CACHE STRING "Flags used for linking binaries during coverage builds."
	    FORCE)
	SET(CMAKE_SHARED_LINKER_FLAGS_COVERAGE
	    ""
	    CACHE STRING "Flags used by the shared libraries linker during coverage builds."
	    FORCE)
	MARK_AS_ADVANCED(
	    CMAKE_C_FLAGS_COVERAGE
	    CMAKE_EXE_LINKER_FLAGS_COVERAGE
	    CMAKE_SHARED_LINKER_FLAGS_COVERAGE )

	IF(NOT GCOV_PATH)
		MESSAGE(FATAL_ERROR "gcov not found! Aborting...")
	ENDIF() # NOT GCOV_PATH
	IF(NOT LCOV_PATH)
		MESSAGE(FATAL_ERROR "lcov not found! Aborting...")
	ENDIF() # NOT LCOV_PATH
	IF(NOT GENHTML_PATH)
		MESSAGE(FATAL_ERROR "genhtml not found! Aborting...")
	ENDIF() # NOT GENHTML_PATH
	IF(NOT CMAKE_COMPILER_IS_GNUCC AND NOT CMAKE_COMPILER_IS_GNUCXX)
		# Clang version 3.0.0 and greater now supports gcov as well.
		MESSAGE(WARNING "Compiler is not GNU gcc! Clang Version 3.0.0 and greater supports gcov as well, but older versions don't.")
		IF(NOT "${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" AND NOT "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
			MESSAGE(FATAL_ERROR "Compiler is not GNU gcc! Aborting...")
		ENDIF()
	ENDIF() # NOT CMAKE_COMPILER_IS_GNUCXX

	# Setup target
	ADD_CUSTOM_TARGET(coverage
		# Cleanup lcov
		COMMAND ${LCOV_PATH} --rc lcov_branch_coverage=1 --directory  ${CMAKE_BINARY_DIR}/src --zerocounters

		# Run tests
		COMMAND ${COVERAGE_CMD}

		# Capturing lcov counters and generating report
		COMMAND ${LCOV_PATH} --rc lcov_branch_coverage=1 --directory ${CMAKE_BINARY_DIR}/src --capture --output-file ${COVERAGE_OUT}.info
		COMMAND ${LCOV_PATH} --rc lcov_branch_coverage=1 --output-file ${COVERAGE_OUT}.info --remove ${COVERAGE_OUT}.info src/libqes/*
		COMMAND ${LCOV_PATH} --rc lcov_branch_coverage=1 --output-file ${COVERAGE_OUT}.info --remove ${COVERAGE_OUT}.info src/datrie/*
		COMMAND ${GENHTML_PATH} --branch-coverage -o ${COVERAGE_OUT} ${COVERAGE_OUT}.info
		COMMAND ${CMAKE_COMMAND} -E remove ${COVERAGE_OUT}.info

		WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
		COMMENT "Resetting code coverage counters to zero.\nProcessing code coverage counters and generating report."
	)

	# Show info where to find the report
	ADD_CUSTOM_COMMAND(TARGET coverage POST_BUILD
		COMMAND ;
		COMMENT "Open ${COVERAGE_OUT}/index.html in your browser to view the coverage report."
	)

ENDIF() # build type coverage
