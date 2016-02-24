Calculating SNR
----------------

Calculating either signal to noise ratios for various sources, or
5-sigma point source limiting magnitudes for LSST can be accomplished
using standard SNR equations together with available
information on the expected LSST camera and telescope components.

The appropriate methodology to calculate SNR values for PSF-optimized
photometry is outlined in the LSST Change Controlled Document
`LSE-40 <http://ls.st/lse-40>`_, although this document is awaiting
updates to match new throughput curves and updated information on how
we're handling the PSF profile which means the actual values
calculated in this document are outdated.

The algorithms described in `LSE-40 <http://ls.st/lse-40>`_ are implemented in the LSST
`sims_photUtils <http://github.com/lsst/sims_photUtils>`_ package. In
particular, the
`SignalToNoise
<https://github.com/lsst/sims_photUtils/blob/master/python/lsst/sims/photUtils/SignalToNoise.py>`_
module calculates signal to noise ratios and limiting magnitudes (m5)
values. The SNR calculation per 30 second visit (composed of 2 back-to-back 15 second exposures)
can be summarized as follows:

.. math::
    SNR = \frac{C } {\sqrt{C/g + ( B/g + \sigma^2_{instr}) \, n_{eff}}}

    n_{eff} = 2.266 \, (FWHM_{eff} / pixelScale)^2

where C = total source counts, B = sky background counts per pixel,
and g = gain (expected gain is 2.3 electron/ADU, but for purposes of
calculating SNR or m5, it can safely be assumed to be 1).

The instrumental noise, :math:`\sigma_{instr}`, can be calculated by

.. math::
   \sigma_{instr}^2 = (readNoise^2 + (darkCurrent * expTime)) * n_{exp}

where the current requirement place upper limits of 0.2 photo-electrons/second/pixel
on the dark current and 8.8 photo-electrons/pixel/exposure on the readnoise. The current
LSST observing plan is to take back-to-back exposures of the same field, each
exposure 15 seconds long, for a total of :math:`n_{exp}`=2 exposures per 30 second
long "visit". The total instrumental noise per visit is then 12.7 photo-electrons.

The effective number of pixels in the PSF, :math:`n_{eff}`, is
calculated assuming a single gaussian PSF function, and so we
calculate :math:`FWHM_{eff}` from the PSF determined from measuring
the observed von Karman profile (which has wider wings than a
gaussian), :math:`FWHM_{geom}`:

.. math::
     FWHM_{eff} = (FWHM_{geom} - 0.052) / 0.822

:math:`FWHM_{geom}` is typically slightly smaller than
:math:`FWHM_{eff}`. The conversion factor was calculated by
generating PSF profiles using raytrace software with models of the
LSST mirrors and camera system and will be described in a planned
update of `LSE-40 <http://ls.st/lse-40>`_ and the `LSST Overview Paper <http://arxiv.org/pdf/0805.2366.pdf>`_.
The expected median :math:`FWHM_{eff}` at zenith in the various LSST
bandpasses is

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

where this includes the expected/modeled telescope contribution as well as the distribution of IQ measurements
from an on-site DIMM.

The total counts in the focal plane from any source can be calculated by multiplying the source
spectrum, :math:`F_\nu(\lambda)` at the top of the atmosphere in Janskys, by the fractional
probability of reaching the focal plane and being converted into
electrons and integrating over wavelength (:math:`S(\lambda)`):

.. math::
   C = \frac {expTime \,  effArea} {gain \, h} \int { F_\nu(\lambda) \, S(\lambda)  / \lambda  d\lambda }

where expTime = exposure time in seconds (typically 30 seconds for LSST), effArea
= effective collecting area in cm^2 (effective area-weighted diameter for the LSST primary,
when occultation from the secondary and tertiary mirrors and
vignetting effects are included, is 6.423 m), and h = Planck constant. We
can also use the above formula, together with a conversion from counts
to AB magnitudes, to calculate the "instrumental zeropoint" in each
bandpass, the magnitude which would produce one ADU-count per second. The throughput curves used for this analysis are
based on the throughput components in the `syseng_throughputs <https://github.com/lsst-pst/syseng_throughputs>`_ repository.
There is more information on the origin of these throughput
curves and other key number data in the section 'Data Sources' below.

+------+--------------------------------------------+
|Filter|Instrumental Zeropoint (exptime=1s, gain=1) |
+------+--------------------------------------------+
|u     |     26.50                                  |
+------+--------------------------------------------+
|g     |     28.30                                  |
+------+--------------------------------------------+
|r     |      28.13                                 |
+------+--------------------------------------------+
|i     |      27.79                                 |
+------+--------------------------------------------+
|z     |    27.40                                   |
+------+--------------------------------------------+
|y     |    26.58                                   |
+------+--------------------------------------------+

The expected sky brightness at zenith, in dark sky, has been
calculated in each LSST bandpass by generating a dark sky spectrum,
using data from UVES and Gemini near-IR combined with an ESO sky
spectrum, with a slight normalization in the u and y bands to match the median dark sky values
reported by SDSS. The resulting zenith, dark sky brightness values are
in good agreement with other measurements from CTIO and ESO.

+------+--------------------------------+
|Filter|Sky brightness (mag/arcsecond^2)|
+------+--------------------------------+
|u     |     22.95                      |
+------+--------------------------------+
|g     |     22.24                      |
+------+--------------------------------+
|r     |     21.20                      |
+------+--------------------------------+
|i     |     20.47                      |
+------+--------------------------------+
|z     |    19.60                       |
+------+--------------------------------+
|y     |    18.63                       |
+------+--------------------------------+

The zeropoints above could be used to calculate approximate background
sky counts, or exact values could be calculated using the spectrum
available at `darksky.dat
<https://github.com/lsst-pst/syseng_throughputs/blob/master/siteProperties/darksky.dat>`_.
To calculate the counts from the sky per pixel, we can calculate counts per square arcsecond 
and convert to per pixel using the platescale, 0.2 "/pixel.

With all of these values, we can calculate the "m5" for each bandpass,
the :math:`5\sigma` limiting magnitude for point sources in the dark
sky, zenith case. (Note, this is implemented in the ``calcm5`` method of the
`SignalToNoise
<https://github.com/lsst/sims_photUtils/blob/master/python/lsst/sims/photUtils/SignalToNoise.py>`_
module in `sims_photUtils
<https://github.com/lsst/sims_photUtils>`_). The resulting values are

+------+------+
|Filter|m5    |
+------+------+
|u     |23.60 |
+------+------+
|g     |24.83 |
+------+------+
|r     |24.38 |
+------+------+
|i     |23.92 |
+------+------+
|z     |23.35 |
+------+------+
|y     |22.44 |
+------+------+


Calculating m5 values in the LSST Operations Simulator
-------------------------------------------------------

To rapidly calculate the m5 values reported with each visit in the
outputs from the Operations Simulator, the SNR formulas above are
used to calculate two values, :math:`C_m` and :math:`dC_m^{inf}`. These
values can then be used to calculate m5 under a wide range of sky
brightness, seeing, airmass, and exposure times.

.. math::
   m5 = C_m + dC_m + 0.50\,(m_{sky} - 21.0) + 2.5 log_{10}(0.7 /
   FWHM_{eff}) \\
   + 1.25 log_{10}(expTime / 30.0) - k_{atm}\,(X-1.0)

   dC_m = dC_m^{inf} - 1.25 log_{10}(1 + (10^{(0.8\, dC_m^{inf} -
   1)}/Tscale)

   Tscale = expTime / 30.0 * 10.0^{-0.4*(m_{sky} - m_{darksky})}

The values for :math:`C_m` and :math:`dC_m^{inf}` can be calculated from the m5 value
of a dark sky, zenith visit.

.. math::
   C_m = m5 - 0.5\,(m_{darksky} - 21.0) + 2.5 log_{10}(0.7 / FWHM_{eff}) + 1.25 log_{10}(expTime / 30.0)

where :math:`m_{darksky}` is the dark sky background value in the
bandpass, as described in the table above. A related :math:`C_m^{inf}`
can be calculated using an m5 value generated by assuming that the
instrument noise per exposure is 0: the difference between
:math:`C_m^{inf}` and :math:`C_m` is :math:`dC_m^{inf}`. This term
accounts for the transition between instrument noise limited
observations and sky background limited observations as the
exposure time changes. For most LSST bandpasses, we are
sky-noise dominated even in 15 second exposures, but in the u
band, the sky background is low enough that the exposures become
read noise limited.

The :math:`k_{atm}` term captures the extinction of the atmosphere and how it
varies with airmass. It can be calculated as :math:`k_{atm} =
-2.5 log_{10} (T_b / \Sigma_b)`, where :math:`T_b` is the sum of the
total system throughput in a particular bandpass and :math:`\Sigma_b`
is the sum of the hardware throughput in a particular bandpass
(without the atmosphere).

+------+------+-------+-----+
|Filter|Cm    |dCm_inf|k_atm|
+------+------+-------+-----+
|u     |22.91 | 0.57  |0.50 |
+------+------+-------+-----+
|g     |24.45 | 0.12  |0.21 |
+------+------+-------+-----+
|r     |24.46 | 0.06  |0.13 |
+------+------+-------+-----+
|i     |24.33 | 0.05  |0.10 |
+------+------+-------+-----+
|z     |24.17 | 0.03  |0.07 |
+------+------+-------+-----+
|y     |23.71 | 0.02  |0.18 |
+------+------+-------+-----+

These values are used within OpSim to calculate m5 values for each
pointing in the ``calc_m5`` function in `gen_output.py
<https://github.com/lsst/sims_operations/blob/master/tools/schema_tools/gen_output.py>`_
within the `sims_operations
<https://github.com/lsst/sims_operations>`_ codebase.

The remaining required inputs to calculate m5 in OpSim are the sky
brightness and the seeing, as the airmass and exposure time will come
from the scheduling data itself.

The sky brightness is currently
calculated using a V-band sky brightness model based on Krisciunas &
Schafer (1991) `(K&S) <http://adsabs.harvard.edu/abs/1991PASP..103.1033K>`_,
which is then adjusted to give sky brightness values in various
bandpasses using color terms that depend on the phase of the
moon. The V-band sky brightness calculations are implemented in the
`AstronomicalSky.py <https://github.com/lsst/sims_operations/blob/master/python/lsst/sims/operations/AstronomicalSky.py>`_
module of OpSim, and the per-filter adjustments based on lunar phase
are done in
`Filters.py <https://github.com/lsst/sims_operations/blob/master/python/lsst/sims/operations/Filters.py>`_.
The current OpSim model simply sets y band skybrightness to 17.3 and implements a
step-function for twilight if the altitude of the sun is above -18
degrees, setting the sky brightness to 17.0 in z and y (and the
scheduler is then constrained to observed in z and y during this time,
currently). In the near future we will be updating the OpSim sky
brightness model, to a new
`sims_skybrightness <https://github.com/lsst/sims_skybrightness>`_
model that more closely follows the `ESO sky calculator
<https://www.eso.org/observing/etc/bin/gen/form?INS.MODE=swspectr+INS.NAME=SKYCALC>`_
along with an empirical model for twilight. The sims_skybrightness model has
been validated with nearly a year of on-site all-sky measurements. The current model
has various flaws compared to the upcoming new model, but for the most
part these flaws result in a brighter sky brightness value being used
currently than the more realistic sims_skybrightness model predicts
(see `comparison <https://community.lsst.org/t/comparing-eso-sky-model-to-current-opsim-sky-values/489>`_).

The input seeing data used in OpSim are the atmosphere-only FWHM at
500 nm at zenith,  based on three years of on-site DIMM
measurements. The raw atmospheric FWHM values (:math:`FWHM_{500}`) are adjusted to
the image quality delivered by the entire system by

.. math::
   FWHM_{sys}(X) = \sqrt{(telSeeing \, X^{0.6})^2 + opticalDesign^2 + cameraSeeing^2}

   FWHM_{atm}(X) = FWHM_{500} \, (\frac{500nm}{\lambda_{eff}})^{0.3} \,   (X)^{0.6}

   FWHM_{eff}(X) = 1.16 \sqrt{FWHM_{sys}^2 + 1.04 \, FWHM_{atm}^2}


where the system contributions are telSeeing = 0.25”, opticalDesign =
0.08”, and cameraSeeing = 0.30”. :math:`\lambda_{eff}` is the
effective wavelength for each filter:  366, 482, 622, 754, 869 and
971 nm respectively for u, g, r, i, z, y.
 

Data Sources and References
------------------------------------

Change controlled documents:
 * LSE-40 : "Photon Rates and SNR Calculations" <http://ls.st/lse-40>
 * LSE-29 : "LSST System Requirements" <http://ls.st/lse-29>
 * LSE-30 : "Observatory System Specifications" <http://ls.st/lse-30>
 * LSE-59 : "Camera Subsystem Requirements" <http://ls.st/lse-59>

Official project documents not under change control -
 * The LSST Overview Paper <http://www.lsst.org/content/lsst-science-drivers-reference-design-and-anticipated-data-products>
 * LSST Key Numbers <http://lsst.org/scientists/keynumbers>
 * LSST-PST Syseng_throughputs components git repository  <https://github.com/lsst-pst/syseng_throughputs>

+---------------------------------------------------------+--------+------------------------------------------------------+
|Primary mirror clear aperture                            | 6.423 m| LSE-29, LSR-REQ-0003, LSST Key Numbers               |
+---------------------------------------------------------+--------+------------------------------------------------------+
|Diameter of field of view                                | 3.5 deg| LSE-29, LSR-REQ-0004                                 |
+---------------------------------------------------------+--------+------------------------------------------------------+
|Delivered Image Quality                                  | 0.65"  | Overview Paper, fig. 1 (Site DIMM + telescope model) |
+---------------------------------------------------------+--------+------------------------------------------------------+
|Total instrumental noise per exposure                    | 9 e-   | LSE-59, CAM-REQ-0020 (readnoise and dark current)    |
+---------------------------------------------------------+--------+------------------------------------------------------+
|Focal plane coverage (fill factor in active area of FOV) |  >90%  | LSE-30, OSS-REQ-0259                                 |
+---------------------------------------------------------+--------+------------------------------------------------------+
|Focal plane coverage (fill factor in active area of FOV) | 91%    | Calculated from focal plane models                   |
+---------------------------------------------------------+--------+------------------------------------------------------+

The area-weighted clear aperture is 6.423 m across the entire field of view, although this varies with location. Near the center,
the clear aperture is 6.7 m, while near the edge of the field of view it rolls off by about 10%. 6.423 m is the area-weighted
average across the full field of view.

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
distribution in the `throughputs <https://github.com/lsst/throughputs>`_ github repository, along
with jupyter notebook examples of calculating SNR using these curves and the sims_photUtils package,
such as `this notebook <https://github.com/lsst/throughputs/blob/Update-from-syseng_/examples/Calculating%20SNR.ipynb>`_.

The dark sky sky brightness values come from a dark sky, zenith
spectrum which produces broadband dark sky background measurements
consistent with observed values at SDSS and other sites. We have a new
`skybrightness <https://github.com/lsst/sims_skybrightness>`_ package in development which is also in general agreement with
these dark sky values. The new sky brightness simulator includes
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
