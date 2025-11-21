cmake_minimum_required(VERSION 3.22)
project(faust-vst-template)

#############
#  Options  #
#############

set(FAUST_SOURCES_DIR "${PROJECT_SOURCE_DIR}/faust-code" CACHE STRING "Directory with .dsp files to compile")
set(VST2_SDK_PATH "${PROJECT_SOURCE_DIR}/vstsdk2.4" CACHE STRING "Location of VST 2.x SDK")

# options adapted from faust2faustvst script
option(FAUST_META "Include Faust metadata in generated plugin" ON)
option(FAUST_MIDICC "Enable direct mapping of MIDI CCs in the plugin" OFF)
set(FAUST_NVOICES 0 CACHE STRING "Number of synth voices. Only for synths, if set to 0 will generate an FX instead")

# Hardcoded options for Apple bundle

set(BUNDLE_ID "com.test.faustvst")
set(BUNDLE_NAME "FaustVst")
set(BUNDLE_COPYRIGHT "ACME @2025")


##################
#  VsT SDK       #
##################

set(VST_INCLUDE_DIRS
    "${VST2_SDK_PATH}"
    "${VST2_SDK_PATH}/pluginterfaces/vst2.x/"
    "${VST2_SDK_PATH}/public.sdk/source/vst2.x/"
)

if (UNIX)
    # API fix for POSIX builds
    add_definitions( -D__cdecl= )
endif()

set(VST2_SDK_PLUGIN_SOURCES
    "${VST2_SDK_PATH}/public.sdk/source/vst2.x/audioeffectx.cpp"
    "${VST2_SDK_PATH}/public.sdk/source/vst2.x/audioeffect.cpp"
    "${VST2_SDK_PATH}/public.sdk/source/vst2.x/vstplugmain.cpp"
)

function (add_vst2_plugin_target target source)
    set(TARGET_SOURCES
        ${source}
        ${VST2_SDK_PLUGIN_SOURCES}
    )
    add_library("${target}" MODULE ${TARGET_SOURCES})
    target_include_directories("${target}" PRIVATE ${VST_INCLUDE_DIRS} ${CMAKE_CURRENT_BINARY_DIR})
    target_compile_features("${target}" PRIVATE cxx_std_17)
    target_compile_options("${target}" PRIVATE -Wno-unused -Wno-multichar -Wno-write-strings -Wno-narrowing)
    target_compile_options("${target}" PRIVATE -ffast-math -ftree-vectorize)
    if(${FAUST_META})
        target_compile_definitions("${target}" PRIVATE -DFAUST_META=1)
    endif()
    if(${FAUST_MIDICC})
        target_compile_definitions("${target}" PRIVATE -DFAUST_MIDICC=1)
    endif()
    if(FAUST_NVOICES GREATER 0)
        target_compile_definitions("${target}" PRIVATE -DNVOICES=${FAUST_NVOICES})
    endif()

    if (APPLE)
        set_target_properties("${target}" PROPERTIES
            BUNDLE true
            BUNDLE_EXTENSION "vst"
            XCODE_ATTRIBUTE_WRAPPER_EXTENSION "vst"
            XCODE_ATTRIBUTE_PRODUCT_BUNDLE_IDENTIFIER "${BUNDLE_ID}"
            MACOSX_BUNDLE_INFO_PLIST "${CMAKE_CURRENT_SOURCE_DIR}/info.plist.in"
            MACOSX_BUNDLE_BUNDLE_NAME "${BUNDLE_NAME}"
            MACOSX_BUNDLE_GUI_IDENTIFIER "${BUNDLE_ID}"
            MACOSX_BUNDLE_ICON_FILE ""
            MACOSX_BUNDLE_SHORT_VERSION_STRING "${CMAKE_PROJECT_VERSION}"
            MACOSX_BUNDLE_COPYRIGHT "${BUNDLE_COPYRIGHT}"
        )

        add_custom_command(
            TARGET "${target}"
            POST_BUILD
            COMMAND install_name_tool
            ARGS -add_rpath @executable_path/../Frameworks $<TARGET_FILE:${target}>
        )

    elseif (UNIX)
        set_target_properties("${target}" PROPERTIES VERSION ${CMAKE_PROJECT_VERSION})
    endif()

endfunction()

##################
#  Faust helper  #
##################

function (build_faust_target faust_source)
    get_filename_component(faust_basename ${faust_source} NAME_WE)
    set(FAUST_CPP_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/${faust_basename}_gen.cpp")
    message("${FAUST_CPP_OUTPUT}")

    add_custom_command(
        OUTPUT  ${FAUST_CPP_OUTPUT}
        COMMAND faust -uim -vec -i -a faustvst.cpp -o ${FAUST_CPP_OUTPUT} ${faust_source}
        DEPENDS ${faust_source}
        WORKING_DIRECTORY ${FAUST_SOURCES_DIR}
    )
    add_vst2_plugin_target(${faust_basename} ${FAUST_CPP_OUTPUT})
endfunction()

###################
#  Build targets  #
###################

# All .dsp files in ${FAUST_SOURCES_DIR}

file(GLOB faust_sources "${FAUST_SOURCES_DIR}/*.dsp")
foreach(file ${faust_sources})
    build_faust_target(${file})
endforeach()

