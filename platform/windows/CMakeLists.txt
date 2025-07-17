file(RELATIVE_PATH CURRENT_RELATIVE_PATH "${CMAKE_SOURCE_DIR}" "${CMAKE_CURRENT_SOURCE_DIR}")

set(PLATFORM_DOC_CLASSES "EditorExportPlatformWindows")
set(PLATFORM_DOC_CLASS_PATH "${CURRENT_RELATIVE_PATH}/doc_classes")

foreach(DOC_CLASS ${PLATFORM_DOC_CLASSES})
	list(APPEND DOC_CLASS_PATHS "${DOC_CLASS},${PLATFORM_DOC_CLASS_PATH}")
endforeach()

set(DOC_CLASS_PATHS ${DOC_CLASS_PATHS} PARENT_SCOPE)

if(REX_PLATFORM STREQUAL "windows")
	set(TARGET_WIN_VERSION "0x0601" CACHE STRING "Targeted Windows version, >= 0x0601 (Windows 7)")
	set(WINDOWS_SUBSYSTEM "gui" CACHE STRING "Windows subsystem: gui, console")
	option(USE_MINGW "Use the Mingw compiler, even if MSVC is installed." OFF)
	option(USE_LLVM "Use the LLVM compiler" OFF)
	option(USE_STATIC_CPP "Link MinGW/MSVC C++ runtime libraries statically" ON)
	option(USE_ASAN "Use address sanitizer (ASAN)" OFF)
	option(USE_UBSAN "Use LLVM compiler undefined behavior sanitizer (UBSAN)" OFF)
	option(DEBUG_CRT "Compile with MSVC's debug CRT (/MDd)" OFF)
	option(INCREMENTAL_LINK "Use MSVC incremental linking. May increase or decrease build times." OFF)
	option(SILENCE_MSVC "Silence MSVC's cl/link stdout bloat, redirecting any errors to stderr." ON)
	set(ANGLE_LIBS "" CACHE STRING "Path to the ANGLE static libraries")
	if (MSVC)
		set(WINDOWS_CRASH_HANDLER ${CMAKE_CURRENT_SOURCE_DIR}/crash_handler_windows_seh.cpp)
	else()
		set(WINDOWS_CRASH_HANDLER ${CMAKE_CURRENT_SOURCE_DIR}/crash_handler_windows_signal.cpp)
	endif()

	add_executable(rex_engine
		${CMAKE_CURRENT_SOURCE_DIR}/godot_windows.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/os_windows.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/display_server_windows.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/key_mapping_windows.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/joypad_windows.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/tts_windows.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/windows_terminal_logger.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/windows_utils.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/native_menu_windows.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/gl_manager_windows_native.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/gl_manager_windows_angle.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/wgl_detect_version.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/rendering_context_driver_vulkan_windows.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/drop_target_windows.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/godot_res.rc
		${WINDOWS_CRASH_HANDLER}
	)
	if (MSVC)
		if (DEBUG_CRT)
			target_compile_options(rex_engine PUBLIC /MDd)
		else()
			if (USE_STATIC_CPP)
				target_compile_options(rex_engine PUBLIC /MT)
			else()
				target_compile_options(rex_engine PUBLIC /MD)
			endif()
		endif()

		if (NOT INCREMENTAL_LINK)
			target_link_options(rex_engine PUBLIC /INCREMENTAL:NO)
		endif()

		target_compile_options(rex_engine PUBLIC
			/fp:strict
			/Gd
			/GR
			/nologo
			/utf-8
			/bigobj
		)

		if (ACCESSKIT)
			# Convert TARGET_WIN_VERSION to integer for comparison
			string(SUBSTRING "${TARGET_WIN_VERSION}" 2 -1 HEX_VERSION)
			math(EXPR DEC_VERSION "0${HEX_VERSION}" OUTPUT_FORMAT DECIMAL)
			if(DEC_VERSION LESS 602)  # 0x0602 = 602 in decimal
				message(STATUS "AccessKit enabled, targeted Windows version changed to Windows 8 (0x0602).")
				set(TARGET_WIN_VERSION "0x0602" CACHE STRING "Target Windows version" FORCE)
			endif()
		endif()

		target_compile_definitions(rex_engine PUBLIC
            WINDOWS_ENABLED
            WASAPI_ENABLED
            WINMIDI_ENABLED
            TYPED_METHOD_BIND
            WIN32
			WINVER=${TARGET_WIN_VERSION}
            _WIN32_WINNT=${TARGET_WIN_VERSION}
			NOMINMAX
		)
		if (ARCH STREQUAL "x86_64")
			target_compile_definitions(rex_engine PUBLIC
				_WIN64
			)
		endif()

		set(PREBUILT_LIB_EXTRA_SUFFIX "")
		if (USE_ASAN)
			set(CMAKE_EXECUTABLE_SUFFIX ".san${CMAKE_EXECUTABLE_SUFFIX}")
			set(PREBUILT_LIB_EXTRA_SUFFIX ".san")
			target_compile_options(rex_engine PUBLIC /fsanitize=address)
			target_compile_definitions(rex_engine PUBLIC SANITIZERS_ENABLED)
			target_link_options(rex_engine PUBLIC /INFERASANLIBS)
		endif()

		set(SYSTEM_LIBRARIES
			winmm
			dsound
			kernel32
			ole32
			oleaut32
			sapi
			user32
			gdi32
			IPHLPAPI
			Shlwapi
			wsock32
			Ws2_32
			shell32
			advapi32
			dinput8
			dxguid
			imm32
			bcrypt
			Crypt32
			Avrt
			dwmapi
			dwrite
			wbemuuid
			ntdll
		)

		if (DEBUG_FEATURES)
			list(APPEND SYSTEM_LIBRARIES
				psapi
				dbghelp
			)
		endif()

		if (WINDOWS_SUBSYSTEM STREQUAL "gui")
			add_executable(rex_console
				${CMAKE_CURRENT_SOURCE_DIR}/console_wrapper_windows.cpp
				${CMAKE_CURRENT_SOURCE_DIR}/godot_res_wrap.rc
			)
			clone_library(rex_engine rex_console)
			disable_warnings_for_target(rex_console)

			target_link_libraries(rex_console PRIVATE ${SYSTEM_LIBRARIES})

			target_link_options(rex_console PRIVATE
				/SUBSYSTEM:CONSOLE
				version.lib
			)
			set_target_properties(rex_console PROPERTIES
				OUTPUT_NAME "console.${OUTPUT_NAME}"
				RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin"
			)
			add_dependencies(rex_console rex_engine)
		endif()

		if (ACCESSKIT)
			# TODO
		endif()

		if (VULKAN)
			target_compile_definitions(rex_engine PUBLIC
				VULKAN_ENABLED
				RD_ENABLED
			)
			if (NOT USE_VOLK)
				list(APPEND SYSTEM_LIBRARIES vulkan)
			endif()
		endif()

		if (D3D12)
			# TODO
		endif()

		if (OPENGL3)
			target_compile_definitions(rex_engine PUBLIC
				GLES3_ENABLED
			)
			if (NOT ANGLE_LIBS STREQUAL "")
				target_compile_definitions(rex_engine PUBLIC
					EGL_STATIC
				)
				set(APPEND SYSTEM_LIBRARIES
					${ANGLE_LIBS}libANGLE.windows.${ARCH}${PREBUILT_LIB_EXTRA_SUFFIX}
					${ANGLE_LIBS}libEGL.windows.${ARCH}${PREBUILT_LIB_EXTRA_SUFFIX}
					${ANGLE_LIBS}libGLES.windows.${ARCH}${PREBUILT_LIB_EXTRA_SUFFIX}
					dxgi
					d3d9
					d3d11
				)
			endif()
			target_include_directories(rex_engine PUBLIC
				${CMAKE_SOURCE_DIR}/thirdparty/angle/include/
			)
		endif()


	endif()

	set(SYSTEM_LIBRARIES ${SYSTEM_LIBRARIES} PARENT_SCOPE)
endif()
