.. _prism:

Rectangular prisms
==================

One of the geometric bodies that Harmonica is able to forward model are
*rectangular prisms*.
We can compute their gravitational fields through the
:func:`harmonica.prism_gravity` function.

Rectangular prisms can only be defined in Cartesian coordinates, therefore they
are intended to be used to forward model bodies in a small geographic region,
in which we can neglect the curvature of the Earth.

Each prism can be defined as a tuple containing its boundaries in the following
order: *west*, *east*, *south*, *north*, *bottom*, *top* in Cartesian
coordinates and in meters.

.. jupyter-execute::
   :hide-code:

   import harmonica as hm

Lets define a single prism and compute the gravitational potential
it generates on a computation point located at 500 meters above its uppermost
surface.

Define the prism and its density (in kg per cubic meters):

.. jupyter-execute::

   prism = (-100, 100, -100, 100, -1000, 500)
   density = 3000

Define a computation point located 500 meters above the *top* surface of the
prism:

.. jupyter-execute::

   coordinates = (0, 0, 1000)

And finally, compute the gravitational potential field it generates on the
computation point:

.. jupyter-execute::

   potential = hm.prism_gravity(coordinates, prism, density, field="potential")
   print(potential, "J/kg")


Passing multiple prisms
^^^^^^^^^^^^^^^^^^^^^^^

We can compute the gravitational field of a set of prisms by passing a list of
them, where each prism is defined as mentioned before, and then making
a single call of the :func:`harmonica.prism_gravity` function.

Lets define a set of four prisms, along with their densities:

.. jupyter-execute::

   prisms = [
       [2e3, 3e3, 2e3, 3e3, -10e3, -1e3],
       [3e3, 4e3, 7e3, 8e3, -9e3, -1e3],
       [7e3, 8e3, 1e3, 2e3, -7e3, -1e3],
       [8e3, 9e3, 6e3, 7e3, -8e3, -1e3],
   ]
   densities = [2670, 3300, 2900, 2980]

We can define a set of computation points located on a regular grid at zero
height:

.. jupyter-execute::

   import verde as vd

   coordinates = vd.grid_coordinates(
       region=(0, 10e3, 0, 10e3), shape=(40, 40), extra_coords=0
   )

And finally calculate the vertical component of the gravitational acceleration
generated by the whole set of prisms on every computation point:

.. jupyter-execute::

   g_z = hm.prism_gravity(coordinates, prisms, densities, field="g_z")

.. note::

   When passing multiple prisms and coordinates to
   :func:`harmonica.prism_gravity` we calculate the field in parallel using
   multiple CPUs, speeding up the computation.

Lets plot this gravitational field:

.. jupyter-execute::

   import matplotlib.pyplot as plt

   plt.pcolormesh(coordinates[0], coordinates[1], g_z)
   plt.colorbar(label="mGal")
   plt.gca().set_aspect("equal")
   plt.xlabel("easting [m]")
   plt.ylabel("northing [m]")
   plt.show()


.. _prism_layer:

Prism layer
^^^^^^^^^^^

One of the most common usage of prisms is to model geologic structures.
Harmonica offers the possibility to define a layer of prisms through the
:func:`harmonica.prism_layer` function: a regular grid of
prisms of equal size along the horizontal dimensions and with variable top and
bottom boundaries.
It returns a :class:`xarray.Dataset` with the coordinates of the centers of the
prisms and their corresponding physical properties.

The :class:`harmonica.DatasetAccessorPrismsLayer` Dataset accessor can be used
to obtain some properties of the layer like its shape and size or the
boundaries of any prism in the layer.
Moreover, we can use the :meth:`harmonica.DatasetAcessorPrismsLayer.gravity`
method to compute the gravitational field of the prism layer on any set of
computation points.

Lets create a simple prism layer, whose lowermost boundaries will be set on
zero and their uppermost boundary will approximate a sinusoidal function.
We can start by setting the region of the layer and the horizontal dimensions
of the prisms:

.. jupyter-execute::

   region = (0, 100e3, -40e3, 40e3)
   spacing = 2000

Then we can define a regular grid where the centers of the prisms will fall:

.. jupyter-execute::

   easting, northing = vd.grid_coordinates(region=region, spacing=spacing)

We need to define a 2D array for the uppermost *surface* of the layer. We will
sample a trigonometric function for this simple example:

.. jupyter-execute::

   import numpy as np

   wavelength = 24 * spacing
   surface = np.abs(np.sin(easting * 2 * np.pi / wavelength))

Lets assign the same density to each prism through a 2d array with the same
value: 2700 kg per cubic meter.

.. jupyter-execute::

   density = np.full_like(surface, 2700)

Now we can define the prism layer specifying the reference level to zero:

.. jupyter-execute::

   prisms = hm.prism_layer(
       coordinates=(easting, northing),
       surface=surface,
       reference=0,
       properties={"density": density},
   )

Lets define a grid of observation points at 1 km above the zeroth height:

.. jupyter-execute::

   region_pad = vd.pad_region(region, 10e3)
   coordinates = vd.grid_coordinates(
       region_pad, spacing=spacing, extra_coords=1e3
   )


And compute the gravitational field generated by the prism layer on them:

.. jupyter-execute::

   gravity = prisms.prism_layer.gravity(coordinates, field="g_z")

Finally, lets plot the gravitational field:

.. jupyter-execute::

   plt.pcolormesh(*coordinates[:2], gravity)
   plt.colorbar(label="mGal", shrink=0.8)
   plt.gca().set_aspect("equal")
   plt.ticklabel_format(axis="both", style="sci", scilimits=(0, 0))
   plt.title("Gravity acceleration of a layer of prisms")
   plt.xlabel("easting [m]")
   plt.ylabel("northing [m]")
   plt.tight_layout()
   plt.show()
