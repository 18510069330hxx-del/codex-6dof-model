# Initial Rotor Tilt Simulation Plan

## Cases

- zero_0
- stagger_x_3
- stagger_x_5
- inward_y_3
- inward_y_5
- outward_y_5

## Flight Conditions

1. Hover
2. Forward flight at 20 m/s

## Tilt Definitions

- stagger_x: model motors 1/4/5 tilt +X, motors 2/3/6 tilt -X
- inward_y: model motors 1/3/5 tilt +Y, motors 2/4/6 tilt -Y
- outward_y: model motors 1/3/5 tilt -Y, motors 2/4/6 tilt +Y

## First Run Order

1. 0 deg hover smoke check
2. 0 deg 20 m/s forward-flight aero check
3. Candidate tilt cases

## Main Outputs

- rotor speed
- power margin
- body force and moment
- aerodynamic force and moment
- angular acceleration
- p/q/r response
- coupling
- saturation or limit issues
