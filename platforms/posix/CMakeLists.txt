
add_subdirectory(src)

include_directories(${CMAKE_CURRENT_BINARY_DIR})

get_property(module_libraries GLOBAL PROPERTY PX4_MODULE_LIBRARIES)

px4_posix_generate_builtin_commands(
	OUT apps
	MODULE_LIST ${module_libraries}
)

px4_add_executable(px4
	src/main.cpp
	apps.cpp
	)

target_link_libraries(px4
	PRIVATE
		${module_libraries}
		df_driver_framework
		${df_driver_libs}
		pthread m
)

if (NOT APPLE)
	target_link_libraries(px4 PRIVATE rt)
endif()

target_link_libraries(px4 PRIVATE modules__uORB)

if ("${BOARD}" STREQUAL "rpi")
	add_custom_target(upload
		COMMAND scp -r $<TARGET_FILE:px4> ${PX4_SOURCE_DIR}/posix-configs/rpi/*.config ${PX4_SOURCE_DIR}/ROMFS pi@navio:/home/pi
		DEPENDS px4
		COMMENT "uploading px4"
		USES_TERMINAL
		)

elseif ("${BOARD}" STREQUAL "bebop")

	add_custom_target(upload
		COMMAND ${CMAKE_STRIP} -R .comment -R .gnu.version -o $<TARGET_FILE:px4>.stripped $<TARGET_FILE:px4>
		COMMAND ${PX4_SOURCE_DIR}/Tools/adb_upload_to_bebop.sh $<TARGET_FILE:px4>.stripped /usr/bin
		DEPENDS px4
		COMMENT "uploading px4"
		USES_TERMINAL
		)

elseif ("${BOARD}" STREQUAL "sitl")

	include(sitl_target)
	if(BUILD_TESTING)
		include(sitl_tests)
	endif()

elseif (("${BOARD}" STREQUAL "eagle") OR ("${BOARD}" STREQUAL "excelsior"))

	set(APPS_TARGET_PATH /home/linaro)
	add_custom_target(upload
		COMMAND adb wait-for-device
		COMMAND adb push $<TARGET_FILE:px4> ${APPS_TARGET_PATH}
		COMMAND adb push ${PX4_SOURCE_DIR}/posix-configs/eagle/flight/mainapp.config ${APPS_TARGET_PATH}
		COMMAND adb shell sync
		DEPENDS px4
		COMMENT "uploading px4"
		USES_TERMINAL
	)
endif()

#=============================================================================
# install
#

install(TARGETS px4 DESTINATION ${CMAKE_INSTALL_BINDIR})
install(DIRECTORY ${PROJECT_SOURCE_DIR}/ROMFS DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME})
install(DIRECTORY ${PROJECT_SOURCE_DIR}/posix-configs DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME})
