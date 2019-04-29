cmake_minimum_required(VERSION 3.12)

set(PACKAGE_NAME libnfc_nci_linux)
set(PACKAGE ${PACKAGE_NAME})
set(PACKAGE_VERSION 1.0)
set(VERSION ${PACKAGE_VERSION})
set(PACKAGE_TARNAME "${PACKAGE} ${VERSION}")

project(${PACKAGE_NAME} VERSION ${PACKAGE_VERSION} LANGUAGES C CXX)

option(BUILD_SHARED_LIBS "" OFF)
option(ENABLE_LLCP1_3 "enable LLCP1.3 (secured P2P). requires OPENSSLDIR to be set." OFF)
set(ACTIVE_TML "ALT" CACHE STRING "choose which type of TML to have enabled")
set_property(CACHE ACTIVE_TML PROPERTY STRINGS ABEND ALT I2C LPCUSBSIO)

if(ENABLE_LLCP1_3)
    message(FATAL_ERROR "LLCP 1.3 is currently not supported.")
endif()

foreach(_tml ABEND LPCUSBSIO)
    if(ACTIVE_TML STREQUAL ${_tml})
        message(FATAL_ERROR "The chosen ACTIVE_TML '${_tml}' is not supported. Only ALT and I2C are currently supported.")
    endif()
endforeach()

include(CheckIncludeFile)
check_include_file("dlfcn.h" HAVE_DLFCN_H)
check_include_file("fcntl.h" HAVE_FCNTL_H)
check_include_file("inttypes.h" HAVE_INTTYPES_H)
check_include_file("libudev.h" HAVE_LIBUDEV_H)
check_include_file("malloc.h" HAVE_MALLOC_H)
check_include_file("memory.h" HAVE_MEMORY_H)
check_include_file("stdint.h" HAVE_STDINT_H)
check_include_file("stdlib.h" HAVE_STDLIB_H)
check_include_file("string.h" HAVE_STRING_H)
check_include_file("strings.h" HAVE_STRINGS_H)
check_include_file("termios.h" HAVE_TERMIOS_H)
check_include_file("unistd.h" HAVE_UNISTD_H)
check_include_file("sys/ioctl.h" HAVE_SYS_IOCTL_H)
check_include_file("sys/stat.h" HAVE_SYS_STAT_H)
check_include_file("sys/time.h" HAVE_SYS_TIME_H)
check_include_file("sys/types.h" HAVE_SYS_TYPES_H)

check_include_file("stdbool.h" HAVE_STDBOOL_H)
if(HAVE_STDBOOL_H)
    set(HAVE__BOOL 1)
endif()

include(CheckFunctionExists)
check_function_exists("dlopen" HAVE_DLOPEN)
check_function_exists("clock_gettime" HAVE_CLOCK_GETTIME)
check_function_exists("gettimeofday" HAVE_GETTIMEOFDAY)
check_function_exists("malloc" HAVE_MALLOC)
check_function_exists("memmove" HAVE_MEMMOVE)
check_function_exists("memset" HAVE_MEMSET)
check_function_exists("select" HAVE_SELECT)
check_function_exists("strerror" HAVE_STRERROR)
check_function_exists("strstr" HAVE_STRSTR)
check_function_exists("realloc" HAVE_REALLOC)

if(NOT HAVE_MALLOC)
    set(malloc rpl_malloc)
endif()
if(NOT HAVE_REALLOC)
    set(realloc rpl_realloc)
endif()

include(CheckCSourceCompiles)
check_c_source_compiles("
    inline testFunc() {}
    int main() {
        testFunc();
        return 0;
    }"
    HAS_INLINE
)

if(NOT HAS_INLINE)
    check_c_source_compiles("
        __inline__ testFunc() {}
        int main() {
            testFunc();
            return 0;
        }"
        HAS___INLINE__
    )

    if(HAS___INLINE__)
        set(inline __inline__)
    else()
        check_c_source_compiles("
            __inline testFunc() {}
            int main() {
                testFunc();
                return 0;
            }"
            HAS___INLINE
        )

        if(HAS__INLINE)
            set(inline __inline)
        endif()
    endif()
endif()

include(CheckTypeSize)
foreach(_type int8_t int16_t int32_t uint8_t uint16_t uint32_t uint64_t size_t ssize_t)
    string(TOUPPER ${_type} _type_upper)
    check_type_size(${_type} ${_type_upper})

    if(NOT ${_type_upper})
        set(${_type} true)
    endif()
endforeach()

configure_file(cmake/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/include/config.h)

if(ENABLE_LLCP1_3)
else()
    message(STATUS "LLCP1.3 support is disabled (no secure P2P)")
endif()

add_library(${PACKAGE_NAME})

set_target_properties(${PACKAGE_NAME} PROPERTIES
    POSITION_INDEPENDENT_CODE true
)

target_compile_options(${PACKAGE_NAME} PRIVATE
    -w
)

target_compile_definitions(${PACKAGE_NAME} PRIVATE
    HAVE_CONFIG_H
    NFC_NXP_NOT_OPEN_INCLUDED=TRUE
    NXP_HW_SELF_TEST
    NXP_NFC_NATIVE_ENABLE_HCE=TRUE
    PH_NCI_NXP_HAL_ENABLE_FW_DOWNLOAD=FALSE
    NFC_NXP_LLCP_SECURED_P2P=FALSE
    $<$<STREQUAL:${ACTIVE_TML},ALT>:PHFL_TML_ALT_NFC>
)

find_package(Threads)
target_link_libraries(${PACKAGE_NAME} PUBLIC
    ${CMAKE_THREAD_LIBS_INIT}
    -ldl
    -lrt
)

file(GLOB_RECURSE PROJECT_HEADERS LIST_DIRECTORIES false "src/libnfc-nci/*.h")
target_sources(${PACKAGE_NAME} PRIVATE
    ${PROJECT_HEADERS}
)

target_include_directories(${PACKAGE_NAME} PRIVATE
    ${CMAKE_CURRENT_BINARY_DIR}/include
    src/libnfc-nci/include
    src/libnfc-nci/gki/ulinux
    src/libnfc-nci/gki/common
    src/libnfc-nci/nfa/include
    src/libnfc-nci/nfa/int
    src/libnfc-nci/nfc/include
    src/libnfc-nci/nfc/int
    src/libnfc-nci/hal/include
    src/libnfc-nci/hal/int
)

target_sources(${PACKAGE_NAME} PRIVATE
    src/libnfc-nci/nfa/rw/nfa_rw_main.c
    src/libnfc-nci/nfa/rw/nfa_rw_api.c
    src/libnfc-nci/nfa/rw/nfa_rw_act.c
    src/libnfc-nci/nfa/hci/nfa_hci_ci.c
    src/libnfc-nci/nfa/hci/nfa_hci_api.c
    src/libnfc-nci/nfa/hci/nfa_hci_act.c
    src/libnfc-nci/nfa/hci/nfa_hci_main.c
    src/libnfc-nci/nfa/hci/nfa_hci_utils.c
    src/libnfc-nci/nfa/ce/nfa_ce_act.c
    src/libnfc-nci/nfa/ce/nfa_ce_main.c
    src/libnfc-nci/nfa/ce/nfa_ce_api.c
    src/libnfc-nci/nfa/cho/nfa_cho_api.c
    src/libnfc-nci/nfa/cho/nfa_cho_main.c
    src/libnfc-nci/nfa/cho/nfa_cho_sm.c
    src/libnfc-nci/nfa/cho/nfa_cho_util.c
    src/libnfc-nci/nfa/p2p/nfa_p2p_act.c
    src/libnfc-nci/nfa/p2p/nfa_p2p_api.c
    src/libnfc-nci/nfa/p2p/nfa_p2p_main.c
    src/libnfc-nci/nfa/snep/nfa_snep_act.c
    src/libnfc-nci/nfa/snep/nfa_snep_api.c
    src/libnfc-nci/nfa/snep/nfa_snep_default.c
    src/libnfc-nci/nfa/snep/nfa_snep_main.c
    src/libnfc-nci/nfa/sys/nfa_sys_cback.c
    src/libnfc-nci/nfa/sys/nfa_sys_ptim.c
    src/libnfc-nci/nfa/sys/nfa_sys_cfg.c
    src/libnfc-nci/nfa/sys/nfa_sys_main.c
    src/libnfc-nci/nfa/dm/nfa_dm_discover.c
    src/libnfc-nci/nfa/dm/nfa_dm_act.c
    src/libnfc-nci/nfa/dm/nfa_dm_main.c
    src/libnfc-nci/nfa/dm/nfa_dm_ndef.c
    src/libnfc-nci/nfa/dm/nfa_dm_api.c
    src/libnfc-nci/nfa/dm/nfa_dm_cfg.c
    src/libnfc-nci/nfa/ee/nfa_ee_api.c
    src/libnfc-nci/nfa/ee/nfa_ee_main.c
    src/libnfc-nci/nfa/ee/nfa_ee_act.c
    src/libnfc-nci/adaptation/libmain.c
    src/libnfc-nci/gki/common/gki_debug.c
    src/libnfc-nci/gki/common/gki_time.c
    src/libnfc-nci/gki/common/gki_buffer.c
    src/libnfc-nci/gki/ulinux/gki_ulinux.c
    src/libnfc-nci/nfc/cipher/cipher.c
    src/libnfc-nci/nfc/llcp/llcp_sdp.c
    src/libnfc-nci/nfc/llcp/llcp_link.c
    src/libnfc-nci/nfc/llcp/llcp_api.c
    src/libnfc-nci/nfc/llcp/llcp_util.c
    src/libnfc-nci/nfc/llcp/llcp_dlc.c
    src/libnfc-nci/nfc/llcp/llcp_main.c
    src/libnfc-nci/nfc/nfc/nfc_vs.c
    src/libnfc-nci/nfc/nfc/nfc_task.c
    src/libnfc-nci/nfc/nfc/nfc_main.c
    src/libnfc-nci/nfc/nfc/nfc_ncif.c
    src/libnfc-nci/nfc/nfc/nfc_test.c
    src/libnfc-nci/nfc/nfc/nfc_ee.c
    src/libnfc-nci/nfc/nfc/nfc_utils.c
    src/libnfc-nci/nfc/nci/nci_hrcv.c
    src/libnfc-nci/nfc/nci/nci_hmsgs.c
    src/libnfc-nci/nfc/tags/rw_i93.c
    src/libnfc-nci/nfc/tags/rw_t4t.c
    src/libnfc-nci/nfc/tags/tags_int.c
    src/libnfc-nci/nfc/tags/ce_t3t.c
    src/libnfc-nci/nfc/tags/rw_t2t.c
    src/libnfc-nci/nfc/tags/rw_t2t_ndef.c
    src/libnfc-nci/nfc/tags/rw_t1t.c
    src/libnfc-nci/nfc/tags/ce_t4t.c
    src/libnfc-nci/nfc/tags/rw_main.c
    src/libnfc-nci/nfc/tags/rw_t3t.c
    src/libnfc-nci/nfc/tags/ce_main.c
    src/libnfc-nci/nfc/tags/rw_t1t_ndef.c
    src/libnfc-nci/nfc/ndef/ndef_utils.c
    src/libnfc-nci/nfc/ndef/ndef_cho_utils.c
    src/libnfc-nci/adaptation/NfcAdaptation.cpp
    src/libnfc-nci/adaptation/config.cpp
    src/libnfc-nci/adaptation/OverrideLog.cpp
    src/libnfc-nci/adaptation/android_logmsg.cpp
    src/libnfc-nci/adaptation/CrcChecksum.cpp
)

if(ACTIVE_TML STREQUAL ALT OR ACTIVE_TML STREQUAL I2C)
    target_include_directories(${PACKAGE_NAME} PRIVATE
        src/halimpl/include
        src/halimpl/pn54x/utils
        src/halimpl/pn54x/inc
        src/halimpl/pn54x/common
        src/halimpl/pn54x/dnld
        src/halimpl/pn54x/hal
        src/halimpl/pn54x/log
        src/halimpl/pn54x/tml
        src/halimpl/pn54x/tml/i2c
        src/halimpl/pn54x/core
        src/halimpl/pn54x/self-test
    )
    target_sources(${PACKAGE_NAME} PRIVATE
        src/halimpl/pn54x/tml/phDal4Nfc_messageQueueLib.c
        src/halimpl/pn54x/tml/phOsalNfc_Timer.c
        src/halimpl/pn54x/tml/i2c/phTmlNfc_i2c.c
        src/halimpl/pn54x/tml/i2c/phTmlNfc.c
        src/halimpl/pn54x/dnld/phNxpNciHal_Dnld.c
        src/halimpl/pn54x/dnld/phDnldNfc_Internal.c
        src/halimpl/pn54x/dnld/phDnldNfc.c
        src/halimpl/pn54x/dnld/phDnldNfc_Utils.c
        src/halimpl/pn54x/log/phNxpLog.c
        src/halimpl/pn54x/self-test/phNxpNciHal_SelfTest.c
        src/halimpl/pn54x/hal/phNxpNciHal_NfcDepSWPrio.c
        src/halimpl/pn54x/hal/phNxpNciHal_dta.c
        src/halimpl/pn54x/hal/phNxpNciHal_ext.c
        src/halimpl/pn54x/hal/phNxpNciHal_Kovio.c
        src/halimpl/pn54x/hal/phNxpNciHal.c
        src/halimpl/pn54x/utils/phNxpNciHal_utils.c
        src/halimpl/pn54x/utils/phNxpConfig.cpp
    )
endif()

target_include_directories(${PACKAGE_NAME}
    PUBLIC
        src/include
    PRIVATE
        src/service/interface
        src/service/utils
        src/service/extns/inc
        src/service/extns/src/common
        src/service/extns/src/mifare
)
target_sources(${PACKAGE_NAME} PRIVATE
    src/service/utils/Mutex.cpp
    src/service/utils/IntervalTimer.cpp
    src/service/utils/CondVar.cpp
    src/service/interface/nativeNfcTag.cpp
    src/service/interface/NfcTag.cpp
    src/service/interface/nativeNdef.cpp
    src/service/interface/nativeNfcHandover.cpp
    src/service/interface/nativeNfcManager.cpp
    src/service/interface/nativeNfcSnep.cpp
    src/service/interface/nativeNfcLlcp.cpp
    src/service/interface/RoutingManager.cpp
    src/service/extns/src/mifare/phFriNfc_SmtCrdFmt.c
    src/service/extns/src/mifare/phNxpExtns_MifareStd.c
    src/service/extns/src/mifare/phFriNfc_MifStdFormat.c
    src/service/extns/src/mifare/phFriNfc_MifareStdMap.c
    src/service/extns/src/phNxpExtns.c
    src/service/linux_nfc_api.c
    src/service/linux_nfc_factory_api.c
)

add_subdirectory(demoapp)
