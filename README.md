# NASA-MEaSUREs-deep-clouds
Code base for constructing deep convective cloud database. Assembled by Eric Wilcox, Desert Research Institute, January 2022.
Code is designed to run specifically on NASA Discover computer.
Compile script is ccmeasures_clouds_v12.
Run script is measures_slurm_call_css.
Outside Discover, `conda env create -f environment.yml` installs the compilers, HDF4, netCDF and the Python packages from conda-forge; then set HDF_INCLUDE and HDF_LIBRARY in ccmeasures_clouds_v12 to $CONDA_PREFIX/include and $CONDA_PREFIX/lib.
