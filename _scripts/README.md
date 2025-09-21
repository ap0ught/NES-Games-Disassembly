# Scripts Directory

This directory contains the shared build system scripts used by all NES game disassemblies in this repository. These scripts provide a standardized way to assemble 6502 assembly source code into byte-perfect NES ROM files.

## Overview

The build system is designed to:
- Automatically install and configure required dependencies (cc65 cross-assembler, Lua 5.3)
- Process assembly source files through a Lua preprocessing step
- Assemble the code using the cc65 toolchain
- Generate NES ROM files with proper headers and graphics data
- Verify output ROM files match original checksums

## Script Files

### `os_support.sh`
**Purpose**: Cross-platform compatibility and utility functions

Provides:
- Color-coded output functions (`echoinfo`, `echowarn`, `echoerror`)
- Operating system detection (macOS, Linux)
- Package installation support (apt-get, yum)
- Date/time utilities for different platforms
- Error handling and validation functions

### `env.sh`
**Purpose**: Environment setup and dependency management

Handles:
- Automatic detection of cc65 cross-assembler
- Automatic detection of Lua 5.3 interpreter
- Interactive installation of missing dependencies
- PATH configuration for build tools
- Version management for cc65 (v2.19) and Lua (v5.3.6)

### `assemble_header.sh`
**Purpose**: Pre-assembly preparation and validation

Performs:
- Dependency verification (cc65 and Lua availability)
- Lua preprocessing of assembly files via `preparations.lua`
- Assembly of individual bank files using ca65
- Handling of ignored/excluded assembly files
- Generation of listing and debug files (when enabled)

### `assemble_standard.sh`
**Purpose**: Main ROM assembly and linking

Executes:
- Linking of assembled object files using ld65 linker
- Combination of NES header, PRG ROM, and CHR ROM data
- Generation of final NES ROM file
- Debug file creation (when enabled)

### `assemble_footer.sh`
**Purpose**: Post-assembly cleanup and verification

Provides:
- Cleanup of temporary files and build artifacts
- ROM file size validation
- SHA-1 checksum verification against original ROMs
- Backup and diff file management
- Success/failure reporting

## Usage

These scripts are automatically sourced by each game's `assemble.sh` script in the following order:

1. `os_support.sh` - Load utility functions
2. `env.sh` - Setup build environment
3. `assemble_header.sh` - Prepare and assemble source files
4. `assemble_standard.sh` - Link and create ROM
5. `assemble_footer.sh` - Cleanup and verify

## Configuration Variables

Game-specific `assemble.sh` scripts configure the build process using these variables:

- `NES_OUTPUT_SIMPLE_NAME` - Base name for output ROM file
- `NES_OUTPUT_FILE_SIZE` - Expected ROM file size in bytes
- `NES_OUTPUT_FILE_SHA1_ORIGINAL` - SHA-1 checksum of original ROM
- `NES_OUTPUT_FAST_ASSEMBLY` - Enable fast build mode (skip listings)
- `NES_IGNORE_COMPILE_ASM_ARRAY` - Array of assembly files to skip

## Dependencies

The scripts automatically handle installation of:
- **cc65 v2.19**: 6502 cross-assembler toolchain (note: some systems may install v2.18)
- **Lua 5.3.x**: Script preprocessing engine
- **System packages**: readline-dev (Linux), build tools

## Platform Support

Tested and supported on:
- **Linux**: Ubuntu, Debian, CentOS, RHEL
- **macOS**: Recent versions with Xcode command line tools
- **Windows**: MinGW/MSYS2 environment

## Build Process Flow

1. **Environment Check**: Verify cc65 and Lua are available
2. **Preprocessing**: Run Lua script to prepare assembly files
3. **Assembly**: Compile each bank file to object code
4. **Linking**: Combine objects into program ROM
5. **ROM Creation**: Merge header + PRG + CHR into NES file
6. **Verification**: Check size and checksum match original

All builds complete in under 2 seconds and produce byte-perfect ROM files.