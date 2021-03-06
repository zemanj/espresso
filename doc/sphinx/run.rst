Running the simulation
======================

``integrate``: Running the simulation
-------------------------------------

To run the integrator use call the
:meth:`espressomd.integrate.Integrator.run` method of the
:class:`espressomd._system.integrator` instance::

    system.integrator.run(steps=number_of_steps)

where ``number_of_steps`` is the number of time steps the integrator
should perform.

|es| uses the Velocity Verlet algorithm for the integration of the equations
of motion.

Note that this implementation of the Velocity Verlet algorithm reuses
forces, that is, they are computed once in the middle of the time step,
but used twice, at the beginning and end. However, in the first time
step after setting up, there are no forces present yet. Therefore, has
to compute them before the first time step. That has two consequences:
first, random forces are redrawn, resulting in a narrower distribution
of the random forces, which we compensate by stretching. Second,
coupling forces of e.g. the Lattice Boltzmann fluid cannot be computed
and are therefore lacking in the first half time step. In order to
minimize these effects, has a quite conservative heuristics to decide
whether a change makes it necessary to recompute forces before the first
time step. Therefore, calling hundred times
:meth:`espressomd.integrate.Integrator.run` with ``steps=1`` does the
same as with ``steps=100``, apart from some small calling overhead.

However, for checkpointing, there is no way for to tell that the forces
that you read back in actually match the parameters that are set.
Therefore, would recompute the forces before the first time step, which
makes it essentially impossible to checkpoint LB simulations, where it
is vital to keep the coupling forces. To work around this, there is
an additional parameter, which tells integrate to not recalculate
the forces for the first time step, but use that the values still stored
with the particles. Use this only if you are absolutely sure that the
forces stored match your current setup!

The opposite problem occurs when timing interactions: In this case, one
would like to recompute the forces, despite the fact that they are
already correctly calculated. To this aim, the option can be used to
enforce force recalculation.

``minimize_energy``: Run steepest descent minimization
------------------------------------------------------

In Python the ``minimize_energy`` functionality can be imported from
:mod:`espressomd.minimize_energy` as class
:class:`espressomd.minimize_energy.MinimizeEnergy`. Alternatively it
is already part of the :class:`espressomd._system.System` class object
and can be called from there (second variant)::

    espressomd.minimize_energy.init(
        f_max = <double>,
        gamma = <double>,
        max_steps = <double>,
        max_displacement = <double>)
    espressomd.minimize_energy.minimize()

    system.minimize_energy.init(
        f_max = <double>,
        gamma = <double>,
        max_steps = <double>,
        max_displacement = <double>)
    system.minimize_energy.minimize()

This command runs a steepest descent energy minimization on the system.
Please note that the behaviour is undefined if either a thermostat,
Maggs electrostatics or Lattice-Boltzmann is activated. It runs a simple
steepest descent algorithm:

Iterate

.. math:: p_i = p_i + \min(\texttt{gamma} \times F_i, \texttt{max_displacement}),

while the maximal force is bigger than or for at most ``max_steps`` times. The energy
is relaxed by ``gamma``, while the change per coordinate per step is limited to ``max_displacement``.
The combination of and can be used to get an poor man’s adaptive update.
Rotational degrees of freedom are treated similarly: each particle is
rotated around an axis parallel to the torque acting on the particle.
Please be aware of the fact that this needs not to converge to a local
minimum in periodic boundary conditions. Translational and rotational
coordinates that are fixed using the ``fix`` command or the
``ROTATION_PER_PARTICLE`` feature are not altered.

``change_volume_and_rescale_particles``: Changing the box volume
----------------------------------------------------------------

This is implemented in
:meth:`espressomd._system.System.change_volume_and_rescale_particles`
with the parameters ``d_new`` for the new length and ``dir`` for the
coordinates to work on and ``"xyz"`` for isotropic.

Changes the volume of either a cubic simulation box to the new volume or
its given x-/y-/z-/xyz-extension to the new box-length, and
isotropically adjusts the particles coordinates as well. The function
returns the new volume of the deformed simulation box.

Stopping particles
------------------

To stop particles you can use the functionality implemented in the
:mod:`espressomd.galilei` module.  The corresponding class
:class:`espressomd.galilei.GalileiTransform` which is wrapped inside
the :class:`espressomd.system.System` instance as
:class:`espressomd._system.System.galilei` has two functions:

- :meth:`espressomd.galilei.GalileiTransform.kill_particle_motion`:
   halts all particles in the current simulation, setting their
   velocities to zero, as well as their angular momentum if the
   feature ``ROTATION`` has been compiled in.

- :meth:`espressomd.galilei.GalileiTransform.kill_particle_forces`:
   sets all forces on the particles to zero, as well as all torques if
   the feature ``ROTATION`` has been compiled in.

Multi-timestepping
------------------

Required feature: ``MULTI_TIMESTEP``

The multi-timestepping integrator allows to run two concurrent
integration time steps within a simulation, associating beads with
either the large :attr:`espressomd._system.System.time_step` or the
other :attr:`espressomd._system.System.smaller_time_step`. Setting
:attr:`espressomd._system.System.smaller_time_step` to a positive
value turns on the multi-timestepping algorithm. The ratio
:attr:`espressomd._system.System.time_step`/:attr:`espressomd._system.System.smaller_time_step`
*must* be an integer. Beads are by default associated with
:attr:`espressomd._system.System.time_step`, corresponding to the
particle property
:attr:`espressomd.particle_data.ParticleHandle.smaller_timestep` set
to 0. Setting to
:attr:`espressomd.particle_data.ParticleHandle.smaller_timestep` to 1
associates the particle to the
:attr:`espressomd._system.System.smaller_time_step` integration. The
integrator can be used in the NVE ensemble, as well as with the
Langevin thermostat and the modified Andersen barostat for NVT and NPT
simulations, respectively. See :cite:`bereau15` for more details.

Reaction Ensemble
-----------------

.. note:: Required features: REACTION\_ENSEMBLE

The reaction ensemble ::cite:`smith94a,turner2008simulation` allows to simulate
chemical reactions which can be represented by the general equation:

.. math::

   \mathrm{\nu_1 S_1 +\ \dots\  \nu_l S_l\ \rightleftharpoons\ \nu_m S_m +\ \dots\ \nu_z S_z } \,,
       \label{general-eq}

where :math:`\nu_i` is the stoichiometric coefficient of species
:math:`S_i`. By convention, stiochiometric coefficents of he
species on the left-hand side of Eq.[general-eq] (*reactants*) attain
negative values, and those on the right-hand side (*products*) attain
positive values, so that Eq.[general-eq] can be equivalently written as

.. math::

   \mathrm{\sum_i \nu_i S_i = 0} \,.
       \label{general-eq-sum}


The equilibrium constant of the reaction is then given as

.. math::

   K = \exp(-\Delta_{\mathrm{r}}G / k_B T)
       \quad\text{with}\quad
       \Delta_{\mathrm{r}}G^{\ominus} = \sum_i \nu_i \mu_i^{\ominus}\,.
       \label{Keq}


Here :math:`k_B` is the Boltzmann constant, :math:`T` is temperature,
:math:`\Delta_{\mathrm{r}}G^{\ominus}` standard Gibbs free energy change
of the reaction, and :math:`\mu_i^{\ominus}` the standard Chemical
potential of species :math:`i`. Note that thermodynamic equilibrium is
independent of the direction in which we write Eq.[general-eq]. If it is
written with left and righ-hand side swapped, the stoichiometric
coefficients and :math:`\Delta_{\mathrm{r}}G^{\ominus}` attain opposite
signs, and the equilibrium constant attains the inverse value. Further,
note that the equilibrium constant :math:`K` in Eq. [Keq] is the
dimensionless *thermodynamic* equilibrium constant :math:`K_\mathrm{p}`.
Apparent, concentration-based equilibrium constants can also be found in
literature. To be used as input for the reaction ensemble, they need to
be converted to thermodynamic constants as described in texbooks of
Physical Chemistry. As a special case, all stoichiometric coefficients
on one side of Eq.[general-eq] can be zero. Such reaction is equivalent
to exchange with a reservoir, and the simulation in reaction ensemble
becomes equivalent with the grandcanonical simulation. A simulation in
the reaction ensemble consists of two types of moves: the reaction move
and the configuration move. The configuration move is carried out by a
suitable molecular dynamics or a Monte Carlo scheme. The
``reacton_ensemble`` command of takes care only of the reaction moves.
In the *forward* reaction, the appropriate number of reactants (given by
:math:`\nu_i`) is removed from the system, and the concomitant number of
products is inserted into the system. In the *reverse* reaction,
reactants and products exchange their roles. The acceptance probability
:math:`P^{\xi}` for move from state :math:`o` to :math:`n` reaction
ensemble is given by the criterion ::cite:`smith94a`

.. math::

   P^{\xi} = \text{min}\biggl(1,V^{\bar\nu\xi}\Gamma^{\xi}e^{-\beta\Delta E_\mathrm{pot}}\prod_{i=1}\frac{N_i^0!}{(N_i^0+\nu_{i}\xi)!}
       \label{eq:Pacc}
       \biggr),

where :math:`\Delta E_\mathrm{pot}=E_\mathrm{pot new}-E_\mathrm{pot old}` is the interaction energy change,
:math:`\beta=1/k_\mathrm{B}T`, :math:`V` is the simulation box volume,
:math:`\bar\nu = \sum_i
\nu_i`. The extent of reaction, :math:`\xi=1` for the forward, and
:math:`\xi=-1` for the reverse direction. The parameter :math:`\Gamma`
is related to :math:`K` as

.. math::

   \Gamma = K \biggl(\frac{p^{\ominus}}{N_{\mathrm{A}}k_\mathrm{B}T}\biggr)^{\bar\nu},
         \label{eq:Gamma}

where :math:`N_{\mathrm{A}}` is the Avogadro number and
:math:`p^{\ominus}=1 atm` is the standard pressure. It is often
convenient and in some cases even necessary (dissociation reaction of
polyelectrolytes) that some particles representing reactants are not
removed from or placed at randomly in the system (think about reacting monomers in a polymer), but their identity is changed to that of the
products (and vice versa in the reverse direction). The replacement rule is that for any given reactant type it is replaced by the corresponding product type (corresponding means here in terms of order in the reaction equation that was provided) as long as the corresponding coefficients allow it.
For a description of the available methods see :mod:`espressomd.reaction_ensemble`


Wang-Landau Reaction Ensemble
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note:: Required features: REACTION_ENSEMBLE

Since you might be interested in thermodynamic properties of a reacting
system you may use the Wang-Landau algorithm ::cite:`wang01a`
to obtain them ::cite:`landsgesell16a`. Here the 1/t Wang-Landau
algorithm ::cite:`belardinelli07a` is implemented since it
does not suffer from systematic errors. Additionally to the above
commands for the reaction ensemble use the following commands for the
Wang-Landau reaction ensemble. For a description of the available methods see :mod:`espressomd.reaction_ensemble`:

Constant pH method
~~~~~~~~~~~~~~~~~~

.. note:: Required features: REACTION_ENSEMBLE

In the constant pH method due to Reed and Reed
::cite:`reed92a` it is possible to set the chemical potential
of :math:`H^{+}` ions, assuming the simulated system is coupled to an
infinite reservoir. This value is the used to simulate dissociation
equilibrium of acids and bases. Under certain conditions, the constant
pH method can yield equivalent results as the reaction ensemble. For
more information see ::cite:`landsgesell16b`. However, it
treats the chemical potential of :math:`H^{+}` ions and their actual
number in the simulation box as independent variables, which can lead to
serious artifacts. The constant pH method significantly differs in its
derivation compared to the Reaction Ensemble. The constant pH method can
be used by initializing the reactions of interest with the commands
mentioned for the reaction ensemble. However with the difference that
you do not provide the dimensionless reaction constant but directly the
*apparent reaction constant* (from the law of mass action) :math:`K_a`
which can in general carry a unit. For an example file for how to setup
a Constant pH simulation, see a file in the testcases. The following
commands for the constant pH method are available. For a description of the available methods see :mod:`espressomd.reaction_ensemble`:

Grand Canonical Ensemble
------------------------

Since the Reaction Ensemble acceptance transition probability can be
derived from the grand canonical acceptance transition probability we
can use the reaction ensemble to implement grand canonical simulation
moves. This is done via adding reactions that only have reactants (for the
deletion of particles) or only have products (for the creation of
particles). There exists a one to one mapping of the expressions in the
grand canonical transition probabilities and the expressions in the
reaction ensemble transition probabilities.

How to add the water autodissociation to a simulation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

With the above trick of grand canonical simulation moves include we can
include the autodissociation of water into the system. In order to add
the water autodissociation 


.. math::

   \mathrm{2 H_2O \rightleftharpoons\ H_3O^+ + OH^- } \,,


add the following ex nihilo reactions to Espresso. (:math:`\emptyset`, read ex
nihilo). Ex nihilo means that the reaction has no reactants or products.
Therefore, if :math:`\emptyset` is a product, particles vanish and if
:math:`\emptyset` is an reactant, then particles are created ex nihilo:

.. math::

   \mathrm{\emptyset \rightleftharpoons\ H_3O^+ + OH^- } \,, \text{ with reaction constant K}

.. math::

   \mathrm{H_3O^+ + OH^- \rightleftharpoons\ \emptyset} \,, \text{ with reaction constant 1/K},

where K is given implicitly as a function of the apparent dissociation
constant :math:`K_w=10^{-14} \rm{mol^2/l^2}=x\cdot \rm{1/(\sigma^3)^2}`:
:math:`K=(x\cdot \rm{1/(\sigma^3)^2})/(\beta P^0)^{\overline{\nu}}` with
:math:`\overline{\nu}=2` for the dissociation reaction and where x is
the value of the apparent dissociation constant that is converted from
:math:`\rm{mol^2/l^2}` to a number density in :math:`1/(\sigma^3)^2`,
where :math:`\sigma` is the simulation length unit. If :math:`\beta` and
:math:`P^0` are provided in simulation units this will make :math:`K`
dimensionless. As a test for the autodissociation of water a big
simulation box can be set up and the autodissociation reaction can be
performed. Then the box should fill with the correct number of protons
and hydroxide ions (check for the number of protons and hydroxide ions
in the given simulation volume and compare this to the expected value at
pH 7). Further the :math:`pK_w=14` should be reproduced -also in the
case of an initial excess of acid or base in the simulation box. Note
that this only works for big enough volumes.
