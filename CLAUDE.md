# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

LG4X-V2 is an open-source GUI application for X-ray photoemission spectroscopy (XPS) curve fitting based on Python's lmfit package. It's a PyQt5-based desktop application that streamlines XPS data analysis through an interactive GUI for peak fitting, background subtraction, and spectral analysis.

**Current Branch**: `database` (work in progress)
**Main Branch**: `master` (use for PRs)

## Development Setup

### Installation

```bash
git clone https://github.com/Julian-Hochhaus/LG4X-V2.git
cd LG4X-V2/
pip install -r requirements.txt
```

### Running the Application

```bash
python3 Python/main.py
```

### Dependencies

The project requires:
- Python 3.8-3.11 (tested on 3.9.5+)
- Key packages: PyQt5 (>=5.15), lmfit (>=1.1.0), lmfitxps (>=4.0.9), matplotlib (>=3.6), numpy (>=1.22), pandas (>=2.0), scipy (>=1.6)
- See `requirements.txt` for full list

### Configuration

Application settings are stored in `config/config.ini`:
- GUI settings (resolution, dark mode, two-window mode)
- Import settings (CSV separator, columns, header row)
- Modify these through the GUI settings menu or directly in the file

## Architecture

### Core Components

1. **`Python/main.py`**: Main application entry point (~6600 lines)
   - `PrettyWidget` class: Main window controller
   - Handles GUI initialization (single or dual-window mode)
   - Manages fitting workflow, data import/export, and parameter management
   - Version tracking: `__version__ = "2.4.2"`

2. **`Python/helpers.py`**: Core fitting and modeling utilities
   - Autoscaling functions
   - Model construction helpers for various peak shapes (Gaussian, Lorentzian, Voigt, Doniach, etc.)
   - Background model helpers (Shirley, Tougaard, Polynomial, Slope)
   - Uses lmfit's built-in models and lmfitxps extensions

3. **`Python/gui_helpers.py`**: GUI construction utilities
   - Layout creators (top row, bottom layouts, canvas setup)
   - Table management for parameter display and editing
   - Menu bar and toolbar construction
   - UI component factories

4. **`Python/vamas.py` & `Python/vamas_export.py`**: VAMAS file format support
   - Import ISO VAMAS format files (.vms/.npl)
   - Export decomposed VAMAS data to tab-separated text files
   - Based on Kane O'Donnell's implementation

5. **`Python/periodictable.py`**: Periodic table window for peak identification
   - XPS peak energy and sensitivity database
   - Based on clusterid project

6. **`Python/database_handler.py`**: SQLite database integration (database branch)
   - Schema: Projects → Spectra → Data/Fits
   - Database location: `../Databases/lg4x-v2-univers.db`
   - Currently under development on `database` branch

### Data Flow

1. **Import**: CSV, text, or VAMAS files → `data_arr` dictionary
2. **Setup**: Background and peak parameters configured in tables → `pre` list structure
3. **Fitting**: lmfit optimization using selected models → `result` object
4. **Export**:
   - Parameters → `_fit.txt` (text file)
   - Spectral data → `_fit.csv` (contains raw data, background, components, sum curve)

### Background Types (dictBG)

- `"0"`: Static Shirley BG
- `"100"`: Active Shirley BG
- `"1"`: Static Tougaard BG
- `"101"`: Active Tougaard BG
- `"2"`: Polynomial BG
- `"3"`: Arctan
- `"4"`: Error function
- `"5"`: CutOff
- `"6"`: Slope BG

### Peak Models

Supported via lmfit: Gaussian, Lorentzian, Voigt, PseudoVoigt, ExponentialGaussian, SkewedGaussian, SkewedVoigt, BreitWigner, Lognormal, Doniach

Extended via lmfitxps: ConvGaussianDoniachSinglett, ConvGaussianDoniachDublett, FermiEdge

## File Formats

### Preset Files (`_pars.dat`)

Structure: `[BG_type_index, [BG_table_params], [Fit_table_params]]`

### Export CSV Structure

1. Column 1-2: Raw x, y data
2. Column 3: Intensity minus background
3. Column 4: Sum over all components (no background)
4. Column 5: Background (includes polynomial)
5. Column 6: Polynomial background only
6. Column 7: Sum curve (components + backgrounds)
7. Column 8+: Individual fit components

## Important Directories

- `Python/`: Main source code
- `Example/`: Example XPS data files
- `Preset/`: Preset fitting parameter files
- `test/`: Test data files (Au4f, FK, Survey spectra)
- `Databases/`: SQLite database (database branch)
- `config/`: Application configuration
- `Logs/`: Application logs (rotating, max 4MB, 5 backups)

## GUI Modes

The application supports two display modes:
- **Single-window mode**: All controls and plot in one window
- **Two-window mode**: Separate windows for controls and visualization
  - Configured via `config.ini`: `two_window_mode = True/False`
  - Allows multi-monitor setups

## Testing

Test data files are located in `test/` directory:
- `Highres_Au4f_hv_180_testfile.csv`
- `Highres_FK_hv_180.csv`
- `Survey_Au_hv_700.csv`

No automated test suite currently exists. Testing is manual via the GUI.

## Development Notes

- The codebase is actively developed; use released versions for production use
- Current work focuses on database integration (see `database` branch)
- Logging is configured with rotation in `Logs/app.log`
- Flatpak support: Special paths for containerized environment (`io.github.julian_hochhaus.LG4X_V2`)
- Style: Uses matplotlib's `seaborn-v0_8-colorblind` style
- The application can run in Flatpak container (detected via `os.environ.get("container")`)

## Common Patterns

### Adding a New Peak Model

1. Import the model in `helpers.py` from lmfit or lmfitxps
2. Add model creation logic in the model construction functions
3. Update the Fit table dropdown in `gui_helpers.py` with model identifier
4. Document parameters in GUI tooltips/labels

### Modifying Background Methods

1. Background implementations are in lmfitxps.backgrounds or lmfitxps.models
2. Add to `dictBG` in `main.py`
3. Create background parameter handling in table construction
4. Update BG selection dropdown in GUI

### Parameter Constraints

- Amplitude ratios (`amp_ratio`) and peak differences (`ctr_diff`) can reference other peaks
- Checkboxes in tables enable value fixing or bound constraints
- Empty cells don't affect optimization
- Parameters are stored hierarchically: `pre = [[], [], [], []]`

## Citation

If using this code in research, cite via Zenodo DOI: 10.5281/zenodo.7777422
