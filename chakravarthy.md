'''freefem
//
// chakravarthy.geo
// Gretchen Schulke
// gretchen.schulke@duke.edu
//
// This file can be used with FreeFem to create a mesh
//

assert(mpisize == 1); // Must be run with 1 processor
include "settings.idp"
real n0 = getARGV("-n0", 50);
real n1 = getARGV("-n1", 20);
real r = getARGV("-r", 1.0);

string meshout = getARGV("-mo", "chakravarthy.msh"); // mesh filename
if(meshout.rfind(".msh") < 0) meshout = meshout + ".msh"; // add extension if not provided

//  Define boarders 
//  o-----3------o
//  |            |
//  |            |
//  |            |
//  |            |
//  |            |
//  |            |
//  4            2
//  |            |
//  |            |
//  |            |
//  |            |
//  |            |
//  o------1-----o


border C01(t=0, 1){ x=30*r*t;     y=0;      label=BCinflow;  }
border C02(t=0, 1){ x=30*r;       y=80*r*t; label=BCwall;    }
border C03(t=0, 1){ x=30*r*(1-t); y=80*r;   label=BCoutlet; }
border C04(t=0, 1){ x=0;          y=80*r*(1-t); label=BCaxis;}

// Assemble mesh
mesh Thg = buildmesh(C01(n1) + C02(n0) + C03(n1) + C04(n0));

int[int] meshlabels = labels(Thg);
cout << "\tMesh: " << Thg.nv << " vertices, " << Thg.nt << " triangles, " << Thg.nbe << " boundary elements, " << meshlabels.n << " labeled boundaries." << endl;
cout << "  Saving mesh '" + meshout + "'." << endl;
savemesh(Thg, meshout);
'''