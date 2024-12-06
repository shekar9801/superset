## README: Data Preparation Pipeline
This script processes telemetry data from multiple sites, applies transformations and feature engineering, and outputs the processed data in a structured format. 
Below is a detailed explanation of the script and how to use it.

Purpose
The script performs the following tasks:

1. Load telemetry data from CSV files.
2. Create fault labels based on specific conditions.
3. Divide data into time slots.
4. Compute statistical metrics for each parameter in 15-minute intervals.
5. Save the processed data in a new CSV format for downstream analysis.


### File Structure

 1. Input Data:
      Folder: 15_site_data/telemetry
      Example: Site-<site_id>.csv

 2. Output Data:
      Folder: version 1 data
      Example: Site-<site_id>.csv

### Setup
Dependencies

   Ensure the following Python libraries are installed:
   1.pandas
   2.numpy
   3.math
   4.os
   5.datetime (part of Python standard library) 
   6.warnings

Install any missing dependencies using pip install <library_name>.

## Code Overview

Class: Data_Preparation:

1. __init__(self, input_file_path=None)
   Initializes the class with a default input file path for telemetry data or uses a user-provided path.

2. load_data(self, file_path)
   Loads CSV data, renames columns for consistency, converts timestamp to datetime, and optimizes memory by converting float64 columns to float32.

3. creating_ac_fault_labels(self, raw_data)
   Adds a new column fault. A fault is marked when:
   ac_voltage is outside the range 225–275 and energy is 0.

4. creating_slots(self, telemetry_data2, minutes)
   Divides data into time slots (e.g., 15-minute intervals).
   Adds columns for date, slot, and statistical metrics (mean, median, std, range, etc.).
   Returns the slotted data sorted by site and timestamp.

5. arrange_data_minute_wise(self, data_pre)
   Aggregates data minute-by-minute within a 15-minute slot. Computes:
   Statistical features: min, max, mean, median, std, range, coefficient of variation (CV).


### Usage
1. Initialize:
   Update the input_file_path variable to point to the folder containing telemetry data files.

2. Run the Script:
   The script iterates over site IDs, processes each site's telemetry data, and saves the output.

3. File Output:
   Processed data is saved in the folder version 1 data. Each site's file is named Site-<site_id>.csv.

### Output Data Structure

Processed data columns:
1. Metadata:
   site_id, serial_num, start_time, end_time, date, slot

2. Raw Features:
   ac_voltage, ac_frequency, dc_voltage, dc_current, temperature, energy

3. Statistical Features:
   Min, Max, Mean, Median, Std, Range, CV for each of ac_voltage, ac_frequency, dc_voltage, dc_current, temperature, energy

4. Fault Indicator:
   fault: Binary flag (1 for fault, 0 otherwise)

### Example Workflow
1. Place telemetry data in the folder 15_site_data/telemetry.
2. Update the site_id list in the main function as needed.
3. Run the script. The output will be generated in version 1 data.


### Notes
1. Ensure the input files are properly formatted with a timestamp column and numeric columns like ac_voltage, energy, etc.
2. Adjust the slot interval in creating_slots if different intervals (e.g., 30 minutes) are required.
3. The script is designed for large datasets and optimizes memory usage with float32.
