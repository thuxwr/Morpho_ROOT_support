# Introduction

[Morpho](https://morpho.readthedocs.io/en/latest/index.html) provides an interface with [PyStan](https://pystan.readthedocs.io/en/latest/), which is the Python interface of [Stan](http://mc-stan.org/) and can be used for Bayesian statistical modelling and computation.
Stan is especially efficient in performing fits with a large number of parameters, while it requires an analytical form of probability distribution.
Since numerical calculation can be easily handled using [ROOT](https://root.cern), it will bring about great convenience if users can adopt ROOT functions in Stan models.
This work makes it possible to use external c++ functions (containing those from ROOT) in modelling with Stan via Morpho.

# Preparation

Since the current version of PyStan does not allow external functions in modelling, it is required to install the modified version of PyStan by executing:
```bash
    pip install git+https://github.com/thuxwr/pystan.git@root_supported
```

Then one needs to install the modified version of Morpho by:
```bash
    pip install git+https://github.com/thuxwr/morpho.git@root_supported
```

It is also required to install ROOT 6, with instructions on https://root.cern.ch/building-root. If ROOT environment is set properly, one should be able to execute:
```bash
    echo $ROOTSYS
```
and see his/her path to ROOT.

After installing all the above packages, one need to add ROOT headers in his/her PyStan global model header "model_header.hpp", which is located in
```bash
    path_to_pystan/stan/src/stan/model
```

For example, if TGraph class is used in one's Stan model, then "TGraph.h" should be included in "model_header.hpp". The "model_header.hpp" may look like this:
```c++
    #ifndef STAN_MODEL_MODEL_HEADER_HPP
    #define STAN_MODEL_MODEL_HEADER_HPP                                                                                       

    #include <stan/math.hpp>

    #include <stan/io/cmd_line.hpp>
    #include <stan/io/dump.hpp>
    #include <stan/io/program_reader.hpp>
    #include <stan/io/reader.hpp>
    #include <stan/io/writer.hpp>

    #include <stan/lang/rethrow_located.hpp>
    #include <stan/model/prob_grad.hpp>
    #include <stan/model/indexing.hpp>
    #include <stan/services/util/create_rng.hpp>

    #include <boost/exception/all.hpp>
    #include <boost/random/additive_combine.hpp>
    #include <boost/random/linear_congruential.hpp>

    #include <cmath>
    #include <cstddef>
    #include <fstream>
    #include <iostream>
    #include <sstream>
    #include <stdexcept>
    #include <utility>
    #include <vector>

    /* Add all ROOT headers here. */
    #include "TH1D.h"
    #include "TGraph.h"
    #include "TCanvas.h"
    #include "TF1.h"
    #include "TH2D.h"
    #include "TLegend.h"
    #include "TLatex.h"
    #include "TFile.h"
    
    #endif                                                           

```

# Use ROOT functions in model.stan
After finishing all the above preparations, it is possible to use external ROOT functions in the modelling. Reading the article on http://mc-stan.org/rstan/articles/external.html in advance is suggested. In **model_name.stan**, the structure is as follows:
```c++
    functions {
      type_name your_function(type_name variables); //In which type_name is real or int
    }
    
    model {
      /* use your_function */
    }
```
and in **header_name.hpp**, the following functions should be written:
```c++
    type_name your_function(type_name variable, std::ostream* pstream) { 
    //If type_name is real in model_name.stan, here type_name should be double; if "int" in model_name.stan, here "int".
      /* details */
      return given_type_of_value;
    }
    
    var your_function(const var& variable, std::ostream* pstream) {
      /* Calculate return value, and the first derivative. */
      /* several lines of calculation ...... */
      return var(new precomp_v_vari(return_value, variable.vi_, first_derivative));
    }
```
If the type of variable in user-defined function is **int**, it is not necessary to return first derivative to Stan, which means in **header_name.hpp**, one can simply write:
```c++
    var your_function(const int& variable, std::ostream* pstream) {
      return an_integer_value;
    }
```
If one wants to use a self-defined function with multiple real variables, he or she may refer to https://github.com/stan-dev/math/wiki/Adding-a-new-function-with-known-gradients.

It is noted that a template typename can be adopted to calculate the first derivative automatically. To do so, the **header_name.hpp** should be:
```c++
    template <typename T>
    typename boost::math::tools::promote_args<T>::type
    your_function(const T& variable, std::ostream* pstream) {
      return some_value;
    }
```
You can find such a header in the example provided below.

# Configure external functions in Morpho
There are several new parameters provided in Morpho config files, which are:
```python
    allow_undefined = False # If an external code is to be used, this parameter must be True.
    include_dirs = [<string>] # Include directories for all headers, except for the ROOT headers, which is loaded automatically.
    includes = [<string>] # User-defined headers.
```
These parameters should be used the same way as other parameters while configuring Morpho processors.

# An example 
An example of using ROOT to calculate numerical integration is provided in https://github.com/thuxwr/morpho/tree/root_supported/examples/integral. One can run the example by executing:
```bash
    python integral/scripts/morpho_integral.py
```

The model used in this example relies on a numerical probability density function (given by histogram in **models/model_input/spectrum.root**). 
Random sampling is done according to the numerical pdf, and all events above a fixed threshold are counted. 
By inputting the threshold and total counts above threshold, this example tries to fit the total times of random sampling. 
The **Integral()** function in **TH1D** class is adopted to calculate the integration above threshold, and the method of calculating the first derivative is given in **models/integral.hpp** for reference.
      
