
add_executable(nfcDemoApp main.c tools.c tools.h)
target_include_directories(nfcDemoApp PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
target_link_libraries(nfcDemoApp PRIVATE libnfc_nci_linux)
target_compile_options(nfcDemoApp PRIVATE -w)
