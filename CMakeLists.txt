cmake_minimum_required (VERSION 3.0)
project (cef C CXX)


MACRO(DEFINE_CEF_CHECKSUM_SHA1_PACKAGE OS VERNO HASH)
    set(CEF_CHECKSUM_SHA1_PACKAGE_${OS}_${VERNO} ${HASH})
ENDMACRO()

if(NOT CEF_VERSION)
    set(CEF_VERSION "3.3282.1732.g973997f")
endif()
DEFINE_CEF_CHECKSUM_SHA1_PACKAGE(LINUX64    "3.3282.1732.g973997f"  "91d61dc41ad5b0e6fb08bdc319a8234d57438fd1")

IF(CMAKE_SIZEOF_VOID_P MATCHES "8")
    set(_CEF_OS "LINUX64")
    set(_CEF_OS_PKG "linux64")
ELSE()
    set(_CEF_OS "LINUX32")
    set(_CEF_OS_PKG "linux32")
ENDIF()

set(CEF_DOWNLOAD_URL_BASE "http://opensource.spotify.com/cefbuilds")
set(CEF_PACKAGE_EXT ".tar.bz2")
set(CEF_PACKAGE_BASE "cef_binary_${CEF_VERSION}_${_CEF_OS_PKG}")
SET(CEF_DOWNLOAD_URL "${CEF_DOWNLOAD_URL_BASE}/${CEF_PACKAGE_BASE}${CEF_PACKAGE_EXT}" CACHE STRING "Override download URL for ${_CEF_OS}" FORCE)
SET(CEF_CHECKSUM_SHA1_PACKAGE "${CEF_CHECKSUM_SHA1_PACKAGE_${_CEF_OS}_${CEF_VERSION}}" CACHE STRING "Override checksum for download package for ${_CEF_OS}" FORCE)

SET(CEF_DIR "${CMAKE_CURRENT_BINARY_DIR}/pkg" CACHE PATH "Directory of the CEF package")
SET(CEF_WRAPPER_SRC_DIR "${CEF_DIR}/libcef_dll" CACHE PATH "Directory of the CEF wrapper static library sources")

FUNCTION(CEF_DOWNLOAD_PACKAGE)
	SET(_download_required 0)
	#SET(_download_required 1)
	SET(_extract_required 0)
	get_filename_component(_cef_package_file "${CEF_DOWNLOAD_URL}" NAME)
	# need to use string(REPLACE) instead of get_filename_component(NAME_WE) because it would
	# also remove the version number from the filename, since it removes the longest extension.
	string(REPLACE ${CEF_PACKAGE_EXT} "" _cef_package_file_we "${_cef_package_file}")

	SET(_dest "${CMAKE_CURRENT_BINARY_DIR}/${_cef_package_file}")
	SET(_local_source_file "/tmp/${_cef_package_file}")
	set(_local_pkg_dir "/tmp/${CEF_PACKAGE_BASE}")
	IF(NOT EXISTS "${_dest}" OR _download_required)
        if(EXISTS ${_local_pkg_dir} AND 0)
            message(STATUS "Copy CEF package dir from ${_local_pkg_dir} to ${CEF_DIR}")
            file(COPY ${_local_pkg_dir} DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")
            message(STATUS "rename ${CMAKE_CURRENT_BINARY_DIR}/${CEF_PACKAGE_BASE} ${CEF_DIR}")
            file(RENAME "${CMAKE_CURRENT_BINARY_DIR}/${CEF_PACKAGE_BASE}" "${CEF_DIR}")
            SET(_extract_required 0)
		elseif(EXISTS ${_local_source_file})
            message(STATUS "Copy CEF from ${_local_source_file} to ${_dest}")
            file(COPY ${_local_source_file} DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")
            SET(_extract_required 1)
        else()
            message(STATUS "Download CEF ${CEF_DOWNLOAD_URL} to ${_dest}")
            FILE(DOWNLOAD "${CEF_DOWNLOAD_URL}" "${_dest}" TLS_VERIFY OFF
                    EXPECTED_HASH SHA1=${CEF_CHECKSUM_SHA1_PACKAGE}
                    STATUS _download_status
                    SHOW_PROGRESS)
            LIST(GET _download_status 0 _download_status_code)
            SET(_extract_required 1)
            IF(NOT _download_status_code)
                MESSAGE(STATUS "CEF package ${_cef_package_file} (SHA1=${CEF_CHECKSUM_SHA1_PACKAGE}) downloaded")
            ELSE()
                MESSAGE(FATAL_ERROR "Failed to download CEF package ${CEF_DOWNLOAD_URL}(SHA1=${CEF_CHECKSUM_SHA1_PACKAGE}) to ${_dest}")
            ENDIF()
        endif()
    endif()

	IF(NOT EXISTS "${CEF_DIR}")
        MESSAGE(STATUS "CEF directory ${CEF_DIR} does not exist")
		SET(_extract_required 1)
    ELSE()
        if(EXISTS ${CEF_DIR}/.cef_version)
            file(READ ${CEF_DIR}/.cef_version _current_cef_version)
        else()
            set(_current_cef_version "unknown")
        endif()
        if(_current_cef_version STREQUAL ${CEF_VERSION})
            MESSAGE(STATUS "Current CEF \"${_current_cef_version}\" matches package version \"${CEF_VERSION}\", no update required.")
        else()
            MESSAGE(STATUS "Current CEF \"${_current_cef_version}\" does not match package version \"${CEF_VERSION}\" and must be updated.")
            SET(_extract_required 1)
        endif()
	ENDIF()

	IF(_extract_required)
        MESSAGE(STATUS "Extract CEF package ${_cef_package_file} to ${CEF_DIR}...")
		file(REMOVE_RECURSE "${CEF_DIR}")
		file(REMOVE_RECURSE "${CMAKE_CURRENT_BINARY_DIR}/tmp")
		IF(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/tmp")
			FILE(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/tmp")
		endif()
		if(EXISTS "${CEF_DIR}")
			MESSAGE(FATAL_ERROR "Unable to remove directory ${CEF_DIR}. Please remove directory manually and restart CMake.")
		endif()
        execute_process(COMMAND "tar" "xfj" "${_dest}" "-C" "${CMAKE_CURRENT_BINARY_DIR}/tmp"
                        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/tmp"
                        RESULT_VARIABLE _cef_extract_result
                        OUTPUT_VARIABLE _cef_extract_output
                        ERROR_VARIABLE _cef_extract_error
                        )
		IF(NOT _cef_extract_result)
			# exit code is zero, so everything is fine
			IF(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/tmp/${_cef_package_file_we}")
				MESSAGE(FATAL_ERROR "Downloaded package from ${CEF_DOWNLOAD_URL} does not contain a ${_cef_package_file_we} directory")
			ELSE()
				FILE(RENAME "${CMAKE_CURRENT_BINARY_DIR}/tmp/${_cef_package_file_we}" "${CEF_DIR}")
				MESSAGE(STATUS "CEF package ${_cef_package_file} has been download and extracted to ${CEF_DIR}")
				file(REMOVE_RECURSE "${CMAKE_CURRENT_BINARY_DIR}/tmp")
                file(WRITE ${CEF_DIR}/.cef_version ${CEF_VERSION})
			ENDIF()
		ELSE()
			MESSAGE(WARNING "Failed to download ${_cef_package_file} and extract CEF package ${_dest}. Error ${_cef_extract_error}, ${_cef_extract_output}")
		ENDIF()
	ENDIF()
ENDFUNCTION(CEF_DOWNLOAD_PACKAGE)


FUNCTION(CEF_PREPARE_SDK)


    IF(NOT EXISTS ${CEF_DIR}/Debug/libcefd.so AND EXISTS ${CEF_DIR}/Debug/libcef.so)
        FILE(RENAME ${CEF_DIR}/Debug/libcef.so ${CEF_DIR}/Debug/libcefd.so)
        execute_process(COMMAND "patchelf" --set-soname libcefd.so ${CEF_DIR}/Debug/libcefd.so
                        WORKING_DIRECTORY ${CEF_DIR}/Debug
                        RESULT_VARIABLE _patchelf_result
                        OUTPUT_VARIABLE _patchelf_output
                        )
    ENDIF()

	SET(_cef_lib_dir ${CMAKE_CURRENT_BINARY_DIR}/lib)

    IF(NOT EXISTS ${_cef_lib_dir})
        FILE(MAKE_DIRECTORY "${_cef_lib_dir}")
    ENDIF()

    IF(EXISTS ${CEF_DIR}/tests)
        FILE(REMOVE_RECURSE ${CEF_DIR}/tests)
    ENDIF()

	SET(_cef_bin_files)

    execute_process(COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CEF_DIR}/Release/libcef.so ${_cef_lib_dir}/libcef.so)
    execute_process(COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CEF_DIR}/Debug/libcefd.so ${_cef_lib_dir}/libcefd.so)

ENDFUNCTION(CEF_PREPARE_SDK)

FUNCTION(CEF_BUILD_WRAPPER)
    set(CEF_ROOT ${CEF_DIR})
    add_subdirectory(${CEF_DIR})
ENDFUNCTION(CEF_BUILD_WRAPPER)

FUNCTION(CEF_INSTALL_FILES)
    SET(_cef_lib_dir ${CMAKE_CURRENT_BINARY_DIR}/lib)
    set(_cef_res_dir ${CEF_DIR}/Resources)
    set(_cef_rel_dir ${CEF_DIR}/Release)
    set(_cef_dbg_dir ${CEF_DIR}/Debug)
    set(_cef_src_dir ${CEF_DIR}/libcef_dll)
    set(_cef_cmake_dir ${CEF_DIR}/cmake)

    install(DIRECTORY ${CEF_DIR}/include DESTINATION .)
    install(FILES ${_cef_lib_dir}/libcef.so ${_cef_lib_dir}/libcefd.so DESTINATION lib)

    install(FILES
        ${_cef_rel_dir}/libEGL.so
        ${_cef_rel_dir}/libGLESv2.so
        ${_cef_rel_dir}/libwidevinecdmadapter.so
        # Only required for setuid sandboxes
        # See https://code.google.com/p/chromium/wiki/LinuxSandboxing#The_setuid_sandbox
        ${_cef_rel_dir}/chrome-sandbox
        ${_cef_rel_dir}/natives_blob.bin
        ${_cef_rel_dir}/snapshot_blob.bin
        ${_cef_rel_dir}/v8_context_snapshot.bin
        DESTINATION lib/cef.rel
        )
    install(FILES
        ${_cef_rel_dir}/swiftshader/libEGL.so
        ${_cef_rel_dir}/swiftshader/libGLESv2.so
        DESTINATION lib/cef.rel/swiftshader
        )

    install(FILES
        ${_cef_dbg_dir}/libEGL.so
        ${_cef_dbg_dir}/libGLESv2.so
        ${_cef_dbg_dir}/libwidevinecdmadapter.so
        # Only required for setuid sandboxes
        # See https://code.google.com/p/chromium/wiki/LinuxSandboxing#The_setuid_sandbox
        ${_cef_dbg_dir}/chrome-sandbox
        ${_cef_dbg_dir}/natives_blob.bin
        ${_cef_dbg_dir}/snapshot_blob.bin
        ${_cef_dbg_dir}/v8_context_snapshot.bin
        DESTINATION lib/cef.dbg
        )
    install(FILES
        ${_cef_dbg_dir}/swiftshader/libEGL.so
        ${_cef_dbg_dir}/swiftshader/libGLESv2.so
        DESTINATION lib/cef.dbg/swiftshader
        )

    install(FILES
                ${_cef_res_dir}/cef.pak
                ${_cef_res_dir}/cef_100_percent.pak
                ${_cef_res_dir}/cef_200_percent.pak
                ${_cef_res_dir}/cef_extensions.pak
                ${_cef_res_dir}/devtools_resources.pak
            DESTINATION
                share/cef/Resources
            )
    install(DIRECTORY
                ${_cef_res_dir}/locales
            DESTINATION
                share/cef/Resources
            )

    install(FILES
                ${_cef_res_dir}/icudtl.dat
            DESTINATION
                lib/cef.rel
        )
    install(FILES
                ${_cef_res_dir}/icudtl.dat
            DESTINATION
                lib/cef.dbg
        )
    install(DIRECTORY
                ${_cef_src_dir}
            DESTINATION
                src/cef
            )
    install(DIRECTORY
                ${_cef_cmake_dir}
            DESTINATION
                src/cef
            )
    install(FILES
                ${CEF_DIR}/CMakeLists.txt
            DESTINATION
                src/cef
            )
    get_filename_component(_obj_dir_name "${CMAKE_BINARY_DIR}" NAME)
    install(FILES
                ${CMAKE_CURRENT_BINARY_DIR}/${_obj_dir_name}/pkg/libcef_dll_wrapper/libcef_dll_wrapper.a
            DESTINATION
                lib
        )
ENDFUNCTION(CEF_INSTALL_FILES)

MESSAGE(STATUS "CEF_DOWNLOAD_URL=${CEF_DOWNLOAD_URL}")

CEF_DOWNLOAD_PACKAGE()
CEF_PREPARE_SDK()
CEF_BUILD_WRAPPER()
CEF_INSTALL_FILES()
