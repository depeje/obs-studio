project(linux-v4l2)

if(DISABLE_V4L2)
	message(STATUS "v4l2 plugin disabled")
	return()
endif()


if(ENABLE_V4L2)
	find_package(Libv4l2 REQUIRED)
else()
	find_package(Libv4l2)
	if(NOT LIBV4L2_FOUND)
		message(STATUS "libv4l2 not found, disabling v4l2 plugin")
		return()
	endif()
endif()

if(DISABLE_UDEV)
	add_definitions(-DHAVE_UDEV)
else()
	find_package(UDev)
	if(NOT UDEV_FOUND)
		message(STATUS "udev disabled for v4l2 plugin")
	else()
		set(linux-v4l2-udev_SOURCES
			v4l2-udev.c
		)
	endif()
endif()

find_package(ImageMagick QUIET COMPONENTS MagickCore)

if(NOT ImageMagick_MagickCore_FOUND AND NOT FFMPEG_AVCODEC_FOUND)
	message(FATAL_ERROR "Either MagickCore or Libavcodec is required, but neither were found.")
elseif(NOT ImageMagick_MagickCore_FOUND AND LIBOBS_PREFER_IMAGEMAGICK)
	message(FATAL_ERROR "ImageMagick support was requested, but was not found.")
endif()


if(NOT FFMPEG_AVCODEC_FOUND OR (ImageMagick_MagickCore_FOUND AND LIBOBS_PREFER_IMAGEMAGICK))
	message(STATUS "Using ImageMagick for v4l2 MJPEG loading")

	if(${ImageMagick_VERSION_STRING} LESS 7)
		add_definitions(-DLIBOBS_IMAGEMAGICK_DIR_STYLE_6L)
	elseif(${ImageMagick_VERSION_STRING} GREATER_EQUAL 7)
		add_definitions(-DLIBOBS_IMAGEMAGICK_DIR_STYLE_7GE)
	endif()

	set(v4l2_mjpeg_loading_SOURCES
		v4l2-decompress-magick.c)
	set(v4l2_mjpeg_loading_LIBRARIES
		${ImageMagick_LIBRARIES})

	include_directories(${ImageMagick_INCLUDE_DIRS})
else()
	message(STATUS "Using libavcodec for v4l2 MJPEG loading")

	set(v4l2_mjpeg_loading_SOURCES
		v4l2-decompress-ffmpeg.c)
	set(v4l2_mjpeg_loading_LIBRARIES
		${FFMPEG_AVCODEC_LIBRARIES})
endif()


include_directories(
	SYSTEM "${CMAKE_SOURCE_DIR}/libobs"
	${LIBV4L2_INCLUDE_DIRS}
)

set(linux-v4l2_SOURCES
	linux-v4l2.c
	v4l2-controls.c
	v4l2-input.c
	v4l2-helpers.c
	${v4l2_mjpeg_loading_SOURCES}
	${linux-v4l2-udev_SOURCES}
)

add_library(linux-v4l2 MODULE
	${linux-v4l2_SOURCES}
)
target_link_libraries(linux-v4l2
	libobs
	${LIBV4L2_LIBRARIES}
	${UDEV_LIBRARIES}
	${v4l2_mjpeg_loading_LIBRARIES}
)
set_target_properties(linux-v4l2 PROPERTIES FOLDER "plugins")


install_obs_plugin_with_data(linux-v4l2 data)
