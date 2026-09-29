# AGENTS.md

## Project Purpose

This repository is the working bench for the 6B/6H six-rotor 6DoF dynamics model and rotor tilt angle assessment.

The main engineering question is:

> Using the current 6B 0-degree no-tilt configuration as the baseline, decide whether introducing rotor tilt has enough whole-aircraft benefit to justify its aerodynamic, propulsion, control, and structural cost.

The work should support two levels of conclusions:

1. Initial configuration assessment: compare the aircraft body, propulsion demand, force/moment capability, coupling, and margins under different tilt cases.
2. Detailed control assessment: after the initial result is meaningful, evaluate whether flight-control Mixer and PID adaptation are required.

## Collaboration Model

- ChatGPT is used to clarify engineering logic, plan cases, interpret results, and prepare reports.
- Codex is used to edit models, write MATLAB scripts, run batch simulations, organize data, and produce repeatable outputs.
- The repository is the single source of truth for model versions, input data, scripts, simulation results, notes, and report material.

Do not treat scattered desktop files, chat screenshots, or temporary MATLAB states as authoritative unless they are copied into this repository and referenced by scripts or documentation.

## Recommended Repository Structure

Use this structure unless there is a good reason to keep an existing layout:

```text
codex-6dof-model/
├─ model/                 # Simulink models and controlled model backups
├─ scripts/               # MATLAB scripts for setup, batch runs, checks, and plotting
├─ data/
│  ├─ propulsion/          # Thrust/RPM, torque/RPM, power/RPM, limits
│  ├─ aero/                # Aerodynamic force/moment data and coefficient tables
│  ├─ flight_control/      # Mixer, motor order, control parameters, limit logic
│  └─ cases/               # Simulation case definitions
├─ results/                # Simulation outputs, CSV files, MAT files, figures
├─ docs/
│  ├─ plan/                # Work plans, task decomposition, assumptions
│  ├─ notes/               # Meeting notes and engineering decisions
│  └─ report/              # Review PPT/report drafts and conclusion tables
└─ AGENTS.md
```

If the repo already has a different structure, keep existing useful files and migrate gradually. Do not move many files at once unless the user asks for repository cleanup.

## Core Engineering Rules

1. Always compare every tilt case against the current 0-degree baseline.
2. Keep mass, CG, inertia, propulsion limits, environment, and flight condition fixed when comparing tilt angles.
3. Separate configuration effects from flight-control adaptation effects.
4. Do not present a tilted case using the 0-degree Mixer as the final closed-loop conclusion. That case is only a risk or mismatch check.
5. Do not change multiple uncertain assumptions at the same time. If propulsion, aero, Mixer, and tilt definitions are all changing, document each change clearly.
6. Every result must state the model version, case definition, key assumptions, and input data version.
7. If a result depends on a temporary hard-coded value in a MATLAB Function block, document it and plan to move it into a case/config file later.

## Simulink Model Rules

Before modifying any `.slx` model:

1. Create a timestamped backup in `model/backups/` or next to the model if the repo structure is not ready.
2. Record why the change is being made.
3. After editing, run a basic model update or smoke check if MATLAB is available.
4. Do not overwrite the original reference model.

The current working model is expected to include these conceptual modules:

- `motor_model`: converts rotor speeds into body force and body moment.
- `mixer_fcn`: converts total thrust and moment commands into rotor speed commands or actuator commands.
- `Aero_Model`: converts flight condition and aerodynamic data into aerodynamic force and aerodynamic moment.
- `Actuator_Dynamics`: applies motor response and rate limits.
- 6DoF dynamics block: integrates body force and moment into aircraft motion.

For rotor tilt assessment, the first required model change is in `motor_model`:

- Rotor thrust must be a 3D vector, not always fixed along body `-Z`.
- Force and moment must be computed using `M = r x F`.
- Rotor reaction torque should follow the tilted rotor axis direction.

Do not deeply redesign `mixer_fcn` during the first configuration assessment unless explicitly requested.

## Current Model Architecture Requirements

The current initial-study model should use this force and moment architecture:

```text
motor_model -> F_body -> Reshape_F ┐
Aero_Model -> F_aero -------------- ├-> ForceSum -> 6DoF force input
GravBlock -> F_grav --------------- ┘

motor_model -> M_body -> Reshape_M ┐
Aero_Model -> M_aero -------------- ├-> MomentSum -> 6DoF moment input
```

This means:

```text
F_total = F_motor + F_aero + F_gravity
M_total = M_motor + M_aero
```

`Aero_Model` must output both:

- `F_aero`
- `M_aero`

`F_aero` goes into `ForceSum`.
`M_aero` goes into `MomentSum` and is summed with motor moment before entering the 6DoF moment input.

The current aerodynamic moment data are already about CG. Do not add an extra `r x F` correction unless a future aero dataset explicitly uses another moment reference point.

## Known Motor Numbering

The local dynamics model motor order is:

```text
model motor 1 = M0
model motor 2 = M5
model motor 3 = M1
model motor 4 = M4
model motor 5 = M2
model motor 6 = M3
```

Equivalent mapping:

```text
[M0, M1, M2, M3, M4, M5] = [model 1, model 3, model 5, model 6, model 4, model 2]
```

Do not reorder `motor_coord` just to match flight-control document order. The coordinate table should stay in the actual local model order unless the whole model is deliberately refactored.

Important sign convention:

- In the local dynamics model, reverse rotation has been treated as `+1`.
- In the flight-control document, reverse rotation is marked as `-1`.

Therefore, do not blindly copy flight-control `s_i` signs into the dynamics model. Always check yaw moment direction after changing `motor_dir`.

## Current Initial Tilt Case Definitions

For the first-round assessment, use the current 0-degree configuration as the baseline.

Minimum initial cases:

```text
0 deg baseline
front/rear stagger tilt 3 deg
front/rear stagger tilt 5 deg
inward tilt 3 deg
inward tilt 5 deg
outward tilt 5 deg
```

Current local definitions in the MATLAB/Simulink model motor numbering:

- Front/rear stagger tilt:
  - model motors 1, 4, 5 tilt forward along `+X`
  - model motors 2, 3, 6 tilt backward along `-X`
- Inward tilt:
  - model motors 1, 3, 5 tilt toward `+Y`
  - model motors 2, 4, 6 tilt toward `-Y`
- Outward tilt:
  - model motors 1, 3, 5 tilt toward `-Y`
  - model motors 2, 4, 6 tilt toward `+Y`

Use clear names in scripts and reports:

```text
zero_0
stagger_x_3
stagger_x_5
inward_y_3
inward_y_5
outward_y_5
```

Do not call the stagger case "all forward tilt"; it is not all-forward tilt.

`motor_model` should support these first-round tilt modes:

```text
tilt_mode = 0: no tilt
tilt_mode = 1: staggered X tilt
tilt_mode = 2: inward Y tilt
tilt_mode = 3: outward Y tilt
```

Only `tilt_mode` and `tilt_deg` should be changed between first-round cases unless the user explicitly asks for deeper model changes.

## Initial Flight Conditions

First-round simulations should stay small and explainable:

1. Hover.
2. Forward flight at `20 m/s`.

For forward flight, use the current aerodynamic force and moment data at pitch angles `0 deg` and `5 deg`. If interpolation is used, document the interpolation method. If both pitch states are run separately, report them as a small aerodynamic sensitivity range.

Do not expand into large parameter scans until the 0-degree baseline and the first-round tilt cases are understood.

## Propulsion Data Rules

Propulsion data may include:

- thrust vs RPM
- torque vs RPM
- shaft power vs RPM
- electrical input power vs RPM
- continuous and transient speed limits
- continuous and transient power limits
- motor response or time constant

Use propulsion data consistently:

- thrust vs RPM updates thrust coefficient or lookup used by `motor_model`
- torque vs RPM updates reaction torque coefficient or lookup
- shaft/electrical power data supports power estimation and limit checks
- continuous limits are usually assessment constraints, not always hard simulation clamps
- transient limits can be used for short maneuver capability, but must be clearly labeled

Known current propulsion constraints:

```text
continuous maximum rotor speed: 1330 rpm
transient maximum rotor speed: 1750 rpm
transient allowed duration: 30 s
single propulsion system continuous electrical input power: 44 kW
```

Do not silently replace all speed limits with 1330 rpm if the simulation is evaluating transient control capability. Instead, report both continuous margin and transient margin when relevant.

## Aerodynamic Data Rules

Aerodynamic inputs should state:

- coordinate system and sign convention
- reference area `Sref`
- reference length for pitch/yaw/roll moments
- moment reference point
- whether values are forces/moments directly or coefficients
- corresponding speed, pitch angle, sideslip, and configuration

If using coefficients:

```text
Fx = q * Sref * Cx
Fy = q * Sref * Cy
Fz = q * Sref * Cz
Mx = q * Sref * Bref * Cl
My = q * Sref * Cref * Cm
Mz = q * Sref * Bref * Cn
q  = 0.5 * rho * V^2
```

If the aerodynamic moment is not about CG, convert it before feeding the 6DoF moment input:

```text
M_CG = M_ref + cross(r_ref_to_CG, F_aero)
```

For hover, fixed-wing-like aerodynamic coefficients may not be meaningful unless the aero team specifically includes rotor wake, downwash, or interference effects.

## Current Aerodynamic Dataset

The current aerodynamic data are already converted to the local model body axes.

Dataset assumptions:

- forward speed: `20 m/s`
- coordinate system: local model body axes
- force unit: `N`
- moment unit: `N*m`
- moment reference point: CG
- geometry: `0 deg` no-tilt aircraft
- rotor wake/slipstream: included in the aero data

At `20 m/s` and pitch angle `0 deg`:

```text
F_aero = [-872.34; 15.33; -1788.918] N
M_aero = [23.102; 2694.1325; 15.7031] N*m
```

At `20 m/s` and pitch angle `5 deg`:

```text
F_aero = [-933.79; -25.627; -1539.22] N
M_aero = [68.8477; 2599.56; 61.048] N*m
```

For the initial tilt assessment, use these aerodynamic loads for all tilt cases and state the assumption clearly:

> Tilt-induced secondary aerodynamic/wake changes are not recomputed in this first round. Tilt effects enter through rotor thrust direction changes in `motor_model`.

For hover, do not use this 20 m/s aerodynamic dataset. Unless a separate hover aero/wake dataset is provided, use:

```text
F_aero = [0; 0; 0]
M_aero = [0; 0; 0]
```

The current `Aero_Model` should use `V_body` and `euler`, interpolate between the `0 deg` and `5 deg` pitch data, and output zero aerodynamic load at very low speed. Do not add `rotor_tilt_deg` to `Aero_Model` unless future aero data are provided for each tilt configuration.

## Flight-Control and Mixer Rules

The flight-control document describes the current 0-degree 6H Mixer in normalized `T/R/P/Y` channels. It does not automatically replace the dynamics model.

Use these levels clearly:

1. Configuration/body assessment:
   - focus on force, moment, power, control capability, and saturation margin
   - `mixer_fcn` may remain simple or close to the current model
2. 0-degree control baseline check:
   - verify motor order, roll direction, pitch direction, and yaw direction
3. Tilt-adapted control assessment:
   - use a Mixer adapted to the tilted rotor geometry
   - then evaluate whether PID adjustment is needed

For any closed-loop conclusion, specify which control setup was used:

```text
0-degree Mixer + original PID
tilt-adapted Mixer + original PID
tilt-adapted Mixer + retuned PID
```

## Direction Checks

After any change to motor order, `motor_dir`, Mixer, or rotor force direction, perform at least these checks:

```text
positive roll command  -> expected sign of Mx / p
positive pitch command -> expected sign of My / q
positive yaw command   -> expected sign of Mz / r
```

If a simple yaw perturbation also causes roll or pitch moment, do not immediately assume it is wrong. First check whether the perturbation pattern is a pure yaw allocation for the actual geometry.

## Results and Reporting Rules

Each simulation result should be saved with:

- case name
- model file/version
- script name
- date/time
- mass, CG, inertia version
- propulsion data version
- aerodynamic data version
- tilt definition
- flight condition
- control setup

For each tilt case, summarize in this format:

```text
case name:
  main benefit:
  main cost:
  power/RPM margin:
  force and moment capability:
  roll/pitch/yaw angular acceleration:
  coupling:
  saturation or limit issue:
  task suitability:
  recommendation:
  required follow-up:
```

Avoid conclusions like "5 degrees is better" without saying better in what dimension and under which assumptions.

## Safety and Change Control

- Do not run a large simulation sweep unless the user explicitly asks for it.
- Do not delete or overwrite old result files unless the user explicitly asks for cleanup.
- Do not modify original source data from propulsion, aero, flight control, or structure teams.
- Do not treat guessed thresholds as formal acceptance criteria. If a threshold is not from a design requirement or specialist input, label it as a temporary screening criterion.
- When in doubt, keep the model change smaller and make the assumption visible.

## Current Recommended Next Step

For the immediate first-round tilt study:

1. Confirm `motor_model` supports the current `tilt_mode` definitions: `zero_0`, `stagger_x_3`, `stagger_x_5`, `inward_y_3`, `inward_y_5`, and `outward_y_5`.
2. Confirm `Aero_Model` outputs both `F_aero` and `M_aero`, and that `M_aero` is connected through `MomentSum` into the 6DoF moment input.
3. Run 0-degree hover first as a smoke check.
4. Run 0-degree `20 m/s` forward flight as an aero-integration smoke check.
5. Run the five candidate tilt cases.
6. Compare power, rotor speed, force/moment, angular acceleration, coupling, and saturation margin.
7. Only after the first comparison is understood, decide whether to build a tilt-adapted Mixer.
