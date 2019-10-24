cmake_minimum_required(VERSION 3.10)
project(faust-vst-template)

#############
#  Options  #
#############

set(FAUST_SOURCES_DIR "${PROJECT_SOURCE_DIR}/faust-code" CACHE STRING "Directory with .dsp files to compile")
set(VST2_SDK_PATH "${PROJECT_SOURCE_DIR}/vstsdk2.4" CACHE STRING "Location of VST 2.x SDK")

# options adapted from faust2faustvst script
option(FAUST_META "Include Faust metadata in generated plugin" ON)
option(FAUST_MIDICC "Enable direct mapping of MIDI CCs in the plugin" OFF)
option(FAUST_NVOICES "Number of synth voices (instruments only)" 8)

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
        "${source}"
        "${VST2_SDK_PLUGIN_SOURCES}"
    )
    add_library(${target} MODULE ${TARGET_SOURCES})
    target_include_directories(${target} PRIVATE ${VST_INCLUDE_DIRS})
    target_compile_features(${target} PRIVATE cxx_std_17)
    target_compile_options(${target} PRIVATE -Wno-unused -Wno-multichar -Wno-write-strings -Wno-narrowing)
    target_compile_options(${target} PRIVATE -ffast-math -ftree-vectorize)
    if(${FAUST_META})
        target_compile_definitions(${target} PRIVATE -DFAUST_META=1)
    endif()
    if(${FAUST_MIDICC})
        target_compile_definitions(${target} PRIVATE -DFAUST_MIDICC=1)
    endif()
    target_compile_definitions(${target} PRIVATE -DFAUST_NVOICES=${FAUST_NVOICES})
    set_target_properties(${target} PROPERTIES
        # Remove leading 'lib' from generated .so file
        PREFIX ""
    )
endfunction()

##################
#  Faust helper  #
##################

function (build_faust_target faust_source)
    get_filename_component(faust_basename ${faust_source} NAME_WE)
    set(FAUST_CPP_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/${faust_basename}_gen.cpp")
    message("${FAUST_CPP_OUTPUT}")
    execute_process(
        COMMAND faust -i -a faustvst.cpp -o ${FAUST_CPP_OUTPUT} ${faust_source}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
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

