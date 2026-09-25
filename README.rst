
Demo 3: Moving Fins
===================

The following example demonstrates how to use pyCart and Cart3D to analyze a
configuration with moving components.  The example is similar to
:ref:`pycart-ex-arrow`, except that the fins go inside the body of the bullet.

    .. figure:: fins01.png
        :width: 4in
        
        Example showing fin 4 deflected 15 degrees after intersecting with body

This allows for the fins to be rotated about some hinge axis, and then once the
fins are in the correct position, Cart3D's ``intersect`` tool can be used to
transform this self-intersecting surface into a single water-tight geometry.


To get started, clone this repo and run the following easy commands:

    .. code-block:: console

        $ git clone https://github.com/nasa-ddalle/pycart03-fins.git
        $ cd pycart03-fins
        $ ./copy-files.py
        $ cd work/

This will copy all of the files into a newly created ``work/`` folder. Follow
the instructions below by entering that ``work/`` folder; the purpose is that
you can easily delete the ``work/`` folder and restart the tutorial at any
time.

Let's run a case to see how this works.

    .. code-block:: none
    
        $ pycart -f fins.json -I 7
        Case Config/Run Directory       Status  Iterations  Que CPU Time
        ---- -------------------------- ------- ----------- --- --------
        7    d2+05_d4-15/m1.75a1.0r30.0 ---     /           .
          Group name: 'd2+05_d4-15' (index 4)
          Preparing surface triangulation...
          Reading tri file(s):
            inputs/bullet.tri
            inputs/fin1.tri
            inputs/fin2.tri
            inputs/fin3.tri
            inputs/fin4.tri
         > autoInputs -r 8 -t Components.c.tri -maxR 10 -nDiv 4
             (PWD = 'd2+05_d4-15/')
             (STDOUT = 'autoInputs.out')
             (STDERR = 'autoInputs.out')
         > intersect -i Components.tri -o Components.o.tri
             (PWD = 'd2+05_d4-15/')
             (STDOUT = 'intersect.out')
             (STDERR = 'intersect.out')
            Using PyVista for cleanup of 'Components.o.tri', tol=1.0e-06
            Mapping CompIDs for intersected Components.o.tri -> Components.i.tri
        Using template for 'input.cntl' file
             Starting case 'd2+05_d4-15/m1.75a1.0r30.0'
         > flowCart -his -clic -N 200 -y_is_spanwise -limiter 2 -T -cfl 1.1 -mg 3 -no_fmg -binaryIO -tm 0
             (PWD = 'd2+05_d4-15/m1.75a1.0r30.0/')
             (STDOUT = 'flowCart.out')
             (STDERR = 'flowCart.err')

        Submitted or ran 1 job(s).

        ---=1,

The output is pretty similar to a case without the fin deflections, but there
are a few differences that are helpful to explain. The first is the name of the
case, ``d2+05_d4-15/m1.75a1.0r30.0``, which is especially different in that the
name of the group folder contains the fin deflections. That is all controlled in
the pyCart input file ``fins.json``, and we will discuss it shortly. This
example is configured with the fin deflections in the folder name because each
set of cases with same fin positions can use the same mesh, and keys marked
``"Group": true`` are automatically folded into the group folder name.

The next difference is that pyCart builds the mesh from a series of
triangulation files: ``Components.tri`` and ``Components.c.tri`` on input, and
the final ``Components.i.tri`` on output.  The reason for the input pair is that
``intersect`` requires each body to have a single component ID, which destroys
the surface component naming that is defined in our inputs (like splitting off
the nose cap, body, and base of the bullet shape into separate components).  So
``Components.tri`` has only five components (the bullet shape and one for
each fin) while ``Components.c.tri`` has seven components.

Then ``intersect`` is run with the command shown above, which generates
``Components.o.tri``. This file also has only five component IDs, and these
are mapped back into the original component ID numbering by comparing to
``Components.c.tri`` to generate the final triangulation
``Components.i.tri`` with its seven component IDs.  CAPE now also cleans up
the raw ``intersect`` output with PyVista (eliminating tiny slivers and other
artifacts; the cleaned file is written as ``Components.u.tri``) before
mapping the component IDs.

Otherwise, the solution proceeds in the same manner as a non-intersecting case. 
Let's take a closer look at the ``"Mesh"`` and ``"RunMatrix"`` sections of the
pyCart input file ``fins.json`` to explain how this was set up.

    .. code-block:: javascript
    
        "Mesh": {
            // Surface triangulation
            "TriFile": [
                "inputs/bullet.tri",
                "inputs/fin1.tri",
                "inputs/fin2.tri",
                "inputs/fin3.tri",
                "inputs/fin4.tri"
            ]
        },
        
The ``"Mesh"`` section is relatively simple but contains a little bit more
information than the default section: *TriFile* is now a *list* of ``tri``
files rather than a single file. The individual water-tight volumes are
split into separate ``tri`` files, which provides pyCart two layers of
information about how to split up the surface. Each ``tri`` file may contain
multiple component IDs (in this case, only ``bullet.tri`` contains more
than one component ID), but each file should contain a single closed surface.
Then pyCart combines all these triangulations before intersecting them. The
``"intersect": true`` switch that turns on this process now lives in the
``"RunControl"`` section (not ``"Mesh"``); if it is not set to ``true``, using
multiple triangulation files has little effect.

    .. code-block:: javascript
    
        // RunMatrix (i.e. run matrix) description
        "RunMatrix": {
            // Global run matrix definitions
            "Keys": ["Mach", "alpha_t", "phi", "d2", "d4"],
            "File": "inputs/matrix.csv",
            "GroupMesh": true,
            // Customized key definitions
            "Definitions": {
                // Rotate fin 2
                "d2": {
                    "Group": true,
                    "Type": "rotation",
                    "CompID": "fin2",
                    "Vector": [[7.2,0,0], [7.2,-1,0]],
                    "Value": "float",
                    "Format": "%+03i_"
                },
                // Rotate fin 4
                "d4": {
                    "Group": true,
                    "Type": "rotation",
                    "CompID": "fin4",
                    "Vector": [[7.2,0,0], [7.2,1,0]],
                    "Value": "float",
                    "Format": "%+03i"
                }
            }
        }
        
The ``"RunMatrix"`` section, which defines the run matrix input variables, is
more interesting, so let's go through the settings one-by-one. The *Keys* input
sets the list of input variables. The first three are common variables for many
configurations and as such are automatically recognized by pyCart. The *File*
parameter simply points to a file that contains the values of each input
variable at which to run the configuration.

Setting *GroupMesh* to true tells pyCart that the run matrix can be split into
groups of cases such that each case in one group can use the same mesh. The
name of each group folder is then built from the keys whose definitions set
*Group* to ``true``---here the two fin-rotation keys *d2* and *d4*, giving a
group folder such as ``d2+05_d4-15``. (Earlier versions of this example used a
*GroupPrefix* setting to name the group folders; that option is now legacy and
is ignored, having been superseded by the *config* trajectory key and the
*Group* flag on individual keys.)

The last parameter, *Definitions*, is the interesting part of this example.
Because *Mach*, *alpha_t*, and *phi* are such common input variables (called
"trajectory keys" in CAPE terminology) that we can rely on the default
definitions.  (Default trajectory key definitions can be altered by editing the
file ``$CAPE/pycart/options/pyCart.default.json``.)  The other two parameters
are fin rotations, which require customization.

The trajectory key *d2* is set up to rotate fin #2. We set *Group* to ``true``
because cases with the same fin deflections can use the same mesh. The *Type*
is set to ``"rotation"``, which pyCart recognizes and reduces some of our work
in defining it here. We set *CompID* to ``"fin2"``, which tells pyCart to
rotate any triangles in the component defined as ``"fin2"`` in the
``Config.xml`` file. Then *Vector* gives a list of two points that define a
vector about which to rotate the points.

Finally, *Format* sets a ``printf`` style format string for how the value is
printed in the folder name.  It's set to integer in this example, which would
create problems for fin deflection angles like ``2.1``.  Anyway, this example
shows how to set up general component rotations very quickly.


