=======================================
How To Write An LLVM Register Allocator
=======================================

Introduction
============

This document is meant to be a quick start for developers who want to write a register allocator using the LLVM infrastructure.

In this tutorial, will be shown the method of writing a LLVM register allocator using the interface ``RegAllocBase``, which is the simplest way, for those who want to create a allocator without this interface, they can find this tutorial helpful in some way also.

It's assumed that you already know what is a pass in the LLVM infrastructure, in the case you don't know, it's highly recommended to look `Writing an LLVM Pass <http://www.llvm.org/docs/WritingAnLLVMPass.html>`_ and of course, basics concepts for register allocation, intermediate representation of programs in compilers, SSA-form, liveness analysis and basic knowledge on computer organization and architecture.

Register Allocation
===================

Register Allocation is executed during the Code Generation phase and consists in mapping a program with a unbounded number of virtual registers (like in the LLVM IR) to a program that contains a bounded (possibly small) number of physical registers of some particular architecture. Each target architecture has a different number of physical registers. If the number of physical registers is not enough to accommodate all the virtual registers, some of them will have to be mapped into memory. These virtuals are called spilled virtuals. For more information about how the LLVM infrastructure represents registers, take a look at the `Register Allocation Section <http://www.llvm.org/docs/CodeGenerator.html#register-allocator>`_ in the LLVM docs.

The LLVM infrastructure provides a series of classes and tools to make the process of writing a register allocator more easier.

Writing An Register Allocator on LLVM
=====================================

In this section, will be shown the basic classes that are related to register allocation and how to write a register allocation extending the ``RegAllocBase`` interface.

Virtual Register and the SSA-form
---------------------------------

The virtual registers in the LLVM are represented by the ``LiveInterval`` class and each virtual register represents a unique variable with one single definition, this fact is due to the strict SSA-form, used to represent the LLVM IR. The SSA-form is destructed before the register allocation pass, but the Liveness Analysis is maintained in the SSA-form.

If you want to get all the virtual registers, you can use the following code:

.. code-block:: c++

   for (unsigned i = 0, e = MRI->getNumVirtRegs(); i != e; ++i) {
    // reg ID
    unsigned Reg = TargetRegisterInfo::index2VirtReg(i);
    // if is not a DEBUG register
    if (MRI->reg_nodbg_empty(Reg))
      continue;
    // get the respective LiveInterval
    LiveInterval *VirtReg = &LIS->getInterval(Reg);
  }

Where ``MRI`` corresponds to an instance of the ``MachineRegisterInfo`` class (contains information about virtual registers) and ``LIS`` corresponds to the Liveness Analysis pass, represented by the class ``LiveIntervals``.

This procedure is already performed previously in the ``RegAllocBase`` interface, by the method ``seedLiveRegs``.

Irregularities in Architectures
-------------------------------

Some architectures are not regular (such as x86) and their registers can be represented in different ways. The registers ``AH``, ``AX`` and ``EAX`` share the same physical location, but they have different sizes. The LLVM deals with this by representing physical registers as register units, where each unit is an alias.

You can traverse through this units, given a physical register ``PhysReg`` and a ninstance of the class ``TargetRegisterInfo`` named ``TRI``, using the following code:

.. code-block:: c++

   for (MCRegUnitIterator Units(PhysReg, TRI); Units.isValid(); ++Units) {
   	// do something
   }

Other irregularity is pre-coloring, which consist in some physical registers of the architecture that are reserved for some operations like parameter passing, return values and others. One way to handle such registers is using the ``LiveRegMatrix`` class, which provides mechanisms to check if some physical register is reserved when you check for a interference, if it is reserved, the kind of interference returned will be ``IK_RegUnit``. Other way is to call the ``freezeReservedRegs`` method of the ``MachineRegisterInfo`` class before the register allocation, this method will make the reserved physical register unaccessible during register allocation.

Interference Graph
------------------

Some register allocators uses the graph coloring approach, to do this they need a structure called Interference Graph, usually the nodes of such graphs are variables and each edge represents a interference between two or more variables, i.e., two or more variables which live the same time. To build a interference graph you'll need to do the following steps:

* Add a node for each register unit and pre-color it.
* Add a node for each virtual register.
* Check for interferences between an virtual register and the others virtual registers, also between an virtual register and register units, if a interference exists, add an edge. This interference can be checked through the ``overlaps`` method of the ``LiveInterval`` class.

This approach is to use the classical representation of the interference graph, LLVM provides a class named ``LiveRegMatrix`` that already performs a similar function. The difference is that this structure will check for interferences on-the-fly between a virtual register and the virtual registers assigned to some physical register. The ``LiveRegMatrix`` has four types of interferences (see the `LiveRegMatrix source <http://www.llvm.org/doxygen/LiveRegMatrix_8cpp_source.html>`_ for reference).

To check the interference between a virtual register ``VirtReg`` and some physical register ``PhysReg``, you can use an instance of the ``LiveRegMatrix`` named ``Matrix`` and also the ``AllocationOrder`` class, which provides a order of available physical registers that best fit for some virtual register. The arguments passed to the ``AllocationOrder`` constructor are an integer identifier of the virtual register, an instance of the ``VirtRegMap`` class and an instance of the ``RegisterClassInfo``.

.. code-block:: c++
   
   AllocationOrder Order(VirtReg->reg, *VRM, RegClassInfo);
   while (unsigned PhysReg = Order.next()) {
    // Check for interference in PhysReg
    switch (Matrix->checkInterference(*VirtReg, PhysReg)) {
    case LiveRegMatrix::IK_Free:
      // do something
      continue;

    case LiveRegMatrix::IK_VirtReg:
      // do something
      continue;

    default:
      // do something
      continue;
    }
   }

The ``LiveRegMatrix`` can also be used to collect all interferences of some virtual register ``VirtReg`` through the ``query`` method:

.. code-block:: c++

   // Collect interferences assigned to any alias of the physical register.
   for (MCRegUnitIterator Units(PhysReg, TRI); Units.isValid(); ++Units) {
    // build a query
    LiveIntervalUnion::Query &Q = Matrix->query(VirtReg, *Units);
    // collect all interfering virtual registers assigned to PhysReg
    Q.collectInterferingVRegs();
    // if some of the interferences cannot be spilled
    if (Q.seenUnspillableVReg()) {
     // do something
    }
    // iterate through interferences
    for (unsigned i=0, e=Q.interferingVRegs().size(); i != e ; ++i) {
     LiveInterval *Intf = Q.interferingVRegs()[i];
     // do something
    }
   }

The The ``LiveRegMatrix`` also provides methods for assign virtual registers to physical registers and unassign virtual registers from physical registers.

Spill
-----

To apply spill to an virtual register, the class ``InlineSpiller`` can be used, this class implements the ``Spiller`` interface. The method used to apply spill to a virtual register is named ``spill`` and takes as parameter a instance of the class ``LiveRangeEdit``. An instance of the ``LiveRangeEdit`` class has to be created each time the allocator decide to apply spill or split some virtual register, in order to create a new virtual register and preserves the original. The ``LiveRangeEdit`` constructor takes as parameters: the virtual register that will be modified, an array to insert splitted virtual registers, an pointer to the current function, an pointer to the Liveness Analysis and an instance of the ``VirtRegMap`` class.

.. code-block:: c++
   
   // Spill some virtual register
   LiveRangeEdit LRE(&VirtReg, SplitVRegs, *MF, *LIS, VRM);
   spiller().spill(LRE);

After spill has been inserted, the pass of Liveness Analysis is automatically called to update the ``LiveIntervals`` information.

The spill cost of each virtual register is already computed before the register allocation pass and it's stored in the ``weight`` attribute of the ``LiveInterval`` class, for more information see the `CalcSpillWights <http://llvm.org/doxygen/CalcSpillWeights_8h.html>`_ file.

Using the ``RegAllocBase`` Interface
------------------------------------

The files that correspond to the header (``RegAllocBase.h``) and implementation (``RegAllocBase.cpp``) of the ``RegAllocBase`` interface are in the ``llvm/lib/CodeGen`` directory.

The ``RegAllocBase.h`` provides the methods that need to be overridden in order to implement the logic of the register allocator and attributes that stores useful information to the register allocation pass.

Some of the attributes are the following, to get more deeper information about each one, you can access the `doxygen <http://llvm.org/doxygen/>`_ documentation of LLVM:

* **TRI**: ``TargetRegisterInfo`` instance, provides information about the register in the target architecture.
* **MRI**: ``MachineRegisterInfo`` instance, provides information about the virtual and physical registers.
* **VRM**: ``VirtRegMap`` instance, maps virtual register to physical registers and also to stack slots.
* **LIS**: ``LiveIntervals`` instance, provides information about the Liveness Analysis.
* **Matrix**: ``LiveRegMatrix`` instance, provides on-the-fly interference information and indirect assignment and unassignment of virtual registers.
* **RegClassInfo**: ``RegisterClassInfo`` instance, provides information about target register classes dynamically.

To implement the logic of the register allocator that have been designed, you'll need override the following methods.

The ``spiller`` Method
~~~~~~~~~~~~~~~~~~~~~~

This methods returns an instance of some class that implements the ``Spiller`` interface, like the ``InlineSpiller`` class.

Declaration
^^^^^^^^^^^

.. code-block:: c++
   
   /// Inline Spiller
   Spiller &spiller() override;

The ``enqueue`` Method
~~~~~~~~~~~~~~~~~~~~~~

This method dictates the logic to the insertion of new virtual registers in the structure that you are using to store them. This method is called in the ``seedLiveRegs`` method and at each spill insertion, in order to store the new LiveInterval that has been created.

Declaration
^^^^^^^^^^^

.. code-block:: c++
   
   /// Put a new VirtReg for later assignment
   void enqueue(LiveInterval *LI) override;

The ``dequeue`` Method
~~~~~~~~~~~~~~~~~~~~~~

This method dictates the order to the removal of virtual registers of the structure that you are using to store them, so that virtual register will be assigned then. This method is called in the ``allocatePhysRegs`` method until some virtual register have not has been assigned yet.

Declaration
^^^^^^^^^^^

.. code-block:: c++
   
   /// Select a VirtReg for assignment
   LiveInterval *dequeue() override;

The ``selectOrSplit`` Method
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This method implements the logic of the heuristic applied to the register allocator, at each call the method will return an available physical register for some virtual register or it will split it. The split is not mandatory, but if you apply, you'll have to append the splitted virtual registers in the ``splitLRVs`` array. If you do not apply split, probably you'll have to spill the virtual register in the implementation of this method.

Declaration
^^^^^^^^^^^

.. code-block:: c++
   
   // Each call must guarantee forward progress by returning an available PhysReg or new set of split live virtual registers.
   // It is up to the splitter to converge quickly toward fully spilled live ranges.
   unsigned selectOrSplit(LiveInterval &VirtReg,
                         SmallVectorImpl<unsigned> &splitLRVs) override;

The ``aboutToRemoveInterval`` Method
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This method will be called before some the removal of some virtual register (i.e. ``LiveInterval`` instance). In this method you'll update some property or data struct that depends on information about the respective virtual register.

Declaration
^^^^^^^^^^^

.. code-block:: c++
   
   /// Method called when the allocator is about to remove a LiveInterval.
   void aboutToRemoveInterval(LiveInterval &LI) override;

Built in Methods
~~~~~~~~~~~~~~~~

Besides the methods that need to be overridden, the ``RegAllocBase`` interface has some methods that already have a implementation in the ``RegAllocBase.cpp`` file, this methods are the following ones.

The ``init`` Method
^^^^^^^^^^^^^^^^^^^

This methods initiates the attributes of the interface, makes the reserved register inaccessible and collects dynamic information about the register classes of the target architecture. The method has the following signature:

.. code-block:: c++

  // A RegAlloc pass should call this before allocatePhysRegs.
  void init(VirtRegMap &vrm, LiveIntervals &lis, LiveRegMatrix &mat);

The ``seedLiveRegs`` Method
^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method collects all the available virtual registers before the register allocation and stores them through the ``enqueue`` method. It's a private method and has the following signature:

.. code-block:: c++

  void seedLiveRegs();

The ``allocatePhysRegs()`` Method
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method is responsible for perform the allocation, while the ``dequeue`` returns a valid virtual register, the assignment to a physical register will be performed, in the case that a available physical register has not been found, the method will continue to the next iteration or will call ``enqueue``, in the case where live interval splitting has been applied. The method has the following signature:

.. code-block:: c++

  // The top-level driver. The output is a VirtRegMap that us updated with
  // physical register assignments.
  void allocatePhysRegs();

Once you have created the logic of your register allocator, you'll need to register him in the LLVM ``PassManager``, this can be done using an `existing pass registry for a new register allocator <http://www.llvm.org/docs/WritingAnLLVMPass.html#using-existing-registries>`_.

It's worth to remember that you have to inherit from an ``MachineXXXXPass`` also, where ``XXXX`` depends on the scope that you'll work with your register allocator, like ``Module``, ``Function`` or ``BasicBlock``.

Once you have registered your register allocation pass, you can use him in tools like ``llc``:

.. code-block:: console
   
   $ llc -help
     ...
     -regalloc                    - Register allocator to use (default=linearscan)
     =linearscan                -   linear scan register allocator
     =local                     -   local register allocator
     =simple                    -   simple register allocator
     =myregalloc                -   my register allocator help string
     ...

Tips
====

How To Start
------------

One good example to learn how the register allocation pass works on LLVM, using the ``RegAllocBase`` interface, is checking at the source code of the ``basic`` register allocator (``RegAllocBasic.cpp``) under the ``llvm/lib/CodeGen/`` directory.

This allocator have a simple implementation, which is great for those who wants a kickoff to write a register allocator using LLVM.

Statistics
----------

If you want to collect some statistics in your register allocation pass, you can use the LLVM `STATISTIC <http://www.llvm.org/docs/ProgrammersManual.html#Statistic>`_ macro.

Debug
-----

If you want to print ``debug`` messages in your register allocation pass, you can use the LLVM `DEBUG <http://www.llvm.org/docs/ProgrammersManual.html#the-debug-macro-and-debug-option>`_ macro.

Timing
------

The LLVM infrastructure automatically measures the runtime of passes executed during the compilation when you use the ``-time-passes`` option in tools like ``llc``, if you want the runtime of an specific region of your register allocator code you can use:

.. code-block:: c++
   
   NamedRegionTimer T("Code Region", TimerGroupName, TimePassesIsEnabled);

Where the ``TimeGroupName`` instance is accessible only if you use the ``RegAllocBase`` interface.

Command Line Options
--------------------

If you want to add some command line options to tweak your register allocator, you can use the `Command Line Library <http://www.llvm.org/docs/CommandLine.html>`_.