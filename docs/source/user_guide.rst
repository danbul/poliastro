User guide
==========

Defining the orbit: :py:class:`~poliastro.twobody.orbit.Orbit` objects
----------------------------------------------------------------------

The core of poliastro are the :py:class:`~poliastro.twobody.orbit.Orbit` objects
inside the :py:mod:`poliastro.twobody` module. They store all the required
information to define an orbit:

* The body acting as the central body of the orbit, for example the Earth.
* The position and velocity vectors or the orbital elements.
* The time at which the orbit is defined.

First of all, we have to import the relevant modules and classes:

.. code-block:: python

    import numpy as np
    import matplotlib.pyplot as plt
    plt.ion()  # To immediately show plots

    from astropy import units as u

    from poliastro.bodies import Earth, Mars, Sun
    from poliastro.twobody import Orbit

    plt.style.use("seaborn")  # Recommended

From position and velocity
~~~~~~~~~~~~~~~~~~~~~~~~~~

There are several methods available to create
:py:class:`~poliastro.twobody.orbit.Orbit` objects. For example, if we have the
position and velocity vectors we can use
:py:meth:`~poliastro.twobody.orbit.Orbit.from_vectors`:

.. code-block:: python

    # Data from Curtis, example 4.3
    r = [-6045, -3490, 2500] * u.km
    v = [-3.457, 6.618, 2.533] * u.km / u.s

    ss = Orbit.from_vectors(Earth, r, v)

And that's it! Notice a couple of things:

* Defining vectorial physical quantities using Astropy units is very easy.
  The list is automatically converted to a :py:mod:`astropy.units.Quantity`,
  which is actually a subclass of NumPy arrays.
* If we display the orbit we just created, we get a string with the radius of
  pericenter, radius of apocenter, inclination, reference frame and attractor::

    >>> ss
    7283 x 10293 km x 153.2 deg (GCRS) orbit around Earth (♁)

* If no time is specified, then a default value is assigned::

    >>> ss.epoch
    <Time object: scale='utc' format='jyear_str' value=J2000.000>
    >>> ss.epoch.iso
    '2000-01-01 12:00:00.000'

* The reference frame of the orbit will be one pseudo-inertial frame around the
  attractor. You can retrieve it using the :py:attr:`~poliastro.twobody.orbit.Orbit.frame` property:

    >>> ss.frame
    <GCRS Frame (obstime=J2000.000, obsgeoloc=(0., 0., 0.) m, obsgeovel=(0., 0., 0.) m / s)>

.. _`International Celestial Reference System or ICRS`: http://web.archive.org/web/20170920023932/http://aa.usno.navy.mil:80/faq/docs/ICRS_doc.php

.. note::

  At the moment, there is no explicit way to set the reference system of an orbit. This
  is the focus of our next releases, so we will likely introduce changes in the near
  future. Please subscribe to `this issue <https://github.com/poliastro/poliastro/issues/257>`_
  to receive updates in your inbox.

Intermezzo: quick visualization of the orbit
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. figure:: _static/curtis.png
   :align: right
   :figwidth: 350
   :alt: Plot of the orbit

If we're working on interactive mode (for example, using the wonderful IPython
notebook) we can immediately plot the current state::

    from poliastro.plotting import plot
    plot(ss)

This plot is made in the so called *perifocal frame*, which means:

* we're visualizing the plane of the orbit itself,
* the \\(x\\) axis points to the pericenter, and
* the \\(y\\) axis is turned \\(90 \\mathrm{^\\circ}\\) in the
  direction of the orbit.

The dotted line represents the *osculating orbit*:
the instantaneous Keplerian orbit at that point. This is relevant in the
context of perturbations, when the object shall deviate from its Keplerian
orbit.

.. warning::

  Be aware that, outside the Jupyter notebook (i.e. a normal Python interpreter
  or program) you might need to call :code:`plt.show()` after the plotting
  commands or :code:`plt.ion()` before them or they won't show. Check out the
  `Matplotlib FAQ`_ for more information.

.. _`Matplotlib FAQ`: http://matplotlib.org/faq/usage_faq.html#non-interactive-example

From classical orbital elements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can also define a :py:class:`~poliastro.twobody.orbit.Orbit` using a set of
six parameters called orbital elements. Although there are several of
these element sets, each one with its advantages and drawbacks, right now
poliastro supports the *classical orbital elements*:

* Semimajor axis \\(a\\).
* Eccentricity \\(e\\).
* Inclination \\(i\\).
* Right ascension of the ascending node \\(\\Omega\\).
* Argument of pericenter \\(\\omega\\).
* True anomaly \\(\\nu\\).

In this case, we'd use the method
:py:meth:`~poliastro.twobody.orbit.Orbit.from_classical`:

.. code-block:: python

    # Data for Mars at J2000 from JPL HORIZONS
    a = 1.523679 * u.AU
    ecc = 0.093315 * u.one
    inc = 1.85 * u.deg
    raan = 49.562 * u.deg
    argp = 286.537 * u.deg
    nu = 23.33 * u.deg
    
    ss = Orbit.from_classical(Sun, a, ecc, inc, raan, argp, nu)

Notice that whether we create a ``Orbit`` from \\(r\\) and \\(v\\) or from
elements we can access many mathematical properties individually using the
:py:attr:`~poliastro.twobody.orbit.Orbit.state` property of
:py:class:`~poliastro.twobody.orbit.Orbit` objects::

    >>> ss.state.period.to(u.day)
    <Quantity 686.9713888628166 d>
    >>> ss.state.v
    <Quantity [  1.16420211, 26.29603612,  0.52229379] km / s>

To see a complete list of properties, check out the
:py:class:`poliastro.twobody.orbit.Orbit` class on the API reference.

Moving forward in time: propagation
-----------------------------------

Now that we have defined an orbit, we might be interested in computing
how is it going to evolve in the future. In the context of orbital
mechanics, this process is known as **propagation**, and can be
performed with the ``propagate`` method of
:py:class:`~poliastro.twobody.orbit.Orbit` objects::

    >>> from poliastro.examples import iss
    >>> iss
    6772 x 6790 km x 51.6 deg (GCRS) orbit around Earth (♁)
    >>> iss.epoch
    <Time object: scale='utc' format='iso' value=2013-03-18 12:00:00.000>
    >>> iss.nu.to(u.deg)
    <Quantity 46.595804677061956 deg>
    >>> iss.n.to(u.deg / u.min)
    <Quantity 3.887010576192155 deg / min>

Using the :py:meth:`~poliastro.twobody.orbit.Orbit.propagate` method
we can now retrieve the position of the ISS after some time::

    >>> iss_30m = iss.propagate(30 * u.min)
    >>> iss_30m.epoch  # Notice we advanced the epoch!
    <Time object: scale='utc' format='iso' value=2013-03-18 12:30:00.000>
    >>> iss_30m.nu.to(u.deg)
    <Quantity 163.1409357544868 deg>

For more advanced propagation options, check out the
:py:mod:`poliastro.twobody.propagation` module.

Studying non-keplerian orbits: perturbations
--------------------------------------------

Apart from the Keplerian propagators, poliastro also allows
the user to define custom perturbation accelerations to study
non Keplerian orbits, thanks to Cowell's method::

    >>> from poliastro.twobody.propagation import cowell
    >>> from numba import njit
    >>> r0 = [-2384.46, 5729.01, 3050.46] * u.km
    >>> v0 = [-7.36138, -2.98997, 1.64354] * u.km / u.s
    >>> initial = Orbit.from_vectors(Earth, r0, v0)
    >>> @njit
    ... def accel(t0, state, k):
    ...     """Constant acceleration aligned with the velocity. """
    ...     v_vec = state[3:]
    ...     norm_v = (v_vec * v_vec).sum() ** .5
    ...     return 1e-5 * v_vec / norm_v
    ...
    >>> initial.propagate(3 * u.day, method=cowell, ad=accel)
    18255 x 21848 km x 28.0 deg (GCRS) orbit around Earth (♁)

Some natural perturbations are available in poliastro to be used
directly in this way. For instance,
let us examine the effect of J2 perturbation::

    >>> from poliastro.twobody.perturbations import J2_perturbation
    >>> tof = (48.0 * u.h).to(u.s)
    >>> final = initial.propagate(tof, method=cowell, ad=J2_perturbation, J2=Earth.J2.value, R=Earth.R.to(u.km).value)

The J2 perturbation changes the orbit parameters (from Curtis example 12.2)::

    >>> ((final.raan - initial.raan) / tof).to(u.deg / u.h)
    <Quantity -0.17232668 deg / h>
    >>> ((final.argp - initial.argp) / tof).to(u.deg / u.h)
    <Quantity 0.28220397 deg / h>

For more available perturbation options, see the
:py:mod:`poliastro.twobody.perturbations` module.

Studying artificial perturbations: thrust
--------------------------------------------

In addition to natural perturbations, poliastro also has
built-in artificial perturbations (thrusts) aimed
at intentional change of some orbital elements. 
Let us simultaineously change eccentricy and inclination::

    >>> from poliastro.twobody.thrust import change_inc_ecc
    >>> from poliastro.twobody import Orbit
    >>> from poliastro.bodies import Earth
    >>> from poliastro.twobody.propagation import cowell
    >>> from astropy import units as u
    >>> from astropy.time import Time
    >>> ecc_0, ecc_f = 0.4, 0.0
    >>> a = 42164
    >>> inc_0, inc_f = 0.0, (20.0 * u.deg).to(u.rad).value
    >>> argp = 0.0
    >>> f = 2.4e-7
    >>> k = Earth.k.to(u.km**3 / u.s**2).value
    >>> s0 = Orbit.from_classical(Earth, a * u.km, ecc_0 * u.one, inc_0 * u.deg, 0 * u.deg, argp * u.deg, 0 * u.deg, epoch=Time(0, format='jd', scale='tdb'))
    >>> a_d, _, _, t_f = change_inc_ecc(s0, ecc_f, inc_f, f)
    >>> sf = s0.propagate(t_f * u.s, method=cowell, ad=a_d, rtol=1e-8)

The thrust changes orbit parameters as desired (within errors)::

    >>> sf.inc, sf.ecc
    (<Quantity 0.34719734 rad>, <Quantity 0.00894513>)

For more available perturbation options, see the
:py:mod:`poliastro.twobody.thrust` module.

Changing the orbit: :py:class:`~poliastro.maneuver.Maneuver` objects
--------------------------------------------------------------------

poliastro helps us define several in-plane and general out-of-plane
maneuvers with the :py:class:`~poliastro.maneuver.Maneuver` class inside the
:py:mod:`poliastro.maneuver` module.

Each ``Maneuver`` consists on a list of impulses \\(\\Delta v_i\\)
(changes in velocity) each one applied at a certain instant \\(t_i\\). The
simplest maneuver is a single change of velocity without delay: you can
recreate it either using the :py:meth:`~poliastro.maneuver.Maneuver.impulse`
method or instantiating it directly.

.. code-block:: python

    from poliastro.maneuver import Maneuver

    dv = [5, 0, 0] * u.m / u.s
    
    man = Maneuver.impulse(dv)
    man = Maneuver((0 * u.s, dv))  # Equivalent

There are other useful methods you can use to compute common in-plane
maneuvers, notably :py:meth:`~poliastro.maneuver.Maneuver.hohmann` and
:py:meth:`~poliastro.maneuver.Maneuver.bielliptic` for `Hohmann`_ and
`bielliptic`_ transfers respectively. Both return the corresponding
``Maneuver`` object, which in turn you can use to calculate the total cost
in terms of velocity change (\\(\\sum \|\\Delta v_i|\\)) and the transfer
time::

    >>> ss_i = Orbit.circular(Earth, alt=700 * u.km)
    >>> ss_i
    7078 x 7078 km x 0.0 deg (GCRS) orbit around Earth (♁)
    >>> hoh = Maneuver.hohmann(ss_i, 36000 * u.km)
    >>> hoh.get_total_cost()
    <Quantity 3.6173981270031357 km / s>
    >>> hoh.get_total_time()
    <Quantity 15729.741535747102 s>

You can also retrieve the individual vectorial impulses::

    >>> hoh.impulses[0]
    (<Quantity 0 s>, <Quantity [ 0.        , 2.19739818, 0.        ] km / s>)
    >>> hoh[0]  # Equivalent
    (<Quantity 0 s>, <Quantity [ 0.        , 2.19739818, 0.        ] km / s>)
    >>> tuple(val.decompose([u.km, u.s]) for val in hoh[1])
    (<Quantity 15729.741535747102 s>, <Quantity [ 0.        , 1.41999995, 0.        ] km / s>)

.. _Hohmann: http://en.wikipedia.org/wiki/Hohmann_transfer_orbit
.. _bielliptic: http://en.wikipedia.org/wiki/Bi-elliptic_transfer

To actually retrieve the resulting ``Orbit`` after performing a maneuver, use
the method :py:meth:`~poliastro.twobody.orbit.Orbit.apply_maneuver`::

    >>> ss_f = ss_i.apply_maneuver(hoh)
    >>> ss_f
    36000 x 36000 km x 0.0 deg (GCRS) orbit around Earth (♁)

More advanced plotting: :py:class:`~poliastro.plotting.OrbitPlotter` objects
----------------------------------------------------------------------------

We previously saw the :py:func:`poliastro.plotting.plot` function to easily
plot orbits. Now we'd like to plot several orbits in one graph (for example,
the maneuver we computed in the previous section). For this purpose, we
have :py:class:`~poliastro.plotting.OrbitPlotter` objects in the
:py:mod:`~poliastro.plotting` module.

These objects hold the perifocal plane of the first ``Orbit`` we plot in
them, projecting any further trajectories on this plane. This allows to
easily visualize in two dimensions:

.. code-block:: python

    from poliastro.plotting import OrbitPlotter
    
    op = OrbitPlotter()
    ss_a, ss_f = ss_i.apply_maneuver(hoh, intermediate=True)
    op.plot(ss_i, label="Initial orbit")
    op.plot(ss_a, label="Transfer orbit")
    op.plot(ss_f, label="Final orbit")

Which produces this beautiful plot:

.. figure:: _static/hohmann.png
   :align: center
   :alt: Hohmann transfer
   
   Plot of a Hohmann transfer.

Where are the planets? Computing ephemerides
--------------------------------------------

.. versionadded:: 0.3.0

Thanks to Astropy and jplephem, poliastro can now read Satellite
Planet Kernel (SPK) files, part of NASA's SPICE toolkit. This means that
we can query the position and velocity of the planets of the Solar System.

The method :py:meth:`~poliastro.twobody.orbit.Orbit.get_body_ephem` will return
a planetary orbit using low precision ephemerides available in
Astropy and an :py:mod:`astropy.time.Time`:

.. code-block:: python

    from astropy import time
    epoch = time.Time("2015-05-09 10:43")  # UTC by default

And finally, retrieve the planet orbit::

    >>> from poliastro import ephem
    >>> Orbit.from_body_ephem(Earth, epoch)
    1 x 1 AU x 23.4 deg (ICRS) orbit around Sun (☉)

This does not require any external download. If on the other hand we want
to use higher precision ephemerides, we can tell Astropy to do so::

    >>> from astropy.coordinates import solar_system_ephemeris
    >>> solar_system_ephemeris.set("jpl")
    Downloading http://naif.jpl.nasa.gov/pub/naif/generic_kernels/spk/planets/de430.bsp
    |==========>-------------------------------|  23M/119M (19.54%) ETA    59s22ss23

This in turn will download the ephemerides files from NASA and use them
for future computations. For more information, check out
`Astropy documentation on ephemerides`_.

.. _Astropy documentation on ephemerides: http://docs.astropy.org/en/v2.0.4/coordinates/solarsystem.html

.. note:: The position and velocity vectors are given with respect to the
    Solar System Barycenter in the **International Celestial Reference Frame**
    (ICRF), which means approximately equatorial coordinates.

Traveling through space: solving the Lambert problem
----------------------------------------------------

The determination of an orbit given two position vectors and the time of
flight is known in celestial mechanics as **Lambert's problem**, also
known as two point boundary value problem. This contrasts with Kepler's
problem or propagation, which is rather an initial value problem.

The package :py:obj:`poliastro.iod` allows as to solve Lambert's problem,
provided the main attractor's gravitational constant, the two position
vectors and the time of flight. As you can imagine, being able to compute
the positions of the planets as we saw in the previous section is the
perfect complement to this feature!

For instance, this is a simplified version of the example
`Going to Mars with Python using poliastro`_, where the orbit of the
Mars Science Laboratory mission (rover Curiosity) is determined:

.. code-block:: python

    date_launch = time.Time('2011-11-26 15:02', scale='utc')
    date_arrival = time.Time('2012-08-06 05:17', scale='utc')
    tof = date_arrival - date_launch

    ss0 = Orbit.from_body_ephem(Earth, date_launch)
    ssf = Orbit.from_body_ephem(Mars, date_arrival)

    from poliastro import iod
    (v0, v), = iod.lambert(Sun.k, ss0.r, ssf.r, tof)

And these are the results::

    >>> v0
    <Quantity [-29.29150998, 14.53326521,  5.41691336] km / s>
    >>> v
    <Quantity [ 17.6154992 ,-10.99830723, -4.20796062] km / s>

.. figure:: _static/msl.png
   :align: center
   :alt: MSL orbit

   Mars Science Laboratory orbit.

.. _`Going to Mars with Python using poliastro`: http://nbviewer.ipython.org/github/poliastro/poliastro/blob/master/examples/Going%20to%20Mars%20with%20Python%20using%20poliastro.ipynb

Working with NEOs
-----------------
`NEOs (Near Earth Objects)`_ are asteroids and comets whose orbits are near to earth (obvious, isn't it?).
More correctly, their perihelion (closest approach to the Sun) is less than 1.3 astronomical units (≈ 200 * 10\ :sup:`6` km).
Currently, they are being an important subject of study for scientists around the world, due to their status as the relatively
unchanged remains from the solar system formation process.

Because of that, a new module related to NEOs has been added to ``poliastro``
as part of `SOCIS 2017 project`_.

For the moment, it is possible to search NEOs by name (also using wildcards),
and get their orbits straight from NASA APIs, using :py:func:`~poliastro.neos.orbit_from_name`.
For example, we can get `Apophis asteroid (99942 Apophis)`_ orbit with one command, and plot it:

.. code-block:: python

    from poliastro.neos import neows

    apophis_orbit = neows.orbit_from_name('apophis')  # Also '99942' or '99942 apophis' works
    earth_orbit =  Orbit.from_body_ephem(Earth)

    op = OrbitPlotter()
    op.plot(earth_orbit, label='Earth')
    op.plot(apophis_orbit, label='Apophis')

.. figure:: _static/neos.png
   :align: center
   :alt: Apophis asteroid orbit
   
   Apophis asteroid orbit compared to Earth orbit.

.. _`SOCIS 2017 project`: https://github.com/poliastro/poliastro/wiki/SOCIS-2017
.. _`NEOs (Near Earth Objects)`: https://en.wikipedia.org/wiki/Near-Earth_object
.. _`Apophis asteroid (99942 Apophis)`: https://en.wikipedia.org/wiki/99942_Apophis

*Per Python ad astra* ;)
