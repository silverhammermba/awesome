# This module is designed to allow the Awesome documentation examples to be
# tested.
#
# It shims enough of the Awesome C API to allow code to be executed without an
# actual X server or running Awesome process. These tests are not genuine
# integration tests, but they are the next best thing.
#
# As secondary goals, this module also generates images of the test result
# where relevant. Those images are used by the documentation and help the
# developers track user interface regressions and glitches. Finally, it also
# helps to find broken code.
cmake_minimum_required(VERSION 3.0.0)

# Get and update the LUA_PATH so the scripts can be executed without Awesome.
execute_process(COMMAND lua -e print\(package.path\) OUTPUT_VARIABLE "LUA_PATH_")

# Add the main awesome lua libraries.
set(LUA_PATH2_ "\
${CMAKE_SOURCE_DIR}/lib/?.lua;\
${CMAKE_SOURCE_DIR}/lib/?/init.lua;\
${CMAKE_SOURCE_DIR}/lib/?;"
)

# Add the C API shims.
set(LUA_PATH3_ "\
${CMAKE_SOURCE_DIR}/tests/examples/shims/?.lua;\
${CMAKE_SOURCE_DIR}/tests/examples/shims/?/init.lua;\
${CMAKE_SOURCE_DIR}/tests/examples/shims/?;"
)

# Done in 3 variables to avoid CMake from implicitly converting into a list.
set(ENV{LUA_PATH} "${LUA_PATH3_}${LUA_PATH2_}${LUA_PATH_}")

# Unset environment variables that would get in the way of evaluating LUA_PATH.
unset(ENV{LUA_PATH_5_1})
unset(ENV{LUA_PATH_5_2})
unset(ENV{LUA_PATH_5_3})

# The documentation images directory.
set(IMAGE_DIR "${CMAKE_BINARY_DIR}/doc/images")
file(MAKE_DIRECTORY "${IMAGE_DIR}")

# Escape potentially multiline strings to be part of the API doc.
#  * add "--" in front of each lines
#  * add a custom prefix in front of each lines
#  * drop empty lines
#  * convert " " lines into empty lines
#  * drop lines ending with "--DOC_SOMETHING", they are handled elsewhere
function(escape_string variable content escaped_content line_prefix)
    # If DOC_HIDE_ALL is present, do nothing.
    if(variable MATCHES "--DOC_HIDE_ALL")
        return()
    endif()
    string(REGEX REPLACE "\n" ";" var_lines "${variable}")

    set(tmp_output ${content})
    foreach (LINE ${var_lines})
        if(NOT LINE MATCHES "^.+--DOC_[A-Z]+$")
            set(tmp_output ${tmp_output}\n--${line_prefix}${LINE})
        endif()
    endforeach()

    set(${escaped_content} ${tmp_output} PARENT_SCOPE)
endfunction()

# Extract lines with the --DOC_HEADER marker.
function(extract_header variable pre_output post_output)
    string(REGEX REPLACE "\n" ";" var_lines "${variable}")

    set(IS_POST 0)

    foreach (LINE ${var_lines})
        # This function doesn't escape the lines, so make sure they are
        # already comments.
        if(LINE MATCHES "^--.*--DOC_HEADER")
            # Remove the header tag
            string(REGEX REPLACE "[ ]*--DOC_HEADER" "" LINE "${LINE}")

            # ldoc is picky about what happens after the first --@, so split
            # the output on that.
            if (NOT IS_POST AND LINE MATCHES "^--[ ]*@")
                set(IS_POST 1)
            endif()

            if (IS_POST)
                set(tmp_post_output ${tmp_post_output}\n${line_prefix}${LINE})
            else()
                set(tmp_pre_output ${tmp_pre_output}\n${line_prefix}${LINE})
            endif()
        endif()
    endforeach()

    set(${post_output} "${tmp_post_output}\n--" PARENT_SCOPE)
    set(${pre_output} "${tmp_pre_output}\n--" PARENT_SCOPE)
endfunction()

# Read a code file and convert it to ldoc usage example.
#  * add "--" in front of each lines
#  * drop empty lines
#  * convert " " lines into empty lines
#  * drop lines ending with "--DOC_HIDE"
function(escape_code path escaped_content pre_header post_header)
    file(READ ${path} path)

    escape_string("${path}\n" "" escaped_code "")

    extract_header("${path}" example_pre_header example_post_header)

    set(${escaped_content} ${escaped_code} PARENT_SCOPE)
    set(${pre_header} ${example_pre_header} PARENT_SCOPE)
    set(${post_header} ${example_post_header} PARENT_SCOPE)
endfunction()

# Find the template.lua that is closest to the given file. For example, if a
# template.lua is present in the same directory, its path will be returned. If
# one is present in the parent directory, that path is returned etc.
function(find_template result_variable file)
    get_filename_component(path "${file}" DIRECTORY)

    while(NOT EXISTS "${path}/template.lua")
        set(last_path "${path}")
        get_filename_component(path "${path}" DIRECTORY)
        if(last_path STREQUAL path)
            message(FATAL_ERROR "Failed to find template.lua for ${file}")
        endif()
    endwhile()

    set(${result_variable} "${path}/template.lua" PARENT_SCOPE)
endfunction()

# Get the namespace of a file.
function(get_namespace result_variable file)
    get_filename_component(path "${file}" DIRECTORY)
    string(LENGTH "${SOURCE_DIR}/tests/examples" prefix_length)
    string(REPLACE "/" "_" namespace "${path}")
    string(SUBSTRING "${namespace}" "${prefix_length}" -1 namespace)

    set(${result_variable} "${namespace}" PARENT_SCOPE)
endfunction()

# Execute a Lua file.
function(run_test test_path namespace escaped_content)
    find_template(template "${test_path}")

    # Get the file name without the extension.
    get_filename_component(${test_path} TEST_FILE_NAME NAME)
    set(IMAGE_PATH "${IMAGE_DIR}/AUTOGEN${namespace}_${TEST_FILE_NAME}")

    # Use the example testing as coverage source.
    if (USE_LCOV)
        set(LCOV_PATH ${SOURCE_DIR}/.luacov)
    endif()

    # Execute the script, leave the image extension decision to the test.
    # SVG is preferred, but PNG is better suited for some tests, like bitmap
    # patterns.
    execute_process(
        COMMAND lua ${template} ${test_path} ${IMAGE_PATH} ${LCOV_PATH}
        OUTPUT_VARIABLE TEST_OUTPUT
        ERROR_VARIABLE  TEST_ERROR
    )

    # If there is something on stderr, exit.
    if (NOT TEST_ERROR STREQUAL "")
        message("${TEST_OUTPUT}")
        message("${TEST_ERROR}")
        if (STRICT_TESTS)
            message(FATAL_ERROR ${test_path} " An example test failed, aborting.")
        endif()
    endif()

    # Read the code and turn it into an usage example.
    escape_code(${test_path} TEST_CODE TEST_PRE_HEADER TEST_POST_HEADER)

    # Build the documentation.
    set(TEST_DOC_CONTENT "${TEST_PRE_HEADER}")

    # If the image has been created, then add it.
    if(EXISTS "${IMAGE_PATH}.svg")
        escape_string(
            "![Usage example](../images/AUTOGEN${namespace}_${TEST_FILE_NAME}.svg)\n"
            "${TEST_DOC_CONTENT}" TEST_DOC_CONTENT ""
        )
    elseif(EXISTS "${IMAGE_PATH}.png")
        escape_string(
            "![Usage example](../images/AUTOGEN${namespace}_${TEST_FILE_NAME}.png)\n"
            "${TEST_DOC_CONTENT}" TEST_DOC_CONTENT ""
        )
    endif()

    # If there is an output, assume it is relevant and add it to the
    # documentation under the image.
    if(NOT ${TEST_OUTPUT} STREQUAL "")
        set(TEST_DOC_CONTENT
            "${TEST_DOC_CONTENT}\n--\n--**Usage example output**:\n--"
        )

        # Markdown requires an empty line before and after, and 4 spaces.
        escape_string(
            "\n${TEST_OUTPUT}"
            "${TEST_DOC_CONTENT}" TEST_DOC_CONTENT "    "
        )
        set(TEST_DOC_CONTENT "${TEST_DOC_CONTENT}\n--")
    endif()

    # If there is some @* content, append it.
    set(TEST_DOC_CONTENT "${TEST_DOC_CONTENT}${TEST_POST_HEADER}")

    # Only add it if there is something to display.
    if(NOT ${TEST_CODE} STREQUAL "\n--")
        # Do not use the @usage tag, but 4 spaces.
        file(READ ${test_path} tmp_content)
        if(NOT tmp_content MATCHES "--DOC_NO_USAGE")
            set(DOC_PREFIX "@usage")
        endif()

        escape_string(
            " ${DOC_PREFIX}"
            "${TEST_DOC_CONTENT}" TEST_DOC_CONTENT ""
        )

        set(TEST_DOC_CONTENT "${TEST_DOC_CONTENT}${TEST_CODE}")
    endif()

    # Export the outout to the parent scope.
    set(${escaped_content} "${TEST_DOC_CONTENT}" PARENT_SCOPE)

endfunction()

# Find and run all test files.
file(GLOB_RECURSE test_files LIST_DIRECTORIES false
    "${SOURCE_DIR}/tests/examples/*.lua")
foreach(file ${test_files})
    if ((NOT "${file}" MATCHES ".*/shims/.*")
        AND (NOT "${file}" MATCHES ".*/template.lua"))

        file(RELATIVE_PATH relative_file "${SOURCE_DIR}" "${file}")
        message(STATUS "Running ${relative_file}...")

        # Get the file name without the extension.
        get_filename_component(TEST_FILE_NAME ${file} NAME_WE)

        get_namespace(namespace "${file}")

        run_test("${file}" "${namespace}" ESCAPED_CODE_EXAMPLE)

        # Set the test name.
        set(TEST_NAME DOC${namespace}_${TEST_FILE_NAME}_EXAMPLE)

        # Anything called @DOC_`namespace`_EXAMPLE@
        # in the Lua or C sources will be replaced by the content of that
        # variable during the pre-processing.
        # While at it, replace \" created by CMake by ',
        # &quot; wont work in <code>.
        string(REPLACE "\"" "'" ${TEST_NAME} ${ESCAPED_CODE_EXAMPLE})
    endif()
endforeach()

message(STATUS "Example tests passed.")
