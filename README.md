# Buoyant Jets Chackravarthy etal. JFM (2018)
This file shows an example `ff-bifbox` workflow for reproducing the results of the paper:
```tex

@article{Chakravarthy_2018, 
  title={Global stability of buoyant jets and plumes}, 
  volume={835}, 
  DOI={10.1017/jfm.2017.764}, 
  journal={Journal of Fluid Mechanics}, 
  author={Chakravarthy, R. V. K. and Lesshafft, L. and Huerre, P.}, 
  year={2018}, 
  pages={654–673}
} 
```
The commands below illustrate how to perform a weakly nonlinear analysis of the 2D incompressible flow around a cylinder and an open cavity using `ff-bifbox`.

## Setup environment for `ff-bifbox`
1. Navigate to the main `ff-bifbox` directory.
```sh
cd ~/your/path/to/ff-bifbox/
```
2. Export working directory and number of processors for easy reference.
```sh
export workdir=examples/BuoyantJets
export nproc=4
```
3. Create symbolic links for governing equations and solver settings.
```sh
ln -sf examples/BuoyantJets/eqns_chakravarthy_etal_2017.idp eqns.idp
ln -sf examples/BuoyantJets/settings_chakravarthy_etal_2017.idp settings.idp
```

## Build initial meshes
`ff-bifbox` uses FreeFEM for adaptive meshing during the solution process, but it needs an initial mesh to adaptively refine.
#### build initial mesh using GMSH
```sh
FreeFem++-mpi -v 0 importgmsh.md -gmshdir examples/BuoyantJets -dir $workdir -mi jet.geo
```

#### build initial mesh using BAMG 
'''sh
FreeFem++-mpi -v 0 examples/BuoyantJet/chakravarthy.md -mo $workdir/jet
'''

## Perform parallel computations using `ff-bifbox`
### Zeroth order
1. Compute base states on the created meshes at $Re=10$ from default guess
```sh
ff-mpirun -np $nproc basecompute.md -v 0 -dir $workdir -mi jet.msh -fo jetbase Re 200 Pr 0.7 Ri 10 S 7
```
//OLD BELOW HERE 
2. Continue base state along the parameter $1/Re$ with adaptive remeshing
```sh
ff-mpirun -np $nproc basecontinue.md -v 0 -dir $workdir -fi cylinder.base -fo cylinder -param 1/Re -h0 -1 -scount 2 -maxcount 8 -mo cylinder -thetamax 5
ff-mpirun -np $nproc basecontinue.md -v 0 -dir $workdir -fi cavity.base -fo cavity -param 1/Re -h0 -1 -scount 4 -maxcount 16 -mo cavity
```

3. Compute base states at $Re=50$ (cylinder) and $Re=4000$ (cavity) with guess from continuation
```sh
ff-mpirun -np $nproc basecompute.md -v 0 -dir $workdir -fi cylinder_8.base -fo cylinder50 -1/Re 0.021
ff-mpirun -np $nproc basecompute.md -v 0 -dir $workdir -fi cavity_16.base -fo cavity4000 -1/Re 0.00025
```

### First & second order
1. Compute leading direct eigenmode at $Re=50$ (cylinder) and $Re=4000$ (cavity)
```sh
ff-mpirun -np $nproc modecompute.md -v 0 -dir $workdir -fi cylinder50.base -fo cylinder50 -eps_target 0.1+0.8i -sym 1 -eps_pos_gen_non_hermitian
ff-mpirun -np $nproc modecompute.md -v 0 -dir $workdir -fi cavity4000.base -fo cavity4000 -eps_target 0.1+8.0i -sym 0 -eps_pos_gen_non_hermitian
```
NOTE: Here, the `-sym` argument specifies the asymmetric (1) or symmetric (0) reflective symmetry across the boundary `BCaxis`.

2. Compute the critical point and critical base/direct/adjoint solution
```sh
ff-mpirun -np $nproc hopfcompute.md -v 0 -dir $workdir -fi cylinder50.mode -fo cylinder -param 1/Re -nf 0
ff-mpirun -np $nproc hopfcompute.md -v 0 -dir $workdir -fi cavity4000.mode -fo cavity -param 1/Re -nf 0
```

3. Adapt the mesh to the critical solution, save `.vtu` files for Paraview
```sh
ff-mpirun -np $nproc hopfcompute.md -v 0 -dir $workdir -fi cylinder.hopf -fo cylinderadapt -mo cylinderhopf -adaptto bda -param 1/Re -thetamax 5 -pv 1 -wnl 1 
ff-mpirun -np $nproc hopfcompute.md -v 0 -dir $workdir -fi cavity.hopf -fo cavityadapt -mo cavityhopf -adaptto bda -param 1/Re -pv 1 -wnl 1
```
NOTE: the normalizations for the direct and adjoint eigenmodes (and therefore also the weakly-nonlinear corrections) used by `ff-bifbox` are different than the normalizations used by Sipp and Lebedev. This causes the results to differ by a complex scaling factor.


### Harmonic Balance
1. Continue periodic orbit from initial Hopf bifurcations using 2nd-order Harmonic Balance (Caution: memory intensive!)
```sh
ff-mpirun -np $nproc porbcontinue.md -v 0 -dir $workdir -fi cylinder.hopf -fo cylinderNh2 -Nh 2 -mo cylinderporb -param 1/Re -thetamax 5 -h0 -1 -scount 5 -maxcount 10
ff-mpirun -np $nproc porbcontinue.md -v 0 -dir $workdir -fi cavity.hopf -fo cavityNh2 -Nh 2 -mo cavityporb -param 1/Re -h0 -1 -scount 4 -maxcount 8
```

2. Compute periodic orbits at $Re=50$ (cylinder) and $Re=5000$ (cavity) using 3rd-order Harmonic Balance with block Jacobi solver (Caution: memory intensive!)
```sh
ff-mpirun -np $nproc porbcompute.md -v 0 -dir $workdir -fi cylinderNh2_10.porb -fo cylinder50Nh3 -Nh 3 -1/Re 0.02 -blocks 3
ff-mpirun -np $nproc porbcompute.md -v 0 -dir $workdir -fi cavityNh2_8.porb -fo cavity5000Nh3 -Nh 3 -1/Re 0.0002 -blocks 3
```
NOTE: Sipp & Lebedev do not perform harmonic balance analysis. See Fabre et al., Appl. Mech. Rev. 2018 and Meliga, JFM, 2017 for reference results from the cylinder and cavity geometries, respectively.


### Time-domain nonlinear simulation
1. Compute time series over first 10 time units for cavity case after step increase in Reynolds number, adapt mesh to solution, export files to Paraview. 
```sh
ff-mpirun -np $nproc tdnscompute.md -v 0 -dir $workdir -fi cavity4000.base -fo cavity -1/Re 0.0002 -tsdt 0.01 -mo cavitytimeseries -scount 5 -maxcount 1000 -pv 1
```