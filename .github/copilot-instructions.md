# NES Games Disassembly

This repository contains 25+ complete disassemblies of NES (Nintendo Entertainment System) games that are equal to the original ROMs, compilable and editable. Each game can be assembled into a byte-perfect NES ROM file with matching SHA-1 checksums.

Always reference these instructions first and fallback to search or bash commands only when you encounter unexpected information that does not match the info here.

## Working Effectively

### Bootstrap and build dependencies
- Install basic build tools: `sudo apt update && sudo apt install -y build-essential curl tar gzip libreadline-dev bc`
- Install cc65 cross-assembler for 6502 processors:
  ```bash
  cd /tmp && wget https://github.com/cc65/cc65/archive/refs/tags/V2.19.tar.gz
  tar xzf V2.19.tar.gz && cd cc65-2.19
  make && sudo make install PREFIX=/usr/local
  ```
  Build takes 55 seconds. NEVER CANCEL. Set timeout to 120+ seconds.
- Install Lua 5.3: `sudo apt install -y lua5.3 && sudo ln -sf /usr/bin/lua5.3 /usr/bin/lua`
- Verify installation: `cc65 --version && ca65 --version && ld65 --version && lua -v`

### Build individual games
- Navigate to any game directory (e.g., `cd "Excitebike"`)
- Run: `bash assemble.sh`
- Build times: 0.07-1.6 seconds per game. NEVER CANCEL builds.
- Output: `_[gamename].nes` file with correct SHA-1 checksum verification

### Build all games
- All 25 games build successfully in ~7 seconds total
- Test command: Run from repository root to build all games:
  ```bash
  for dir in */; do
    if [ -f "$dir/assemble.sh" ]; then
      echo "Building $(basename "$dir")..."
      cd "$dir" && bash assemble.sh && cd ..
    fi
  done
  ```

### File structure understanding
- Each game directory contains:
  - `assemble.sh` / `assemble.bat` - Build scripts (use .sh on Linux/Mac)
  - `bank_*.asm` - Assembly source files for different ROM banks
  - `bank_ram.inc` - RAM variable definitions  
  - `bank_val.inc` - Constant value definitions
  - `preparations.lua` - Lua preprocessing script
  - `header.bin` - NES ROM header
  - `ld65.cfg` - Linker configuration
  - `CHR_ROM.chr` - Character/graphics data (if present)
- `_scripts/` directory contains shared build logic
- `syntax_6502.xml` - Notepad++ syntax highlighting for 6502 assembly

## Validation

### Manual validation requirements
- ALWAYS run `bash assemble.sh` in a game directory after making changes
- Verify the ROM file is generated with correct size and SHA-1 checksum
- Build validation scenario: Pick any game directory and run:
  ```bash
  cd "Excitebike"
  bash assemble.sh
  ls -la _excitebike.nes  # Should show 24592 bytes
  ```

### Testing scenarios
- Single game build test: Navigate to any game directory, run `bash assemble.sh`, verify success message and ROM generation
- All games build test: Run build loop for all directories with `assemble.sh` files
- Checksum verification: Each successful build reports "Original SHA-1 checksum detected"
- Build artifact cleanup: Temporary files (`copy_*`, `*.o`, `PRG_ROM.bin`) are automatically cleaned up

## Common tasks

### Working with game disassemblies
- Edit assembly files: Modify `.asm` files using any text editor
- Important comments marked with `bzk` text indicate reverse-engineering notes
- The `preparations.lua` script preprocesses files before assembly, removing disassembler output formatting
- Use `syntax_6502.xml` for Notepad++ syntax highlighting

### Build system details
- Build process: lua preprocessing → ca65 assembly → ld65 linking → ROM generation
- Fast assembly mode enabled by default (`NES_OUTPUT_FAST_ASSEMBLY=1`)
- Each game script sources common functions from `_scripts/` directory
- Build scripts automatically check for and install missing dependencies (cc65, lua)
- Windows .exe files in game directories are for Windows builds - ignore on Linux

### Repository navigation
```
Repository root structure:
.
├── README.md                    # Project documentation
├── syntax_6502.xml             # Notepad++ syntax highlighting
├── _scripts/                   # Shared build scripts
│   ├── env.sh                  # Environment setup
│   ├── os_support.sh           # OS compatibility
│   ├── assemble_header.sh      # Build preparation
│   ├── assemble_standard.sh    # Main assembly
│   └── assemble_footer.sh      # Cleanup and verification
├── [Game Name]/               # Individual game directories
│   ├── assemble.sh            # Game build script
│   ├── bank_*.asm            # Assembly source files
│   ├── bank_ram.inc          # RAM definitions
│   ├── bank_val.inc          # Constants
│   ├── preparations.lua      # Lua preprocessor
│   ├── header.bin           # NES header
│   ├── ld65.cfg            # Linker config
│   └── CHR_ROM.chr         # Graphics data (optional)
└── [24 more game directories]
```

### Timing expectations
- Single game build: 0.07-1.6 seconds (average 0.287 seconds)
- All 25 games: ~7 seconds total
- cc65 compilation from source: ~55 seconds
- Dependency installation: 60-120 seconds
- NEVER CANCEL any build operations - they complete very quickly

### Error handling
- Build failures are rare - all 25 games build successfully
- Common issues: Missing dependencies (auto-resolved by scripts)
- Windows executables (.exe/.dll) in game directories can be ignored on Linux
- Use `bash assemble.sh` explicitly rather than `sh assemble.sh` to avoid shell compatibility issues