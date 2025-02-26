# GenSPGL.jl

A Julia solver for large scale minimization problems using any provided norm.
GenSPGL supports implicit arrays(JOLI), explicit arrays, and functions as modelling
operators (**__A__**).

This code is an adaptation of Michael P. Friedlander, Ewout van den Berg, and
Aleksandr Aravkin's MATLAB program [SPGL1](http://www.cs.ubc.ca/~mpf/spgl1/).

Please note that while it is largely based off of SPGL1, this package is NOT a direct port of SPGL1. Various changes have been made to generalize the solver and make it more applicable to SLIM's research on large datasets. 

Noteable changes include: 
* **__A__** can be non-linear
* L1 and L2 projections
* Nuclear or L1 norms and their corresponding dual norms

## Installation

GenSPGL requires the JOLI package. If it isn't already installed run the following from the Julia REPL:

    Pkg.add(PackageSpec(url="https://github.com/slimgroup/JOLI.jl.git",rev="master"))

GenSPGL can be installed using the Julia package manager.
If you have a Github account and installed SSH keys, run the following from the Julia REPL:

    Pkg.add(PackageSpec(url="git@github.com:slimgroup/GenSPGL.jl.git",rev="master"))

Otherwise run: 

    Pkg.add(PackageSpec(url="https://github.com/slimgroup/GenSPGL.jl.git",rev="master"))

## Examples

The scripts for the following examples have been included in the package in the package directory. 
If you don't know the package directory, in the Julia REPL run

    import GenSPGL; joinpath(dirname(pathof(GenSPGL)), "..")

All paths in the rest of this document and relative to the package directory.

### Implicit Array Modelling Operators
**/src/Examples/pres/jl_cs.jl**

The script solves a small compressive sensing problem using Basis Pursuit Denoise. Data is generated by applying a restriction operator, **R**, and the DCT as a sparsifying transform, **S**, on the sparse vector, **x**, such that:

__RSx__ = __b__

To run the example start a Julia instance, load the GenSPGL module, and run the example by entering the following in the Julia REPL.

    julia> using GenSPGL
    julia> x, r, g, info, SNR = ex_cs()

`ex_cs` computes the following problem.

    """
    An example of using implicit JOLI operators to solve a system using GenSPGL
    Use: x, r, g, info, SNR = jl_cs()
    """
    function jl_cs()
        n = 1024
        k = 200

        # Create a system
        p = randperm(n)
        x0 = zeros(n)
        x0[p[1:k]] = sign.(randn(k))

        # Create a modelling op
        F = joDFT(n)
    
        # Create a restriction op
        ind = randperm(n)
        ind = ind[1:Int(floor(0.6*n))]
        R = joRestriction(n,ind)

        # Create data
        b = R*F*x0

        # Solve
        opts = spgOptions(optTol = 1e-4,
                         verbosity = 1)

        @time x, r, g, info = spgl1(R*F, vec(b), tau = 0., sigma = 1e-3, options = opts) 

        # Calc SNR of recovery
        SNR = -20*log10(norm(x0-x)/norm(x0));
   
        return x, r, g, info, SNR
    end
    
For additional information of the ouput, see the package documentation.

    help?> spgl1

A MATLAB version of the script is provided to compare accuracy and performance. 

### Factorization Based Rank Minimization 
**/src/Examples/compare/jl_complex.jl**

This example loads a subsampled frequency slice from a seismic data set, and recovers it using factorization based rank-minimization framework.

GenSPGL supports function handles as modelling operators given in place of explicit/implicit arrays. In addition to this, the package supports applying any norm on **x** by allowing users to define their own projection, dual norm, and primal norm functions. This can easily be done by creating an spgOptions type and changing the default options. Any unspecified parameters will be set to their default values, and if no spgOptions type is given to spgl1 the default options will be used. 

In this example user defined projection, dual norm, and primal norm functions are used instead of the defaults.

    opts = spgOptions(project     = TraceNorm_project,
                      primal_norm = TraceNorm_primal,
                      dual_norm   = TraceNorm_dual)



To run the example, if you have not already done so, load GenSPGL

    julia> using GenSPGL

And call the example function

    julia> x, r, g, info = jl_complex()
        
In addition to returning the output, this function will write the results into the example directory as "xLS_jl.mat". 
        
**/GenSPGL/src/Examples/compare/mat_complex.m** solves the same problem using the MATLAB implementation of the code, and is provided as a means of comparison. 
            
## Performance Notes

* GenSPGL is currently in development, so the module has not been precompiled. Measuring the perfomance of spgl1 the first time it is called is really measuring the perfomance of Julia's JIT compiler. To get a true sense of the packages perfomance, profile the code after it has been ran atleast once.

* Julia calls Open BLAS for linear aglebra routines, and defaults to 16 threads. Each machine should be tested, but we have found that, in general, setting the number of threads to the number of memory channels gives the best performance. Open BLAS threads can be set using:  

        julia> BLAS.set_num_threads()
                                                                                                
                                                                                                



