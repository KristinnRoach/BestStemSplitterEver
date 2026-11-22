# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BestStemSplitterEver is an audio stem separation tool that:
- Detects musical key and BPM using librosa
- Separates audio into stems (vocals, bass, drums, other, and optionally piano/guitar) using Demucs
- Further splits drum stems into individual components (kick, snare, hats, toms) using a custom Drumsep integration
- Organizes output files with configurable naming patterns

## Development Commands

### Setup
```bash
# Initial setup (downloads model and installs dependencies)
./setup.sh              # macOS/Linux
./setup.bat             # Windows
python setup.py         # Direct Python execution
```

### Running the Application
```bash
# Basic usage
python stem_splitter.py <audio_file>

# With custom config
python stem_splitter.py <audio_file> --config my_config.yaml

# Override output directory
python stem_splitter.py <audio_file> --output ~/Desktop/MySongStems

# Use different Demucs model
python stem_splitter.py <audio_file> --model htdemucs

# Using shell scripts
./run.sh <audio_file>   # macOS/Linux
run.bat <audio_file>    # Windows
```

### Testing
```bash
# Verify installation and check all dependencies
python test.py
```

## Architecture

### Core Processing Pipeline

The main workflow in `stem_splitter.py` follows this sequence:

1. **Audio Analysis** (`detect_key_and_tempo()`): Uses librosa to analyze the audio file
   - Detects BPM using beat tracking
   - Detects musical key using chroma features and Krumhansl-Kessler profiles
   - Generates Camelot notation for DJ-friendly key representation

2. **Stem Separation** (via Demucs): Separates audio into basic stems
   - Model selection affects available stems (4-source vs 6-source models)
   - Default model: `htdemucs_6s` (includes piano and guitar)
   - Output temporarily stored in `temp_dir` before processing

3. **Drum Component Separation** (`run_drumsep()`): Further breaks down the drums stem
   - Uses custom Demucs model (ID: 49469ca8) specifically trained for drum separation
   - Separates into: kick (bombo), snare/clap (redoblante), hats (platillos), toms
   - Spanish filenames are mapped to English in the main script

4. **File Organization** (`format_filename()`): Renames and organizes output
   - Configurable filename templates with {key}, {camelot}, {bpm}, {name}, {stem} placeholders
   - Optional per-song folder organization
   - Automatic cleanup of temporary Demucs output

### Key Components

**stem_splitter.py** (main entry point):
- Orchestrates the entire pipeline
- Handles configuration loading and command-line arguments
- Cross-platform compatibility (Windows, macOS, Linux)
- Uses `run_command()` wrapper for subprocess execution with error handling

**drumsep/drumsep.py** (drum separation module):
- Wrapper around Demucs with custom drum separation model
- Can be called directly or imported as a module
- Model stored in `drumsep/model/49469ca8.th` (downloaded during setup)

**config.yaml** (runtime configuration):
- Defines paths for temp and output directories
- Specifies Demucs model to use
- Controls output organization and filename formatting

**setup.py** (initialization):
- Downloads the drum separation model from Google Drive using gdown
- Model ID: `1-Dm666ScPkg8Gt2-lK3Ua0xOudWHZBGC`
- Creates model directory structure

### Important Technical Details

**PyTorch Version Constraint**: The drum separation model requires PyTorch 2.5.1 or earlier. Newer versions may have compatibility issues with the model loading mechanism. If encountering model loading errors, set `TORCH_WEIGHTS_ONLY=0`.

**Cross-Platform Handling**:
- File path handling uses `os.path` for compatibility
- Platform-specific folder opening (`open_folder()` function)
- Shell script wrappers for both Unix and Windows

**Configuration Resolution**:
- Relative paths in config are resolved relative to the script directory
- User home directory expansion with `~` is supported
- Command-line arguments override config file settings

**Model Outputs**:
- Demucs creates folders named `<model_name>/<song_name>/` in temp directory
- Available stems depend on model: htdemucs/htdemucs_ft/mdx_extra (4 sources) vs htdemucs_6s (6 sources)
- Drum separation creates a folder with model ID `49469ca8` containing separated components

## Demucs Model Options

- `htdemucs`: 4 sources (vocals, drums, bass, other)
- `htdemucs_6s`: 6 sources (adds piano, guitar) - DEFAULT
- `htdemucs_ft`: Fine-tuned 4 sources
- `mdx_extra`: Higher quality, slower
- `mdx_extra_q`: Quantized mdx_extra

## Configuration File Structure

The `config.yaml` supports:
- `paths.temp_dir`: Temporary storage for Demucs output
- `paths.output_dir`: Final stem destination
- `tools.drumsep_dir`: Location of drumsep module (relative or absolute)
- `tools.demucs_model`: Which Demucs model to use
- `output.organize_by_song`: Create per-song folders (boolean)
- `output.include_key_bpm`: Add key/BPM to filenames (boolean)
- `output.filename_format`: Template string for output files
