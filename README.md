# winscape

**winscape** is a Golang implementation of the *Bring Your Own Vulnerable Driver* (BYOVD) attack paradigm. It leverages the Bitdefender API Utility driver (`bdapiutil64.sys`) to obtain kernel-level privileges for terminating security-related processes. This project builds upon two prior works:

* The original proof-of-concept developed by [BlackSnufkin](https://github.com/BlackSnufkin/BYOVD/tree/main/BdApiUtil-Killer).
* The extended Golang rewrite [dead-av](https://github.com/carved4/dead-av), from which **winscape** continues development with further enhancements and refinements.

## Overview

This tool exploits multiple vulnerabilities in the Bitdefender API Utility driver to achieve kernel-assisted process termination. Once executed, it performs the following tasks:

* Embeds and extracts the vulnerable driver.
* Installs the driver as a Windows service.
* Periodically enumerates active processes.
* Identifies processes associated with security software (e.g., EDR, antivirus, monitoring agents).
* Terminates identified processes using kernel-level `DeviceIoControl` calls.
* Currently targets 441 security products across a wide range of vendors.

## Technical Approach

### Driver Exploitation

The attack leverages several vulnerabilities present in `bdapiutil64.sys`, including:

* **Arbitrary process termination** via `IOCTL` code `0x800024b4`.
* **Controllable process handles** through `ObOpenObjectByPointer`.
* **Arbitrary memory read/write primitives** exposed through multiple `IOCTL` codes.
* **File creation with attacker-controlled `ObjectName`** via `IoCreateFile`.

### Attack Workflow

1. The embedded vulnerable driver is extracted to the filesystem.
2. The driver is registered and launched as a Windows service using the Service Control Manager (SCM).
3. A handle to the driver device (`\\.\\bdapiutil`) is obtained.
4. Running processes are enumerated using `CreateToolhelp32Snapshot`.
5. For each targeted process:

   * The process identifier (PID) is passed to the driver via `DeviceIoControl` using `IOCTL 0x800024b4`.
   * The driver forcibly terminates the process at kernel privilege level.
6. This scanning and termination loop is repeated at two-second intervals.

### Targeted Software

The tool is designed to neutralize major endpoint protection and monitoring solutions, including but not limited to:

* **Microsoft Defender** (e.g., `MsMpEng.exe`, `MpCmdRun.exe`, `NisSrv.exe`)
* **CrowdStrike Falcon** (e.g., `CSFalconService.exe`, `CSFalconContainer.exe`)
* **SentinelOne** (e.g., `SentinelAgent.exe`, `SentinelHelperService.exe`)
* **Carbon Black** (e.g., `cb.exe`, `cbcomms.exe`, `cbstream.exe`)
* **Symantec** (e.g., `ccSvcHst.exe`, `NortonSecurity.exe`, `sepmaster.exe`)
* **Kaspersky** (e.g., `avp.exe`, `avpui.exe`, `klnagent.exe`)
* **Trend Micro** (e.g., `tmbmsrv.exe`, `tmccsf.exe`, `tmlwfmgr.exe`)
* **Bitdefender** (e.g., `bdagent.exe`, `vsserv.exe`, `bdservicehost.exe`)
* **Malwarebytes** (e.g., `mbamservice.exe`, `mbamtray.exe`)
* **Analysis Tools** (e.g., `ProcessHacker.exe`, `ProcExp.exe`, `Wireshark.exe`)

## Usage

```bash
# Compilation
go build -o winscape.exe cmd/main.go

# Execution (requires administrative privileges)
./winscape.exe
```

**Note:** Administrator privileges are required due to the need for driver installation and kernel-level interactions.

## Exploitation Vectors in `bdapiutil64.sys`

### 1. Arbitrary Process Termination

```json
{
  "title": "Arbitrary process termination",
  "description": "ZwTerminateProcess with controllable handle",
  "eval": {
    "IoControlCode": "0x800024b4",
    "InputBufferLength": "0x4"
  }
}
```

### 2. Memory Read/Write Primitives

```json
{
  "title": "Read/write controllable address",
  "description": "Direct memory access",
  "eval": {
    "IoControlCode": "0x800021a0",
    "InputBufferLength": "0x10"
  }
}
```

### 3. File System Manipulation

```json
{
  "title": "Controllable ObjectName in ObjectAttributes",
  "description": "IoCreateFile invocation",
  "eval": {
    "IoControlCode": "0x8000264c",
    "InputBufferLength": "0x208"
  }
}
```

## Acknowledgments

* Original vulnerability research and proof-of-concept implementation by [@BlackSnufkin](https://github.com/BlackSnufkin/BYOVD/tree/main/BdApiUtil-Killer).
* Golang reimplementation and initial extensions in [dead-av](https://github.com/carved4/dead-av).
* **winscape** continues this line of development, introducing additional features such as:

  * Embedded driver extraction.
  * Expanded process termination targeting.
  * Kernel module enumeration and retrieval of the `ntoskrnl` base address.

