cmake_minimum_required(VERSION 3.17.0 FATAL_ERROR)

# set meta data: I am treating is for now a fork as upstream does not merge any patches anymore. Hence I am putting myself as
# author here and use my fork's URL.
project(pianobooster)
set(META_PROJECT_NAME ${PROJECT_NAME})
set(META_PROJECT_TYPE application)
set(META_PROJECT_LICENSE "GPL-3.0-or-later")
set(META_APP_NAME "Piano Booster")
set(META_APP_CATEGORIES "Music;Education;")
set(META_APP_AUTHOR "Martchus, L. J. Barman, Fabien Givors and others")
set(META_APP_DOMAIN "https://github.com/Martchus")
set(META_APP_URL "${META_APP_DOMAIN}/${META_PROJECT_NAME}")
set(META_APP_DESCRIPTION "Learn the piano just by playing a game")
set(META_VERSION_MAJOR 1)
set(META_VERSION_MINOR 0)
set(META_VERSION_PATCH 1)
set(META_RELEASE_DATE "2023-05-06")

set(SRC_FILES
    MidiFile.cpp
    MidiTrack.cpp
    Song.cpp
    Conductor.cpp
    Util.cpp
    Chord.cpp
    Tempo.cpp
    MidiDevice.cpp
    MidiDeviceRt.cpp)
set(HEADER_FILES
    MidiFile.h
    MidiTrack.h
    Song.h
    Conductor.h
    Rating.h
    Util.h
    Chord.h
    Tempo.h
    MidiDevice.h
    MidiDeviceRt.h)
set(WIDGETS_SRC_FILES
    QtMain.cpp
    QtWindow.cpp
    GuiTopBar.cpp
    GuiSidePanel.cpp
    GuiMidiSetupDialog.cpp
    GuiKeyboardSetupDialog.cpp
    GuiPreferencesDialog.cpp
    GuiSongDetailsDialog.cpp
    GuiLoopingPopup.cpp
    GlView.cpp
    StavePosition.cpp
    Score.cpp
    Cfg.cpp
    Piano.cpp
    Draw.cpp
    Scroll.cpp
    Notation.cpp
    TrackList.cpp
    Rating.cpp
    Bar.cpp
    Settings.cpp
    Merge.cpp
    application.qrc
    images/pianobooster.ico)
set(WIDGETS_HEADER_FILES
    QtWindow.h
    GuiTopBar.h
    GuiSidePanel.h
    GuiMidiSetupDialog.h
    GuiKeyboardSetupDialog.h
    GuiPreferencesDialog.h
    GuiSongDetailsDialog.h
    GuiLoopingPopup.h
    GlView.h
    StavePosition.h
    Score.h
    Cfg.h
    Piano.h
    Draw.h
    Scroll.h
    Notation.h
    TrackList.h
    Rating.h
    Bar.h
    Settings.h
    Merge.h)
set(WIDGETS_UI_FILES
    GuiTopBar.ui
    GuiSidePanel.ui
    GuiMidiSetupDialog.ui
    GuiKeyboardSetupDialog.ui
    GuiPreferencesDialog.ui
    GuiSongDetailsDialog.ui
    GuiLoopingPopup.ui)
set(CMAKE_MODULE_FILES ../CMakeLists.txt ../translations/CMakeLists.txt)
set(WINDOWS_ICON_PATH "${CMAKE_CURRENT_SOURCE_DIR}/images/pianobooster.ico")
set(REQUIRED_ICONS
    application-exit
    color-profile
    dialog-cancel
    dialog-ok
    dialog-ok-apply
    document-open
    edit-clear
    globe
    help-about-symbolic
    help-hint
    help-keybord-shortcuts
    music-note-16th
    preferences-desktop-icons
    preferences-desktop-locale
    qtcreator
    system-file-manager
    system-run
    system-search
    view-fullscreen
    view-left-pane-symbolic
    window-close
)

option(WITH_INTERNAL_FLUIDSYNTH "Build with an internal FluidSynth sound generator" ON)
option(USE_FTGL "Build with ftgl for notes localization" ON)
option(USE_SYSTEM_FONT "Build with system font" OFF)
option(USE_JACK "Build with Jack (Only required for BSD Unix)" OFF)
if (${CMAKE_SYSTEM} MATCHES "Linux")
    option(USE_BUNDLED_RTMIDI "Build with bundled rtmidi (for older distributions only)" OFF)
else ()
    option(USE_BUNDLED_RTMIDI "Build with bundled rtmidi" ON)
endif ()

# find c++utilities
find_package(${PACKAGE_NAMESPACE_PREFIX}c++utilities${CONFIGURATION_PACKAGE_SUFFIX} 5.20.0 REQUIRED)
use_cpp_utilities(VISIBILITY PUBLIC)

# find qtutilities
find_package(${PACKAGE_NAMESPACE_PREFIX}qtutilities${CONFIGURATION_PACKAGE_SUFFIX_QTUTILITIES} 6.13.0 REQUIRED)
use_qt_utilities()

# configure prefix and data dir
if (NOT DATA_DIR)
    set(DATA_DIR "share/games/pianobooster")
endif ()
add_compile_options("-DDATA_DIR=\"${DATA_DIR}\"")
set(CMAKE_INSTALL_DATAROOTDIR "share/games")  # used by c++/qtutilities

# apply basic configuration
include(BasicConfig)

# find OpenGL
set(OpenGL_GL_PREFERENCE "GLVND")
find_package(OpenGL REQUIRED)

# find pkg-config
if (NOT WIN32 OR MINGW)
    # the `pkg_check_modules` function is created with this call
    include(FindPkgConfig)
    find_package(PkgConfig REQUIRED)
endif ()

# find FTGL
if (USE_FTGL)
    if (WIN32)
        find_package(FTGL)

        # add include path for freetype2 as ftgl uses freetype2 headers but doesn't pull in the include path automatically
        if (FTGL_FOUND AND COMMAND pkg_check_modules)
            pkg_check_modules(FREETYPE2 freetype2)
            if (FREETYPE2_FOUND)
                list(APPEND FTGL_INCLUDE_DIR "${FREETYPE2_INCLUDE_DIRS}")
            endif ()
        endif ()
    endif ()
    if (NOT FTGL_FOUND)
        pkg_check_modules(FTGL ftgl)
        if (NOT FTGL_FOUND)
            message(FATAL_ERROR "FTGL was not found")
        endif (NOT FTGL_FOUND)
        set(FTGL_INCLUDE_DIR ${FTGL_INCLUDE_DIRS})
        set(FTGL_LIBRARY ${FTGL_LIBRARIES})
    endif ()
else (USE_FTGL)
    add_compile_options("-DNO_USE_FTGL")
endif (USE_FTGL)

# configure translations
if (NO_LANGS)
    add_compile_options("-DNO_LANGS")
else ()
    add_subdirectory(../translations ../build/translations-subdir)
endif ()

# find Qt libraries
list(
    APPEND
    ADDITIONAL_QT_MODULES
    Core
    Gui
    Widgets
    OpenGL
    Xml)
if (QT_PACKAGE_PREFIX MATCHES ".*Qt([0-9]+).*")
    list(APPEND ADDITIONAL_QT_MODULES OpenGLWidgets)
endif ()
include(QtGuiConfig)
include(QtConfig)

# add platform-specific libraries
if (${CMAKE_SYSTEM} MATCHES "Linux")
    add_definitions(-D__LINUX_ALSA__)
    link_libraries(asound)
    link_libraries(pthread)
    link_libraries(GL)
endif (${CMAKE_SYSTEM} MATCHES "Linux")
if (${CMAKE_SYSTEM} MATCHES "Windows")
    add_definitions(-D__WINDOWS_MM__ -D_WIN32)
    link_libraries(winmm opengl32)
endif (${CMAKE_SYSTEM} MATCHES "Windows")
if (${CMAKE_SYSTEM} MATCHES "Darwin")
    add_definitions(-D__MACOSX_CORE__)
    add_definitions(-std=c++11)
    link_libraries("-framework CoreMidi -framework CoreAudio -framework CoreFoundation -framework OpenGL")
endif (${CMAKE_SYSTEM} MATCHES "Darwin")

# find FluidSynth
if (WITH_INTERNAL_FLUIDSYNTH)
    if (NOT WIN32)
        pkg_check_modules(FLUIDSYNTH fluidsynth)

        if (NOT FLUIDSYNTH_FOUND)
            message(FATAL_ERROR "FLUIDSYNTH was not found")
        endif (NOT FLUIDSYNTH_FOUND)
        include_directories(${FLUIDSYNTH_INCLUDE_DIRS})
        set(FLUIDSYNTH_LIBRARY ${FLUIDSYNTH_LIBRARIES})
    else ()
        # find FluidSynth and link against its imported target
        find_package(FluidSynth REQUIRED)
        set(FLUIDSYNTH_LIBRARY FluidSynth::libfluidsynth)

        # find additional dependencies that are pulled in by the imported target but not automatically made available
        find_package(OpenMP REQUIRED)
        if (COMMAND pkg_check_modules)
            pkg_check_modules(GLIB IMPORTED_TARGET glib-2.0 gthread-2.0)
            pkg_check_modules(LIBSNDFILE IMPORTED_TARGET sndfile)
            pkg_check_modules(PORTAUDIO IMPORTED_TARGET portaudio-2.0)
            pkg_check_modules(READLINE IMPORTED_TARGET readline)
        endif ()
    endif ()
    add_definitions(-DWITH_INTERNAL_FLUIDSYNTH)
    message("Building with internal fluidsynth")
    list(APPEND SRC_FILES MidiDeviceFluidSynth.cpp)
endif (WITH_INTERNAL_FLUIDSYNTH)

# find JACK
if (USE_JACK)
    # Check for Jack
    find_library(JACK_LIB jack)
    pkg_check_modules(JACK jack)
    if (NOT JACK_FOUND)
        message(FATAL_ERROR "JACK was not found")
    endif (NOT JACK_FOUND)

    set(JACK_INCLUDE_DIR ${JACK_INCLUDE_DIRS})
    set(JACK_LIBRARY ${JACK_LIBRARIES})
    if (JACK_FOUND)
        add_definitions(-D__UNIX_JACK__)
    endif (JACK_FOUND)
endif (USE_JACK)

# find RtMidi
if (USE_BUNDLED_RTMIDI)
    message(STATUS "Building using bundled rtmidi")
    add_compile_options("-DUSE_BUNDLED_RTMIDI")
    include_directories("3rdparty")
    list(APPEND SRC_FILES 3rdparty/rtmidi/RtMidi.cpp)
    list(APPEND HEADER_FILES 3rdparty/rtmidi/RtMidi.h)
    # don't treat warnings for this 3rdparty code as fatal (even if we'd normally would treat warnings as fatal)
    set_property(
        SOURCE 3rdparty/rtmidi/RtMidi.cpp
        APPEND
        PROPERTY COMPILE_OPTIONS -Wno-error=conversion -Wno-error=sign-conversion -Wno-error=old-style-cast)
else ()
    pkg_check_modules(RTMIDI rtmidi)
    if (NOT RTMIDI_FOUND)
        message(FATAL_ERROR "rtmidi not found (Try building with option USE_BUNDLED_RTMIDI=ON)")
    endif (NOT RTMIDI_FOUND)
    include_directories(${RTMIDI_INCLUDE_DIRS})
    link_directories(${RTMIDI_LIBRARY_DIRS})
endif ()

# configure MacOS-specific bundling
if (APPLE)
    # the property added the pianobooster icon to Info.plist
    set(MACOSX_BUNDLE_ICON_FILE pianobooster.icns)

    # Tells CMake where to find and install the pianobooster icon
    set(pianobooster_ICON ${CMAKE_CURRENT_SOURCE_DIR}/../icons/MacOS/pianobooster.icns)
    set_source_files_properties(${pianobooster_ICON} PROPERTIES MACOSX_PACKAGE_LOCATION "Resources")

    set(pianobooster_TRANSLATIONS ${CMAKE_CURRENT_BINARY_DIR}/translations/langs.json)
    set_source_files_properties(${pianobooster_TRANSLATIONS} PROPERTIES MACOSX_PACKAGE_LOCATION "Resources/translations")

    set(pianobooster_MUSIC ${CMAKE_CURRENT_SOURCE_DIR}/../music/BoosterMusicBooks.zip)
    set_source_files_properties(${pianobooster_MUSIC} PROPERTIES MACOSX_PACKAGE_LOCATION "Resources/music")
    set(pianobooster_FONT ${CMAKE_CURRENT_SOURCE_DIR}/fonts/DejaVuSans.ttf)
    set_source_files_properties(${pianobooster_FONT} PROPERTIES MACOSX_PACKAGE_LOCATION "Resources/fonts")

    add_custom_target(install-translations cp -R -v ${CMAKE_CURRENT_BINARY_DIR}/translations/*.qm
                                           ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.app/Contents/Resources/translations)
endif ()

# configure executable
include(WindowsResources)
include(AppTarget)
target_include_directories("${META_TARGET_NAME}" PRIVATE ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_BINARY_DIR}
                                                         ${OPENGL_INCLUDE_DIR} ${FTGL_INCLUDE_DIR})
target_link_directories("${META_TARGET_NAME}" PRIVATE ${FTGL_LIBRARY_DIRS} ${JACK_LIBRARY_DIRS} ${FLUIDSYNTH_LIBRARY_DIRS})
target_link_libraries("${META_TARGET_NAME}" PRIVATE ${OPENGL_LIBRARIES} ${FTGL_LIBRARY} ${RTMIDI_LIBRARIES} ${JACK_LIBRARY}
                                                    ${FLUIDSYNTH_LIBRARY} ${RTMIDI_LIBRARY})

# install various additional files
if (NOT APPLE)
    install(FILES ../pianobooster.desktop DESTINATION share/applications)
    install(TARGETS "${META_TARGET_NAME}" RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif (NOT APPLE)
if (NOT NO_DOCS)
    install(FILES ../README.md DESTINATION share/doc/pianobooster/)
    install(FILES ../doc/faq.md DESTINATION share/doc/pianobooster/)
endif ()
if (NOT NO_LICENSE)
    install(FILES ../license.txt DESTINATION share/licenses/pianobooster/)
endif ()
if (NOT NO_CHANGELOG)
    install(FILES ../Changelog.txt DESTINATION share/doc/pianobooster/)
endif ()
if (WITH_MAN)
    install(FILES ../pianobooster.6 DESTINATION share/man/man6/)
endif ()
install(FILES ../icons/hicolor/32x32/pianobooster.png DESTINATION share/icons/hicolor/32x32/apps)
install(FILES ../icons/hicolor/48x48/pianobooster.png DESTINATION share/icons/hicolor/48x48/apps)
install(FILES ../icons/hicolor/64x64/pianobooster.png DESTINATION share/icons/hicolor/64x64/apps)
install(FILES ../icons/hicolor/scalable/pianobooster.svg DESTINATION share/icons/hicolor/scalable/apps)

# configure and install music books
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/../music/BoosterMusicBooks.zip ${CMAKE_CURRENT_BINARY_DIR}/ COPYONLY)
if (UNIX AND NOT APPLE)
    install(FILES ../music/BoosterMusicBooks.zip DESTINATION ${DATA_DIR}/music)
endif (UNIX AND NOT APPLE)
if (WIN32)
    install(FILES ../music/BoosterMusicBooks.zip DESTINATION .)
endif (WIN32)

# install fonts
if (NOT USE_SYSTEM_FONT)
    configure_file(fonts/DejaVuSans.ttf ${CMAKE_CURRENT_BINARY_DIR}/fonts/DejaVuSans.ttf COPYONLY)
    install(FILES fonts/DejaVuSans.ttf DESTINATION ${DATA_DIR}/fonts/)
endif (NOT USE_SYSTEM_FONT)

# configure debugging flags
if (DEBUG_LOG_TRACE)
    add_definitions(-DPB_LOG_TRACE)
endif ()
if (DEBUG_LOG_TIMING)
    add_definitions(-DPB_LOG_TIMING)
endif ()

include(ConfigHeader)
