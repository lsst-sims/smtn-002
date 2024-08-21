# Calculating LSST limiting magnitudes and SNR

This technote documents the various m5 calculation tools and provides some summary information to facilitate calculating SNR and m5 values. The throughput curves used for this analysis are from  **`v1.9`** of the syseng_throughput repo, which includes the 'triple silver' mirror coatings, and as-measured mirror, filter and lens throughputs.

## Calculating SNR

Calculating either signal to noise ratios for various sources, or
5-sigma point source limiting magnitudes for LSST can be accomplished
using standard SNR equations together with available
information on the expected LSST camera and telescope components.

The appropriate methodology to calculate SNR values for PSF-optimized
photometry is outlined in the LSST Change Controlled Document
[LSE-40](https://ls.st/lse-40), and partially summarized below. Note
that LSE-40 was written with outdated throughput curves and
with an outdated understanding of the PSF profile which means the 
actual numerical values  calculated in that document are should not be used.

The SNR calculation can be summarized as follows:

$$
SNR = \frac{C } {\sqrt{C/g + ( B/g + \sigma^2_{instr}) \, n_{eff}}}

n_{eff} = 2.266 \, (FWHM_{eff} / pixelScale)^2
$$

where C = total source counts, B = sky background counts per
pixel, $\sigma_{instr}$ is the instrumental noise per pixel (all
in ADU) and g = gain. The LSST expected gain is 2.3 electron/ADU, but for purposes of
calculating SNR or m5, it can safely be assumed to be 1, which has the
nice property that then all quantities are equivalent in ADU or
photo-electrons.

### Source Counts

The total counts (in ADU) in the focal plane from any source can be calculated by multiplying the source
spectrum, $F_\nu(\lambda)$ at the top of the atmosphere in Janskys, by the fractional
probability of reaching the focal plane and being converted into
electrons, then integrating over wavelength ($S(\lambda)$):

$$
C = \frac {expTime \,  effArea} {g \, h} \int { F_\nu(\lambda) \, \frac{S(\lambda)}{\lambda}  d\lambda }
$$

where expTime = exposure time in seconds (typically 30 seconds for LSST), effArea
= effective collecting area in cm^2 (effective area-weighted clear aperture diameter for the LSST primary,
when occultation from the secondary and tertiary mirrors and
vignetting effects are included, is 6.423 m), and h = Planck
constant. The fractional throughput curves, $S(\lambda)$, for
each component in the LSST hardware system plus a standard
atmosphere can be found in
the LSST [syseng_throughputs](https://github.com/lsst-pst/syseng_throughputs) github repository. The python
code in syseng_throughputs provides an easy way to combine the 
individual throughput components, resulting in a total throughput curve
for each bandpass.

### Photometric Zeropoints

We can also use the above formula to calculate the 'instrumental zeropoint with the standard atmosphere' in each bandpass. This is
the AB magnitude which would produce one count per second (note this
value depends on the gain used; here we use gain=1, so the counts in
ADU = counts in photo-electrons).

| Filter    | Zeropoint (1s) |
|:---|---------------:|
| u  |          26.52 |
| g  |          28.51 |
| r  |          28.36 |
| i  |          28.17 |
| z  |          27.78 |
| y  |          26.82 |


### Sky Counts

When calculating sky background counts per pixel, instead of using the
entire hardware system plus atmosphere, the $F_\nu(\lambda)$
value for the sky spectrum should be multiplied by only the
hardware.[^skynote] The skybrightness in magnitudes per sq arcsecond then
is used to calculate counts per sq arcsecond, and converted to counts
per pixel using the pixelScale, 0.2"/pixel.

The expected sky brightness at zenith, in dark sky, has been
calculated in each LSST bandpass by generating a dark sky spectrum,
using data from UVES and Gemini near-IR combined with an ESO sky
spectrum, with a slight normalization in the u and y bands to match the median dark sky values
reported by SDSS. The resulting zenith, dark sky brightness values are
in good agreement with other measurements from CTIO and ESO.

| Filter | Sky (mag/arcsec^2) |
|:-------|-------------------:|
| u      |              23.05 |
| g      |              22.25 |
| r      |               21.2 |
| i      |              20.46 |
| z      |              19.61 |
| y      |               18.6 |

The instrumental zeropoints above could be used to calculate approximate background
sky counts per arcsecond sq or exact values could be calculated using
the calibrated spectrum
available at [darksky.dat](https://github.com/lsst-pst/syseng_throughputs/blob/main/siteProperties/darksky.dat).

### Instrumental Noise

The instrumental noise per pixel, $\sigma_{instr}$, can be calculated as

$$
\sigma_{instr}^2 = (readNoise^2 + (darkCurrent * expTime)) * n_{exp}
$$

where the LSST requirements place upper limits of 0.2 photo-electrons/second/pixel
on the dark current and 8.8 photo-electrons/pixel/exposure on the
total readnoise from the camera (sensors plus electronics).
Tests of vendor prototypes sensors are consistent with these
requirements.

The current LSST observing plan is to take back-to-back exposures of the same field, each
exposure 15 seconds long, for a total of $n_{exp}$ =2 exposures per 30 second
long "visit". The total instrumental noise per exposure is  9
photo-electrons. The combined total instrumental noise per visit is then 12.7 photo-electrons.

### Source footprint ($n_{eff}$)

Optimal source count extraction means matching the photometry
footprint to the PSF of the source. Raytrace experiments using models
of the LSST mirors and focal plane and atmosphere, as well as
observations from existing telescopes, indicate that the PSF for point
sources should be similar to a von Karman profile. The details of the profile
depend independently on the size of the atmospheric IQ and the
hardware IQ. The conversion factors are described in the [Document 20160](https://ls.st/document-20160)
 by Bo Xin, George Angeli, and Zeljko Ivezic.

Because the SNR calculation only depends on the number of pixels
contained in the footprint on the focal plane (to determine the sky
noise and instrumental noise contributions), we calculate $FWHM_{eff}$:
the FWHM of a single gaussian which contains the same number of pixels
as the von Karman profile. This must be calculated for the appropriate atmosphere and hardware
contributions in a given observation.

$$
FWHM_{sys}(X) = X^{0.6} \, \sqrt{telSeeing^2 + opticalDesign^2 + cameraSeeing^2}

FWHM_{eff}(X) = 1.16 \sqrt{FWHM_{sys}^2 + 1.04 \, FWHM_{atm}^2}
$$

where requirements place the system contributions at telSeeing = 0.25”, opticalDesign =
0.08”, and cameraSeeing = 0.30”. We can then just calculate $n_{eff}$ using a single gaussian profile,

$$
n_{eff} = 2.266 \, (FWHM_{eff} / pixelScale)^2.
$$

For purposes where the physical size of the PSF is important, such as
modeling moving object trailing losses or galaxy shape measurements, we can
also calculate $FWHM_{geom}$,

$$
FWHM_{geom} = 0.822\,FWHM_{eff} + 0.052
$$

$FWHM_{geom}$ is typically slightly smaller than
$FWHM_{eff}$.

The expected fiducial $FWHM_{eff}$ at zenith in the various LSST
bandpasses (based on the fiducial atmospheric seeing value expected from the SRD) is


| Filter | $FWHM_{eff}$ |
|:-------|-------------:|
| u      |         0.92 |
| g      |         0.87 |
| r      |         0.83 |
| i      |         0.80 |
| z      |         0.78 |
| y      |         0.76 |

where this includes the expected (and modeled) telescope contribution as well as the distribution of IQ measurements
from an on-site DIMM.

### Calculating m5

With all of these values, we can calculate  the $5\sigma$
limiting magnitude for point sources (m5) in each bandpass, in the dark
sky, zenith case, assuming visits consist of a single 30s exposure.
The resulting values are

| Filter |    m5 |
|:-------|------:|
| u      | 23.70 |
| g      | 24.97 |
| r      | 24.52 |
| i      | 24.13 |
| z      | 23.56 |
| y      | 22.55 |

It is worth noting that the final exposure times in the LSST survey may vary from
a simple 1x30s visit. In particular, in filters other than u band, visits should be
assumed to be 2x15s (instead of 1x30s); this makes a small difference in bands other
than u (which is why we use 1x30s for the calculation above, as visits are expected
to be 30s long in u band).

It is also worth referring to [PSTN-054](https://pstn-054.lsst.io) for a more in-depth
update on expected m5 values, including accounting for the effects of observing over a range
of conditions during operations. Due to different seeing distributions, skybrightness distributions,
and airmass distributions, median expected m5 depths diverge from those above. (although note that at present, PSTN-054 uses v1.7 of the throughput curves).

### Useful github repositories

The algorithms described in [LSE-40](https://ls.st/lse-40) are implemented in the LSST
[rubin_sim.photUtils](https://github.com/lsst/rubin_sim) package,
available on github. In particular, the
[SignalToNoise](https://github.com/lsst/rubin_sim/blob/main/rubin_sim/phot_utils/signaltonoise.py)
module calculates signal to noise ratios and limiting magnitudes (m5)
values. Here is a
[jupyter notebook example](https://github.com/lsst/rubin_sim_notebooks/blob/main/photometry/calculating_snr.ipynb)
using this code to calculate SNR in a variety of situations.

The throughput curves used for this analysis were the throughput curves in the
[syseng_throughputs](https://github.com/lsst-pst/syseng_throughputs) repository.
These throughput curves are then propagated to `rubin_sim_data`, in a modified format that
incorporates average losses over time (instead of maintaining these as separate components) and combines the individual components into 'total' and 'hardware' throughput curves.
There is more information on the origin of these throughput
curves and other key number data in the section 'Data Sources' below.

[^skynote]: The atmosphere should not be included in the calculation of
    the expected counts in the focal plane, as the sky emission
    comes from various layers in the atmosphere - a completely
    proper treatment would involve a radiative transfer model that
    includes emission and absorption over the entire
    atmosphere. Instead the standard treatment is to generate a
    sky brightness and sky spectrum that correspond to the
    skybrightness at the pupil of the telescope, and then just
    multiply this by $S_{hardware}(\lambda)$ to generate the
    focal plane counts

## Calculating m5 values in the LSST Operations Simulator

To rapidly calculate the m5 values reported with each visit in the
outputs from the Operations Simulator, the SNR formulas above are
used to calculate two values, $C_m$ and $dC_m^{inf}$. These
values can then be used to calculate m5 under a wide range of sky
brightness, seeing, airmass, and exposure times.

$$
m5 = C_m + dC_m + 0.50\,(m_{sky} - 21.0) + 2.5 log_{10}(0.7 / FWHM_{eff}) \\ + 1.25 log_{10}(expTime / 30.0) - k_{atm}\,(X-1.0)

dC_m = dC_m^{inf} - 1.25 log_{10}(1 + {(10^{(0.8\, dC_m^{inf})} -
1)}/Tscale)

Tscale = expTime / 30.0 * 10.0^{-0.4*(m_{sky} - m_{darksky})}
$$

The $dC_m^{inf}$ term accounts for the transition between instrument noise limited
observations and sky background limited observations as the
exposure time or sky brightness varies. For most LSST bandpasses, we are
sky-noise dominated even in 15 second exposures, but in the u
band, the sky background is low enough that the exposures become
read noise limited (thus $dC_m^{inf}$ has an associated base exposure time used
in its calculation and applied to $Tscale$; this is 15s above).
The $k_{atm}$ term captures the extinction of the atmosphere and how it
varies with airmass. It can be calculated as $k_{atm} =
-2.5 log_{10} (T_b / \Sigma_b)$, where $T_b$ is the sum of the
total system throughput in a particular bandpass and $\Sigma_b$
is the sum of the hardware throughput in a particular bandpass
(without the atmosphere).

| Filter |    Cm |   dCm_infinity | kAtm |
|:--|------:|---------------:|-----:|
| u | 22.97 |           0.54 | 0.47 |
| g | 24.58 |           0.09 | 0.21 |
| r | 24.6  |           0.04 | 0.13 |
| i | 24.54 |           0.03 | 0.10 |
| z | 24.37 |           0.02 | 0.07 |
| y | 23.84 |           0.02 | 0.17 |

These values can be used with the `rubin_scheduler.utils`
[m5_scale](https://github.com/lsst/rubin_scheduler/blob/main/rubin_scheduler/utils/m5_flat_sed.py#L7) function
to calculate m5 values under varying exposure times, skybrightness or seeing.

For each pointing in OpSim, the skybrightness and seeing come from various
simulated telemetry streams, and airmass and exposure time come from
the scheduling data itself. The skybrightness comes from `rubin_sim.skybrightness`. It
is based on the [ESO sky calculator](https://www.eso.org/observing/etc/bin/gen/form?INS.MODE=swspectr+INS.NAME=SKYCALC)
along with an empirical model for twilight. The rubin_sim skybrightness model has
been validated with nearly a year of on-site all-sky measurements.
The seeing  comes from `rubin_scheduler.site_models`
package, which uses 10 years of seeing data from Cerro Pachon as inputs.
The seeing model generates atmosphere-only FWHM at 500nm at zenith; these
raw atmospheric FWHM values ($FWHM_{500}$) are adjusted to
the image quality delivered by the entire system by

$$
FWHM_{sys}(X) = \sqrt{telSeeing^2 + opticalDesign^2 + cameraSeeing^2} \, (X)^{0.6}

FWHM_{atm}(X) = FWHM_{500} \, (\frac{500nm}{\lambda_{eff}})^{0.3} \,   (X)^{0.6}

FWHM_{eff}(X) = 1.16 \sqrt{FWHM_{sys}^2 + 1.04 \, FWHM_{atm}^2}
$$

where the system contributions are telSeeing = 0.25”, opticalDesign =
0.08”, and cameraSeeing = 0.30”. $\lambda_{eff}$ is the
effective wavelength for each filter:  366, 482, 622, 754, 869 and
971 nm respectively for u, g, r, i, z, y.

### Calculating C_m values

The values for $C_m$ and $dC_m^{inf}$ can be calculated using the m5 value
of a dark sky, zenith visit.

$$
C_m = m5 - 0.5\,(m_{darksky} - 21.0) + 2.5 log_{10}(0.7 / FWHM_{eff}) + 1.25 log_{10}(expTime / 30.0)
$$

where $m_{darksky}$ is the dark sky background value in the
bandpass, as described in the table above. A related $C_m^{inf}$
can be calculated using an m5 value generated by assuming that the
instrument noise per exposure is 0. The difference between
$C_m^{inf}$ and $C_m$ is $dC_m^{inf}$.

## Data Sources and References

Change controlled documents:
: - LSE-40 : "Photon Rates and SNR Calculations" <https://ls.st/lse-40> (useful for SNR eqns, but do not use the outdated values from this document)
  - LSE-29 : "LSST System Requirements" <https://ls.st/lse-29>
  - LSE-30 : "Observatory System Specifications" <https://ls.st/lse-30>
  - LSE-59 : "Camera Subsystem Requirements" <https://ls.st/lse-59>

Official project documents not under change control -
: - The LSST Overview Paper <http://ls.st/document-5462>
  - LSST Key Numbers <http://lsst.org/scientists/keynumbers>
  - LSST-PST Syseng_throughputs components git repository  <https://github.com/lsst-pst/syseng_throughputs>
  - SMTN-002 <https://smtn-002.lsst.io>  (this document)
  - PSTN-054 <https://pstn-054.lsst.io>
  - Atmospheric and Delivered Image Quality in OpSim  <https://ls.st/document-20160>

| Additional Data                               | Value | Reference |
|:---------------------------------------------------------|------:|---------------:|
| Primary mirror clear aperture [^areanote]                | 6.423 m| LSE-29, LSR-REQ-0003, LSST Key Numbers               |
| Total instrumental noise per exposure                    | 9 e-   | LSE-59, CAM-REQ-0020 (readnoise and dark current)    |
| Diameter of field of view                                | 3.5 deg| LSE-29, LSR-REQ-0004                                 |
| Focal plane coverage (fill factor in active area of FOV) |  >90%  | LSE-30, OSS-REQ-0259                                 |
| Focal plane coverage (fill factor in active area of FOV) | 91%    | Calculated from focal plane models                   |


[^areanote]: The area-weighted clear aperture is 6.423 m across the entire field of view, although this varies with location. Near the center, the clear aperture is 6.7 m, while near the edge of the field of view it rolls off by about 10%. 6.423 m is the area-weighted average across the full field of view.

Throughput curves come from the [syseng_throughputs github repo](https://github.com/lsst-pst/syseng_throughputs).
The throughput curves in the syseng_throughputs repository track
the expected performance of the components of the LSST systems.
There are versions of these throughput curves packaged for
distribution in the [throughputs](https://github.com/lsst/throughputs) github repository,
as well as with [rubin_sim_data](https://s3df.slac.stanford.edu/data/rubin/sim-data/rubin_sim_data/).

The dark sky sky brightness values come from a dark sky, zenith
spectrum which produces broadband dark sky background measurements
consistent with observed values at SDSS and other sites. The skybrightness
from `rubin_sim.skybrightness` is also in general agreement with
these dark sky values. The `rubin_sim` skybrightness simulator includes
twilight sky brightness, as well as explicit components contributed by
the moon, zodiacal light, airglow and sky emission lines - it is based
on the [ESO sky calculator](https://www.eso.org/observing/etc/bin/gen/form?INS.MODE=swspectr+INS.NAME=SKYCALC)
with the addition of a twilight sky model based on observational data
from the LSST site.

The conversion from atmospheric FWHM to delivered image quality is
based on ray-trace simulations by Bo Xin (LSST Systems
Engineering). The atmospheric FWHM measurements come from an on-site
DIMM, described in more depth in the Site Selection documents. The
DIMM measurements were cross-checked with measurements coming from
nearby atmospheric monitoring systems from other observatories.
