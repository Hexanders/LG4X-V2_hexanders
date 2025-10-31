# Database Integration: "To Rule Them All"

## Executive Summary

This document provides a comprehensive analysis of database integration for LG4X-V2, including analysis of current implementation, detailed schema suggestions, and implementation recommendations for tracking parameters, user information, GUI configurations, and user interactions.

---

## 1. Current State Analysis

### 1.1 Fork Changes Overview

Your fork (Hexanders/LG4X-V2_hexanders) has made significant progress on the `database` branch:

**Key Commits:**
- `8342355`: Initial database implementation
- `cebe015`: Database initialization functions
- `1ed2786`, `8f87ca6`: Settings dialog with tabs for better organization
- `0c1e6f9`: Merge conflict resolution

**Files Modified/Added:**
- `Python/database_handler.py`: Basic schema initialization
- `Python/helpers.py`: Added `SettingsDialog` with tabbed interface
- `Databases/lg4x-v2-univers.db`: SQLite database file (currently empty)
- `devolopment/sql_data_base_sketch.svg`: Schema visualization

### 1.2 Current Database Schema

The existing schema in `database_handler.py` is minimal:

```sql
-- Projects (top level)
CREATE TABLE Project (
    ID INTEGER PRIMARY KEY,
    Name TEXT,
    Comment TEXT,
    Spec_id INT,
    FOREIGN KEY (Spec_id) REFERENCES Spectra (SID)
);

-- Spectra (middle level)
CREATE TABLE Spectra (
    SID INTEGER PRIMARY KEY,
    creationDate TIMESTAMP,
    lastChange TIMESTAMP,
    name TEXT,
    comment TEXT,
    Data_id INT,
    Fit_id INT,
    FOREIGN KEY (Data_id) REFERENCES Data (DID),
    FOREIGN KEY (Fit_id) REFERENCES Fits (FID)
);

-- Data (raw data reference)
CREATE TABLE Data (
    DID INTEGER PRIMARY KEY,
    comment TEXT,
    FilePath TEXT
);

-- Fits (fit parameters reference)
CREATE TABLE Fits (
    FID INTEGER PRIMARY KEY,
    comment TEXT,
    FilePath TEXT
);
```

**Current Limitations:**
1. No parameter versioning/history tracking
2. No user management system
3. GUI configurations still in `config.ini` only
4. No interaction/activity logging
5. Fit parameters stored only as file paths, not in database
6. No relationship tracking between related fits
7. No metadata for analysis workflow
8. No search/query optimization

---

## 2. Comprehensive Database Schema Recommendations

### 2.1 Core Principle: Separation of Concerns

The database should be organized into distinct domains:
1. **User Management**: Authentication, profiles, preferences
2. **Project Organization**: Hierarchical project structure
3. **Data Management**: Raw data, metadata, relationships
4. **Fit Management**: Parameters, history, versioning
5. **Configuration**: GUI state, user preferences, application settings
6. **Activity Logging**: User interactions, audit trail
7. **Analysis Workflow**: Templates, presets, common workflows

### 2.2 Detailed Schema Design

#### 2.2.1 User Management Schema

```sql
-- Users table
CREATE TABLE Users (
    user_id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    email TEXT UNIQUE,
    full_name TEXT,
    affiliation TEXT,  -- Research institution/company
    orcid TEXT,        -- ORCID identifier for researchers
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    is_active BOOLEAN DEFAULT 1,
    password_hash TEXT,  -- If authentication is needed
    CONSTRAINT valid_email CHECK (email IS NULL OR email LIKE '%@%')
);

-- User preferences (extensible key-value store)
CREATE TABLE UserPreferences (
    pref_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    preference_key TEXT NOT NULL,
    preference_value TEXT,
    preference_type TEXT CHECK(preference_type IN ('string', 'int', 'float', 'bool', 'json')),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    UNIQUE(user_id, preference_key)
);

-- User sessions (for tracking work sessions)
CREATE TABLE UserSessions (
    session_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    session_start TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    session_end TIMESTAMP,
    ip_address TEXT,
    app_version TEXT,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE
);
```

#### 2.2.2 Enhanced Project & Data Management

```sql
-- Projects (enhanced)
CREATE TABLE Projects (
    project_id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_name TEXT NOT NULL,
    project_description TEXT,
    owner_user_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_archived BOOLEAN DEFAULT 0,
    project_metadata TEXT,  -- JSON for extensible metadata
    FOREIGN KEY (owner_user_id) REFERENCES Users(user_id) ON DELETE RESTRICT
);

-- Project collaborators (many-to-many)
CREATE TABLE ProjectCollaborators (
    collab_id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    role TEXT CHECK(role IN ('owner', 'editor', 'viewer')),
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (project_id) REFERENCES Projects(project_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    UNIQUE(project_id, user_id)
);

-- Samples (new concept for organizing spectra)
CREATE TABLE Samples (
    sample_id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER NOT NULL,
    sample_name TEXT NOT NULL,
    sample_description TEXT,
    preparation_date DATE,
    preparation_method TEXT,
    material_composition TEXT,
    metadata TEXT,  -- JSON for custom fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (project_id) REFERENCES Projects(project_id) ON DELETE CASCADE
);

-- Spectra (enhanced)
CREATE TABLE Spectra (
    spectrum_id INTEGER PRIMARY KEY AUTOINCREMENT,
    sample_id INTEGER,
    spectrum_name TEXT NOT NULL,
    spectrum_type TEXT CHECK(spectrum_type IN ('XPS', 'UPS', 'NEXAFS', 'Auger', 'Survey', 'Other')),
    acquisition_date TIMESTAMP,
    beamline TEXT,
    photon_energy REAL,       -- hν
    pass_energy REAL,
    work_function REAL,       -- φ
    energy_scale TEXT CHECK(energy_scale IN ('BE', 'KE')),
    analyzer TEXT,
    comments TEXT,
    raw_data_file TEXT,       -- Original file path
    data_format TEXT,         -- csv, vms, txt
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata TEXT,            -- JSON for instrument-specific parameters
    FOREIGN KEY (sample_id) REFERENCES Samples(sample_id) ON DELETE SET NULL
);

-- Spectrum data points (optional: store actual data in DB)
CREATE TABLE SpectrumDataPoints (
    datapoint_id INTEGER PRIMARY KEY AUTOINCREMENT,
    spectrum_id INTEGER NOT NULL,
    energy REAL NOT NULL,     -- x-axis
    intensity REAL NOT NULL,  -- y-axis
    point_index INTEGER,      -- Order in spectrum
    FOREIGN KEY (spectrum_id) REFERENCES Spectra(spectrum_id) ON DELETE CASCADE
);
CREATE INDEX idx_spectrum_data ON SpectrumDataPoints(spectrum_id, point_index);

-- Data processing steps (track background subtraction, normalization, etc.)
CREATE TABLE DataProcessingSteps (
    step_id INTEGER PRIMARY KEY AUTOINCREMENT,
    spectrum_id INTEGER NOT NULL,
    step_type TEXT,           -- 'background_subtraction', 'normalization', 'smoothing'
    step_order INTEGER,
    parameters TEXT,          -- JSON with step parameters
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    applied_by INTEGER,
    FOREIGN KEY (spectrum_id) REFERENCES Spectra(spectrum_id) ON DELETE CASCADE,
    FOREIGN KEY (applied_by) REFERENCES Users(user_id) ON DELETE SET NULL
);
```

#### 2.2.3 Comprehensive Fit Management with History

```sql
-- Fit sessions (groups related fitting attempts)
CREATE TABLE FitSessions (
    session_id INTEGER PRIMARY KEY AUTOINCREMENT,
    spectrum_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    session_name TEXT,
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    status TEXT CHECK(status IN ('in_progress', 'completed', 'abandoned')),
    FOREIGN KEY (spectrum_id) REFERENCES Spectra(spectrum_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE SET NULL
);

-- Fits (individual fit attempts with versioning)
CREATE TABLE Fits (
    fit_id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id INTEGER,
    spectrum_id INTEGER NOT NULL,
    fit_version INTEGER DEFAULT 1,
    fit_name TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER,
    parent_fit_id INTEGER,    -- For tracking fit lineage

    -- Fit quality metrics
    chi_square REAL,
    reduced_chi_square REAL,
    r_squared REAL,
    aic REAL,                 -- Akaike Information Criterion
    bic REAL,                 -- Bayesian Information Criterion

    -- Fit settings
    bg_type TEXT,             -- 'Shirley', 'Tougaard', etc.
    bg_params TEXT,           -- JSON with background parameters
    x_min REAL,
    x_max REAL,
    num_peaks INTEGER,
    optimizer TEXT,           -- 'leastsq', 'least_squares', etc.
    max_iterations INTEGER,
    tolerance REAL,

    -- Results
    fit_status TEXT,          -- 'success', 'failed', 'aborted'
    fit_message TEXT,
    execution_time REAL,      -- seconds

    -- Storage
    result_file_path TEXT,    -- Path to _fit.txt
    data_file_path TEXT,      -- Path to _fit.csv
    preset_file_path TEXT,    -- Path to _pars.dat if saved

    -- Flags
    is_published BOOLEAN DEFAULT 0,
    is_favorite BOOLEAN DEFAULT 0,

    notes TEXT,
    metadata TEXT,            -- JSON for extensible data

    FOREIGN KEY (session_id) REFERENCES FitSessions(session_id) ON DELETE SET NULL,
    FOREIGN KEY (spectrum_id) REFERENCES Spectra(spectrum_id) ON DELETE CASCADE,
    FOREIGN KEY (created_by) REFERENCES Users(user_id) ON DELETE SET NULL,
    FOREIGN KEY (parent_fit_id) REFERENCES Fits(fit_id) ON DELETE SET NULL
);
CREATE INDEX idx_fits_spectrum ON Fits(spectrum_id, created_at DESC);
CREATE INDEX idx_fits_session ON Fits(session_id);

-- Peaks (individual peak parameters)
CREATE TABLE Peaks (
    peak_id INTEGER PRIMARY KEY AUTOINCREMENT,
    fit_id INTEGER NOT NULL,
    peak_number INTEGER NOT NULL,
    peak_label TEXT,          -- e.g., 'C1', 'Au4f7/2'

    -- Peak model
    model_type TEXT,          -- 'gaussian', 'lorentzian', 'voigt', etc.

    -- Peak parameters (store both initial and fitted)
    amplitude REAL,
    amplitude_stderr REAL,
    amplitude_initial REAL,

    center REAL,
    center_stderr REAL,
    center_initial REAL,

    sigma REAL,
    sigma_stderr REAL,
    sigma_initial REAL,

    gamma REAL,               -- For Voigt, Lorentzian, etc.
    gamma_stderr REAL,
    gamma_initial REAL,

    fwhm REAL,                -- Calculated FWHM
    height REAL,              -- Peak height
    area REAL,                -- Integrated area
    area_stderr REAL,

    -- Constraints
    is_constrained BOOLEAN DEFAULT 0,
    constraint_type TEXT,     -- 'fixed', 'ratio', 'difference'
    constraint_reference INTEGER,  -- Reference peak for ratios/differences
    constraint_value REAL,

    -- Additional model-specific parameters (JSON)
    extra_params TEXT,

    FOREIGN KEY (fit_id) REFERENCES Fits(fit_id) ON DELETE CASCADE,
    FOREIGN KEY (constraint_reference) REFERENCES Peaks(peak_id) ON DELETE SET NULL,
    UNIQUE(fit_id, peak_number)
);
CREATE INDEX idx_peaks_fit ON Peaks(fit_id);

-- Parameter history (track how parameters evolved during fit)
CREATE TABLE ParameterHistory (
    history_id INTEGER PRIMARY KEY AUTOINCREMENT,
    fit_id INTEGER NOT NULL,
    iteration INTEGER,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    parameter_name TEXT,      -- 'peak_1_amplitude', 'bg_c0', etc.
    parameter_value REAL,
    chi_square REAL,          -- Fit quality at this iteration
    FOREIGN KEY (fit_id) REFERENCES Fits(fit_id) ON DELETE CASCADE
);
CREATE INDEX idx_param_history ON ParameterHistory(fit_id, iteration);

-- Fit comparisons (for comparing multiple fits)
CREATE TABLE FitComparisons (
    comparison_id INTEGER PRIMARY KEY AUTOINCREMENT,
    comparison_name TEXT,
    created_by INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    notes TEXT,
    FOREIGN KEY (created_by) REFERENCES Users(user_id) ON DELETE SET NULL
);

CREATE TABLE FitComparisonMembers (
    member_id INTEGER PRIMARY KEY AUTOINCREMENT,
    comparison_id INTEGER NOT NULL,
    fit_id INTEGER NOT NULL,
    FOREIGN KEY (comparison_id) REFERENCES FitComparisons(comparison_id) ON DELETE CASCADE,
    FOREIGN KEY (fit_id) REFERENCES Fits(fit_id) ON DELETE CASCADE,
    UNIQUE(comparison_id, fit_id)
);
```

#### 2.2.4 Configuration Management

```sql
-- GUI configurations (replaces/supplements config.ini)
CREATE TABLE GUIConfigurations (
    config_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,          -- NULL for global configs
    config_category TEXT,     -- 'GUI', 'Import', 'Plot', 'Export'
    config_key TEXT NOT NULL,
    config_value TEXT,
    config_type TEXT CHECK(config_type IN ('string', 'int', 'float', 'bool', 'json')),
    is_global BOOLEAN DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    UNIQUE(user_id, config_category, config_key)
);

-- Window states (for restoring GUI state)
CREATE TABLE WindowStates (
    state_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    window_name TEXT,         -- 'main', 'settings', 'periodic_table'
    window_geometry TEXT,     -- JSON: {x, y, width, height}
    window_state TEXT,        -- 'normal', 'maximized', 'minimized'
    splitter_sizes TEXT,      -- JSON: for splitter positions
    last_used TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE
);

-- Recent files/paths (for quick access)
CREATE TABLE RecentItems (
    item_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    item_type TEXT CHECK(item_type IN ('file', 'directory', 'project', 'preset')),
    item_path TEXT NOT NULL,
    last_accessed TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    access_count INTEGER DEFAULT 1,
    pinned BOOLEAN DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE
);
CREATE INDEX idx_recent_items ON RecentItems(user_id, last_accessed DESC);
```

#### 2.2.5 Presets and Templates

```sql
-- Fit presets (reusable fitting configurations)
CREATE TABLE FitPresets (
    preset_id INTEGER PRIMARY KEY AUTOINCREMENT,
    preset_name TEXT NOT NULL,
    preset_description TEXT,
    created_by INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Preset content
    bg_type TEXT,
    bg_params TEXT,           -- JSON
    peak_models TEXT,         -- JSON array of peak configurations
    constraints TEXT,         -- JSON for constraints

    -- Applicability
    spectrum_type TEXT,       -- 'XPS', 'UPS', etc.
    element TEXT,             -- e.g., 'C', 'Au', 'Ag'
    orbital TEXT,             -- e.g., '1s', '4f'

    is_public BOOLEAN DEFAULT 0,
    is_template BOOLEAN DEFAULT 0,
    times_used INTEGER DEFAULT 0,

    preset_file_path TEXT,    -- Reference to .dat file

    FOREIGN KEY (created_by) REFERENCES Users(user_id) ON DELETE SET NULL
);

-- Preset usage tracking
CREATE TABLE PresetUsageLog (
    usage_id INTEGER PRIMARY KEY AUTOINCREMENT,
    preset_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    fit_id INTEGER,
    used_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (preset_id) REFERENCES FitPresets(preset_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (fit_id) REFERENCES Fits(fit_id) ON DELETE SET NULL
);
```

#### 2.2.6 Activity & Interaction Logging

```sql
-- User activity log (comprehensive interaction tracking)
CREATE TABLE ActivityLog (
    activity_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    session_id INTEGER,
    activity_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    activity_type TEXT,       -- 'file_import', 'fit_run', 'parameter_change', etc.
    activity_category TEXT,   -- 'data', 'fitting', 'export', 'ui'
    target_type TEXT,         -- 'spectrum', 'fit', 'project', etc.
    target_id INTEGER,        -- ID of the target object
    action_details TEXT,      -- JSON with detailed information
    success BOOLEAN,
    error_message TEXT,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE SET NULL,
    FOREIGN KEY (session_id) REFERENCES UserSessions(session_id) ON DELETE CASCADE
);
CREATE INDEX idx_activity_user_time ON ActivityLog(user_id, activity_timestamp DESC);
CREATE INDEX idx_activity_type ON ActivityLog(activity_type, activity_timestamp DESC);

-- Error log (for debugging and quality improvement)
CREATE TABLE ErrorLog (
    error_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    error_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    error_type TEXT,
    error_message TEXT,
    stack_trace TEXT,
    app_version TEXT,
    python_version TEXT,
    os_info TEXT,
    context TEXT,             -- JSON with additional context
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE SET NULL
);

-- Performance metrics (for optimization)
CREATE TABLE PerformanceMetrics (
    metric_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    operation_type TEXT,      -- 'fit', 'import', 'export', 'background_calc'
    duration_ms INTEGER,
    memory_mb REAL,
    cpu_percent REAL,
    data_size INTEGER,        -- Number of data points processed
    additional_info TEXT,     -- JSON
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE SET NULL
);
```

#### 2.2.7 Tags and Organization

```sql
-- Tags (for flexible organization)
CREATE TABLE Tags (
    tag_id INTEGER PRIMARY KEY AUTOINCREMENT,
    tag_name TEXT UNIQUE NOT NULL,
    tag_color TEXT,           -- Hex color for UI
    tag_description TEXT,
    created_by INTEGER,
    FOREIGN KEY (created_by) REFERENCES Users(user_id) ON DELETE SET NULL
);

-- Taggable items (polymorphic tagging)
CREATE TABLE TagAssignments (
    assignment_id INTEGER PRIMARY KEY AUTOINCREMENT,
    tag_id INTEGER NOT NULL,
    entity_type TEXT CHECK(entity_type IN ('project', 'sample', 'spectrum', 'fit', 'preset')),
    entity_id INTEGER NOT NULL,
    assigned_by INTEGER,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (tag_id) REFERENCES Tags(tag_id) ON DELETE CASCADE,
    FOREIGN KEY (assigned_by) REFERENCES Users(user_id) ON DELETE SET NULL,
    UNIQUE(tag_id, entity_type, entity_id)
);
CREATE INDEX idx_tag_assignments ON TagAssignments(entity_type, entity_id);

-- Notes/annotations (can be attached to various entities)
CREATE TABLE Annotations (
    annotation_id INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_type TEXT CHECK(entity_type IN ('project', 'sample', 'spectrum', 'fit', 'peak')),
    entity_id INTEGER NOT NULL,
    annotation_text TEXT NOT NULL,
    created_by INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_important BOOLEAN DEFAULT 0,
    FOREIGN KEY (created_by) REFERENCES Users(user_id) ON DELETE CASCADE
);
CREATE INDEX idx_annotations ON Annotations(entity_type, entity_id);
```

---

## 3. Implementation Strategy

### 3.1 Migration Path

**Phase 1: Foundation (Weeks 1-2)**
- Implement user management system (basic)
- Enhance existing Projects/Spectra tables
- Set up proper database connection handling
- Create database manager class with connection pooling

**Phase 2: Core Functionality (Weeks 3-4)**
- Implement Fits and Peaks tables
- Add parameter history tracking
- Migrate config.ini settings to database
- Implement configuration management API

**Phase 3: Advanced Features (Weeks 5-6)**
- Add activity logging
- Implement preset management in database
- Add tags and annotations
- Create search and query optimization

**Phase 4: Integration & Polish (Weeks 7-8)**
- Full GUI integration
- Performance optimization
- Migration tools for existing data
- Documentation and testing

### 3.2 Code Organization

```
Python/
├── database/
│   ├── __init__.py
│   ├── connection.py        # Database connection management
│   ├── models.py            # SQLAlchemy ORM models (if using ORM)
│   ├── schema.py            # Schema definitions and migrations
│   ├── repositories/        # Data access layer
│   │   ├── user_repository.py
│   │   ├── project_repository.py
│   │   ├── spectrum_repository.py
│   │   ├── fit_repository.py
│   │   └── config_repository.py
│   ├── migrations/          # Database version migrations
│   └── queries.py           # Common SQL queries
├── services/
│   ├── user_service.py      # Business logic for users
│   ├── fit_service.py       # Fitting workflow logic
│   └── config_service.py    # Configuration management
└── utils/
    └── db_utils.py          # Database utilities
```

### 3.3 Database Connection Best Practices

```python
# Python/database/connection.py

import sqlite3
import threading
from contextlib import contextmanager
from typing import Optional

class DatabaseManager:
    """Thread-safe database connection manager."""

    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        if not hasattr(self, 'initialized'):
            self.db_path = None
            self.initialized = True

    def initialize(self, db_path: str):
        """Initialize database connection."""
        self.db_path = db_path
        self._init_schema()

    @contextmanager
    def get_connection(self):
        """Get a database connection with automatic cleanup."""
        conn = sqlite3.connect(self.db_path)
        conn.execute("PRAGMA foreign_keys = ON")
        conn.row_factory = sqlite3.Row  # Enable column access by name
        try:
            yield conn
            conn.commit()
        except Exception as e:
            conn.rollback()
            raise
        finally:
            conn.close()

    def execute_query(self, query: str, params: tuple = ()):
        """Execute a query and return results."""
        with self.get_connection() as conn:
            cursor = conn.execute(query, params)
            return cursor.fetchall()

    def execute_many(self, query: str, params_list: list):
        """Execute multiple queries efficiently."""
        with self.get_connection() as conn:
            conn.executemany(query, params_list)

    def _init_schema(self):
        """Initialize database schema if needed."""
        # Run schema creation scripts
        pass
```

### 3.4 Parameter History Implementation Example

```python
# Python/services/fit_service.py

class FitService:
    """Service for managing fits and tracking parameter history."""

    def __init__(self, db_manager: DatabaseManager):
        self.db = db_manager

    def create_fit(self, spectrum_id: int, user_id: int,
                   fit_params: dict) -> int:
        """Create a new fit and return fit_id."""
        query = """
            INSERT INTO Fits (
                spectrum_id, created_by, bg_type, bg_params,
                x_min, x_max, num_peaks
            ) VALUES (?, ?, ?, ?, ?, ?, ?)
        """
        with self.db.get_connection() as conn:
            cursor = conn.execute(query, (
                spectrum_id, user_id,
                fit_params['bg_type'],
                json.dumps(fit_params['bg_params']),
                fit_params['x_min'],
                fit_params['x_max'],
                fit_params['num_peaks']
            ))
            return cursor.lastrowid

    def track_parameter_change(self, fit_id: int, iteration: int,
                               param_name: str, param_value: float,
                               chi_square: float):
        """Track parameter evolution during fitting."""
        query = """
            INSERT INTO ParameterHistory (
                fit_id, iteration, parameter_name,
                parameter_value, chi_square
            ) VALUES (?, ?, ?, ?, ?)
        """
        self.db.execute_query(query, (
            fit_id, iteration, param_name,
            param_value, chi_square
        ))

    def get_parameter_history(self, fit_id: int,
                             param_name: Optional[str] = None):
        """Retrieve parameter evolution history."""
        if param_name:
            query = """
                SELECT iteration, parameter_value, chi_square, timestamp
                FROM ParameterHistory
                WHERE fit_id = ? AND parameter_name = ?
                ORDER BY iteration
            """
            return self.db.execute_query(query, (fit_id, param_name))
        else:
            query = """
                SELECT iteration, parameter_name, parameter_value,
                       chi_square, timestamp
                FROM ParameterHistory
                WHERE fit_id = ?
                ORDER BY iteration, parameter_name
            """
            return self.db.execute_query(query, (fit_id,))

    def save_peaks(self, fit_id: int, peaks_data: list):
        """Save all peaks for a fit."""
        query = """
            INSERT INTO Peaks (
                fit_id, peak_number, peak_label, model_type,
                amplitude, amplitude_stderr, center, center_stderr,
                sigma, sigma_stderr, fwhm, area, area_stderr
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """
        params_list = [
            (fit_id, peak['number'], peak['label'], peak['model'],
             peak['amp'], peak['amp_err'], peak['center'], peak['center_err'],
             peak['sigma'], peak['sigma_err'], peak['fwhm'],
             peak['area'], peak['area_err'])
            for peak in peaks_data
        ]
        self.db.execute_many(query, params_list)
```

### 3.5 Configuration Management Example

```python
# Python/services/config_service.py

class ConfigService:
    """Service for managing GUI configurations."""

    def __init__(self, db_manager: DatabaseManager):
        self.db = db_manager
        self._cache = {}

    def get_config(self, user_id: Optional[int],
                   category: str, key: str, default=None):
        """Get configuration value with fallback to global/default."""
        cache_key = f"{user_id}:{category}:{key}"
        if cache_key in self._cache:
            return self._cache[cache_key]

        # Try user-specific config
        if user_id:
            query = """
                SELECT config_value, config_type
                FROM GUIConfigurations
                WHERE user_id = ? AND config_category = ?
                  AND config_key = ?
            """
            result = self.db.execute_query(query, (user_id, category, key))
            if result:
                value = self._convert_value(result[0])
                self._cache[cache_key] = value
                return value

        # Fallback to global config
        query = """
            SELECT config_value, config_type
            FROM GUIConfigurations
            WHERE user_id IS NULL AND config_category = ?
              AND config_key = ?
        """
        result = self.db.execute_query(query, (category, key))
        if result:
            value = self._convert_value(result[0])
            self._cache[cache_key] = value
            return value

        return default

    def set_config(self, user_id: Optional[int], category: str,
                   key: str, value, config_type: str = 'string'):
        """Set configuration value."""
        query = """
            INSERT INTO GUIConfigurations (
                user_id, config_category, config_key,
                config_value, config_type
            ) VALUES (?, ?, ?, ?, ?)
            ON CONFLICT(user_id, config_category, config_key)
            DO UPDATE SET config_value = ?, updated_at = CURRENT_TIMESTAMP
        """
        str_value = str(value)
        self.db.execute_query(query, (
            user_id, category, key, str_value, config_type, str_value
        ))

        # Invalidate cache
        cache_key = f"{user_id}:{category}:{key}"
        self._cache.pop(cache_key, None)

    def migrate_from_ini(self, config_parser, user_id: int):
        """Migrate settings from config.ini to database."""
        for section in config_parser.sections():
            for key in config_parser[section]:
                value = config_parser[section][key]

                # Determine type
                if value.lower() in ('true', 'false'):
                    config_type = 'bool'
                elif value.isdigit():
                    config_type = 'int'
                else:
                    try:
                        float(value)
                        config_type = 'float'
                    except ValueError:
                        config_type = 'string'

                self.set_config(user_id, section, key, value, config_type)

    def _convert_value(self, row):
        """Convert stored value to appropriate Python type."""
        value, value_type = row['config_value'], row['config_type']

        if value_type == 'int':
            return int(value)
        elif value_type == 'float':
            return float(value)
        elif value_type == 'bool':
            return value.lower() in ('true', '1', 'yes')
        elif value_type == 'json':
            return json.loads(value)
        return value
```

### 3.6 Activity Logging Integration

```python
# Python/utils/activity_logger.py

class ActivityLogger:
    """Centralized activity logging."""

    def __init__(self, db_manager: DatabaseManager):
        self.db = db_manager

    def log_activity(self, user_id: Optional[int],
                     activity_type: str, category: str,
                     target_type: str = None, target_id: int = None,
                     details: dict = None, success: bool = True,
                     error_message: str = None):
        """Log user activity."""
        query = """
            INSERT INTO ActivityLog (
                user_id, activity_type, activity_category,
                target_type, target_id, action_details,
                success, error_message
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        """
        self.db.execute_query(query, (
            user_id, activity_type, category,
            target_type, target_id,
            json.dumps(details) if details else None,
            success, error_message
        ))

    def log_fit_execution(self, user_id: int, fit_id: int,
                         duration_ms: int, chi_square: float,
                         success: bool):
        """Specialized logging for fit execution."""
        self.log_activity(
            user_id=user_id,
            activity_type='fit_executed',
            category='fitting',
            target_type='fit',
            target_id=fit_id,
            details={
                'duration_ms': duration_ms,
                'chi_square': chi_square
            },
            success=success
        )

    def get_user_activity(self, user_id: int,
                         limit: int = 100,
                         activity_type: str = None):
        """Retrieve user activity history."""
        if activity_type:
            query = """
                SELECT * FROM ActivityLog
                WHERE user_id = ? AND activity_type = ?
                ORDER BY activity_timestamp DESC
                LIMIT ?
            """
            return self.db.execute_query(query,
                                        (user_id, activity_type, limit))
        else:
            query = """
                SELECT * FROM ActivityLog
                WHERE user_id = ?
                ORDER BY activity_timestamp DESC
                LIMIT ?
            """
            return self.db.execute_query(query, (user_id, limit))
```

---

## 4. GUI Integration Recommendations

### 4.1 Settings Dialog Enhancement

Extend the existing `SettingsDialog` in `helpers.py` to include database settings:

```python
# Add to SettingsDialog in Python/helpers.py

def __init__(self, mainGUI, config, config_file_path, parent=None):
    # ... existing code ...

    # Database tab
    database_layout = QtWidgets.QVBoxLayout()

    label_db_path = QtWidgets.QLabel("Database Path:")
    self.line_edit_db_path = QtWidgets.QLineEdit()
    self.line_edit_db_path.setText(
        config.get('Database', 'db_path',
                   fallback='../Databases/lg4x-v2-univers.db')
    )

    button_browse_db = QtWidgets.QPushButton("Browse...")
    button_browse_db.clicked.connect(self.browse_database)

    self.checkbox_enable_history = QtWidgets.QCheckBox(
        "Enable parameter history tracking"
    )
    self.checkbox_enable_history.setChecked(
        config.getboolean('Database', 'enable_history', fallback=True)
    )

    self.checkbox_enable_activity_log = QtWidgets.QCheckBox(
        "Enable activity logging"
    )
    self.checkbox_enable_activity_log.setChecked(
        config.getboolean('Database', 'enable_activity_log', fallback=True)
    )

    label_history_freq = QtWidgets.QLabel("History tracking frequency:")
    self.spinbox_history_freq = QtWidgets.QSpinBox()
    self.spinbox_history_freq.setRange(1, 100)
    self.spinbox_history_freq.setValue(
        config.getint('Database', 'history_frequency', fallback=10)
    )
    self.spinbox_history_freq.setSuffix(" iterations")

    database_layout.addWidget(label_db_path)
    db_path_layout = QtWidgets.QHBoxLayout()
    db_path_layout.addWidget(self.line_edit_db_path)
    db_path_layout.addWidget(button_browse_db)
    database_layout.addLayout(db_path_layout)
    database_layout.addWidget(self.checkbox_enable_history)
    database_layout.addWidget(self.checkbox_enable_activity_log)
    database_layout.addWidget(label_history_freq)
    database_layout.addWidget(self.spinbox_history_freq)

    # Update tab3 layout
    tab3.setLayout(database_layout)
```

### 4.2 History Viewer Widget

Create a new widget to visualize parameter history:

```python
# Python/gui_helpers.py

class ParameterHistoryViewer(QtWidgets.QDialog):
    """Dialog for viewing parameter evolution during fitting."""

    def __init__(self, fit_id: int, db_manager: DatabaseManager,
                 parent=None):
        super().__init__(parent)
        self.setWindowTitle("Parameter History")
        self.fit_id = fit_id
        self.db = db_manager

        self.figure, self.ax = plt.subplots(figsize=(10, 6))
        self.canvas = FigureCanvas(self.figure)

        # Parameter selection
        self.combo_param = QtWidgets.QComboBox()
        self.combo_param.currentTextChanged.connect(self.update_plot)

        # Layout
        layout = QtWidgets.QVBoxLayout(self)
        layout.addWidget(QtWidgets.QLabel("Select Parameter:"))
        layout.addWidget(self.combo_param)
        layout.addWidget(self.canvas)

        self.load_parameters()

    def load_parameters(self):
        """Load available parameters for this fit."""
        query = """
            SELECT DISTINCT parameter_name
            FROM ParameterHistory
            WHERE fit_id = ?
            ORDER BY parameter_name
        """
        results = self.db.execute_query(query, (self.fit_id,))
        self.combo_param.addItems([row['parameter_name'] for row in results])

    def update_plot(self, param_name: str):
        """Update plot with selected parameter."""
        if not param_name:
            return

        query = """
            SELECT iteration, parameter_value, chi_square
            FROM ParameterHistory
            WHERE fit_id = ? AND parameter_name = ?
            ORDER BY iteration
        """
        results = self.db.execute_query(query, (self.fit_id, param_name))

        iterations = [row['iteration'] for row in results]
        values = [row['parameter_value'] for row in results]
        chi_squares = [row['chi_square'] for row in results]

        self.ax.clear()
        ax2 = self.ax.twinx()

        self.ax.plot(iterations, values, 'b-', label=param_name)
        ax2.plot(iterations, chi_squares, 'r--', alpha=0.5, label='χ²')

        self.ax.set_xlabel('Iteration')
        self.ax.set_ylabel(param_name, color='b')
        ax2.set_ylabel('χ²', color='r')

        self.ax.legend(loc='upper left')
        ax2.legend(loc='upper right')

        self.canvas.draw()
```

### 4.3 Menu Integration

Add database-related menu items to main window:

```python
# Add to createMenuBar in gui_helpers.py

def createMenuBar(self):
    # ... existing menu items ...

    # Database menu
    db_menu = self.menuBar().addMenu('Database')

    history_action = QtWidgets.QAction('View Parameter History', self)
    history_action.triggered.connect(self.show_parameter_history)
    db_menu.addAction(history_action)

    activity_action = QtWidgets.QAction('View Activity Log', self)
    activity_action.triggered.connect(self.show_activity_log)
    db_menu.addAction(activity_action)

    db_menu.addSeparator()

    backup_action = QtWidgets.QAction('Backup Database', self)
    backup_action.triggered.connect(self.backup_database)
    db_menu.addAction(backup_action)

    optimize_action = QtWidgets.QAction('Optimize Database', self)
    optimize_action.triggered.connect(self.optimize_database)
    db_menu.addAction(optimize_action)
```

---

## 5. Data Migration Strategy

### 5.1 Existing Data Import

Create migration scripts to import existing data:

```python
# Python/database/migrations/import_existing_data.py

def migrate_presets_to_database(preset_dir: str, db_manager: DatabaseManager,
                                user_id: int):
    """Import existing preset files into database."""
    for preset_file in Path(preset_dir).glob('*_pars.dat'):
        with open(preset_file, 'r') as f:
            # Parse preset file format: [bg_type, [bg_params], [fit_params]]
            preset_data = ast.literal_eval(f.read())

            # Extract information
            bg_type = preset_data[0]
            bg_params = preset_data[1]
            peak_params = preset_data[2]

            # Determine spectrum type from filename
            preset_name = preset_file.stem.replace('_pars', '')

            # Insert into database
            query = """
                INSERT INTO FitPresets (
                    preset_name, created_by, bg_type, bg_params,
                    peak_models, preset_file_path
                ) VALUES (?, ?, ?, ?, ?, ?)
            """
            db_manager.execute_query(query, (
                preset_name, user_id, bg_type,
                json.dumps(bg_params),
                json.dumps(peak_params),
                str(preset_file)
            ))

def migrate_config_ini(config_file: str, db_manager: DatabaseManager,
                      user_id: int):
    """Migrate config.ini settings to database."""
    config = configparser.ConfigParser()
    config.read(config_file)

    config_service = ConfigService(db_manager)
    config_service.migrate_from_ini(config, user_id)
```

### 5.2 Backward Compatibility

Maintain compatibility with file-based storage:

```python
# Python/database/compatibility.py

class HybridStorage:
    """Provides backward compatibility with file-based storage."""

    def __init__(self, db_manager: DatabaseManager,
                 use_database: bool = True):
        self.db = db_manager
        self.use_database = use_database

    def save_fit_results(self, fit_id: int, fit_data: dict,
                        file_path: str = None):
        """Save fit results to both database and file."""
        if self.use_database:
            # Save to database
            fit_service = FitService(self.db)
            fit_service.update_fit_results(fit_id, fit_data)

        if file_path or not self.use_database:
            # Save to file (legacy format)
            if not file_path:
                file_path = f"fit_{fit_id}_results.txt"
            with open(file_path, 'w') as f:
                # Write in existing format
                pass

    def load_fit_results(self, fit_id: int = None,
                        file_path: str = None):
        """Load fit results from database or file."""
        if self.use_database and fit_id:
            fit_service = FitService(self.db)
            return fit_service.get_fit_results(fit_id)
        elif file_path:
            # Load from file (legacy format)
            with open(file_path, 'r') as f:
                return self._parse_fit_file(f)
        else:
            raise ValueError("Must provide either fit_id or file_path")
```

---

## 6. Performance Considerations

### 6.1 Indexing Strategy

Key indexes for performance:

```sql
-- Most important indexes for query performance
CREATE INDEX idx_spectra_sample ON Spectra(sample_id, created_at DESC);
CREATE INDEX idx_fits_spectrum_date ON Fits(spectrum_id, created_at DESC);
CREATE INDEX idx_fits_quality ON Fits(chi_square, reduced_chi_square);
CREATE INDEX idx_peaks_fit_number ON Peaks(fit_id, peak_number);
CREATE INDEX idx_activity_user_type ON ActivityLog(user_id, activity_type, activity_timestamp DESC);
CREATE INDEX idx_param_history_fit_iter ON ParameterHistory(fit_id, iteration);
CREATE INDEX idx_config_user_category ON GUIConfigurations(user_id, config_category);

-- Full-text search indexes (if SQLite version supports it)
CREATE VIRTUAL TABLE spectra_fts USING fts5(
    spectrum_name, comments, content=Spectra
);

CREATE VIRTUAL TABLE projects_fts USING fts5(
    project_name, project_description, content=Projects
);
```

### 6.2 Query Optimization

Use views for common complex queries:

```sql
-- View for fit summary with metadata
CREATE VIEW fit_summary AS
SELECT
    f.fit_id,
    f.fit_name,
    f.created_at,
    f.chi_square,
    f.reduced_chi_square,
    f.num_peaks,
    s.spectrum_name,
    s.spectrum_type,
    sa.sample_name,
    p.project_name,
    u.username AS created_by_username
FROM Fits f
JOIN Spectra s ON f.spectrum_id = s.spectrum_id
LEFT JOIN Samples sa ON s.sample_id = sa.sample_id
LEFT JOIN Projects p ON sa.project_id = p.project_id
LEFT JOIN Users u ON f.created_by = u.user_id;

-- View for peak analysis
CREATE VIEW peak_analysis AS
SELECT
    p.peak_id,
    p.fit_id,
    p.peak_number,
    p.peak_label,
    p.model_type,
    p.center,
    p.fwhm,
    p.area,
    f.spectrum_id,
    s.spectrum_name
FROM Peaks p
JOIN Fits f ON p.fit_id = f.fit_id
JOIN Spectra s ON f.spectrum_id = s.spectrum_id;

-- View for recent activity
CREATE VIEW recent_activity_detailed AS
SELECT
    al.activity_id,
    al.activity_timestamp,
    al.activity_type,
    al.activity_category,
    u.username,
    al.action_details,
    al.success
FROM ActivityLog al
LEFT JOIN Users u ON al.user_id = u.user_id
ORDER BY al.activity_timestamp DESC;
```

### 6.3 Batch Operations

For importing large datasets:

```python
def import_spectrum_batch(spectra_data: list, db_manager: DatabaseManager):
    """Import multiple spectra efficiently."""
    query = """
        INSERT INTO Spectra (
            sample_id, spectrum_name, spectrum_type,
            photon_energy, work_function, raw_data_file
        ) VALUES (?, ?, ?, ?, ?, ?)
    """
    params_list = [
        (s['sample_id'], s['name'], s['type'],
         s['hv'], s['wf'], s['file_path'])
        for s in spectra_data
    ]

    db_manager.execute_many(query, params_list)
```

---

## 7. Security Considerations

### 7.1 Data Privacy

- Store user passwords as salted hashes (use bcrypt or argon2)
- Never log sensitive information in ActivityLog
- Implement role-based access control for shared projects
- Regular database backups with encryption

### 7.2 SQL Injection Prevention

Always use parameterized queries:

```python
# GOOD - parameterized query
cursor.execute("SELECT * FROM Users WHERE username = ?", (username,))

# BAD - string concatenation (vulnerable!)
cursor.execute(f"SELECT * FROM Users WHERE username = '{username}'")
```

### 7.3 Database Backup

```python
def backup_database(db_path: str, backup_dir: str):
    """Create timestamped database backup."""
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    backup_path = Path(backup_dir) / f"lg4x_backup_{timestamp}.db"

    # Use SQLite backup API
    source_conn = sqlite3.connect(db_path)
    backup_conn = sqlite3.connect(str(backup_path))

    with backup_conn:
        source_conn.backup(backup_conn)

    source_conn.close()
    backup_conn.close()

    return backup_path
```

---

## 8. Testing Strategy

### 8.1 Unit Tests

```python
# test/test_database.py

import unittest
from Python.database.connection import DatabaseManager
from Python.services.fit_service import FitService

class TestFitService(unittest.TestCase):

    def setUp(self):
        """Create test database."""
        self.db = DatabaseManager()
        self.db.initialize(':memory:')  # In-memory database for testing
        self.fit_service = FitService(self.db)

    def test_create_fit(self):
        """Test fit creation."""
        fit_id = self.fit_service.create_fit(
            spectrum_id=1,
            user_id=1,
            fit_params={
                'bg_type': 'Shirley',
                'bg_params': {'cv': 1.0},
                'x_min': 270,
                'x_max': 300,
                'num_peaks': 3
            }
        )
        self.assertIsNotNone(fit_id)
        self.assertGreater(fit_id, 0)

    def test_parameter_history_tracking(self):
        """Test parameter history recording."""
        fit_id = 1
        self.fit_service.track_parameter_change(
            fit_id=fit_id,
            iteration=1,
            param_name='peak_1_amplitude',
            param_value=1000.5,
            chi_square=125.3
        )

        history = self.fit_service.get_parameter_history(
            fit_id, 'peak_1_amplitude'
        )
        self.assertEqual(len(history), 1)
        self.assertEqual(history[0]['iteration'], 1)
```

### 8.2 Integration Tests

Test database operations with actual GUI:

```python
def test_fit_workflow_with_database():
    """Integration test for complete fitting workflow."""
    # 1. Import spectrum
    # 2. Create fit
    # 3. Track parameters
    # 4. Save results
    # 5. Verify database state
    pass
```

---

## 9. Documentation Requirements

### 9.1 User Documentation

Create user-facing documentation for:
- How to use parameter history viewer
- Understanding activity logs
- Managing projects and samples
- Sharing and collaboration features

### 9.2 Developer Documentation

Document:
- Database schema with ER diagrams
- API reference for database services
- Migration procedures
- Backup and recovery procedures

### 9.3 Schema Documentation

Generate schema documentation automatically:

```bash
# Use tools like SchemaSpy or SQLite schema export
sqlite3 lg4x-v2-univers.db .schema > schema.sql

# Or use Python to generate documentation
python generate_schema_docs.py
```

---

## 10. Next Steps and Priorities

### Priority 1: Critical (Do First)
1. ✅ Enhance database schema (Users, enhanced Fits, Peaks)
2. Implement DatabaseManager with proper connection handling
3. Create FitService with parameter history tracking
4. Integrate database into fit execution workflow
5. Test with existing workflows

### Priority 2: Important (Do Soon)
1. Add ConfigService and migrate config.ini
2. Implement activity logging
3. Create parameter history viewer widget
4. Add preset management in database
5. Create migration tools for existing data

### Priority 3: Nice to Have (Do Later)
1. Add tags and annotations
2. Implement full-text search
3. Create advanced query interfaces
4. Add performance monitoring
5. Implement collaboration features

### Priority 4: Future Enhancements
1. Web-based interface for remote access
2. Cloud synchronization
3. Machine learning integration for parameter suggestions
4. Advanced analytics dashboard
5. Export to other formats (CasaXPS, etc.)

---

## 11. Example Integration Points

### 11.1 Modify Fit Execution in main.py

```python
# In Python/main.py, modify the fit execution:

def fit_curve(self):
    """Execute fitting with database integration."""
    # Create fit record
    fit_service = FitService(self.db_manager)
    fit_id = fit_service.create_fit(
        spectrum_id=self.current_spectrum_id,
        user_id=self.current_user_id,
        fit_params=self.get_fit_parameters()
    )

    # Enable parameter tracking
    self.fit_thread.set_fit_id(fit_id)
    self.fit_thread.enable_history_tracking()

    # Execute fit
    self.fit_thread.start()

    # Save results when complete
    self.fit_thread.finished.connect(
        lambda: self.save_fit_results(fit_id)
    )
```

### 11.2 Track Parameters During Optimization

```python
# Modify the FitThread class to track parameters:

class FitThread(QtCore.QThread):
    def __init__(self, parent):
        super().__init__()
        self.parent = parent
        self.fit_id = None
        self.track_history = False
        self.iteration = 0
        self.fit_service = None

    def set_fit_id(self, fit_id: int):
        self.fit_id = fit_id
        self.fit_service = FitService(self.parent.db_manager)

    def enable_history_tracking(self, enabled: bool = True):
        self.track_history = enabled

    def iter_callback(self, params, iteration, resid, *args, **kwargs):
        """Callback during fitting to track parameters."""
        if not self.track_history or not self.fit_id:
            return

        self.iteration += 1
        chi_square = np.sum(resid**2)

        # Track every 10th iteration to avoid overhead
        if self.iteration % 10 == 0:
            for param_name, param in params.items():
                self.fit_service.track_parameter_change(
                    fit_id=self.fit_id,
                    iteration=self.iteration,
                    param_name=param_name,
                    param_value=param.value,
                    chi_square=chi_square
                )

    def run(self):
        """Execute fit with callback."""
        try:
            result = lmfit.minimize(
                self.objective_function,
                self.params,
                method=self.method,
                iter_cb=self.iter_callback if self.track_history else None
            )
            # ... rest of fitting code ...
        except Exception as e:
            logging.error(f"Fit failed: {e}")
```

---

## 12. Conclusion

This comprehensive database integration will transform LG4X-V2 into a professional scientific analysis platform with:

- **Complete traceability**: Every parameter change tracked
- **User management**: Multi-user support with profiles
- **Intelligent organization**: Projects, samples, spectra hierarchy
- **Performance insights**: Track fitting performance and optimize
- **Collaboration**: Share projects and presets
- **Reproducibility**: Full audit trail for scientific publications

The schema is designed to be extensible, allowing future additions without breaking changes. The implementation strategy provides a clear path from the current basic schema to a comprehensive system.

Start with Priority 1 items to get core functionality working, then progressively add features based on user needs and feedback.

---

## Appendix A: SQL Schema Summary

See the complete SQL schema in sections 2.2.1 through 2.2.7.

Total tables: 30+
Key relationships: Projects → Samples → Spectra → Fits → Peaks
Supporting tables: Users, Configuration, Activity Logging, Tags, Presets

## Appendix B: Useful Queries

```sql
-- Find best fits for a spectrum
SELECT fit_id, fit_name, reduced_chi_square, num_peaks
FROM Fits
WHERE spectrum_id = ?
ORDER BY reduced_chi_square ASC
LIMIT 10;

-- Get user statistics
SELECT
    u.username,
    COUNT(DISTINCT f.fit_id) as num_fits,
    COUNT(DISTINCT s.spectrum_id) as num_spectra,
    AVG(f.reduced_chi_square) as avg_quality
FROM Users u
LEFT JOIN Fits f ON u.user_id = f.created_by
LEFT JOIN Spectra s ON f.spectrum_id = s.spectrum_id
GROUP BY u.user_id;

-- Find similar fits (by peak configuration)
SELECT f1.fit_id, f1.num_peaks, f1.reduced_chi_square
FROM Fits f1
JOIN Fits f2 ON f1.spectrum_id = f2.spectrum_id
WHERE f2.fit_id = ? AND f1.fit_id != ?
  AND f1.num_peaks = f2.num_peaks
ORDER BY ABS(f1.reduced_chi_square - f2.reduced_chi_square);
```

---

**Document Version**: 1.0
**Date**: 2025-01-30
**Author**: Claude (Anthropic)
**For**: LG4X-V2 Database Integration Project
