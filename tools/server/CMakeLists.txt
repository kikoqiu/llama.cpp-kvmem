include_directories(${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_BINARY_DIR})

# server-context containing the core server logic, used by llama-server and CLI

set(TARGET server-context)

add_library(${TARGET} STATIC
    server-chat.cpp
    server-chat.h
    server-task.cpp
    server-task.h
    server-queue.cpp
    server-queue.h
    server-common.cpp
    server-common.h
    server-context.cpp
    server-context.h
    server-stream.cpp
    server-stream.h
    server-tools.cpp
    server-tools.h
    server-mcp.cpp
    server-mcp.h
    server-schema.cpp
    server-schema.h
)

if (BUILD_SHARED_LIBS)
    set_target_properties(${TARGET} PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif()

target_include_directories(${TARGET} PRIVATE ../mtmd)
target_include_directories(${TARGET} PRIVATE ${PROJECT_SOURCE_DIR})
target_link_libraries(${TARGET} PUBLIC llama-common mtmd ${CMAKE_THREAD_LIBS_INIT})

# llama-server-impl: server logic, reusable by app

set(TARGET llama-server-impl)

add_library(${TARGET}
    server.cpp
    server-http.cpp
    server-http.h
    server-models.cpp
    server-models.h
)
set_target_properties(${TARGET} PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)

target_include_directories(${TARGET} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_include_directories(${TARGET} PRIVATE ../mtmd ${PROJECT_SOURCE_DIR})
target_link_libraries(${TARGET} PUBLIC server-context llama-ui cpp-httplib ${CMAKE_THREAD_LIBS_INIT})

add_dependencies(${TARGET} llama-ui-assets)

if(LLAMA_TOOLS_INSTALL)
    install(TARGETS ${TARGET} LIBRARY)
endif()

# llama-server executable

set(TARGET llama-server)

add_executable(${TARGET} main.cpp)
install(TARGETS ${TARGET} RUNTIME)

target_link_libraries(${TARGET} PRIVATE llama-server-impl)
target_compile_features(${TARGET} PRIVATE cxx_std_17)

# llama-kvmem-server: KVMem's own single-slot server, built from the out-of-tree
# KVMem sources. Standalone only, so the KVMem root project can define its own
# target with the same name.

if (LLAMA_KVMEM AND LLAMA_STANDALONE)
    set(TARGET llama-kvmem-server)

    add_executable(${TARGET}
        ${LLAMA_KVMEM_ROOT}/tools/llama-kvmem-server.cpp
        ${LLAMA_KVMEM_ROOT}/tools/kvmem-spec.cpp
        ${LLAMA_KVMEM_ROOT}/tools/kvmem-vision.cpp)

    target_include_directories(${TARGET} PRIVATE
        ${LLAMA_KVMEM_ROOT}/src/adapter
        ${LLAMA_KVMEM_ROOT}/tools
        ${PROJECT_SOURCE_DIR}/include
        ${PROJECT_SOURCE_DIR}/common
        ${PROJECT_SOURCE_DIR}/vendor
        ${PROJECT_SOURCE_DIR}/vendor/cpp-httplib
        ../mtmd)

    target_link_libraries(${TARGET} PRIVATE
        llama llama-common kvmem server-context cpp-httplib mtmd ${CMAKE_THREAD_LIBS_INIT})

    target_compile_features(${TARGET} PRIVATE cxx_std_17)

    if(LLAMA_TOOLS_INSTALL)
        install(TARGETS ${TARGET} RUNTIME)
    endif()
endif()
