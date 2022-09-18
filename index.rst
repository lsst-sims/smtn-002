Calculating SNR
===============

Calculating either signal to noise ratios for various sources, or
5-sigma point source limiting magnitudes for LSST can be accomplished
using standard SNR equations together with available
information on the expected LSST camera and telescope components.

The appropriate methodology to calculate SNR values for PSF-optimized
photometry is outlined in the LSST Change Controlled Document
`LSE-40 <http://ls.st/lse-40>`_, and partially summarized below. Note
that LSE-40 is awaiting updates to match new throughput curves and updated information on how
we're handling the PSF profile which means the actual values
calculated in that document are outdated.

The SNR calculation can be summarized as follows:

.. math::
    SNR = \frac{C } {\sqrt{C/g + ( B/g + \sigma^2_{instr}) \, n_{eff}}}

    n_{eff} = 2.266 \, (FWHM_{eff} / pixelScale)^2

where C = total source counts, B = sky background counts per
pixel, :math:`\sigma_{instr}` is the instrumental noise per pixel (all
in ADU) and g = gain. The LSST expected gain is 2.3 electron/ADU, but for purposes of
calculating SNR or m5, it can safely be assumed to be 1, which has the
nice property that then all quantities are equivalent in ADU or
photo-electrons.

Source Counts
-------------

The total counts (in ADU) in the focal plane from any source can be calculated by multiplying the source
spectrum, :math:`F_\nu(\lambda)` at the top of the atmosphere in Janskys, by the fractional
probability of reaching the focal plane and being converted into
electrons and integrating over wavelength (:math:`S(\lambda)`):

.. math::
   C = \frac {expTime \,  effArea} {g \, h} \int { F_\nu(\lambda) \, \frac{S(\lambda)}{\lambda}  d\lambda }

where expTime = exposure time in seconds (typically 30 seconds for LSST), effArea
= effective collecting area in cm^2 (effective area-weighted clear aperture diameter for the LSST primary,
when occultation from the secondary and tertiary mirrors and
vignetting effects are included, is 6.423 m), and h = Planck
constant. The fractional throughput curves, :math:`S(\lambda)`, for
each component in the LSST hardware system plus a standard
atmosphere can be found in
the LSST `syseng_throughputs
<https://github.com/lsst-pst/syseng_throughputs>`_ github repository.

Instrumental Zeropoints
-----------------------

We can also use the above formula to calculate the 'instrumental zeropoint' in each bandpass,
the AB magnitude which would produce one count per second (note this
value depends on the gain used; here we use gain=1, so the counts in
ADU = counts in photo-electrons).

+------+--------------------------------------------+
|Filter|Instrumental Zeropoint (exptime=1s, gain=1) |
+------+--------------------------------------------+
|u     |     27.03                                  |
+------+--------------------------------------------+
|g     |     28.38                                  |
+------+--------------------------------------------+
|r     |      28.16                                 |
+------+--------------------------------------------+
|i     |      27.85                                 |
+------+--------------------------------------------+
|z     |    27.46                                   |
+------+--------------------------------------------+
|y     |    26.68                                   |
+------+--------------------------------------------+

Sky Counts
----------

When calculating sky background counts per pixel, instead of using the
entire hardware system plus atmosphere, the :math:`F_\nu(\lambda)`
value for the sky spectrum should be multiplied by only the
hardware.\ [#skynote]_ The skybrightness in magnitudes per sq arcsecond then
is used to calculate counts per sq arcsecond, and converted to counts
per pixel using the pixelScale, 0.2"/pixel.

The expected sky brightness at zenith, in dark sky, has been
calculated in each LSST bandpass by generating a dark sky spectrum,
using data from UVES and Gemini near-IR combined with an ESO sky
spectrum, with a slight normalization in the u and y bands to match the median dark sky values
reported by SDSS. The resulting zenith, dark sky brightness values are
in good agreement with other measurements from CTIO and ESO.

+------+--------------------------------+
|Filter|Sky brightness (mag/arcsecond^2)|
+------+--------------------------------+
|u     |     22.96                      |
+------+--------------------------------+
|g     |     22.26                      |
+------+--------------------------------+
|r     |     21.20                      |
+------+--------------------------------+
|i     |     20.48                      |
+------+--------------------------------+
|z     |    19.60                       |
+------+--------------------------------+
|y     |    18.61                       |
+------+--------------------------------+

The instrumental zeropoints above could be used to calculate approximate background
sky counts per arcsecond sq or exact values could be calculated using
the calibrated spectrum
available at `darksky.dat
<https://github.com/lsst-pst/syseng_throughputs/blob/master/siteProperties/darksky.dat>`_.

Instrumental Noise
------------------

The instrumental noise per pixel, :math:`\sigma_{instr}`, can be calculated as

.. math::
   \sigma_{instr}^2 = (readNoise^2 + (darkCurrent * expTime)) * n_{exp}

where the LSST requirements place upper limits of 0.2 photo-electrons/second/pixel
on the dark current and 8.8 photo-electrons/pixel/exposure on the
total readnoise from the camera (sensors plus electronics).
Tests of vendor prototypes sensors are consistent with these
requirements.

The current LSST observing plan is to take back-to-back exposures of the same field, each
exposure 15 seconds long, for a total of :math:`n_{exp}` =2 exposures per 30 second
long "visit". The total instrumental noise per exposure is  9
photo-electrons. The combined total instrumental noise per visit is then 12.7 photo-electrons.

Source footprint (:math:`n_{eff}`)
----------------------------------

Optimal source count extraction means matching the photometry
footprint to the PSF of the source. Raytrace experiments using models
of the LSST mirors and focal plane and atmosphere, as well as
observations from existing telescopes, indicate that the PSF for point
sources should be similar to a von Karman profile. The details of the profile
depend independently on the size of the atmospheric IQ and the
hardware IQ. The conversion factors will be described in a planned
update of `LSE-40 <http://ls.st/lse-40>`_ and the `LSST Overview Paper
<http://www.lsst.org/content/lsst-science-drivers-reference-design-and-anticipated-data-products>`_.

Because the SNR calculation only depends on the number of pixels
contained in the footprint on the focal plane (to determine the sky
noise and instrumental noise contributions), we calculate :math:`FWHM_{eff}`:
the FWHM of a single gaussian which contains the same number of pixels
as the von Karman profile. This must be calculated for the appropriate atmosphere and hardware
contributions in a given observation.

.. math::
   FWHM_{sys}(X) = X^{0.6} \, \sqrt{telSeeing^2 + opticalDesign^2 + cameraSeeing^2}

   FWHM_{eff}(X) = 1.16 \sqrt{FWHM_{sys}^2 + 1.04 \, FWHM_{atm}^2}

where requirements place the system contributions at telSeeing = 0.25”, opticalDesign =
0.08”, and cameraSeeing = 0.30”. We can then just calculate :math:`n_{eff}` using a single gaussian profile,

.. math::
   n_{eff} = 2.266 \, (FWHM_{eff} / pixelScale)^2.

For purposes where the physical size of the PSF is important, such as
modeling moving object trailing losses or galaxy shape measurements, we can
also calculate :math:`FWHM_{geom}`,

.. math::
     FWHM_{geom} = 0.822\,FWHM_{eff} + 0.052

:math:`FWHM_{geom}` is typically slightly smaller than
:math:`FWHM_{eff}`.

The expected fiducial :math:`FWHM_{eff}` at zenith in the various LSST
bandpasses (based on the fiducial atmospheric seeing value expected from the SRD) is

+------+-------------------+
|Filter|:math:`FWHM_{eff}` |
+------+-------------------+
|u     | 0.92"             |
+------+-------------------+
|g     | 0.87"             |
+------+-------------------+
|r     | 0.83"             |
+------+-------------------+
|i     | 0.80"             |
+------+-------------------+
|z     | 0.78"             |
+------+-------------------+
|y     | 0.76"             |
+------+-------------------+

where this includes the expected (and modeled) telescope contribution as well as the distribution of IQ measurements
from an on-site DIMM.


Calculating m5
-----------------------------------------------

With all of these values, we can calculate  the :math:`5\sigma`
limiting magnitude for point sources (m5) in each bandpass, in the dark
sky, zenith case, assuming visits consist of a single 30s exposure.
The resulting values are

+------+------+
|Filter|m5    |
+------+------+
|u     |24.07 |
+------+------+
|g     |24.90 |
+------+------+
|r     |24.40 |
+------+------+
|i     |23.96 |
+------+------+
|z     |23.38 |
+------+------+
|y     |22.49 |
+------+------+

It is worth noting that the final exposure times in the LSST survey may vary from
a simple 1x30s visit. In particular, in filters other than u band, visits should be
assumed to be 2x15s (instead of 1x30s); this makes a small difference in bands other
than u (which is why we use 1x30s for the calculation above, as visits are expected
to be 30s long in u band).

It is also worth referring to [PSTN-054](https://pstn-054.lsst.io) for a more in-depth
update on expected m5 values, including accounting for the effects of observing over a range
of conditions during operations. Due to different seeing distributions, skybrightness distributions,
and airmass distributions, median expected m5 depths diverge from those above.


Useful github repositories
--------------------------

The algorithms described in `LSE-40 <http://ls.st/lse-40>`_ are implemented in the LSST
`rubin_sim.photUtils <http://github.com/lsst/rubin_sim>`_ package,
available on github. In particular, the
`SignalToNoise
<https://github.com/lsst/rubin_sim/blob/main/rubin_sim/photUtils/SignalToNoise.py>`_
module calculates signal to noise ratios and limiting magnitudes (m5)
values. Here is a
`jupyter notebook example <https://github.com/lsst/rubin_sim_notebooks/blob/main/photometry/Calculating%20SNR.ipynb>`_
using this code to calculate SNR in a variety of situations.

The throughput curves used for this analysis are
based on the throughput components in
`syseng_throughputs <https://github.com/lsst-pst/syseng_throughputs>`_ repository.
These throughput curves are then propagated to ``rubin_sim_data``, in a modified format that
incorporates average losses over time (instead of maintaining these as separate components).
There is more information on the origin of these throughput
curves and other key number data in the section 'Data Sources' below.


.. [#skynote] The atmosphere should not be included in the calculation of
        the expected counts in the focal plane, as the sky emission
        comes from various layers in the atmosphere - a completely
        proper treatment would involve a radiative transfer model that
        includes emission and absorption over the entire
        atmosphere. Instead the standard treatment is to generate a
        sky brightness and sky spectrum that correspond to the
        skybrightness at the pupil of the telescope, and then just
        multiply this by :math:`S_{hardware}(\lambda)` to generate the
        focal plane counts

Calculating m5 values in the LSST Operations Simulator
======================================================

To rapidly calculate the m5 values reported with each visit in the
outputs from the Operations Simulator, the SNR formulas above are
used to calculate two values, :math:`C_m` and :math:`dC_m^{inf}`. These
values can then be used to calculate m5 under a wide range of sky
brightness, seeing, airmass, and exposure times.

.. math::
   m5 = C_m + dC_m + 0.50\,(m_{sky} - 21.0) + 2.5 log_{10}(0.7 /
   FWHM_{eff}) \\
   + 1.25 log_{10}(expTime / 30.0) - k_{atm}\,(X-1.0)

   dC_m = dC_m^{inf} - 1.25 log_{10}(1 + {(10^{(0.8\, dC_m^{inf})} -
   1)}/Tscale)

   Tscale = expTime / 15.0 * 10.0^{-0.4*(m_{sky} - m_{darksky})}

The :math:`dC_m^{inf}` term accounts for the transition between instrument noise limited
observations and sky background limited observations as the
exposure time or sky brightness varies. For most LSST bandpasses, we are
sky-noise dominated even in 15 second exposures, but in the u
band, the sky background is low enough that the exposures become
read noise limited (thus :math:`dC_m^{inf}` has an associated base exposure time used
in its calculation and applied to :math:`Tscale`; this is 15s above).
The :math:`k_{atm}` term captures the extinction of the atmosphere and how it
varies with airmass. It can be calculated as :math:`k_{atm} =
-2.5 log_{10} (T_b / \Sigma_b)`, where :math:`T_b` is the sum of the
total system throughput in a particular bandpass and :math:`\Sigma_b`
is the sum of the hardware throughput in a particular bandpass
(without the atmosphere).


+------+------+-------+-----+
|Filter|Cm    |dCm_inf|k_atm|
+------+------+-------+-----+
|u     |23.39 | 0.37  |0.50 |
+------+------+-------+-----+
|g     |24.51 | 0.10  |0.21 |
+------+------+-------+-----+
|r     |24.49 | 0.05  |0.13 |
+------+------+-------+-----+
|i     |24.37 | 0.04  |0.10 |
+------+------+-------+-----+
|z     |24.21 | 0.02  |0.07 |
+------+------+-------+-----+
|y     |23.77 | 0.02  |0.17 |
+------+------+-------+-----+

These values can be used with the ``rubin_sim.utils``
`m5_scale <https://github.com/lsst/rubin_sim/blob/main/rubin_sim/utils/m5_flat_sed.py#L7>`_ function
to calculate m5 values under varying exposure times, skybrightness or seeing.

For each pointing in OpSim, the skybrightness and seeing come from various
simulated telemetry streams, and airmass and exposure time come from
the scheduling data itself. The skybrightness comes from ``rubin_sim.skybrightness``. It
is based on the `ESO sky calculator
<https://www.eso.org/observing/etc/bin/gen/form?INS.MODE=swspectr+INS.NAME=SKYCALC>`_
along with an empirical model for twilight. The rubin_sim skybrightness model has
been validated with nearly a year of on-site all-sky measurements.
The seeing  comes from ``rubin_sim.site_models``
package, which uses 10 years of seeing data from Cerro Pachon as inputs.
The seeing model generates atmosphere-only FWHM at 500nm at zenith; these
raw atmospheric FWHM values (:math:`FWHM_{500}`) are adjusted to
the image quality delivered by the entire system by

.. math::
   FWHM_{sys}(X) = \sqrt{telSeeing^2 + opticalDesign^2 + cameraSeeing^2} \, (X)^{0.6}

   FWHM_{atm}(X) = FWHM_{500} \, (\frac{500nm}{\lambda_{eff}})^{0.3} \,   (X)^{0.6}

   FWHM_{eff}(X) = 1.16 \sqrt{FWHM_{sys}^2 + 1.04 \, FWHM_{atm}^2}


where the system contributions are telSeeing = 0.25”, opticalDesign =
0.08”, and cameraSeeing = 0.30”. :math:`\lambda_{eff}` is the
effective wavelength for each filter:  366, 482, 622, 754, 869 and
971 nm respectively for u, g, r, i, z, y.

Calculating C_m values
----------------------

The values for :math:`C_m` and :math:`dC_m^{inf}` can be calculated using the m5 value
of a dark sky, zenith visit.

.. math::
   C_m = m5 - 0.5\,(m_{darksky} - 21.0) + 2.5 log_{10}(0.7 / FWHM_{eff}) + 1.25 log_{10}(expTime / 30.0)

where :math:`m_{darksky}` is the dark sky background value in the
bandpass, as described in the table above. A related :math:`C_m^{inf}`
can be calculated using an m5 value generated by assuming that the
instrument noise per exposure is 0. The difference between
:math:`C_m^{inf}` and :math:`C_m` is :math:`dC_m^{inf}`.


Data Sources and References
===========================

Change controlled documents:
 * LSE-40 : "Photon Rates and SNR Calculations" http://ls.st/lse-40 (useful for SNR eqns, but do not use the outdated values from this document)
 * LSE-29 : "LSST System Requirements" http://ls.st/lse-29
 * LSE-30 : "Observatory System Specifications" http://ls.st/lse-30
 * LSE-59 : "Camera Subsystem Requirements" http://ls.st/lse-59

Official project documents not under change control -
 * The LSST Overview Paper http://ls.st/document-5462
 * LSST Key Numbers http://lsst.org/scientists/keynumbers
 * LSST-PST Syseng_throughputs components git repository  https://github.com/lsst-pst/syseng_throughputs
 * SMTN-002 https://smtn-002.lsst.io  (this documnent)
 * PSTN-054 https://pstn-054.lsst.io

+---------------------------------------------------------+--------+------------------------------------------------------+
|Primary mirror clear aperture [#areanote]_               | 6.423 m| LSE-29, LSR-REQ-0003, LSST Key Numbers               |
+---------------------------------------------------------+--------+------------------------------------------------------+
|Median delivered Image Quality                           | 0.65"  | Overview Paper, fig. 1 (Site DIMM + telescope model) |
+---------------------------------------------------------+--------+------------------------------------------------------+
|Total instrumental noise per exposure                    | 9 e-   | LSE-59, CAM-REQ-0020 (readnoise and dark current)    |
+---------------------------------------------------------+--------+------------------------------------------------------+
|Diameter of field of view                                | 3.5 deg| LSE-29, LSR-REQ-0004                                 |
+---------------------------------------------------------+--------+------------------------------------------------------+
|Focal plane coverage (fill factor in active area of FOV) |  >90%  | LSE-30, OSS-REQ-0259                                 |
+---------------------------------------------------------+--------+------------------------------------------------------+
|Focal plane coverage (fill factor in active area of FOV) | 91%    | Calculated from focal plane models                   |
+---------------------------------------------------------+--------+------------------------------------------------------+

.. [#areanote] The area-weighted clear aperture is 6.423 m across the entire field of view, although this varies with location. Near the center, the clear aperture is 6.7 m, while near the edge of the field of view it rolls off by about 10%. 6.423 m is the area-weighted average across the full field of view.

Throughput curves: `syseng_throughputs github repo <https://github.com/lsst-pst/syseng_throughputs>`_:

    The QE curve for the CCD is measured from
    prototype devices delivered by the two vendors under
    consideration.  The filter transmission curves match those provided as
    specifications to vendors, and are derived from LSE-30,
    OSS-REQ-0240.
    Mirror reflectivities are based on lab measurements of pristine
    witness samples; the losses  and lens transmission curves are
    based on expected performance curves. The atmospheric transmission
    is based on MODTRAN models of the atmosphere at Cerro Pachon, with
    the addition of a conservative amount of aerosols. The
    throughput curves are consistent with the relevant requirements documents,
    LSE-29 and LSE-30. More information on the throughput curves for
    each component, along with the time-averaged losses applied to
    each component due to surface contamination and condensation, is
    available in the `README <https://github.com/lsst-pst/syseng_throughputs/blob/master/README.md>`_.

The throughput curves in the syseng_throughputs repository track
the expected performance of the components of the LSST systems.
There are versions of these throughput curves packaged for
distribution in the `throughputs <https://github.com/lsst/throughputs>`_ github repository,
as well as with `rubin_sim_data <https://s3df.slac.stanford.edu/data/rubin/sim-data/rubin_sim_data/>_`.

The dark sky sky brightness values come from a dark sky, zenith
spectrum which produces broadband dark sky background measurements
consistent with observed values at SDSS and other sites. The skybrightness
from ``rubin_sim.skybrightness`` is also in general agreement with
these dark sky values. The ``rubin_sim`` skybrightness simulator includes
twilight sky brightness, as well as explicit components contributed by
the moon, zodiacal light, airglow and sky emission lines - it is based
on the `ESO sky calculator
<https://www.eso.org/observing/etc/bin/gen/form?INS.MODE=swspectr+INS.NAME=SKYCALC>`_
with the addition of a twilight sky model based on observational data
from the LSST site.

The conversion from atmospheric FWHM to delivered image quality is
based on ray-trace simulations by Bo Xin (LSST Systems
Engineering). The atmospheric FWHM measurements come from an on-site
DIMM, described in more depth in the Site Selection documents. The
DIMM measurements were cross-checked with measurements coming from
nearby atmospheric monitoring systems from other observatories. 
