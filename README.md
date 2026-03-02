# F1-Tenth
we first understand the meaning of the parameters and how they affect the car
## 1) lookahead_distance (3.0)
What it is: a cap on how far the algorithm “cares” about space.
In code: it clips LiDAR ranges:

proc_ranges = np.where(proc_ranges > lookahead_distance, lookahead_distance, proc_ranges)

### Effect
Bigger → more “long-term” vision, tends to pick smoother directions, can go faster on straights.

Smaller → reacts only to nearby space, can become twitchy and turn late.

Too big risk: might ignore near obstacles because many directions look equally “maxed out” after clipping.
Too small risk: jerky steering, slower.

2) robot_width (0.2032)

What it is: intended width of the car (meters).
In your current code: it’s declared but not used in calculations.

Effect right now: changing it does nothing unless you later use it to inflate bubbles / gap feasibility.

3) obstacle_bubble_radius (0.12)

What it is: the safety radius around the closest obstacle.
In code: it turns into an angle atan(radius / distance) then zeros out those indices.

Effect

Bigger → safer (keeps more distance), but removes more possible paths → may steer away too aggressively and slow down.

Smaller → allows tighter passing and potentially faster lines, but higher crash risk.

4) disparity_threshold (0.6)

What it is: sensitivity for detecting “edges” in LiDAR where distance jumps suddenly.
In code:

range_diffs = abs(diff(proc_ranges))
disparity_indices = where(range_diffs > disparity_threshold)

Effect

Lower (e.g., 0.3) → detects more disparities → inflates more obstacles → safer but can overreact and weave.

Higher (e.g., 0.8) → detects fewer → faster/more direct but may clip obstacle corners.

5) max_speed (10.0)

What it is: the top speed the controller will command.
In code, speed is:

raw_speed = max_speed - abs(steering_angle)*speed_gain
speed = max(min_speed, raw_speed)

Effect

Higher → faster on straights, but if steering is not smooth you’ll crash.

With your current speed formula, high max_speed only helps if steering stays small.

6) min_speed (1.0)

What it is: the slowest speed allowed even in sharp turns.

Effect

Higher → you won’t slow down enough in corners → more crashes.

Lower → safer cornering but slower lap time.

7) max_steering (0.34)

What it is: the hard cap on steering angle (radians).
In code:

steering_angle = clip(..., -max_steering, max_steering)

Effect

Higher → can turn tighter (useful for sharp corners), but can cause oscillation and instability at speed.

Lower → smoother and more stable at high speed, but may fail to make tight turns.

8) disparity_bubble_radius (0.10)

What it is: how much you “inflate” obstacles at disparity points (edges).
This is the “don’t clip corners” safety margin.

Effect

Bigger → safer near obstacle edges, but removes more gap space → can force wider lines and slow down.

Smaller → more aggressive passing, faster, but higher risk of hitting edges.

9) consecutive_valid_gap (5)

In your current code: declared but not used anywhere.

Effect right now: none.

(Usually people use this to require a gap of at least N consecutive scan points before treating it as drivable.)

10) steering_gain (0.4)

What it is: multiplies steering before clipping:

steering_angle = clip(steering_angle * steering_gain, ...)

Effect

Higher → more aggressive turning (responds strongly), can help in tight tracks but can oscillate.

Lower → smoother, more stable, but may understeer (doesn’t turn enough).

11) speed_gain (0.5)

What it is: how strongly turning reduces speed:

raw_speed = max_speed - abs(steering_angle)*speed_gain

Effect

Higher → slows down a lot when steering → safer but slower.

Lower → maintains speed through turns → faster but riskier.

12) field_of_vision (π/2)

What it is: how wide an angle of LiDAR you consider around the front.
Your code zeros out everything outside that window.

Effect

Wider → sees more side space, can pick wider racing lines, but can get distracted by side openings.

Narrower → focuses forward (stable at speed), but might miss side gaps in hairpins.

13) lidarscan_topic (/scan) and 14) drive_topic (/drive)

What they are: ROS topic names.

/scan = where LiDAR data comes in

/drive = where you publish speed + steering

Effect

If these are wrong, your car won’t move (no scan / commands not received).
