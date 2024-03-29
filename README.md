# Breakthrough Listen Periodic Spectral Signals Search

The Breakthrough Listen project aims to conduct a comprehensive search for radio technosignatures from potential extraterrestrial intelligence sources. Different classes of technosignatures require different search strategies. In the past, efforts have focused on searching for continuous narrowband signals that display Doppler drift due to relative motion between the source and the observer [1]. However, broadband signals with periodic spectral modulation patterns are also a promising indicator of ETI, considering that similar signals are common in communication systems on Earth.

Unfortunately, algorithms dedicated to search for such signals are computationally intensive. Furthermore, they typically rely on Fast Fourier Transform techniques, which are impacted by "red" (random walk) noise and perform poorly on modulation patterns with either a small duty cycle (<1%) or a long period (>500 milliseconds). This work aims to develop a pipeline for searching for periodic spectral signals based on the Fast Folding Algorithm, which displays improved performance in these regimes [2].

The initial stage of the project will be to write a python wrapper that uses the `blimpy` [3] and `riptide-ffa` [4] packages, along with necessary plotting tools, to search for and analyze periodic spectral signals across all frequency channels in a single set of 5-minute observations. The next stage of the project involves developing tools to compare a set of ON and OFF observations for a sky-localized signal. The final stage of the project will be to test the pipeline on data collected by the BL program, in one of the first comprehensive searches for this class of signals.

## FFA Algorithm

Consider an evenly sampled time series containing <em>m</em> cycles of a pulsed periodic signal with integer period <em>p</em>. This data can be represented as a two-dimensional array with <em>m</em> rows and <em>p</em> columns. Each row represents a cycle; since all pulses are vertically aligned, phase-coherent folding can be achieved by summing across all of the rows.

Now, increase the period to a non-integer value <em>p + Δp</em>, where <em>0 < Δp < 1</em>. Then, the periodic signal will appear to drift to the right across time, toward higher values of phase. To phase-coherently fold the signal, it is necessary to apply a circular rotation to each pulse to compensate for this drift; in other words, we have to integrate along a diagonal path, parametrized by a slope <em>s</em>. The folding transform of the data is defined as "the set of integrated pulse profiles obtained for all trial values of <em>s</em> with <em>0 ≤ s < m</em>," where <em>s = 0</em> corresponds to a folding period of <em>p</em>, and <em>s = m</em> corresponds to a folding period of roughly <em>p + 1</em> [2]. 

Figure 1 below, taken from Morello et al. (2020), illustrates the folding transform on a dataset with <em>m</em> = 8 and <em>p</em> = 6. The dataset contains an artifical train of pulses with a width of 1 sample, an initial phase of <em>ϕ</em> = 1, and a periodicity such that the signal appears to drift by <em>s</em> = 4 bins across the total observation time. The visible peak in integrated intensity in the folding transform indicates the detection of a candidate periodic signal.
  
 ![Folding Transform](/Pictures/folding_transform.jpeg)

The depth-first implemention of the FFA partitions the rows of the dataset into two arbitrarily sized sections, called the head and the tail. Within each section, the integration path will drift by <em>i</em> and <em>j</em> bins respectively, with an additional possible phase jump <em>b</em> at the boundary, such that <em>s = i + b + j</em>. If <em>H</em>, <em>T</em>, and <em>F</em> represent the folding transforms of the head, tail, and full dataset, then row <em>s</em> of the full folding transform can be obtained by adding together row <em>i</em> of H and row <em>j</em> of T rotated left by <em>i + b</em> bins. Figure 2 below, taken from Morello et al. (2020), summarizes this divide-and-conquer strategy.
  
![Fast Folding Algorithm](/Pictures/fast_folding_algorithm.jpeg)

## Repository Organization

- `bliss.py`
  - The main script for this project, which carries out the FFA analysis using blimpy and riptide. It accepts as input an ON file and any number of optional OFF files, and outputs a text file containing all periodicities detected in each frequency channel above a certain SNR threshold.

- `comparison.py`
  - A script containing functionality for quick comparison of an ON and OFF file prior to analysis. The two techniques tested were based on windowing and kurtosis. This script is not currently used in the pipeline.

- `plotter.py`
  - The secondary script for this project, containing all necessary tools to generate a 6x2 plot for each detected periodic signal. The left column contains the periodograms for each file in the cadence, while the right column contains the corresponding folded profiles. 

- `simulation.py`
  - A script containing functionality for injecting periodic signals into an ON file, and testing to see if they are recovered by the FFA algorithm. Since this functionality has now been included as an option in the main script, this code is currently not used in the pipeline.

- `bliss.sh`
  - A shell script containing the entire pipeline for the BLISS project. It accepts as input a full ON-OFF-ON-OFF-ON-OFF cadence, and runs `bliss.py` three times with each of the ON files and all of the OFF files. Then, it feeds the output text files into `plotter.py`, which combines them and generates a 6x2 plot for each unique detected signal. Furthermore, a six-digit code is assigned to each unique signal, with a 0 indicating that the detected signal is absent from the corresponding ON or OFF file, and a 1 indicating that the detected signal is present in the corresponding ON or OFF file. 

- `simulation.sh`
  - A shell script that is similar to `bliss.sh`, but also injects a periodic signal into each ON file. This signal can then be recovered and inspected in the plots produced by `bliss.py` and `plotter.py`.
  
## Pipeline

![BLISS Pipeline](/Pictures/pipeline.png)

## Results

As a proof of concept, the analysis portion of the project was tested on a single 5-minute ON file collected for the pulsar B0355+54, which has a literature period of 0.156 seconds. Figure 3 below displays the period-frequency space output of `bliss.py`, which accurately recovered a dominant periodicity of 156.4 milliseconds:
  
![Pulsar B0355+54](/Pictures/pulsar_output.png)
 
The + symbols denote harmonics of the dominant periodic signal, reinforcing the signal's validity. Vertical features in the plot correspond to periodic signals that are consistent across all frequency channels, while horizontal features generally arise due to radio frequency interference.
  
After implementing multiprocessing techniques, the pipeline was next tested on a full six-file cadence from the Breakthrough Listen data archives. Figure 4 below displays the period-frequency space output for a typical ON file, which has a lower time resolution than the pulsar data mentioned above.
  
![Source B01](/Pictures/bl_output.png)
  
The △ symbols denote detected periodic signals that are present both in the ON file and on of the three OFF files. As expected, the RFI is flagged, and the dominant periodicity at approximately 8 seconds stands out from the noise. 
  
Figure 5 below displays the output of `plotter.py` for a detected periodicity of approximately 8.5 seconds. A peak is evident in the periodogram of the third ON file, with a discernable signal appearing in the corresponding folded profile. Based on the vertical scales, this pattern was not detected in either of the other ON files, and is not present in any of the OFF files. 
  
![Source B01](/Pictures/bl_plot.png)
  
Note that 7 bins were used to create the folded profiles seen above. A practical lower limit on the bin width is provided by the sampling time of the input data, which is approximately 1 second in the example chosen for this write-up.

## Next Steps
  
1. A useful feature would be the calculation of the peak S/N ratio in the folded profile of a detected signal. This should be similar to the peak S/N ratio in the periodogram computed by `riptide-ffa` for the corresponding frequency channel.

2. Another useful tool would be the implementation of a clustering algorithm to determine whether a signal is present in multiple neighboring frequency channels. This would help further characterize wideband signals.

3. Finally, the main future goal of this project would be to apply the pipeline to large amounts of Breakthrough Listen data to discover and report candidate wideband signals with periodic spectral modulation patterns.

## References

1. Enriquez, J. Emilio, et al. (2018) "The Breakthrough Listen search for intelligent life: 1.1–1.9 GHz observations of 692 nearby stars." https://arxiv.org/pdf/1709.03491.pdf
2. Morello, Vincent, et al. (2020) "Optimal periodicity searching: Revisiting the Fast Folding Algorithm for large-scale pulsar surveys." https://arxiv.org/pdf/2004.03701.pdf
3. https://blimpy.readthedocs.io/en/latest/
4. https://pypi.org/project/riptide-ffa/
