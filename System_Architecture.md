# Real-Time Radar Processing System Architecture

This document provides a detailed breakdown of the system architecture for the mmWave Radar Processing Tool. The system is designed to capture, process, and visualize raw ADC data from a TI mmWave radar in real-time, offering three distinct processing modes: Range Profile, Range-Doppler, and Range-Angle.

## Core Components

The system is built around a set of core components that are shared across all three processing modes. These components handle the initial setup, data acquisition, and basic configuration of the radar system.

### 1. Launcher (`launcher.py`)

The `launcher.py` script serves as the main entry point for the entire application. It provides a graphical user interface (GUI) built with PyQt5 that allows the user to select and launch one of the three radar processing applications. This centralized launcher simplifies the user experience and provides a consistent starting point for all operations.

### 2. Radar Configuration

The radar configuration is handled by a set of files and modules that work together to initialize the radar hardware with the desired parameters.

*   **Configuration File (`config/AWR1843_cfg.cfg`):** This is a plain text file that contains a series of commands for configuring the AWR1843AOP EVM. These commands set up the radar's chirp parameters, frame structure, ADC settings, and data output format. The file is parsed by the `radar_parameters.py` module to extract the configuration values.

*   **`radar_config.py`:** This module contains the `SerialConfig` class, which is responsible for communicating with the radar EVM over a serial (COM) port. It reads the commands from the `.cfg` file and sends them to the radar one by one. It also provides methods for starting and stopping the radar sensor.

*   **`radar_parameters.py`:** This module defines the `RadarParameters` class, which parses the `.cfg` file and calculates a set of derived parameters that are essential for the signal processing pipeline. These parameters include:
    *   Range resolution
    *   Maximum range
    *   Velocity resolution
    *   Maximum velocity
    *   Dimensions of the radar data cube

### 3. Data Acquisition

The data acquisition process is managed by a dedicated thread that receives raw ADC data from the radar and makes it available to the processing pipeline.

*   **`UdpListener` (in `rp_real_time_process.py`, `rd_real_time_process.py`, `ra_real_time_process.py`):** This class, which runs in its own thread, is responsible for listening for UDP packets from the DCA1000 EVM on a specific network port. It receives the raw ADC data, removes the packet headers, and places the binary data into a shared queue for the `DataProcessor` to consume.

*   **Binary Data Queue:** A thread-safe queue that is used to transfer the raw ADC data from the `UdpListener` to the `DataProcessor`. This decouples the data acquisition and data processing tasks, allowing them to run in parallel and preventing data loss.

## Processing Pipelines

The system features three distinct processing pipelines, one for each of the supported radar modes. Each pipeline is implemented in its own set of scripts and is responsible for processing the raw ADC data and generating the corresponding visualization.

### 1. Range Profile

The Range Profile mode provides a 1D visualization of the signal power as a function of distance from the radar.

*   **`rp_main.py`:** The main script for the Range Profile application. It initializes the GUI, starts the `UdpListener` and `DataProcessor` threads, and handles user interactions.

*   **`rp_real_time_process.py`:** Contains the `DataProcessor` class for this mode. It retrieves raw data from the binary queue, reshapes it, and passes it to the `rp_dsp` module for processing.

*   **`rp_dsp.py`:** This module contains the core signal processing functions for the Range Profile mode:
    *   **1D FFT:** A Fast Fourier Transform is applied to the ADC data to transform it from the time domain to the frequency domain, which corresponds to the range domain.
    *   **Pulse Compression:** A matched filter is applied to the data to improve the signal-to-noise ratio (SNR) through pulse compression.
    *   **CFAR Detection:** A Constant False Alarm Rate (CFAR) algorithm is used to detect peaks in the range profile, which correspond to detected objects.
    *   **Clutter Removal:** A static clutter removal algorithm based on Principal Component Analysis (PCA) can be applied to remove stationary background objects.

*   **`rp_app_layout.py`:** Defines the PyQt5 GUI for the Range Profile application. It includes a 1D plot for the range profile, a table to display the range and magnitude of detected objects, and controls for configuring the processing parameters.

### 2. Range-Doppler

The Range-Doppler mode provides a 2D visualization of the signal power as a function of both range and velocity.

*   **`rd_main.py`:** The main script for the Range-Doppler application.

*   **`rd_real_time_process.py`:** Contains the `DataProcessor` for this mode, which calls the `rd_dsp` module for processing.

*   **`rd_dsp.py`:** The signal processing module for the Range-Doppler mode:
    *   **2D FFT:** A 2D FFT is performed on the radar data cube to generate the Range-Doppler map. The first FFT is along the range dimension, and the second is along the Doppler (chirp) dimension.
    *   **2D CFAR Detection:** A 2D version of the CFAR algorithm is used to detect objects in the Range-Doppler map.
    *   **MTI Filtering:** A Moving Target Indicator (MTI) filter can be applied to suppress stationary targets and enhance the detection of moving objects.

*   **`rd_app_layout.py`:** The GUI for the Range-Doppler application. It features a 2D heatmap for the Range-Doppler map and a data table that displays the range, speed, and direction of detected objects.

### 3. Range-Angle

The Range-Angle mode provides a 2D visualization of the signal power as a function of range and angle, allowing for the spatial localization of objects.

*   **`ra_main.py`:** The main script for the Range-Angle application.

*   **`ra_real_time_process.py`:** Contains the `DataProcessor` for this mode, which calls the `ra_dsp` module for processing.

*   **`ra_dsp.py`:** The signal processing module for the Range-Angle mode:
    *   **3D FFT:** A 3D FFT is performed on the radar data cube. The first two dimensions compute the Range-Doppler map, and the third dimension (across the virtual antenna array) is used to estimate the angle of arrival.
    *   **Beamforming:** A beamforming algorithm with steering vectors is used to generate the Range-Angle map for both azimuth and elevation.
    *   **2D CFAR Detection:** The CFAR algorithm is applied to the Range-Angle map to detect objects.

*   **`ra_app_layout.py`:** The GUI for the Range-Angle application. It includes a 2D heatmap for the Range-Angle map and a data table that displays the range and angle of detected objects.

*   **`coordinate_transforms.py`:** A utility module that provides functions for converting polar coordinates (range and angle) to Cartesian coordinates (x and z) for visualization.
