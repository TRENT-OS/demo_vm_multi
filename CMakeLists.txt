#
# Copyright (C) 2023, HENSOLDT Cyber GmbH
#

cmake_minimum_required(VERSION 3.8.2)

project(demo_vm_multi C)

#-------------------------------------------------------------------------------
# use the SDK
find_package(os-sdk REQUIRED)
os_sdk_set_defaults()
set(LibUtilsDefaultZfLogLevel 2 CACHE STRING "")

# Kernel settings
set(KernelArmHypervisorSupport ON CACHE BOOL "" FORCE)
set(KernelArmVtimerUpdateVOffset OFF CACHE BOOL "" FORCE)
set(KernelArmDisableWFIWFETraps ON CACHE BOOL "" FORCE)
set(LibUSB OFF CACHE BOOL "" FORCE)
set(VmInitRdFile ON CACHE BOOL "" FORCE)
set(VmVirtioConsole ON CACHE BOOL "" FORCE)
set(CapDLLoaderMaxObjects 8192 CACHE STRING "" FORCE)
set(VmVirtioNetArping OFF CACHE BOOL "" FORCE)
set(VmVirtioNetVirtqueue ON CACHE BOOL "" FORCE)
set(VmPCISupport ON CACHE BOOL "" FORCE)

include("plat/${PLATFORM}/plat.cmake")

os_sdk_setup()
os_sdk_import_camkes_vm()
os_sdk_import_from_global_components(
    "components/FileServer"
    "components/SerialServer"
    "components/TimeServer"
    "components/VirtQueue"
)

CAmkESAddCPPInclude("plat/${PLATFORM}")


#-------------------------------------------------------------------------------
# config
AddCamkesCPPFlag(cpp_flags CONFIG_VARS VmEmmc2NoDMA VmVchan)

if(NOT KernelPlatformQEMUArmVirt)
    AddCamkesCPPFlag(cpp_flags CONFIG_VARS VmVUSB)
endif()

if(KernelPlatformTK1)
    AddCamkesCPPFlag(cpp_flags CONFIG_VARS Tk1DeviceFwd Tk1Insecure)
endif()


#-------------------------------------------------------------------------------
# VM Images

list(
    APPEND CMAKE_MODULE_PATH
    "${CMAKE_CURRENT_LIST_DIR}/camkes-vm-images"
    "${CMAKE_CURRENT_LIST_DIR}/camkes-vm-linux"
)
find_package(camkes-vm-images REQUIRED)
find_package(camkes-vm-linux REQUIRED)
include("${CAMKES_VM_LINUX_HELPERS_PATH}")

# images for the selected platform
set(VM_IMG_LINUX "${CAMKES_VM_IMAGES_DIR}/${PLATFORM}/linux")
set(VM_IMG_ROOTFS "${CAMKES_VM_IMAGES_DIR}/${PLATFORM}/rootfs.cpio.gz")

# Setup our passthrough vm overlay
AddFileToOverlayDir(
    "inittab"
    ${CMAKE_CURRENT_SOURCE_DIR}/overlay_files/inittab_hvc0
    "etc"
    overlay_vm0
)
AddFileToOverlayDir(
    "S90bridge_setup"
    ${CMAKE_CURRENT_SOURCE_DIR}/overlay_files/vm0_bridge_setup.sh
    "etc/init.d"
    overlay_vm0
)
AddOverlayDirToRootfs(
    overlay_vm0
    ${VM_IMG_ROOTFS}
    "buildroot"
    "rootfs_install"
    vm0_output_overlayed_rootfs_location
    rootfs_target_vm0
    GZIP
)

# Setup our client vm overlays
AddFileToOverlayDir(
    "inittab"
    ${CMAKE_CURRENT_SOURCE_DIR}/overlay_files/inittab_hvc0
    "etc"
    overlay_client_vm
)
AddFileToOverlayDir(
    "S90client_net_vm"
    ${CMAKE_CURRENT_SOURCE_DIR}/overlay_files/vm_client_net_setup.sh
    "etc/init.d"
    overlay_client_vm
)
AddOverlayDirToRootfs(
    overlay_client_vm
    ${VM_IMG_ROOTFS}
    "buildroot"
    "rootfs_install"
    client_output_overlayed_rootfs_location
    rootfs_target_client_vm
    GZIP
)


# Since this needs the file VM_IMG_ROOTFS_PATCHED anyway, there is an implicit
# dependency already. Thus, there is no need to explicitly add a dependency on
# the target 'rootfs_target' here.
DefineCAmkESVMFileServer(
    INSTANCE "fileServer"
    FILES
        "linux:${VM_IMG_LINUX}"
        "linux-initrd-vm0:${vm0_output_overlayed_rootfs_location}"
        "linux-initrd-vm-client:${client_output_overlayed_rootfs_location}"
)

#-------------------------------------------------------------------------------
os_sdk_create_CAmkES_system("vm_multi.camkes")
