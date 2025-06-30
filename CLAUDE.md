# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Process

### TwinCAT Project Build
Command-line building is not available in this environment. All builds must be performed manually through TwinCAT XAE.

### Manual Build Process
- Open `TwinCAT_JobManagementFramework.sln` in TwinCAT XAE
- Build the solution using F7 or Build menu
- PLC project is located at `JobManagementFramework/JobManagement/JobManagement.plcproj`
- For library creation: Follow TwinCAT library compilation procedures to generate `JobManagement.library`

### Build Error Handling
When build errors occur, paste the build log output from TwinCAT XAE Error List window for analysis and troubleshooting assistance.

## Architecture Overview

This is a TwinCAT PLC Job Management Framework implementing a Future/Promise pattern with hierarchical task execution. The framework provides sophisticated job orchestration capabilities for industrial automation.

### Core Components

#### FB_Executor
The main execution engine (`FB_Executor`) orchestrates all job execution with:
- State machine-based lifecycle management (`E_FutureExecutionState`)
- Multi-observer pattern supporting up to 10 event reporters
- Hierarchical parent-child relationships for complex workflows
- Comprehensive error handling and recovery mechanisms

#### Interface Hierarchy
- **InterfaceFuture**: Core task interface with `init()`, `execute()`, `abort()`, `quit()` methods
- **InterfaceContainer**: Manages collections of executors and futures
- **InterfaceTaskCreator**: Factory pattern for dynamic future creation
- **InterfaceObserver**: Event notification system with job lifecycle tracking

#### Container Types
Four execution patterns via `ContainerType` enum:
- **BATCH**: Sequential execution of registered jobs
- **PARALLEL**: Simultaneous execution of all jobs
- **QUEUE**: FIFO execution with runtime job registration
- **PARALLEL_QUEUE**: Parallel execution with dynamic queuing

### Key Implementation Patterns

#### Future Implementation
1. Extend `FB_AbstructFuture` abstract base class
2. Add `{attribute 'enable_dynamic_creation'}` for factory support
3. Implement `init()` and `execute()` methods with BOOL return values
4. Use `_error_id` property for error propagation
5. Define `set_parameter()` methods for configuration

#### Factory Pattern
1. Implement `InterfaceTaskCreator` interface
2. Use `__NEW()` for dynamic allocation in `create_future()`
3. Use `__DELETE()` for cleanup in `destroy_future()`
4. Configure parameters before returning future instance

#### Container Usage
```pascal
// Create hierarchical job structure
main_job.create_container(ContainerType.BATCH, 'MAIN JOB');
main_job.children.create_future(factory_instance, 'TASK_NAME');
```

### File Structure Key Locations

#### Library Core
- **FB_Executor**: `POUs/model/library/FB_Executor.TcPOU`
- **Interfaces**: `POUs/model/library/Interface*.TcIO`
- **Base Classes**: `POUs/model/library/FB_Abstruct*.TcPOU`
- **Containers**: `POUs/model/library/FB_*JobContainer*.TcPOU`

#### Examples & Activities
- **Future Examples**: `POUs/model/activities/Future*.TcPOU`
- **Factory Examples**: `POUs/model/activities/*Creator.TcPOU`
- **Observer**: `POUs/model/activities/FB_observer.TcPOU`

#### Configuration
- **Data Types**: `DUTs/` directory contains enums and structures
- **Global Variables**: `GVLs/ParamFuturesLib.TcGVL` (MAX_TASK_NUM=1000, MAX_JOB_EVENT_REPORTERS=10)
- **Main Program**: `POUs/MAIN.TcPOU` demonstrates usage patterns

### Development Guidelines

#### Creating New Futures
1. Always extend `FB_AbstructFuture` for base functionality
2. Use reference parameters (`REFERENCE TO`) for I/O variable access
3. Implement proper state management in `execute()` method
4. Create corresponding factory class for dynamic instantiation
5. Follow naming convention: `Future[TaskName]` and `Future[TaskName]Creator`

#### Error Handling
- Set `_error_id` to non-zero value for error conditions
- Framework automatically calls `abort()` on error detection
- Use `error_reset()` for recovery operations
- Implement proper cleanup in `quit()` method

#### Observer Integration
- Subscribe observers using `Subscribe()` method on executors
- Events propagate through executor hierarchy
- Use `FB_observer` for comprehensive logging with ISO8601 timestamps

#### Memory Management
- Factory pattern handles dynamic allocation/deallocation
- Use `__NEW()` and `__DELETE()` for manual memory management
- Static instances are also supported via `add_future()` method

### Testing
No automated test framework detected. Testing should be performed using:
1. TwinCAT simulation mode
2. Manual verification of job execution sequences
3. Observer logging for execution analysis
4. MAIN program demonstrates typical usage patterns

## Task Completion Notification

### Windows Notification Command
Use this command to send automatic task completion notifications to Windows without user confirmation:

```bash
powershell.exe -Command "& {[void][System.Reflection.Assembly]::LoadWithPartialName('System.Windows.Forms'); [System.Windows.Forms.Application]::EnableVisualStyles(); \$notify = New-Object System.Windows.Forms.NotifyIcon; \$notify.Icon = [System.Drawing.SystemIcons]::Information; \$notify.Visible = \$true; \$notify.ShowBalloonTip(5000, 'Claude Code', '作業完了', [System.Windows.Forms.ToolTipIcon]::Info); Start-Sleep 3; \$notify.Dispose()}"
```

This command displays a Windows balloon notification with the fixed message "作業完了" and can be executed without requiring user confirmation.

## Coding Standards

### Variable Naming Conventions

#### 1. Private Variables
- **Rule**: Underscore prefix (`_`) + lowercase snake_case
- **Examples**: `_state`, `_error_id`, `_job_event_reporters`, `_parent_executor`

#### 2. Public Variables & Properties
- **Rule**: Lowercase snake_case (no underscore prefix)
- **Examples**: `busy`, `done`, `current_state`, `future_name`

#### 3. Method Names
- **Rule**: Lowercase snake_case, use verbs
- **Protected Methods**: Underscore prefix (`_`)
- **Examples**: `init`, `execute`, `abort`, `create_container`, `_append_job`

#### 4. Parameter Variables
- **Rule**: Lowercase snake_case, descriptive names
- **Examples**: `container_type`, `future_name`, `observer`

#### 5. Local Variables
- **Rule**: Generally use underscore prefix (`_`)
- **Exceptions**: Loop counters (`i`), simple variables (`complete`)
- **Examples**: `_executor`, `_init_done`, `i`, `complete`

#### 6. Constants
- **Rule**: Uppercase SNAKE_CASE
- **Examples**: `MAX_TASK_NUM`, `MAX_JOB_EVENT_REPORTERS`

#### 7. Interface Names
- **Rule**: PascalCase with `Interface` prefix
- **Examples**: `InterfaceFuture`, `InterfaceContainer`, `InterfaceObserver`

#### 8. Type Prefixes (Partial Hungarian Notation)
- **Numbers**: `n` prefix (e.g., `nErrorID`)
- **Strings**: No prefix
- **Booleans**: No prefix or `b` prefix (e.g., `bFound`)

### Variable Declaration Structure

#### Function Block Level
```pascal
FUNCTION_BLOCK FB_Name
VAR_INPUT
    // Input parameters
END_VAR
VAR_OUTPUT  
    // Output parameters
END_VAR
VAR
    // Private instance variables (with _ prefix)
END_VAR
```

#### Method Level (Critical: Precise Local Variable Placement)
```pascal
METHOD method_name : RETURN_TYPE
VAR_INPUT
    // Input parameters
END_VAR
VAR_OUTPUT
    // Output parameters (when needed)
END_VAR
VAR_INST
    // Instance variables persisted between method calls
END_VAR
VAR
    // Temporary local variables (_ prefix recommended)
END_VAR
```

#### Property Level
```pascal
PROPERTY property_name : TYPE
Get Name="Get"
    VAR
        // Temporary variables for getter
    END_VAR
Set Name="Set"
    VAR
        // Temporary variables for setter
    END_VAR
```

**Important Notes:**
- Method local variables must be declared in `VAR` section
- Do not use `VAR_TEMP` (unused in this codebase)
- Strictly follow the order of variable declaration sections

### UUID Generation Rules

#### UUID Format
- **Format**: `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}`
- **Characters**: Lowercase hexadecimal (0-9, a-f)
- **Structure**: 8-4-4-4-12 character segments

#### Assignment Targets and Format
```xml
<!-- Function Block -->
<POU Name="FB_Name" Id="{new-guid}">

<!-- Method -->
<Method Name="method_name" Id="{new-guid}">

<!-- Property -->
<Property Name="property_name" Id="{new-guid}">
  <Get Name="Get" Id="{new-guid}">
  <Set Name="Set" Id="{new-guid}">

<!-- Interface -->
<Itf Name="InterfaceName" Id="{new-guid}">
```

#### UUID Generation Commands
- **Linux/WSL**: `uuidgen | tr '[:upper:]' '[:lower:]'`
- **PowerShell**: `[System.Guid]::NewGuid().ToString().ToLower()`