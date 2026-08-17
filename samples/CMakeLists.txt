##############################################################################
##
##  CMakeLists.txt for the Detours samples.
##
##  Microsoft Research Detours Package
##
##  Copyright (c) Microsoft Corporation.  All rights reserved.
##
##  cping (needs a MIDL-generated RPC proxy/stub), tryman (needs csc.exe/.NET and only
##  works with the VS generator), and opengl (already excluded from NMake's own default
##  build) are intentionally not ported here.
##

# The original NMake build only elevates specific warnings to errors (/we4777 /we4800, applied
# below via DETOURS_SAMPLE_COMPILE_OPTIONS) -- unlike src/tests, it never used a blanket /WX.
# These are unmodified vendor sample sources that predate current MSVC warning levels, so the
# root CMakeLists.txt's global CMAKE_COMPILE_WARNING_AS_ERROR is turned back off just for this
# subdirectory (a normal-variable override, scoped to this add_subdirectory() call only).
set(CMAKE_COMPILE_WARNING_AS_ERROR OFF)

# Mirrors the $(DETOURS_TARGET_PROCESSOR) branching in samples/common.mak / system.mak, which
# several samples key their fixed base addresses and arch-specific sources off of. Only the
# architectures this CMake port actually targets are handled; plain (non-EC) ARM64 isn't a
# supported CMake platform here, unlike the NMake build.
if(CMAKE_GENERATOR_PLATFORM MATCHES "ARM64EC" OR CMAKE_VS_PLATFORM_NAME MATCHES "ARM64EC"
   OR CMAKE_CXX_FLAGS MATCHES "arm64EC" OR CMAKE_C_FLAGS MATCHES "arm64EC")
    set(DETOURS_SAMPLE_ARCH "ARM64EC")
elseif(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(DETOURS_SAMPLE_ARCH "X64")
else()
    set(DETOURS_SAMPLE_ARCH "X86")
endif()

# Matches samples/common.mak's CFLAGS/LINKFLAGS.
set(DETOURS_SAMPLE_COMPILE_OPTIONS /W4 /we4777 /we4800 /Od)
set(DETOURS_SAMPLE_LINK_OPTIONS /INCREMENTAL:NO /PROFILE /RELEASE /NODEFAULTLIB:oldnames.lib)

function(detours_sample_common_settings target)
    if(MSVC)
        target_compile_options(${target} PRIVATE ${DETOURS_SAMPLE_COMPILE_OPTIONS})
        target_compile_definitions(${target} PRIVATE $<$<CONFIG:Debug>:DETOUR_DEBUG=1>)
        target_link_options(${target} PRIVATE ${DETOURS_SAMPLE_LINK_OPTIONS})
    endif()
    # Every sample links against syelog.lib + detours.lib, per samples/common.mak's DEPS/LIBS.
    target_link_libraries(${target} PRIVATE Microsoft::detours syelog)
endfunction()

function(add_detour_sample_exe name)
    cmake_parse_arguments(ARG "" "" "SOURCES;LIBS" ${ARGN})
    add_executable(${name} ${ARG_SOURCES})
    detours_sample_common_settings(${name})
    if(ARG_LIBS)
        target_link_libraries(${name} PRIVATE ${ARG_LIBS})
    endif()
endfunction()

# Samples that bypass this (edll1x/2x/3x, tdll1x..9x, findfunc's target.dll) use plain
# add_library(... SHARED ...) + detours_sample_common_settings() directly instead, since they
# need no export flag, or a single non-default export, rather than this default injection.
function(add_detour_sample_dll name)
    cmake_parse_arguments(ARG "" "" "SOURCES;EXPORTS;LIBS" ${ARGN})
    add_library(${name} SHARED ${ARG_SOURCES})
    detours_sample_common_settings(${name})
    if(ARG_LIBS)
        target_link_libraries(${name} PRIVATE ${ARG_LIBS})
    endif()
    if(NOT ARG_EXPORTS)
        set(ARG_EXPORTS "DetourFinishHelperProcess,@1,NONAME")
    endif()
    foreach(exp IN LISTS ARG_EXPORTS)
        target_link_options(${name} PRIVATE "/EXPORT:${exp}")
    endforeach()
endfunction()

##############################################################################
## syelog - foundational shared dependency almost everything else links.

add_library(syelog STATIC syelog/syelog.cpp)
target_sources(syelog PUBLIC
    FILE_SET HEADERS
    BASE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/syelog
    FILES syelog/syelog.h
)
# syelog.cpp itself #includes detours.h (for the DetourFinishHelperProcess ref counting).
target_link_libraries(syelog PRIVATE Microsoft::detours)
if(MSVC)
    target_compile_options(syelog PRIVATE ${DETOURS_SAMPLE_COMPILE_OPTIONS})
    target_compile_definitions(syelog PRIVATE $<$<CONFIG:Debug>:DETOUR_DEBUG=1>)
endif()

# syelogd doesn't link syelog.lib/detours.lib at all (its own Makefile recipe omits $(LIBS)),
# so it bypasses add_detour_sample_exe rather than picking those up unintentionally.
add_executable(syelogd syelog/syelogd.cpp)
if(MSVC)
    target_compile_options(syelogd PRIVATE ${DETOURS_SAMPLE_COMPILE_OPTIONS})
    target_compile_definitions(syelogd PRIVATE $<$<CONFIG:Debug>:DETOUR_DEBUG=1>)
    target_link_options(syelogd PRIVATE ${DETOURS_SAMPLE_LINK_OPTIONS})
endif()
target_link_libraries(syelogd PRIVATE ws2_32.lib mswsock.lib advapi32.lib)

add_detour_sample_exe(sltest SOURCES syelog/sltest.cpp LIBS kernel32.lib)
add_detour_sample_exe(sltestp SOURCES syelog/sltestp.cpp LIBS kernel32.lib)

##############################################################################
## setdll, withdll - built early because other samples' tests shell out to these binaries.
## The NMake "option:" cross-bitness-copy targets are intentionally not ported.

add_detour_sample_exe(setdll SOURCES setdll/setdll.cpp LIBS kernel32.lib)
add_detour_sample_exe(withdll SOURCES withdll/withdll.cpp LIBS kernel32.lib)

##############################################################################
## The bulk of the "standard" samples.

add_detour_sample_dll(wrotei SOURCES comeasy/wrotei.cpp comeasy/wrotei.rc LIBS kernel32.lib ole32.lib)
add_detour_sample_exe(comeasy SOURCES comeasy/comeasy.cpp LIBS kernel32.lib ole32.lib)
target_link_options(comeasy PRIVATE /FIXED:NO)

add_detour_sample_exe(commem SOURCES commem/commem.cpp LIBS kernel32.lib ole32.lib)

# disas/unk.cpp, arm.asm, ia64.asm are not referenced by any target in the NMake build either.
if(DETOURS_SAMPLE_ARCH STREQUAL "X86")
    set(DETOURS_DISAS_ARCH_SOURCE disas/x86.cpp)
else()
    set(DETOURS_DISAS_ARCH_SOURCE disas/x64.asm)
endif()
add_detour_sample_exe(disas SOURCES disas/disas.cpp ${DETOURS_DISAS_ARCH_SOURCE} LIBS kernel32.lib)
target_link_options(disas PRIVATE /ENTRY:WinMainCRTStartup)

add_library(dtarge SHARED dtest/dtarge.cpp dtest/dtarge.rc)
detours_sample_common_settings(dtarge)
target_link_libraries(dtarge PRIVATE kernel32.lib)
target_link_options(dtarge PRIVATE
    /EXPORT:Target0 /EXPORT:Target1 /EXPORT:Target2 /EXPORT:Target3 /EXPORT:Target4
    /EXPORT:Target5 /EXPORT:Target6 /EXPORT:Target7 /EXPORT:Target8 /EXPORT:Target9
    /EXPORT:Target10 /EXPORT:Target11 /EXPORT:Target12 /EXPORT:Target13 /EXPORT:Target14
    /EXPORT:Target15 /EXPORT:Target16 /EXPORT:TargetV /EXPORT:TargetR
)
add_detour_sample_exe(dtest SOURCES dtest/dtest.cpp LIBS kernel32.lib dtarge)
target_link_options(dtest PRIVATE /ENTRY:WinMainCRTStartup)

add_detour_sample_exe(dumpe SOURCES dumpe/dumpe.cpp LIBS kernel32.lib)
add_detour_sample_exe(dumpi SOURCES dumpi/dumpi.cpp LIBS kernel32.lib)

# echonul.exe doubles as a DLL-style import target (echofx links against it): it needs
# ENABLE_EXPORTS instead of add_library(SHARED), an extra /Zl compile flag, and its link line
# omits Microsoft::detours/syelog entirely (unlike every other sample here).
add_executable(echonul echo/main.cpp echo/echonul.cpp)
if(MSVC)
    target_compile_options(echonul PRIVATE ${DETOURS_SAMPLE_COMPILE_OPTIONS})
    if(NOT DETOURS_SAMPLE_ARCH STREQUAL "ARM64EC")
        # /Zl (omit default library name) also strips the /DEFAULTLIB pragma the ARM64EC
        # compiler otherwise emits for its EC/native exit-thunk dispatch glue, which this
        # exports-heavy target needs (LNK2019 on __os_arm64x_dispatch_* without it) -- an
        # ABI concern that didn't exist when this NMake-era sample picked up /Zl.
        target_compile_options(echonul PRIVATE /Zl)
    endif()
    target_compile_definitions(echonul PRIVATE $<$<CONFIG:Debug>:DETOUR_DEBUG=1>)
    target_link_options(echonul PRIVATE ${DETOURS_SAMPLE_LINK_OPTIONS})
endif()
set_target_properties(echonul PROPERTIES ENABLE_EXPORTS ON)
target_link_options(echonul PRIVATE /EXPORT:Echo)

add_detour_sample_dll(echofx
    SOURCES echo/echofx.cpp echo/echofx.rc
    EXPORTS "DetourFinishHelperProcess,@1,NONAME" "Mine_Echo"
    LIBS kernel32.lib echonul
)

add_detour_sample_exe(region SOURCES region/region.cpp LIBS kernel32.lib)

# trcbld carries a redundant extra /release flag in its Makefile recipe; DETOURS_SAMPLE_LINK_OPTIONS
# already includes /RELEASE, so it's not worth restating here.
add_detour_sample_dll(trcbld SOURCES tracebld/trcbld.cpp tracebld/trcbld.rc LIBS kernel32.lib)
add_detour_sample_exe(tracebld SOURCES tracebld/tracebld.cpp LIBS kernel32.lib)
target_link_options(tracebld PRIVATE /FIXED:NO)

add_detour_sample_dll(trclnk SOURCES tracelnk/trclnk.cpp tracelnk/trclnk.rc LIBS kernel32.lib)
add_detour_sample_dll(trcmem SOURCES tracemem/trcmem.cpp tracemem/trcmem.rc LIBS kernel32.lib)
add_detour_sample_dll(trcreg SOURCES tracereg/trcreg.cpp tracereg/trcreg.rc LIBS kernel32.lib advapi32.lib)
add_detour_sample_dll(trcser SOURCES traceser/trcser.cpp traceser/trcser.rc LIBS kernel32.lib)
add_detour_sample_dll(trcssl SOURCES tracessl/trcssl.cpp tracessl/trcssl.rc LIBS kernel32.lib ws2_32.lib secur32.lib)
add_detour_sample_dll(trctcp SOURCES tracetcp/trctcp.cpp tracetcp/trctcp.rc LIBS kernel32.lib ws2_32.lib)

# testapi.cpp #includes trcapi.cpp and _win32.cpp directly (as does trcapi.cpp with _win32.cpp),
# so each target compiles only its own top-level source, matching the NMake dependency lines.
add_detour_sample_dll(trcapi
    SOURCES traceapi/trcapi.cpp traceapi/trcapi.rc
    LIBS kernel32.lib gdi32.lib user32.lib shell32.lib advapi32.lib ole32.lib ws2_32.lib
)
add_detour_sample_exe(testapi
    SOURCES traceapi/testapi.cpp
    LIBS kernel32.lib gdi32.lib user32.lib shell32.lib advapi32.lib ole32.lib ws2_32.lib
)
target_link_options(testapi PRIVATE /FIXED:NO)

##############################################################################
## Samples needing explicit deviation from the common flags.

add_detour_sample_dll(slept
    SOURCES slept/slept.cpp slept/slept.rc
    EXPORTS "DetourFinishHelperProcess,@1,NONAME" "TimedSleepEx" "UntimedSleepEx" "GetSleptTicks" "TestTicks" "TestTicksEx"
    LIBS kernel32.lib
)
if(MSVC)
    # Re-enables /O2, appended after add_detour_sample_dll's /Od so it wins on the cl.exe command line.
    target_compile_options(slept PRIVATE /O2)
endif()

add_detour_sample_dll(dslept
    SOURCES slept/dslept.cpp slept/dslept.rc
    EXPORTS "DetourFinishHelperProcess,@1,NONAME" "TimedSleepEx" "UntimedSleepEx" "GetSleptTicks"
    LIBS kernel32.lib
)

# verify.cpp is intentionally not listed here: slept.cpp/sleepold.cpp/sleepnew.cpp/sleepbed.cpp/
# dslept.cpp above and below all end with #include "verify.cpp" (textual, no header guard), so
# listing it as its own SOURCES entry would compile it a second time as an independent
# translation unit -- one that lacks the <windows.h> each of those files includes beforehand,
# which detours.h needs to detect the target architecture.
add_detour_sample_exe(sleepold SOURCES slept/sleepold.cpp LIBS kernel32.lib)
target_link_options(sleepold PRIVATE /FIXED:NO)
add_detour_sample_exe(sleepnew SOURCES slept/sleepnew.cpp LIBS kernel32.lib slept)
target_link_options(sleepnew PRIVATE /FIXED:NO)
add_detour_sample_exe(sleepbed SOURCES slept/sleepbed.cpp LIBS kernel32.lib)
target_link_options(sleepbed PRIVATE /FIXED:NO)

add_detour_sample_dll(simple
    SOURCES simple/simple.cpp simple/simple.rc
    EXPORTS "DetourFinishHelperProcess,@1,NONAME" "TimedSleepEx"
    LIBS kernel32.lib
)
add_detour_sample_exe(sleep5 SOURCES simple/sleep5.cpp LIBS kernel32.lib)

add_detour_sample_dll(simple_safe
    SOURCES simple_safe/simple_safe.cpp simple_safe/simple_safe.rc
    EXPORTS "DetourFinishHelperProcess,@1,NONAME" "TimedSleepEx"
    LIBS kernel32.lib
)
if(MSVC)
    target_compile_options(simple_safe PRIVATE /std:c++14)
endif()
# simple_safe/sleep5.cpp is a byte-for-byte copy of simple/sleep5.cpp (NMake builds it twice,
# once per sample directory); kept as a second target/binary to preserve that, under a distinct
# CMake target name since "sleep5" is already taken above.
add_detour_sample_exe(sleep5_safe SOURCES simple_safe/sleep5.cpp LIBS kernel32.lib)
set_target_properties(sleep5_safe PROPERTIES OUTPUT_NAME sleep5_safe)

add_detour_sample_exe(payload SOURCES payload/payload.cpp LIBS kernel32.lib)
add_detour_sample_exe(payloadtarget SOURCES payload/payloadtarget.cpp LIBS kernel32.lib)
if(MSVC)
    target_compile_options(payload PRIVATE /EHsc)
    target_compile_options(payloadtarget PRIVATE /EHsc)
endif()

# X86/X64 only in its own Makefile (excludes ARM64EC too, unlike disas above).
if(DETOURS_SAMPLE_ARCH STREQUAL "X86" OR DETOURS_SAMPLE_ARCH STREQUAL "X64")
    if(DETOURS_SAMPLE_ARCH STREQUAL "X64")
        set(DETOURS_DALLOC_ARCH_SOURCE dynamic_alloc/x64.asm)
    else()
        set(DETOURS_DALLOC_ARCH_SOURCE dynamic_alloc/x86.asm)
    endif()
    # Output is named dalloc(.exe), matching $(TARGET_NAME) in dynamic_alloc/Makefile.
    add_detour_sample_exe(dalloc SOURCES dynamic_alloc/main.cpp ${DETOURS_DALLOC_ARCH_SOURCE} LIBS kernel32.lib user32.lib)
    if(MSVC)
        target_compile_options(dalloc PRIVATE /EHsc)
        if(DETOURS_SAMPLE_ARCH STREQUAL "X86")
            # x86.asm has no .SAFESEH directive (never did in the NMake build either), so it
            # can't be marked as SafeSEH-compliant; since main.cpp's cl.exe-generated object
            # *is* compliant, link.exe would otherwise require -- and fail to find -- SafeSEH
            # info from every module, including this one (LNK2026/LNK1281 on x86 only).
            target_link_options(dalloc PRIVATE /SAFESEH:NO)
        endif()
    endif()
endif()

##############################################################################
## Plain samples with no shared DLL/EXE pairing.

add_detour_sample_exe(member SOURCES member/member.cpp LIBS kernel32.lib)
add_detour_sample_exe(impmunge SOURCES impmunge/impmunge.cpp LIBS kernel32.lib imagehlp.lib)

# X86 only, per its own Makefile.
if(DETOURS_SAMPLE_ARCH STREQUAL "X86")
    add_detour_sample_exe(excep SOURCES excep/excep.cpp excep/firstexc.cpp LIBS kernel32.lib)
    target_link_options(excep PRIVATE /ENTRY:WinMainCRTStartup)
endif()

##############################################################################
## Fixed-base-address samples. edll1x/2x/3x and target.dll bypass add_detour_sample_dll's
## default export injection entirely (no export flag at all, or a single non-default export).

if(DETOURS_SAMPLE_ARCH STREQUAL "ARM64EC")
    set(DETOURS_EINST_EDLL1X_BASE 0x710000000)
    set(DETOURS_EINST_EDLL2X_BASE 0x720000000)
    set(DETOURS_EINST_EDLL3X_BASE 0x730000000)
else()
    set(DETOURS_EINST_EDLL1X_BASE 0x7100000)
    set(DETOURS_EINST_EDLL2X_BASE 0x7200000)
    set(DETOURS_EINST_EDLL3X_BASE 0x7300000)
endif()

add_library(edll1x SHARED einst/edll1x.cpp)
detours_sample_common_settings(edll1x)
target_link_libraries(edll1x PRIVATE kernel32.lib user32.lib)
target_link_options(edll1x PRIVATE /SUBSYSTEM:WINDOWS /BASE:${DETOURS_EINST_EDLL1X_BASE})

add_library(edll2x SHARED einst/edll2x.cpp)
detours_sample_common_settings(edll2x)
target_link_libraries(edll2x PRIVATE kernel32.lib user32.lib)
target_link_options(edll2x PRIVATE /SUBSYSTEM:CONSOLE /BASE:${DETOURS_EINST_EDLL2X_BASE})

add_library(edll3x SHARED einst/edll3x.cpp)
detours_sample_common_settings(edll3x)
target_link_libraries(edll3x PRIVATE kernel32.lib user32.lib)
target_link_options(edll3x PRIVATE /SUBSYSTEM:CONSOLE /BASE:${DETOURS_EINST_EDLL3X_BASE})

add_detour_sample_exe(einst SOURCES einst/einst.cpp LIBS kernel32.lib user32.lib edll1x edll2x edll3x)
target_link_options(einst PRIVATE /ENTRY:WinMainCRTStartup)

if(DETOURS_SAMPLE_ARCH STREQUAL "ARM64EC")
    set(DETOURS_FINDFUNC_TARGET_BASE 0x190000000)
    set(DETOURS_FINDFUNC_EXTEND_BASE 0x1a0000000)
else()
    set(DETOURS_FINDFUNC_TARGET_BASE 0x1900000)
    set(DETOURS_FINDFUNC_EXTEND_BASE 0x1a00000)
endif()

add_library(findfunc_target SHARED findfunc/target.cpp findfunc/target.rc)
set_target_properties(findfunc_target PROPERTIES OUTPUT_NAME target)
detours_sample_common_settings(findfunc_target)
target_link_libraries(findfunc_target PRIVATE kernel32.lib)
target_link_options(findfunc_target PRIVATE /EXPORT:Target /BASE:${DETOURS_FINDFUNC_TARGET_BASE})

add_detour_sample_dll(findfunc_extend SOURCES findfunc/extend.cpp findfunc/extend.rc LIBS kernel32.lib)
set_target_properties(findfunc_extend PROPERTIES OUTPUT_NAME extend)
target_link_options(findfunc_extend PRIVATE /BASE:${DETOURS_FINDFUNC_EXTEND_BASE})

# dbghelp.dll's external-tool-copy step from NMake is intentionally not ported.
add_detour_sample_exe(findfunc SOURCES findfunc/findfunc.cpp LIBS kernel32.lib findfunc_target)
target_link_options(findfunc PRIVATE /FIXED:NO)
add_detour_sample_exe(symtest SOURCES findfunc/symtest.cpp LIBS kernel32.lib findfunc_target)
target_link_options(symtest PRIVATE /FIXED:NO)

# X64 only for this CMake port (NMake also allows IA64, which isn't part of this port at all).
if(DETOURS_SAMPLE_ARCH STREQUAL "X64")
    # /FIXED requires ASLR off; link.exe defaults /DYNAMICBASE to ON regardless of CMake, so it
    # must be disabled explicitly here (the NMake build's own link lines pair the two the same way).
    set(DETOURS_TALLOC_LINK_OPTIONS /INCREMENTAL:NO /FIXED /DYNAMICBASE:NO /RELEASE /NODEFAULTLIB:oldnames.lib)

    # tdllNx.dll link lines use $(LINKFLAGS:/profile=/fixed) -- /FIXED replaces /PROFILE rather
    # than adding to it, so these use DETOURS_TALLOC_LINK_OPTIONS instead of the common list.
    function(add_talloc_dll name base)
        add_library(${name} SHARED talloc/${name}.cpp)
        if(MSVC)
            target_compile_options(${name} PRIVATE ${DETOURS_SAMPLE_COMPILE_OPTIONS})
            target_compile_definitions(${name} PRIVATE $<$<CONFIG:Debug>:DETOUR_DEBUG=1>)
            target_link_options(${name} PRIVATE ${DETOURS_TALLOC_LINK_OPTIONS})
        endif()
        target_link_libraries(${name} PRIVATE Microsoft::detours syelog kernel32.lib psapi.lib)
        set(subsystem CONSOLE)
        if(name STREQUAL "tdll1x")
            set(subsystem WINDOWS)
        endif()
        target_link_options(${name} PRIVATE /SUBSYSTEM:${subsystem} /NOENTRY /BASE:${base})
    endfunction()

    add_talloc_dll(tdll1x 0x280000000)
    add_talloc_dll(tdll2x 0x380000000)
    add_talloc_dll(tdll3x 0x480050000)
    add_talloc_dll(tdll4x 0x580000000)
    add_talloc_dll(tdll5x 0x680000000)
    add_talloc_dll(tdll6x 0x680010000)
    add_talloc_dll(tdll7x 0x680020000)
    add_talloc_dll(tdll8x 0x680030000)
    add_talloc_dll(tdll9x 0x680040000)

    add_detour_sample_exe(talloc
        SOURCES talloc/talloc.cpp
        LIBS kernel32.lib psapi.lib tdll1x tdll2x tdll3x tdll4x tdll5x tdll6x tdll7x tdll8x tdll9x
    )
    target_link_options(talloc PRIVATE /ENTRY:WinMainCRTStartup)
endif()

################################################################# End of File.
