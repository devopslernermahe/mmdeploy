# Copyright (c) OpenMMLab. All rights reserved.
cmake_minimum_required(VERSION 3.14)
project(capis)

include(${CMAKE_SOURCE_DIR}/cmake/common.cmake)

if ("all" IN_LIST MMDEPLOY_CODEBASES)
    set(TASK_LIST "classifier;detector;segmentor;text_detector;text_recognizer;restorer;model")
else ()
    set(TASK_LIST "model")
    if ("mmcls" IN_LIST MMDEPLOY_CODEBASES)
        list(APPEND TASK_LIST "classifier")
    endif ()
    if ("mmdet" IN_LIST MMDEPLOY_CODEBASES)
        list(APPEND TASK_LIST "detector")
    endif ()
    if ("mmseg" IN_LIST MMDEPLOY_CODEBASES)
        list(APPEND TASK_LIST "segmentor")
    endif ()
    if ("mmedit" IN_LIST MMDEPLOY_CODEBASES)
        list(APPEND TASK_LIST "restorer")
    endif ()
    if ("mmocr" IN_LIST MMDEPLOY_CODEBASES)
        list(APPEND TASK_LIST "text_detector")
        list(APPEND TASK_LIST "text_recognizer")
    endif ()
endif ()

foreach (TASK ${TASK_LIST})
    set(TARGET_NAME mmdeploy_${TASK})
    build_target(${TARGET_NAME} ${TASK}.cpp)
    target_link_libraries(${TARGET_NAME} PRIVATE mmdeploy::core)
    target_include_directories(${TARGET_NAME} PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
            $<INSTALL_INTERFACE:include/c>)
    export_target(${TARGET_NAME})

    install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/${TASK}.h
            DESTINATION include/c)

endforeach ()

install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/common.h
        DESTINATION include/c)
