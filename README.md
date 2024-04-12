# FPGA_Audio_Vsiualizer_On-De10-Lite_Using-FFT-Algorithm_Major_project_Pratyusha


The goal is to teach future members about new topics that existing projects do not cover, such as signal processing, digital logic, FPGA’s, and Verilog. 

To do so, members will design a system that uses an FPGA as a digital signals processor. The FPGA will receive audio inputs in real time through an external microphone and display the resulting frequency spectrum on an external display. A block diagram is shown below.

![Block Diagram](https://github.com/kennych418/FPGA_AudioVisualizer/blob/master/pictures/Block%20Diagram.png)


I designed and implemented the system to verify that it is possible on the chosen hardware. This documentation describes all of the steps taken and difficulties encountered while implementing the digital audio visualizer.
![Block Diagram](https://github.com/kennych418/FPGA_AudioVisualizer/blob/master/pictures/System.png)
![Block Diagram](https://github.com/kennych418/FPGA_AudioVisualizer/blob/master/pictures/Ext.Display.png)

## Hardware
* [ Microphone](https://www.adafruit.com/product/3421)
* [DE10-Lite FPGA ](http://www.terasic.com.tw/cgi-bin/page/archive.pl?Language=English&CategoryNo=234&No=1021&PartNo=8)
* VGA Compatible Monitor
* VGA Male to Male Cable
* Wires

I chose the DE10-Lite FPGA because it is fast (50MHz), and provides the necessary I/O for me to interface with the microphone and VGA display. To use the DE10-Lite, you must connect your computer to your FPGA through a USB A to B cable (one is packaged with the DE10-Lite). This will provide power to the FPGA. The table below lists all of the wire connections between the microphone and the FPGA.

| Microphone | FPGA |
|------------|------|
| VCC | GPIO 29 (3.3V pin) |
| GND | GPIO 30 (GND pin) |
| BCLK | Defined by Designer |
| DOUT | Defined by Designer |
| LRCLK | Defined by Designer |

## Implementation details and Files 
All of the synthesizable files are listed below in the heirarchy that they are used in the audio visualizer design.
* FFT
    * clkdiv
    * mic_translator
    * FFT_Processor
        * butterflyunit
    * VGA_generator

The FFT module is the top-level of the audio visualizer. Its inputs are the FPGA's 50MHz base clock (clk), a reset button on the FPGA board (reset), and the microphone's data signal (DOUT). Its outputs are the microphone's clk and control signals (BCLK, LRCLK), and the VGA's clk, control, and data signals (vsync, hsync, r[3:0], g[3:0], b[3:0]).

I used a VGA monitor as my external display because it is simple to use with an FPGA, provides a lot of flexibility, and offers a lot of pixels to visualize data. Additionally, there is a lot of open-source code and online information on how to design a VGA controller using Verilog. I connected the FPGA's VGA port to the monitor through the VGA Male to Male Cable. 

The **mic_translator** module is used to generate the microphone's i2S signals and translate them into a readable format for the FFT_Processor. The mic_translator also stores 16 sets of audio data, which is necessary for a 16 point FFT. The inputs are the 125kHz system clock (clk), a reset button on the FPGA board (reset), and the microphone's data signal (DOUT). The outputs are the microphone's clk and control signals (BCLK, LRCLK), a flag signal that indicates when a new audio sample has been acquired (new_t), and all 16 sets of stored audio data (t0[17:0] - t15[17:0]). It should be noted that the i2S signals' speed is dependant on the clk input, where BCLK is equal to the frequency of the input clk. Therefore, increasing the input clk frequency will increase the sample rate.
The **FFT_Processor** module takes the audio data from the microphone, calculates the 16 point radix 2 decimation in time FFT over four clock cycles, and outputs the corresponding frequency data. The inputs are the 125kHz system clock (clk), a reset button on the FPGA (reset), a flag signal that indicates when a new audio sample has been acquired (new_t), and all 16 sets of stored audio data from the mic_translator (t0[17:0] - t15[17:0]). The outputs are a flag signal that indicates when the FFT calculation is finished, and all 16 sets of frequency data (f0[23:0] - f15[23:0]). 

The audio data from the mic_translator is first sign extended to convert it from 18 bits into 24. This is important because of bit growth, the additions and multiplications that occur throughout the calculation can increase the number of bits required to represent the output. For example, adding two 18 bit numbers requires a 19 bit output. Through simulations, we found that our implementation needs 6 extra bits to accomodate for bit growth. Additionally, the audio data from the mic_translator is zero padded into a 48 bit number. Those zeros represent the imaginary part of the audio data, which is set to 0 since it purely real. The result is formatted as {6'b sign_extension, 18'b audio_data, 24'b 0} which matches the butterflyunit's input format {24'b real, 24'b imag}. 

The FFT_Processor contains 8 butterflyunits that represent a single "layer" of the hardware FFT implementation and requires 4 clock cycles to complete an entire FFT calculation. More information on the hardware FFT implementation can be found [here](http://www.themobilestudio.net/the-fourier-transform-part-14) and a diagram is shown below. Larger FFT calculations would require more butterflyunits and clock cycles. For example, a 32 point FFT would require 16 butterflyunits and 5 clock cycles to complete. 

![Block Diagram](https://github.com/kennych418/FPGA_AudioVisualizer/blob/master/pictures/FFT%20Hardware%20Diagram.png)

The FFT_Processor uses sequential logic, summarized as its "control logic", to determine the inputs and twiddle factors to each butterflyunit. For example, when new formatted audio data is present (new_t high) and the FFT_Processor is inactive (done is high), the control logic will set the new formatted audio data as the next input to the butterflyunit array and set the twiddle factors for the first layer. During the next 3 clock cycles, the control logic will direct the outputs of the butterflyunits back to their inputs and set the twiddle factors for the next layer. This will allow the FFT_Processor to move through each "layer" of the hardware FFT implementation. After the final layer, the FFT_Processor will remain inactive (done is high). The f0[23:0] - f15[23:0] outputs will contain only the real values of the calculation since those are the values we are interested in visualizing. A block diagram is shown below. 
![Block Diagram](https://github.com/kennych418/FPGA_AudioVisualizer/blob/master/pictures/FFT_Processor%20Diagram.png)
The **butterflyunit** module performs the smallest unit operation of the FFT on two inputs. More information on the butterfly unit can be found online. The inputs are the real & imaginary input samples (A_t[47:0] and B_t[47:0]) and the twiddle factor (W[47:0]). The outputs are the real & imaginary output samples (A_f[47:0], B_f[47:0]).

The **VGA_generator** is responsible for visualizing the FFT results onto a monitor. The inputs are a 25MHz VGA clock (clk), a flag signal from the FFT_Processor that notifies when it is done with a new FFT calculation (done), and all 16 sets of frequency data (f0[23:0] - f15[23:0]). The outputs are the VGA's vertical clock (vsync), horizontal clock (hsync), and color data (r[3:0], g[3:0], b[3:0]). You can find more information about the VGA protocol online. Additionally, there are plenty of open source verilog VGA controllers online that are probably less complicated than our implementation. The only unique part of our VGA_generator is the display module, which determines the color of each pixel based on the FFT results.

Since the FFT_Processor and VGA_generator are on two different clock domains, with the former at 125kHz and the later at 25MHz, they are asynchronous relative to each other. As a result, we need to use [synchronizers](http://www-inst.eecs.berkeley.edu/~cs150/sp12/agenda/lec/lec16-synch.pdf) on the frequency data passed between the two. This is done by using registers (f0_reg[23:0] - f15_reg[23:0]) that are clocked by the VGA_generator's 25MHz clock. 

The butterfly_test script is used to help debug the butterflyunit module. To use it, you call the command shown below. It takes the user's values for a_real, a_imag, b_real, and b_imag, calculates the correct butterflyunit output with every twiddle factor for a 16 point FFT, and prints them out. The user can then compare the results with the output of a butterflyunit testbench.



## REFERENCES

[1] S. Winograd, “On computing the discrete Fourier transform,” Math. Comp., 32, pp.175-199, 1978.

[2] S. Winograd, “On the multiplicative complexity of the discrete Fourier transform,” Adv.
Math., 32, pp. 83-117, 1979.

[3] N. Brenner and C. M. Rader, “A new principle for fast Fourier transformation,” IEEE Acoustics Speech and Sig. Proc., no. 24, vol.3, pp. 264-266.

[4] I. Good, “The interaction algorithm and practical Fourier analysis,” J. R. Statist. Soc. B, no.20. vol. 2, pp. 361-372.

[5] L. R. Rabiner, R. W. Schaefer and C. M. Rader, “The chirp-z transform algorithm and its
applications,” Bell System Tech. J., vol.48, pp. 1249-1292, 1969.

## Author
S. Pratyusha,
ECE, Btech Completed,
RGUKT IIIT NUZVID.


