---
title: "Exploring Radar Basics: A Hands-On Guide to the EVALKIT SiRad Simple®"
date: 2024-01-15
draft: false
tags: ["Radar", "FMCW", "Education"]
toc: true
images:
- /siliconRadarTutorial/rangeProfile.png
featuredImagePreview: /siliconRadarTutorial/rangeProfile.png
---

# Exploring Radar Basics: A Hands-On Guide to the EVALKIT SiRad Simple®
The [**EVALKIT SiRad Simple®**](https://siliconradar.com/datasheets/Product_Sheet_SiRad_Simple_V1.2.pdf) is an evaluation board from [Silicon Radar](https://siliconradar.com) (which has now been acquired by Indie Semiconductor actually) designed to use and test one of their FMCW Radar-on-Chip sensors. The evaluation kit communicates via UART and features a ARM Cortex-M4 microcontroller. 

Though this evaluation board is no longer on the market, **there's still good reason to talk about it**. First off, there's some code already developed for this board, and it's available on [GitHub](https://github.com/M-M-Lab/Radar-Systems-Lab/tree/main). I believe it's worth discussing, as it offers educational insights that might be useful to some of you. More importantly, Silicon Radar has released new Radar-on-Chip sensors and their accompanying evaluation boards. The great thing is, the setup and usage of these new boards are quite similar to the one we're discussing, which makes this information relevant and potentially valuable for those exploring radar technology.

## Radar Insights: A Snapshot of TRXˍ120ˍ001
The [**TRXˍ120ˍ001**](https://siliconradar.com/datasheets/Datasheet_TRX_120_001_V1.4.pdf) is a 120-GHz Highly Integrated IQ Transceiver with Antennas in Package. This transceiver radar frontend is ideal for short-range radar systems, integrating both RX and TX patch antennas directly on the chip. It features an impressive bandwidth of up to 7 GHz, all packed into a small 8 mm × 8 mm plastic package. The device operates on a supply voltage of 3.3 V and has a power consumption of around 380 mW in continuous operating mode.

## Hands-On Radar: Developing with MATLAB
This example features code in MATLAB, but the radar can be seamlessly operated with Python code (which I might consider writing at a later time).

### Setting Up the Serial Connection
Begin by identifying available serial ports on your system by using the [MATLAB fuction](https://it.mathworks.com/help/matlab/ref/serialport.html) ```serialportlist()``` and setting the desired baud rate for communication. This step is needed for establishing a connection with the radar board.

```matlab
fprintf("\nRetrieving port ...\n")
sPort = serialportlist;
baudRate = 1000000; % 1e6 for clarity
```
Next, select the serial port where the board is connected from the obtained list and create a serial port object with a specified baud rate for communication speed. The code configures essential parameters for the serial connection, including the number of data bits per character frame, parity checking type, the number of stop bits, and a timeout setting for read operations. Finally, it defines the end-of-line character sequence for the data transmission, ensuring proper data handling and communication with the connected device.
> **Note:** The termination command is given by the board's protocol, which is linked in the following section.

```matlab
% Choose the serial port (e.g., second port in the list)
serialPort = sPort(2);
fprintf("Connecting to %s ...\n",serialPort)

% Create a serial port object
com_port = serialport(serialPort,baudRate);

% Set serial connection parameters
set(com_port, "DataBits", 8);
set(com_port, "Parity", 'none');
set(com_port, "StopBits", 1);
set(com_port, "Timeout", 1);
configureTerminator(com_port,'CR/LF');
```
> **Note for Linux users:** If you're on Ubuntu and encounter issues with serial port permissions, you may need to modify the access rights. To do so, enter the following command in the Terminal: ```sudo chmod 666 /dev/tty*```. This will grant read and write permissions to the serial devices. It would be wiser to modify the permissions only for the specific serial port in use.

### Sending Configuration Commands
Now, it's time to configure the radar. This involves sending specific commands to the radar board. These commands set up the radar's operating parameters, preparing it for data acquisition. For an in-depth understanding of the commands structure, please refer to the evaluation board's [protocol description](https://siliconradar.com/datasheets/Protocol_Description_Easy_Simple_V2.4.pdf) document.

```matlab
fprintf("\nSetting the radar ...\n")
writeline(com_port,"!S08021010")
writeline(com_port,"!K")
writeline(com_port,"!B20000008")
```

Command frames in this system start with an exclamation mark ```!```. There are two primary categories of commands: **short command** and **configuration commands**. Short commands are simple, consisting of a single letter after the exclamation mark, such as ```!K```. On the other hand, configuration commands are more complex and include four types:
- System Configuration ```!S```
- Radar Front End Configuration ```!F```
- PLL Configuration ```!P```
- Baseband Configuration ```!B```

These commands are followed by a series of bits. Each bit, or group of bits, sets a specific parameter of the radar. When sending commands to the board, bits are grouped in fours and represented by decimal digits. Each command type has its own unique bit structure that I am not going to list here, but the full description is given by the evaluation board's [protocol description](https://siliconradar.com/datasheets/Protocol_Description_Easy_Simple_V2.4.pdf) document. 

#### Details about the above commands
- ```!S08021010``` : it starts with ```!S```,  indicating it's a system configuration command. This command configures several settings: it adjusts the data magnitude to a linear scale (though a logarithmic scale option is also available), selects the TSV protocol from three different output protocols provided by the evaluation board, activates one of the two available serial interfaces on the board, specifies that the data format should be RAW (as the board is capable of processing data before sending it via the serial interface, but here the preference is for raw ADC data), and sets the trigger mode to external. This external trigger mode is chosen because the acquisition trigger comes from MATLAB, while the board also has its own self-triggering mechanism. The binary representation of the command is ```0000 1000 0000 0010 0001 0000 0001 0000```.
- ```!K``` : this is a short command to set the bandwidth of the radar to the maximum value.
- ```!B20000008``` : this command starts with ```!B```, thus it's a baseband command. This command enables DC cancellation, turns off the CFAR algorithm (as raw data processing is preferred and the algorithm is unnecessary), sets the number of ramps to 1, specifies the sample count at 64, and chooses the highest ADC sampling frequency. The binary representation of the command is ```0010 0000 0000 0000 0000 0000 0000 1000```.

> **Note:** If you find it complex to manually construct these commands, helpful Python functions are available [at this link](https://github.com/M-M-Lab/Radar-Systems-Lab/blob/main/pythonCode/configurationCommands.py) to aid in the process. Not everything has been fully implemented, therefore some parameters are hardcoded.

### Setting Up Operational Values
In this section, define the key numerical parameters that dictate the radar's performance. This includes the number of samples, bandwidth, speed of light, and a calibration factor. These values are vital for accurate radar operation and data interpretation.

```matlab
nSamples = 64;      % Number of samples
BW = 6.1e9;         % Bandwidth in Hz
c = 3e8;            % Speed of light in m/s
calFactor = 37.5;
```
> **Note:** The variable called ```calFactor``` is basically a fixed overhead on the data. The specific value utilized in this context was determined through empirical measurement and does not align with the datasheet's stated figure. Employing the datasheet's number resulted in a noticeable range offset in our measurements, but **we could've got something wrong**.

### Initializing Plotting Vectors
Finally, prepare for data visualization by initializing vectors that will be used in plotting. This step involves setting up variables for zero padding, calculating the maximum range, and creating a range vector. These preparations are essential for an accurate representation of the radar data.

```matlab
zeroPadding = 4;
nfft = zeroPadding*nSamples;                                       % Number of FFT points
maxRange = ((nSamples + calFactor)*c)/(4*BW);                      % Maximum range
rangeVec = (0:2*maxRange/nfft:maxRange - maxRange/nfft)*100;       % Scaling in cm
```

#### Understanding the computation of the maximum range
The datasheet for the SiRad Simple® provides the equation for the FMCW ramp time as

$$T_{\text{ramp}} = \frac{t_{\text{smp}} \cdot (N_{\text{S}}+ 37.5)}{36 \text{MHz}}$$

where $N_{\text{S}}$ is the number of samples and $t_{\text{smp}}$ denotes the time in clock cycles, and you can reference the datasheet for the specific values. The FMCW equation **[1]**

$$\frac{f_b}{\tau} = \frac{B}{T_{\text{ramp}}}$$

involves $f_b$ as the beat frequency, $\tau$ as the time delay for signal transit, and $B$ as the bandwidth. These parameters are useful to calculate the system's maximum range. Taking into account that

$$
f^{\text{max}}_{b} = \frac{f_s}{2} = \frac{36 \text{MHz}}{2t_{\text{smp}}} \quad \text{and} \quad \tau = \frac{2R_{\text{}}}{c}
$$

where $f_s$ is the sampling frequency of the system, (i.e., the maximum beat frequency is given by half of the sampling frequency) we can substitute the first and third equations into the second to write the maximum range achievable as

$$R_{\text{max}} = \frac{f^{\text{max}}_{b}\cdot T_{\text{ramp}} \cdot c}{2B} = \frac{(N_{\text{S}} + 37.5) \cdot c}{4B}.$$

### Command Transmission and Response Validation
This block sends a ```!N```, which is the command to trigger the acquisition, to the radar board via the serial port. It then enters a loop, continuously reading lines from the serial port until it receives a response that starts with ```!M``` (i.e., the identifier for the raw data), indicating a message containing useful data.

```matlab
writeline(com_port, "!N");
buf = '';
while isempty(buf) || ~startsWith(buf,'!M')
    buf = readline(com_port);
end
```

### Extracting and Converting I/Q Data Components
After receiving the data, the script splits the string by spaces into an array ```frame```, discards the first three elements (which contains the frame counter and the size of the current frame), and converts the remaining string elements to double precision numbers ```data```. These numbers represent the in-phase (I) and quadrature (Q) components of the radar signal, which are alternately placed in the array. The code separates these components into two arrays, ```dataI``` and ```dataQ```, and then combines them into a complex array ```complexData```, which represents the complex radar signal.

```matlab
frame = split(buf);
frame = frame(4:end);
data = str2double(frame);

dataI = data(1:2:end-1);
dataQ = data(2:2:end);
complexData = (dataI + 1i*dataQ).';
```

### FFT, Peak Detection and Plot 
The script checks if the number of received samples matches the expected number ```nSamples``` to discard data affected by reading error. If it does match, it computes the Fast Fourier Transform (FFT) of the complex data to produce a range profile. It finds the maximum value ```m``` and its index ```i``` within the range profile, which can be used to determine the range to the strongest target. In this scene no targets were present, that's why the maximum shown in the picture stays on the clutter around zero.

```matlab
h(1) = plot(ones(1,nfft/2));
h(2) = copyobj(h(1),gca);
xlabel("Distance [cm]")
ylabel("Magnitude [dB]")
xlim([0 rangeVec(end)])
grid on

if length(complexData) == nSamples
    rangeProfile = fft(complexData,nfft);
    curveToPlot = abs(rangeProfile(1:end/2));
    [m,i] = max(curveToPlot);

    set(h(1),'XData',rangeVec,'YData',20*log10(curveToPlot));
    set(h(2),'XData',rangeVec(i),'YData',mag2db(m),'Marker','o','Color','r');
    title('Range Profile',strcat("maximum at ",num2str(round(rangeVec(i),2))," cm"))
else
    error('Wrong number of samples.')
end
```

The code for plotting was initially designed for a MATLAB live script aimed at real-time data acquisition from the radar. This required setting up plot objects at the beginning, outside of a loop. The loop itself was then used for data acquisition and actual plotting of the radar data.

| ![Image 1](/siliconRadarTutorial/rangeProfile.png) |
|:--:|
| **Figure 1:** Acquired range profile. No targets in the acquisition. |

And if you want to save the acquired data, the following block of code will do the job.
```matlab
c = strsplit(strjoin(string(clock),'_'),'.');
file_name = strcat('data_', c(1));
save(file_name,'complexData','BW','nSamples')
fprintf("Saving file: %s\n", file_name)
```

## Concluding
Wrapping up, this guide on the EVALKIT SiRad Simple® is a straightforward and practical walkthrough for those using this specific radar board. The aim was to offer clear, step-by-step instructions into its operation and data handling, making it accessible for users at various levels of expertise. I hope it can be a helpful resource in your exploration of radar systems!

## References
**[1]** James A. Scheer, James L. Kurtz (editors). “Coherent Radar Performance Estimation”. Boston: Artech House (1993), ISBN 0-89006-628-0.