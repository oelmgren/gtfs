# GTFS Bus Stop Predictions Documentation

This document will go through the steps taken to create predictions for bus stop arrival, departure, and dwell times using pings (bus location and timestamp) from throughout the route.

## Original Data
1. Materialized View `vp_lines` </br>
Columns:
```
agency_id 
closest_point
line
speed
stop_id
stop_sequence
timestamp
trip_id
ts
```

2. Materialized View `stops_line` </br>
Columns:
```
agency_id 
arrival_time_s
closest_point
departure_time_s
line
stop_id
stop_name
stop_sequence
trip_id
```


It is important to note that I was only working with an `agency_id` of 0 which corresponds to just the Modesto data. The Modesto data is unique because the pings happen at given locations along each route, whereas other sets of data ping after a certain number of seconds.

## Original Data Pre-Processing 
### `vp_lines`
For vp_lines there are steps a - e to clean the data, get in pst, and add helpful columns. The final product is `vp_lines_e` which has the following columns: 
```
closest_point
timestamp
trip_id
trip_id_upd
trip_percent_complete
variation_id
```
`closest_point` is a `geometry` on where the location is. `trip_id` is the given trip_id by the city of modesto, but is not unique to the date so we created `trip_id_upd` (a hash of the date and `trip_id` concatenated) so that we can easily differentiate individual trips. `trip_percent_complete` is a measure of how far along the trip the ping is, and is determined by taking the `total distance traveled up to that point / total distance traveled on the whole trip`. It is used for comparison to the stops and is helpful in identifying which direction the bus is going in overlapping segments. Lastly, `variation_id` is a hash of a sequence of stops all concatenated. This gives us a unique identifier for every route. 

### `stops_line`
For `stops_line` we needed to take the input data and segment all the `line`'s on a stop by stop basis. This allows us to later snap the pings from `vp_lines` to the segments before and after each stop. 

**Steps** 
```mermaid
flowchart TD
stops_line --simplifying stop_seq to stop_seq_upd--> stops_line_a
stops_line_a --getting variation_id--> stops_line_b
stops_line_b --getting first_point and second_point which are later used to actually split the segments from the route line--> stops_line_segmented_a
stops_line_segmented_a --pre-processing step before cutting points from route line-->stops_line_segmented_c
stops_line_segmented_c --segmentation step--> stops_line_segmented_d
stops_line_segmented_d -- accounting for null, cleaning--> stops_line_segmented_e
```

At this point we have the data in a format that we can effectively use in our predictions algorithm.  

## Arrival and Departure Prediction Algorithm 
1. `segment_buffer_creation`: The most important part of this step is the creation of `buff_startpoint` and `buff_endpoint`. These points are located on the segments between every stop 30 meters from both sides. It is helpful to create indexes on `vp_lines_e`'s `trip_id_upd` and `trip_percent_complete` columns.
2.  `segment_buffer_creation_percentiles`: Builds on `segment_buffer_creation` and adds in a percentile for how far along the trip each buffer point is. This will be compared with the ping percentiles to ensure the bus is moving in the same direction on overlapping parts of a route. Create index's on `stop_point`, `buff_startpoint`, `buff_endpoint`, and `trip_id_upd`. For the start and endpoint use GIST.
3. `predicted_stop_times_buff_base`: Sets up for predictions to happen. Finds closest ping to each buffer point that is: 
  **a)** between the `buff_startpoint` and endpoint when snapped
  **b)** less than 10 meters away from the segment
  **c)** less than 0.1 of difference from the other percentile 
  (```ABS(vp_lines_e.trip_percent_complete - segment_buffer_creation_percentiles.sp_percent_traveled) < 0.1```)
  **d)** less than 1000 meters away from the buffer point. If none these are met than a point is not assigned.
4. `predicted_stop_times_buff`: Simple step to calculate the actual arrival and departure times. These are all the final predictions for the stops (currently excluding the first and last stop on every trip).
5. `dwell_times_buff_zeroed`: Calculates dwell times for all stops. Also sets the arrival time as the departure time when the predicted arrival time is greater than the predicted departure time -- hence "zeroed". Should index: `stop_name` and `trip_id_upd`.
6. `dwell_times_comparison_base`: Sets up comparison of times to `cleaned_actual_stop_times_pst`, our data for the actual stop times from Modesto.
7. `dwell_times_comparison`: The actual comparison of times. 
8. `dwell_times_comparison_routes`: Some statistics on the comparison of times on a route by route basis.
  

