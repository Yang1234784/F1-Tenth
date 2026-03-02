# F1-Tenth

# Key concept of the code
We uses LiDAR data to find the safest and most open direction to drive. It removes unsafe regions near obstacles (safety bubble + disparity extension), then chooses a target direction with the largest available free space and converts that into steering and speed commands.

we first understand the meaning of the parameters and how they affect the car: 
## 1) lookahead_distance
What it is: a cap on how far the algorithm “cares” about space.

In code: it clips LiDAR ranges:

proc_ranges = np.where(proc_ranges > lookahead_distance, lookahead_distance, proc_ranges)

### Effect
Bigger → more “long-term” vision, tends to pick smoother directions, can go faster on straights.

Smaller → reacts only to nearby space, can become twitchy and turn late.

Too big risk: might ignore near obstacles because many directions look equally “maxed out” after clipping.

Too small risk: jerky steering, slower.

## 2) robot_width 
What it is: width of the car (meters).

## 3) obstacle_bubble_radius
What it is: the safety radius around the closest obstacle.

In code: it turns into an angle atan(radius / distance) then zeros out those indices.

### Effect
Bigger → safer (keeps more distance), but removes more possible paths → may steer away too aggressively and slow down.

Smaller → allows tighter passing and potentially faster lines, but higher crash risk.

## 4) disparity_threshold
What it is: sensitivity for detecting “edges” in LiDAR where distance jumps suddenly.

In code:
range_diffs = abs(diff(proc_ranges))

disparity_indices = where(range_diffs > disparity_threshold)

### Effect
Lower (e.g., 0.3) → detects more disparities → inflates more obstacles → safer but can overreact and weave.

Higher (e.g., 0.8) → detects fewer → faster/more direct but may clip obstacle corners.

## 5) max_speed 
What it is: the top speed the controller will command.

In code, speed is:
raw_speed = max_speed - abs(steering_angle)*speed_gain

speed = max(min_speed, raw_speed)

### Effect
Higher → faster on straights, but if steering is not smooth you’ll crash.

## 6) min_speed 
What it is: the slowest speed allowed even in sharp turns.

### Effect
Higher → you won’t slow down enough in corners → more crashes.

Lower → safer cornering but slower lap time.

## 7) max_steering 
What it is: the hard cap on steering angle (radians).

In code:
steering_angle = clip(..., -max_steering, max_steering)

### Effect
Higher → can turn tighter (useful for sharp corners), but can cause oscillation and instability at speed.

Lower → smoother and more stable at high speed, but may fail to make tight turns.

## 8) disparity_bubble_radius
What it is: how much you “inflate” obstacles at disparity points (edges).

### Effect
Bigger → safer near obstacle edges, but removes more gap space → can force wider lines and slow down.

Smaller → more aggressive passing, faster, but higher risk of hitting edges.

## 9) consecutive_valid_gap 
Effect: none.

## 10) steering_gain
What it is: multiplies steering before clipping

In code:
steering_angle = clip(steering_angle * steering_gain, ...)

### Effect
Higher → more aggressive turning (responds strongly), can help in tight tracks but can oscillate.

Lower → smoother, more stable, but may understeer (doesn’t turn enough).

## 11) speed_gain
What it is: how strongly turning reduces speed:

In code: 
raw_speed = max_speed - abs(steering_angle)*speed_gain

### Effect
Higher → slows down a lot when steering → safer but slower.

Lower → maintains speed through turns → faster but riskier.

## 12) field_of_vision (π/2)
What it is: how wide an angle of LiDAR you consider around the front.

### Effect
Wider → sees more side space, can pick wider racing lines, but can get distracted by side openings.

Narrower → focuses forward (stable at speed), but might miss side gaps in hairpins.

--------------------------------------------------------------------------------------------------------------------------------------------------------------
Our first approach is to maximize the speed according to 
raw_speed = max_speed - abs(steering_angle)*speed_gain

speed = max(min_speed, raw_speed)

However, we found that the maximum speed we can achieve is 6.8, after 7, it has high probability colliding. 

So, we increase the lookahead_distance to let the car “see” and plan slightly farther ahead.
To avoid collisions:
1. increase obstacle_bubble_radius because we found that most of the collisions happen on the corners part in the middle of the map
2. decrease disparity_threshold to increase the sensitivity to edges.
3. increase disparity_bubble_radius to inflate the edges because sometimes the collision is due to the width of the car and control delay

After this, we can finally increase our max_speed and min_speed.

On top of that, we increase speed_gain, we think this the main factor that affects our stability.

Other than that, we also increase steering_gain to let the car turns more decisively toward the chosen gap.

Besides tuning on the parameters, we also tried to modify the codes. We added these three lines of codes: 

ranges = np.array(data.ranges, dtype=np.float32) 
ranges[~np.isfinite(ranges)] = 0.0 
ranges = np.clip(ranges, 0.0, data.range_max)

They are LiDAR data-cleaning and safety steps. They make gap-finder more stable because it stops “bad” LiDAR values from messing up obstacle detection and gap choice.
