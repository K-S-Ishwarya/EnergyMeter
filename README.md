## Overview:
EnergyMeter is a proof-of-concept energy monitoring system that provides realtime consumption information and appliance classification using state-of-the art machine learning/neural networks. It was built atop the [EmonPi](https://openenergymonitor.com/emonpi-3/) open-source platform. The experimental setup was built using a power strip to represent a single phase of a residential house. By building this atop an extensible open-source project, we hope to share this work with the energy disaggregation community.

This project was part of University of Washington's [Global Innovation Exchange](https://gixnetwork.org/), as part of TECHIN 514: Hardware/Software Lab 1 as taught by [John Raiti](https://gixnetwork.org/people/john-raiti/). Over the course of one ten-week quarter, students built prototypes that mixed hardware (e.g. embedded systems and sensors) and software (data collection) to be used in machine learning (ML) models.  This class was broken into three milestones to present progress.

## Team: 
[Will Buchanan](https://github.com/buchananwp), [Louis Quicksell](https://github.com/quicksell-louis), Ricky Powell

## Design Objectives: 
Provide a proof of concept residential energy disaggregation feedback mechanism to provide a breakdown of appliance-specific consumption information in realtime, in a nonintrusive manner (e.g. no extra wiring/electrical work) at a low cost. Such a device would involve current and voltage sensors, which would then break down the unique signature of an appliance.

## Problem Statement: 
Few low-cost devices exist to inform appliance-level power consumption in real-time, providing consumers little opportunity to understand where to conserve electricity. Existing plug-level devices (such as the KillaWatt) only measure consumption of individual appliances, and require a plug at each outlet. Nonintrusive Aggregated Load Monitoring (NIALM) algorithms have not been used to their full potential value, reducing the impact of Smart Metering deployment. There is a need for an open-source real-time disaggregation monitoring for whole-home power consumption, as studies have shown that a >12% annualized savings results from realtime appliance-level energy consumption feedback [1].

![image](https://user-images.githubusercontent.com/8934290/54468562-2297be00-474b-11e9-89aa-9c5172b7f9e7.png)

## Project Requirements:
The device should identify appliances being used in real-time based on their unique power consumption signatures, using ML as part of UW's TECHIN 513: Managing Data & Signal Processing, taught by [Josh Fromm](https://jwfromm.com/). The device ideally should utilize state-of-the-art ML techniques to perform the classification, with the results being sent to the user to inform behavior change. 

# Software
The software components of this project collected, cleaned, and fed the data into our classification models. Below are the descriptions of our software and machine learning development efforts.  

## Firmware Modifications
To build the electronic signature of an appliance we modified the existing firmware on the EmonPi to collect the features necessary and sample at a sufficient rate.  Besides voltage and current, these features include phase angle, power factor, real power, reactive power, and apparent power.  We calibrated the voltage and current measurements using [a Kill-A-Watt meter](http://www.p3international.com/products/p4400.html) to verify that our measurements were accurate.  Built-in functions in the Emonpi were enabled to remove any noise present in the measurements.  The measurements are made using the ATMega shield attached to the EmonPi and converted into bits to be sent to the serial port on the pi.  Measurements are sent from the ATMega chip using [the RF12 packet structure](https://jeelabs.org/2011/06/09/rf12-packet-format-and-design/) and we modified the firmware to send a packet containing our feature measurement data.  This packet is read in through the serial port on the pi by a Python script which decodes/cleans the packet and performs the appliance classification.  

### Default EmonPi system diagram
![https://wiki.openenergymonitor.org/index.php/EmonPi](https://user-images.githubusercontent.com/7257165/54460789-cbccbd00-4727-11e9-8ed1-ccdff6085114.png)

### Modified EmonPi system diagram
![energymeter diagram](https://user-images.githubusercontent.com/7257165/54467932-a353bb80-4745-11e9-96e4-4f640c7249b6.png)

## Data Cleaning
[Several functions](https://github.com/quicksell-louis/EnergyMeter/blob/master/scripts/helper_functions.py) were written to decode and format the collected data appropriately. The packets are decoded according to the RF12 structure and fed into a rolling window. A rolling window is used to capture data in the same shape as the time series data samples we used to train our model. Data is padded or trimmed to a set length and centered and scaled to assist with classification. After these transformations, the reshaped windows are fed into the classification model for appliance identification. 

## Classification Model Overview
Our model was constructed with Keras & Tensorflow. This is a multi-class classification problem. There is some overlap among electronic signatures of appliances with similar electrical components (a toaster and a hot plate are both resistive loads and it follows that they would have similar load signatures). If we were implementing classification at a finer level there may be some overlap among different make/models in appliance types, however this is beyond the scope of our project.

## Feature Description
Using the EmonPi, we collected the appliance type (input by user), and the consumption signature of the device in a CSV format. These fields include: time, power factor, phase angle, real/reactive/apparent power, and RMS voltage/current. This constitutes 8 features that are recorded in a rolling window, which is used to perform the classification:

`('Time', 'PF', 'Phase', 'P_real', 'P_reac', 'P_app,', 'V_rms', 'I_rms')`

## Data Collection Procedure 
Data collection was done manually. We wrote a [script](https://github.com/quicksell-louis/EnergyMeter/blob/master/scripts/train.py) which automatically detects that an appliance is turned on and then prompts the user for the type of the appliance being sampled. This is used to label a window of data containing different electronic signature features over a set time period. It contains printouts to view/validate the data, and catch any errors. Each sample is saved to a CSV which is then added to the training dataset during processing.

## Dataset Training
Given the simplicity of our neural network, we initially attempted a run with 20 samples for each device. However, we uncovered numerous errors due to data formatting/processing in our first round of data collection & processing, and achieved only a 60% accuracy rate. This prompted us to rebuild the data processing pipeline, and collect many more samples. After tuning our model, we achieved ~90% accuracy with 100 samples per device, for a total of 700 samples. However, we futher discovered issues in our data collection script, and rewrote it to speed up the process by automatic event detection and window creation, and a state of 'none' to aid classification. We then collected a new dataset containing 800 samples. The model is now near 99% accuracy. Further refinement could be achieved, but efforts would be better spent in the functionality development, mentioned in 'Future Work' section later on this page.

## Dataset Labels
Due our previously defined scope, we chose to limit the detail of the appliance classification to seven appliances for our proof of concept. Labels are used to describe the appliance class (e.g. heatpad, kettle, laptop, induction cooktop, hairdryer). During training data collection, the appliance label is input at the start of each session, and the files are automatically incremented with the run number. There is little room for interpretation regarding appliance labels on behalf of the user, so it is unlikely there is any bias inserted.

## Neural Network Model Design/Architecture
For simplicity, we chose a three-layer dense network. There was no significant performance difference between architectures that warranted additional work. We are still studying our model, but focusing on real-world data collection, and less on model accuracy.
![Image](https://user-images.githubusercontent.com/8934290/54455367-d7fd4e00-4718-11e9-9492-52c09460da44.png)

## Hyperparameters Of Model
* Optimization algorithm: adam
* Training: 20 epochs
* Regularization: relu and softmax (final layer)

## Training/Validation
This is a low-precision application, as it is used to inform behavioral change. The validation accuracy will be sufficient for real-time feedback, though if this were a commercial product much more development would be needed.

## Data Preprocessing & Augmentation 
Due to the challenges of collecting enough data to sufficiently train a neural network, we explored the path of data augmentation to increase our dataset. Our approach to reduce overfitting involved a random application of a stretch or a shrink to the window, inspired by [this paper](https://aaltd16.irisa.fr/files/2016/08/AALTD16_paper_9.pdf). The function chooses a random sample equal to 1/10 of the size of the source array, and replaces sample with pair-wise average of sample indices. This was fed in using the [Keras Image Datagenerator](https://keras.io/preprocessing/image/) object, which performs the preprocessing function during each epoch, thus enhancing our training dataset.

## Deployment
Given the small size / low complexity of the model, we deployed it on the raspberry pi locally. It was saved using the [Keras Model Callback/Checkpoint](https://keras.io/callbacks/)function. Given the time allotted, it is not possible to deploy this product as it stands, being purely a proof of concept. The device is intended to support one household at a time, and will struggle to distinguish between appliances of similar power consumption habits.

## Visualization
We created a dashboard to plot the component features of each appliance's electronic signature.  This dashboard was created in python using [the Plotly web framework, Dash](https://plot.ly/products/dash/).  We were able to run this on our local machines but had some difficulty deploying it using Heroku.  Below are plots for the electronic signature of a water kettle.

![power_plot](https://user-images.githubusercontent.com/7257165/54468410-891bdc80-4749-11e9-934c-ab0bdab256c1.png)

![power_factor_phase_angle_plot](https://user-images.githubusercontent.com/7257165/54468411-891bdc80-4749-11e9-97dc-dfa7759b77d3.png)

![current_plot](https://user-images.githubusercontent.com/7257165/54468412-89b47300-4749-11e9-9017-a1573ab305e9.png)

![voltage_plot](https://user-images.githubusercontent.com/7257165/54468413-89b47300-4749-11e9-84cf-f0b2a9ac7e2e.png)

## Testing
At the time of writing, real-life testing has been limited, primarily due to project timeline constraints. However, full functionality has been demonstrated, and we hope the disaggregation community takes this farther. 

# Hardware 

## Iteration 1
![Iteration 1](https://user-images.githubusercontent.com/8934290/54461480-e869f480-4729-11e9-992c-d563a5bb2567.png)
![image](https://user-images.githubusercontent.com/8934290/54462378-caea5a00-472c-11e9-8b2f-a4a9cac0d015.png)

To begin measuring voltage, we started the project from scratch following the [OpenEnergyMonitor](https://learn.openenergymonitor.org/electricity-monitoring/voltage-sensing/measuring-voltage-with-an-acac-power-adapter) documentation, using a Raspberry Pi, an analog-digital converter, and a plug-in AC-AC adapter. Along the way we learned the fundamentals of AC power, and encountered numerous technical hurdles: ADC sampling rate, aliasing, and noise. After reeevaluating the schedule, we made the decision to transition to an open-source energy monitor, EmonPi. Below is the rationale for our decision:

| Ground-Up Hardware: | EmonPi |
| ------------- | ------------- |
|   |  |
| PROS | PROS  |
| Deep-dive into hardware development  | Accelerated schedule  |
| Learn signal-processing fundamentals  | Integrated Voltage & Current pi shield w/ dedicated ATMega328 & 10-bit ADC  |
| Build custom ADC firmware to suit our sampling needs  | Build atop existing development & learn from others   |
|   | Extensible, open-source platform   |
|   | More time on ML and less bit-banging  |
|   |  |
| CONS | CONS |
| Reinvent the wheel  | Firmware modifications  |
| Moving slowly  | Conversion from EU to US power standards  |
| More time on hardware, less on ML  | Modifying a complex product ecosystem  |
| Less conceptual value-added  |   |

## Iteration 2 
![Iteration 2](https://user-images.githubusercontent.com/535863/54455501-39bdb800-4719-11e9-9e0d-252db1ef78da.png)

We used an [EmonPi](https://github.com/openenergymonitor/emonpi) which is an open-hardware Raspberry Pi and Arduino based energy monitoring unit, 1 [clip-on current sensor](https://openenergymonitor.com/100a-max-clip-on-current-sensor-ct/) to monitor a single-phase circuit, and a plug-in AC-AC adapter to measure RMS voltage. The EmonPi offers a ATMega328 shield that allowed us to build atop a significantly developed arduino firmware library, [EmonLib](https://github.com/openenergymonitor/EmonLib), enabling the increase in both sampling rate and accuracy to produce more calibrated feature sets to be used for ML classification.  

We trained on several common household devices which include: 
* Laptop (Macbook pro)
* Cellphone (iPhone 6)
* Full UV Spectrum Light (aka sad-lamp)
* Desklamp
* Fan
* Kettle
* Monitor


## Iteration 3 (Final)
We chose to keep the hardware unchanged for our third milestone for the following reasons:
* Simplicity keeps the product extensible for future development in the open-source community
* Little room for improvement in the current enclosure design, given we are building atop an existing product
* Hardware is not customer-facing, and resides in a circuit breaker, thus product design is not a priority
* The team discovered halfway through the second iteration that [Sense](https://sense.com/technology.html), a similar product, currently exists on the market, and thus represents a reduced incentive to commercialize. 

## Future Work
Quite a bit of work is needed to bring this to its full potential. Here is a roughly prioritized list:
* Develop concurrent appliance operation functionality. This would allow for overlapping signatures, thus a more realistic use-case
* Collect 'off' state appliance status, to detect when an appliance is turned off
* Develop running tally of consumption per device
* Train on multiple-state appliances (e.g. low, medium, high)
* Understand how appliance state changes over time (e.g. if it is 'warmed up', certain loads have different startup characteristics)
* Increase training dataset samples & diversity, as neural networks need more data to function well
* Build out greater current capacity. Currently limited to 20A & upgrading to a larger scale would require a [burden resistor swap & calibration](https://learn.openenergymonitor.org/electricity-monitoring/ct-sensors/interface-with-arduino?redirected=true)
* Develop two and three-phase monitoring system
* Convert EmonPi scripts to Python3
* Add front-end for visualization 
* Implement behavioral change studies
* Fix MSQTT 'broken pipe' error that results from our having disabled the MSQTT server. This means the pi needs restarting occasionally. Should be an easy fix.

## Bibliography
[1] Armel, K. Carrie, et al. "Is disaggregation the holy grail of energy efficiency? The case of electricity." Energy Policy 52 (2013): 213-234.


