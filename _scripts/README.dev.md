# Scripts Directory - Developer Documentation

This document provides detailed technical information about the build system implementation for developers who need to modify or extend the scripts.

## Architecture Overview

The build system follows a modular design with five specialized scripts that handle different phases of the ROM assembly process. Each script is designed to be sourced by game-specific `assemble.sh` files, creating a standardized build pipeline.

## Detailed Script Analysis

### `os_support.sh` - Cross-Platform Foundation

**Functions:**
```bash
Color_support()     # Initialize color variables for terminal output
echoerror()         # Print red error messages to stderr
echoinfo()          # Print green info messages
echowarn()          # Print yellow warning messages
Return()            # Check exit status and abort on failure
Installer()         # Install packages via apt-get or yum
```

**Platform Detection:**
- Sets `OS_VERSION` via `uname`
- Configures `MD5` command (md5sum vs md5)
- Handles date formatting differences between macOS and Linux
- Sets variables: `today`, `month` for timestamp operations

**Error Handling:**
- `Return()` function provides consistent error checking
- Exits with code 254 on any command failure
- Color-coded output for better user experience

### `env.sh` - Dependency Management

**Key Variables:**
```bash
CC65_VERSION=2.19
CC65_INSTALL_PACKAGE_VERSION=cc65-${CC65_VERSION}
LUA_INSTALL_PACKAGE_VERSION=lua-5.3.6
EXEC_PACKAGES_FOLDER=../_install_packages/bin
INSTALL_PACKAGES_FOLDER=../_install_packages/packages
```

**Installation Process:**
1. **cc65 Compiler:**
   - Downloads from GitHub releases: `https://codeload.github.com/cc65/cc65/tar.gz/refs/tags/V${CC65_VERSION}`
   - Extracts to temporary location
   - Compiles from source using `make`
   - Moves to execution folder for PATH inclusion

2. **Lua Interpreter:**
   - Downloads from official site: `https://www.lua.org/ftp/${LUA_INSTALL_PACKAGE_VERSION}.tar.gz`
   - Platform-specific compilation:
     - macOS: `make macosx test`
     - Linux: `make linux test` (with readline-dev dependency)
   - Installs to execution folder

**PATH Management:**
```bash
export PATH=${PATH}:${CC65_HOME}/bin:${CC65_HOME}/lib:${CC65_HOME}/include:${LUA_HOME}/src
```

### `assemble_header.sh` - Preprocessing and Assembly

**Dependency Verification:**
- Calls `check_cc65_env()` and `check_lua_env()` functions
- Interactive prompts for missing dependencies
- Automatic installation with user confirmation

**Lua Preprocessing:**
```bash
lua preparations.lua
```
- Executes game-specific Lua script
- Typically removes disassembler artifacts
- Prepares clean assembly files with "copy_" prefix

**Assembly Loop:**
```bash
for cp_bank_asm in `ls copy_bank_*.asm`; do
    # Check ignore list
    if [[ "${NES_IGNORE_COMPILE_ASM_ARRAY}" != "" ]]; then
        # Skip files in ignore array
    fi
    
    # Assemble with ca65
    if [ "${NES_OUTPUT_FAST_ASSEMBLY}" -eq 1 ]; then
        ca65 -U ${cp_bank_asm_without_suffix}.asm
    else
        ca65 -U -l ${cp_bank_asm_without_suffix}.lst -g ${cp_bank_asm_without_suffix}.asm
    fi
done
```

**ca65 Options:**
- `-U`: Disable requirement for .import declarations
- `-l`: Generate listing file (.lst)
- `-g`: Generate debug information

### `assemble_standard.sh` - Linking and ROM Creation

**Linking Phase:**
```bash
if [ "${NES_OUTPUT_FAST_ASSEMBLY}" -eq 1 ]; then
    ld65 -C ld65.cfg -o PRG_ROM.bin copy_bank_*.o
else
    ld65 -C ld65.cfg -o PRG_ROM.bin --dbgfile ${NES_OUTPUT_DEBUG_NAME} copy_bank_*.o
fi
```

**ld65 Options:**
- `-C ld65.cfg`: Use game-specific linker configuration
- `-o PRG_ROM.bin`: Output program ROM binary
- `--dbgfile`: Generate debug symbols file

**ROM Assembly:**
```bash
if [ -f "CHR_ROM.chr" ]; then
    cat header.bin PRG_ROM.bin CHR_ROM.chr > ${NES_OUTPUT_SIMPLE_NAME}.nes
else
    cat header.bin PRG_ROM.bin > ${NES_OUTPUT_SIMPLE_NAME}.nes
fi
```

**NES ROM Structure:**
1. **header.bin**: 16-byte iNES header with mapper info
2. **PRG_ROM.bin**: Program code and data
3. **CHR_ROM.chr**: Character/graphics data (optional)

### `assemble_footer.sh` - Verification and Cleanup

**File Operations:**
```bash
# Combine listing files (debug mode only)
if [ "${NES_OUTPUT_FAST_ASSEMBLY}" -eq 0 ]; then
    cat copy_*.lst > ${NES_OUTPUT_LISTING_NAME}
fi

# Cleanup temporary files
rm -f *.o PRG_ROM.bin copy_*
```

**Size Validation:**
```bash
output_rom_zize=`wc -c ${NES_OUTPUT_SIMPLE_NAME}.nes | awk '{print $1}'`
if [ "${output_rom_zize}" -eq ${NES_OUTPUT_FILE_SIZE} ]; then
    # Success path
else
    # Error handling and potential backup restoration
fi
```

**Checksum Verification:**
```bash
# Multi-platform SHA-1 calculation
if [[ $(command -v sha1sum) ]]; then
    sha1_current=$(sha1sum ${NES_OUTPUT_SIMPLE_NAME}.nes)
elif [[ $(command -v shasum) ]]; then
    sha1_current=$(shasum ${NES_OUTPUT_SIMPLE_NAME}.nes)
fi

# Convert to uppercase and compare
sha1_current=$(echo ${sha1_current} | awk '{print $1}' | tr '[:lower:]' '[:upper:]')
if [ "${sha1_current}" == "${NES_OUTPUT_FILE_SHA1_ORIGINAL}" ]; then
    echoinfo "Original SHA-1 checksum detected"
fi
```

**Backup Management:**
- `NES_OUTPUT_FILE_BACKUP=1`: Creates .bak files
- `NES_OUTPUT_FILE_DIFF=1`: Creates .old files for diff comparison
- Automatic backup restoration on build failures

## Build Configuration Variables

### Required Variables
- `NES_OUTPUT_SIMPLE_NAME`: Output ROM filename prefix
- `NES_OUTPUT_FILE_SIZE`: Expected final ROM size in bytes
- `NES_OUTPUT_FILE_SHA1_ORIGINAL`: Original ROM SHA-1 checksum (uppercase)

### Optional Variables
- `NES_OUTPUT_FAST_ASSEMBLY=1`: Skip listing/debug file generation
- `NES_OUTPUT_LISTING_NAME`: Combined listing filename (default: z_listing.asm)
- `NES_OUTPUT_DEBUG_NAME`: Debug symbols filename (default: z_debug.txt)
- `NES_OUTPUT_FILE_BACKUP=0`: Enable .bak file creation
- `NES_OUTPUT_FILE_DIFF=0`: Enable .old file creation for diffs
- `NES_IGNORE_COMPILE_ASM_ARRAY=()`: Array of assembly files to skip

### Special Variables
- `BASH_EXEC_DIR`: Directory containing the assemble.sh script
- All scripts change to this directory before execution

## Error Handling Strategy

**Exit Codes:**
- `254`: Standard error exit code used throughout
- `0`: Success

**Error Sources:**
1. Missing dependencies (cc65, lua)
2. Assembly errors in ca65
3. Linking errors in ld65
4. File size mismatches
5. Checksum verification failures

**Recovery Mechanisms:**
- Automatic dependency installation prompts
- Backup restoration on build failures
- Detailed error messaging with color coding

## Performance Considerations

**Fast Assembly Mode:**
- `NES_OUTPUT_FAST_ASSEMBLY=1` skips listing and debug file generation
- Reduces build time from ~1.6s to ~0.07s for complex games
- Default mode for production builds

**File Management:**
- Temporary files are cleaned up automatically
- Object files (.o) are removed after linking
- Copy files (copy_*) are removed after completion

## Extending the Build System

**Adding New Platforms:**
1. Extend OS detection in `os_support.sh`
2. Add platform-specific date commands
3. Update package installation logic in `Installer()`
4. Test compilation paths in `env.sh`

**Adding New Tools:**
1. Create check function in `env.sh`
2. Add installation logic with version management
3. Update PATH export
4. Call check function in `assemble_header.sh`

**Modifying Assembly Process:**
1. Update ca65/ld65 options in respective scripts
2. Modify file patterns for different assembly structures
3. Extend ignore array functionality for complex exclusions