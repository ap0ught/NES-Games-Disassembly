# Assembly Scripts Documentation

This document explains the `assemble.bat` and `assemble.sh` scripts used to compile The Legend of Zelda NES ROM from disassembled source code.

## Overview

Both scripts serve the same purpose: assembling the disassembled 6502 assembly code back into a functional NES ROM file. The key difference is the target platform:

- **`assemble.bat`** - Windows batch script (262 lines, self-contained)
- **`assemble.sh`** - Unix/Linux/macOS shell script (32 lines, uses modular helper scripts)

## Prerequisites

### Windows (assemble.bat)
- **CC65 toolchain**: ca65 assembler and ld65 linker (must be installed separately)
- **Lua 5.3**: lua53.exe and lua53.dll (must be installed separately)
- **Windows Command Prompt** with batch file support

### Unix/Linux/macOS (assemble.sh)
- **CC65 toolchain**: ca65 assembler and ld65 linker
- **Lua 5.3.x**: lua interpreter
- **Bash shell** environment
- **Standard Unix utilities**: wc, awk, cat, diff, sha1sum/shasum

The shell script can automatically download and install CC65 and Lua if they're not found on the system.

## Configuration Options

Both scripts use the same configuration parameters with different syntax:

### File Settings
| Parameter | Windows | Unix | Description |
|-----------|---------|------|-------------|
| ROM filename | `file_name=_the_legend_of_zelda` | `NES_OUTPUT_SIMPLE_NAME=_the_legend_of_zelda` | Output ROM filename (without .nes extension) |
| Expected file size | `file_size=131088` | `NES_OUTPUT_FILE_SIZE=131088` | Expected ROM size in bytes for validation |
| Original ROM SHA-1 | `sha1_original=DAB79C84...` | `NES_OUTPUT_FILE_SHA1_ORIGINAL="DAB79C84..."` | SHA-1 checksum of original ROM for verification |

### Assembly Options
| Parameter | Windows | Unix | Description |
|-----------|---------|------|-------------|
| Fast assembly | `fast_assembly=1` | `NES_OUTPUT_FAST_ASSEMBLY=1` | 1 = fast mode (no debug files), 0 = debug mode |
| Listing filename | `listing_name=z_listing.asm` | `NES_OUTPUT_LISTING_NAME=z_listing.asm` | Combined listing file (debug mode only) |
| Debug filename | `debug_name=z_debug.txt` | `NES_OUTPUT_DEBUG_NAME=z_debug.txt` | Debug symbols file (debug mode only) |

### Backup Options
| Parameter | Windows | Unix | Description |
|-----------|---------|------|-------------|
| Create backups | `file_backup=0` | `NES_OUTPUT_FILE_BACKUP=0` | 1 = create .bak files, 0 = no backups |
| Check differences | `file_diff=0` | `NES_OUTPUT_FILE_DIFF=0` | 1 = compare with .old copy, 0 = no comparison |

### Additional Unix Options
| Parameter | Description |
|-----------|-------------|
| `NES_IGNORE_COMPILE_ASM_ARRAY` | Array of ASM files to skip during compilation |

## Assembly Process Workflow

### 1. Preparation Phase
- **Windows**: Runs `lua53 preparations.lua` to process source files
- **Unix**: Runs `lua preparations.lua` after checking dependencies

The preparation script processes the disassembled code, removing debugging information and preparing it for assembly.

### 2. Compilation Phase
Both scripts compile multiple bank files using the CA65 assembler:

**Banks compiled:**
- `copy_bank_00.asm` through `copy_bank_06.asm`
- `copy_bank_FF.asm`
- Additional banks: `bank_s1.asm`, `bank_s2.asm`, `bank___BF50_BFF9.asm` (On Unix, these banks are skipped if they are listed in the `NES_IGNORE_COMPILE_ASM_ARRAY` configuration parameter)

**Assembly options:**
- **Fast mode**: `ca65 -U <file>.asm` (no debug info)
- **Debug mode**: `ca65 -U -l <file>.lst -g <file>.asm` (with listing and debug files)

### 3. Linking Phase
The LD65 linker combines all object files into a single binary:

**Fast mode:**
```bash
ld65 -C ld65.cfg -o PRG_ROM.bin copy_bank_*.o
```

**Debug mode:**
```bash
ld65 -C ld65.cfg -o PRG_ROM.bin --dbgfile <debug_name> copy_bank_*.o
```

### 4. ROM Creation Phase
The final ROM is created by concatenating:
1. `header.bin` - NES header (16 bytes)
2. `PRG_ROM.bin` - Program ROM data
3. `CHR_ROM.chr` - Character ROM data (if exists)

**Windows**: `copy /B header.bin + PRG_ROM.bin + CHR_ROM.chr output.nes`
**Unix**: `cat header.bin PRG_ROM.bin CHR_ROM.chr > output.nes`

### 5. Cleanup Phase
- Deletes temporary files: `*.o`, `PRG_ROM.bin`, `copy_*`
- In debug mode, combines listing files into a single file

### 6. Validation Phase
Both scripts perform comprehensive validation:

#### File Size Check
Verifies the assembled ROM matches the expected size (131,088 bytes for Zelda).

#### SHA-1 Checksum Verification
Compares the SHA-1 hash of the assembled ROM with the original:
- **Windows**: Uses `certutil -hashfile` command
- **Unix**: Uses `sha1sum` or `shasum` (macOS)

If checksums match, displays: "Original SHA-1 checksum detected"

### 7. Backup Management
When enabled, provides backup and difference checking:

#### Backup Creation (`file_backup=1`)
- **Success**: Creates `.bak` copy of the ROM
- **Failure**: Restores ROM from `.bak` copy

#### Difference Checking (`file_diff=1`)
- Compares new ROM with `.old` copy
- Reports "Perfect match" or "Differences found"
- Creates new `.old` copy for next comparison

## Usage Examples

### Windows
```cmd
# Basic assembly (fast mode)
assemble.bat

# The script will:
# 1. Display "Assembling into _the_legend_of_zelda.nes"
# 2. Run preparation and compilation
# 3. Show "Assembled successfully" or error details
# 4. Auto-close after 30 seconds on success
```

### Unix/Linux/macOS
```bash
# Basic assembly
sh assemble.sh

# The script will:
# 1. Check and install dependencies if needed
# 2. Run preparation and compilation  
# 3. Display colored status messages
# 4. Show final results with file size validation
```

## Troubleshooting

### Common Issues

**File size mismatch:**
- Check if all source files are present
- Verify no modifications were made to the assembly files
- Ensure `preparations.lua` ran successfully

**SHA-1 checksum doesn't match:**
- This is normal if you've made intentional modifications
- Only means the ROM differs from the original
- ROM should still be functional

**Assembly errors:**
- Check CA65 assembler output for syntax errors
- Verify all `.asm` files are present
- Check file permissions (Unix systems)

**Missing dependencies (Unix):**
```bash
# Manual installation of CC65 (Ubuntu/Debian)
sudo apt-get install cc65

# Manual installation of Lua (Ubuntu/Debian)  
sudo apt-get install lua5.3

# macOS with Homebrew
brew install cc65 lua
```

### Debug Mode

To enable debug mode for troubleshooting:

**Windows**: Set `fast_assembly=0` in assemble.bat
**Unix**: Set `NES_OUTPUT_FAST_ASSEMBLY=0` in assemble.sh

This creates additional files:
- `z_listing.asm` - Combined assembly listing showing addresses and opcodes
- `z_debug.txt` - Debug symbols for use with debuggers
- Individual `.lst` files for each bank

### Error Messages

**Windows:**
- "Error: something's wrong, see above for more info" - Check file size and assembly output
- Auto-restore from backup if `file_backup=1`

**Unix:**
- Colored error messages with specific issue descriptions
- Automatic dependency installation prompts
- Detailed file size and checksum reporting

## File Structure

After successful assembly, you'll have:
```
The Legend of Zelda/
├── _the_legend_of_zelda.nes    # Final ROM file
├── _the_legend_of_zelda.bak    # Backup (if enabled)
├── _the_legend_of_zelda.old    # Previous version (if enabled)
├── z_listing.asm               # Assembly listing (debug mode)
├── z_debug.txt                 # Debug symbols (debug mode)
└── copy_bank_*.asm             # Generated by preparations.lua
```

## Technical Details

### ROM Specifications
- **Total size**: 131,088 bytes
- **Header**: 16 bytes (NES format)
- **PRG ROM**: 128KB (8 banks × 16KB)
- **CHR ROM**: Variable size
- **Mapper**: Depends on original game

### Assembly Toolchain
- **CA65**: 6502 cross-assembler from CC65 package
- **LD65**: Object file linker
- **Lua**: Script processor for source preparation

The scripts use the `-U` flag with CA65 to disable the need for `.import` directives, simplifying the disassembled code structure.

## Customization

To create custom configurations:

1. **Copy the script** to a new filename (e.g., `assemble_custom.bat`)
2. **Modify configuration** variables at the top
3. **Update preparations.lua** if changing the output filename
4. **Run the custom script** instead of the original

This allows multiple build configurations for different ROM variants or hacking projects.