==============================================
FMS, An agent-based Financial Market Simulator
==============================================

FMS is (c) 2008-2012 Jean-Charles Bagneris. See LICENSE for redistribution
information and usual disclaimer.

Thanks for downloading FMS !

What is FMS ?
=============

FMS is an agent-based financial market simulator. The intended audience is
financial markets researchers and experimentators, looking to simulate various
agents behaviours on different types of markets through the resulting
transactions on a fictitious asset. Agents, markets and the environment (the
"world") are Python classes, derived from abstract ones provided with FMS.

As the resulting output (the transactions) is in comma separated values format,
it is easy to use it as an input for whichever processing needed (produce
graphics, import in spreadsheet, crunch in various statistical procedures, ...)

FMS is a command line application, configured through very simple flat files. As
you may write your own agents, markets, engines and world classes, it is as
customizable as it could be. But you do not have to be a programmer to use it:
FMS is provided with a comprehensive set of classes ready to use, and there is
more to come.

If you program your own classes, remember these may be of interest for others:
feel free to drop me an email and to contribute (see `How could I contribute ?`_
below).

What FMS is not
===============

FMS is a simulation tool intended for research only. Thus, FMS is NOT (and will
not be in the foreseeable future):

- a game,
- a portfolio management simulation or helper,
- a learning tool intended for classroom use [1]_,
- a shiny GUI application, providing with coloured graphics and multiple drop
  down menus, smileys and icons,
- a coffee-machine.

.. [1] Although FMS intention is not primarily pedagogical, it *might* be useful 
    in classroom environment with PhD students, for an example.

Rationale and history
=====================

FMS was primarily developed for my own research projects. The idea came from many
other agent based simulation programs, but the design was especially inspired by
Julien Derveeuw thesis (in french) : Derveeuw J., Simulation multi-agents de
marchés financiers, Université des Sciences et Technologies de Lille, 2008 (see
http://cisco.univ-lille1.fr/papers/ for more information). 

FMS is slightly different than Derveeuw's platform in some ways, mainly because
FMS can simulate a multi-stages market, e.g. with a pre-opening period where
orders accumulate, then a fixing, then a continuous order driven market,
starting with the remaining orders after fixing. In addition, FMS is fully
open-source (as far as I know, Derveeuw's platform is freely usable, but sources
are not available) and written in Python_, which (in my opinion) is easier to
learn than Java for researcher whose primary concern is finance, not computer
programming.

.. _Python: http://www.python.org/

Install or uninstall FMS
========================

See INSTALL file.

Quick start
===========

To use FMS, you first need to install it on your system (obviously). Follow the
INSTALL file instruction, and do not forget to run the tests once you are done.

Then, you should describe an experiment for FMS to run. Experiments are
described in config file in the YAML_ format, which is hopefully rather easy to
read and write. A minimal configuration file should contain three items with
their required parameters: world, engines/markets, agents.

.. _YAML: http://www.yaml.org/

World
-----

This is a "global environment" class, providing agents with so called exogenous
information on request. Such information might typically be the level of
interest rates, or energy price, for an example. A NullWorld class is provided
with FMS, it does not provide any special information.

Engines/markets
---------------

The engines/markets tuples describe what you would simply call "the market" in
the real world. Engines are the "traffic controlers" : they give speak to the
agents in a (simulated) synchronous or asynchronous manner, choosing which
agents speak and when at will. For an example, FMS provides with an
AsynchronousRandWReplace engine class, which is asynchronous (market clearing is
required as soon as an agent spoke) and chooses agents randomly, with
replacement. Markets basically are responsible for recording the orders, and
doing the clearing (for an example, auction style "fixing" clearing once in a
while, or continuous book based clearing). FMS provides with two basic market
classes, ContinuousOrderDriven and HighestQtyFixing.

Agents
------

Agents act when the engines give them speak. Acting is either do nothing, or
place an order. Order should at least have a direction (buy or sell) but may in
addition specify price and/or quantity. A ZeroIntelligenceTrader class is
provided: this agent takes fully random decisions.

Putting it all together
-----------------------

Once you have chosen or written your world, engines/markets and agents classes,
you describe those and their parameters in the experiment configuration file.
Examples are provided in the docs/examples directory. The yaml syntax is
available on http://www.yaml.org/. Try with one of the example configuration
files in ``docs/examples`` to begin with.

Choose one of the examples, cd to the config file directory and run::

	startfms.py -v check <config file name>

The ``check`` command is some sort of dry-run : it will perform anything except
running the experiment itself. Thus, it will try to find, import and instanciate
all the classes in your config file, which is probably the best way to check it.
The -v option is the verbose one, hopefully outputting clever error messages if
something went wrong. If it went ok, then run::

	startfms.py -v run <config file name>

This will really run the experiment, outputting transaction data either on the
console or in a comma separated value file, depending on your configuration
file.

What now ?
==========

If you read all this, then you certainly have a good reason to use FMS. If the
world, engines, markets and agent classes included in FMS do not meet your
needs, then you may either write yours, or even (politely) require us to write
it for you. Of course, your problem has to be interesting enough for us to do
this, and the resulting classes would be part of FMS next release. By the way,
if you write yourself an interesting class for FMS, please submit it for
inclusion (you would of course be credited for your work).

How could I contribute ?
========================

Report bugs, write new classes, translate documentation, write documentation and
additional example, request new features, watch or fork `the project`_, use FMS and
let people know you use it. Think of other ways to contribute. Thank you :)

.. _the project: http://github.com/jcbagneris/fms/
