# Monte Carlo Experiment (MCE) 

Monte Carlo Experiment (MCE) is a format specification for *Monte Carlo Experiments*.
The current version of this specification is `0.0.1`.

## 1. Purpose and scope

An **MCE file** is an XML document that states *what* and *how* to simulate a model under *uncertainty* with *Monte Carlo* methods. It contains indications on simulation settings, which input signals are uncertain and how they are distributed, which signals to record, and how to sample. It contains no results.

The format is the contract between programs.

1) A tool is used to configure the experiment writing the MCE file;
2) A tool is used to run the preconfigured *Monte Carlo* experiment;
3) A tool is used to analyse the results.

The format of models in scope are *Modelica*, *FMI 2.0*, *FMI 3.0* and *SSP* systems. However, this can be extended to further models format.

The terms defined in this document are reported below.

- A **parameter** is a model input that the experiment sets.
- A **signal** is a time series that the experiment records.
- A **run** is one simulation with one set of parameter values.

## 2. Structure

The structure of the **MCE file** is presented below in the tree, where `?` means optional, `*` means zero or more, and `+` means one or more.

``` none
experiment                  name, schemaVersion
├─ environment              format (e.g., Modelica), version
│  ├─ model                 name, path
│  ├─ package*+             path (used for Modelica) 
│  ├─ parameterSet*         name (used for SSP)
|  └─ *+                    ...                      
├─ simulation               
│  ├─ runner                name
│  └─ setting*+             name, value
├─ design                   method, samples, seed, path?, start?
├─ recording?               start, interval, interpolation
├─ execution?               timeout, parallel (number of parallel simulations)
├─ syntax?                  separator (to access to internal hierarchical model) 
├─ parameters              
│  ├─ stochastic*+          name
│  │  ├─ distribution       name, distribution attributes
│  │  │  └─ coverage?       only for infinite set distributions 
│  │  └─ target+            scope (can have multiple targets)
│  └─ constant*+            name, value
│     └─ target+            scope (can have multiple targets)
├─ signals
│  └─ signal*+             name
│     └─ target            scope (only one target)
```

## 3. Elements reference

### 3.1 `<experiment>`

`<experiment>` is the root element. `name` labels the experiment.  `schemaVersion` is the XML schemaVersion (e.g., `1.0.0`).

### 3.2 `<environment>`

`<environment>` describes the environment in which each simulation run.

In the following table, we describe the basics attributes.

| Attribute | Description |
| --- | --- |
| `format` | Model format. Examples are `modelica`, `fmi`, `ssp`. |
| `version` | Version of the *format standard*. Examples are *Modelica* language version for a `.mo`, *FMI* version for an *FMU*, *SSP* version. |
| `model path` | Model path. |
| `model name` | Model name. |

The following table lists the attributes that depend on the *model format*.

| Model format | Attribute | Description |
| --- | --- | --- |
| `modelica` | `package path` | Path for a *Modelica* package. |
| `ssp` | `parameterSet name` | Name of an SSV parameter set stored in the `.ssp`, used as baseline values. |

### 3.3 `<simulation>`

`<simulation>` describes the simulation settings in which each simulation run.

`<runner name>` selects the runner that executes the model. `<setting name value>` entries set simulation settings from runner's own vocabulary.

| Runner name | Model format | Settings |
| --- | --- | --- |
| `OMPython` | `modelica` | `startTime`, `stopTime`, `stepSize`, `tolerance`, `solver`, ... |
| `FMPy` | `fmi` | `start_time`, `stop_time`, `step_size`, `relative_tolerance`, `solver`, ... |
| `OMSimulator` | `ssp`, `fmi` | `startTime`, `stopTime`, `tolerance`, ... |

### 3.4 `<design>`

`<design>` describes how the samples of the stochastic parameters are generated.

| Attribute | Description |
| --- | --- |
| `method` | Sampling method. Examples are `sobol`, `halton`, `lhs`, `random`, `corners`, `file`. |
| `samples` | Number of runs. |
| `seed` | Optional. Seed number. |
| `path` | Optional. Required when `method` = `file`. |
| `start` | Optional. Non-negative integer indicating the first run of the sequence. |

With `method` =`corners`, each stochastic parameter takes its lower and upper coverage bound. With `method` = `file`, `samples` must equal the number of data rows and the columns must match the stochastic parameter names exactly, in any order. With `method` = `sobol`, design whose sample count is not a power of two loses its balance properties. With `method` = `lhs`, designs depend on the sample count and cannot be extended by adding runs, therefore the use of `start` do not make sense.

### 3.5 `<recording>` - optional

`<recording>` is an optional element providing details on recording.

| Attribute | Description |
| --- | --- |
| `start` | First recorded time. |
| `interval` | Recording interval. |
| `interpolation` | Recording interpolation when `interval` is different from the interval of the runner. Examples are `linear` or `zoh`. |

Note that these attributes are usefull when the user need to save memory space.

### 3.6 `<execution>` - optional

`<execution>` is an optional element providing details on *Monte Carlo* execution.

| Attribute | Description |
| --- | --- |
| `timeout` | Maximum duration of one run, in seconds. |
| `parallel` | Optional. Positive integer indicating the number of parallel simulations. When it is not declared, the simulations are exectuted sequentially. |

### 3.7 `<syntax>` - optional

`<syntax>` is an optional element specifying details regarding syntax.

In particular, `separator` specify a character to access to model hierarchical level for `scope` defined in `<parameters>` Section and `<signals>` Section. More details can be found at Section 4.

### 3.8 `<parameters>`

`<parameters>` is an element which list all parameters to be tuned.

There are two kind of parameters.

- **`<constant name value>`** is a fixed parameter overriding the model's default value/values.
- **`<stochastic name>`** is a stochastic parameter with a `<distribution>` and its own attributes. It is a random variable being sampled at each *Monte Carlo* run. As the fixed parameters, it overrides the model's default value/values.

Every parameter has a unique `name` and one or more `<target scope="..."/>`.

- `name` is the identity of the input for every consumer. It becomes the column or dataset name in results and the variable name in analysis libraries, so it must be unique. No two parameters may share a target.
- `<target scope="..."/>` define the locations of the values to override.
  - A parameter with several targets applies the same value to several places. For example, in a Modelica model, a new value can be applied to `motor.R` and `generator.R`.
  - Every target must exist in the model and be settable.

#### 3.8.1 `<distribution>`

`<distribution>` is an element which defines a distribution with its own attributes.  

The table below reports basic examples of distributions.

| `name` | Attributes |
| --- | --- |
| `uniform` | `min`, `max` |
| `normal` | `mean`, `sigma`, `<coverage>` |

Further distributions can be added as new name values with their own attributes.

Note that most of the `<design method>` samples values in the parameter-set $[0,1]^d$ where $d$ is the number of stochastic parameters, and then rescale the values using selected distributions.

A special attribute for infinite-set distribution is `<coverage>`. This attribute is needed to prevent samples with infinite coordinate. In fact, this attribute limits the selected distribution to a closed and limited set. For the normal distribution, it can be used in one of the following forms.

- `<coverage probability="P"/>`, where `P` is a value in $[0, 1]$ associated to the interval cutting the tails with that probability.
- `<coverage sigma="n"/>`, where `n` is positive value associate to the interval cutting the tails at $\pm nσ$.
- `<coverage lower="L" upper="U"/>`, where `L` and `U` are associated to the interval cutting the tails for values outside $[L, U]$ interval.

For a truncated distribution with cumulative function $F$ and coverage interval $[x_{min}, x_{max}]$, the value $u$ extracted in the uniform distribution with interval $[0,1]$, and it is mapped as

```latex
x = {F}^{-1}\left[ {F}(x_{min}) + u \cdot \left( {F}(x_{max}) - {F}(x_{min}) \right) \right].
```

This is also valid for distribution with closed and limited set with $[x_{min}, x_{max}]$ as interval.

### 3.9 `<signals>`

`<signals>` is an element which list all signals to be recorded.

Every signal has an unique `name` has exactly one `<target scope="..."/>`.

A parameter recorded as a signal is legal but constant.

## 4. Scope syntax

A **scope** is the full address of a variable inside the model. It is specified for each of the `<parameters>` and `<signals>`.

In *Modelica* `model format,` the scope is full specified with its own syntax. The user can access to hierarchical internal models and variables using the `.` character. This can be used to access to multiple level of hierarchy. Examples are `testbench.plant.motor.mass`, `testbench.plant.k[2]`.

In other model format, the access to hierarchical is not guaranteed by own syntax. For this reason, we need to define a character to access to a deeper hierarchical level. This is done with `syntax separator`, presented in Section 3.7.

As side note, the `.` character is not the right choice for every `model format`. For example, *SSP* `model format` may contain variable names with dots. For this reason, the user need to choose an alternative `separator` like `:`. Using the latter, a variable `myVariable`can be accessed as `myComponent:myVariable`.

## 5. Examples

In this Section, we present some examples of **MCE files**.

### 5.1 Minimal example with *FMI 2.0* format and *FMPy runner*

 ```xml
<experiment schemaVersion="1.0.0" name="VanDerPol">
  <environment format="fmi" version="2.0">
    <model path="models/VanDerPol.fmu"/>
  </environment>
  <simulation>
    <runner name="FMPy"/>
    <setting name="start_time" value="0"/>
    <setting name="stop_time" value="20"/>
  </simulation>
  <design method="sobol" samples="1024"/>
  <parameters>
    <stochastic name="mu">
      <distribution name="normal" mean="1" sigma="0.1">
        <coverage sigma="3"/>
      </distribution>
      <target scope="mu"/>
    </stochastic>
  </parameters>
  <signals>
    <signal name="x0"><target scope="x0"/></signal>
  </signals>
</experiment>
```

### 5.2 Example with *Modelica* format and *OMPython* runner

```xml
<experiment schemaVersion="1.0.0" name="SpringMass">
  <environment format="modelica">
    <model path="./models/SpringMass.mo" name="SpringMass"/>
    <package path="./packages/mechanics/package.mo"/>
  </environment>
  <simulation>
    <runner name="OMPython"/>
    <setting name="startTime" value="0"/>
    <setting name="stopTime" value="10"/>
    <setting name="tolerance" value="1e-06"/>
    <setting name="stepSize" value="0.02"/>
    <setting name="solver" value="dassl"/>
  </simulation>
  <design method="halton" samples="50"/>
  <recording start="2" interval="0.02"/>
  <execution timeout="600"/>
  <parameters>
    <stochastic name="mass">
      <distribution name="uniform" min="8" max="12"/>
      <target scope="SpringMass.m"/>
    </stochastic>
    <stochastic name="damping">
      <distribution name="normal" mean="0.5" sigma="0.05">
        <coverage sigma="3"/>
      </distribution>
      <target scope="SpringMass.d"/>
    </stochastic>
    <stochastic name="stiff">
      <distribution name="normal" mean="100" sigma="5">
        <coverage lower="85" upper="120"/>
      </distribution>
      <target scope="SpringMass.k"/>
    </stochastic>
    <constant name="w_fixed" value="250">
      <target scope="SpringMass.w"/>
    </constant>
  </parameters>
  <signals>
    <signal name="position"><target scope="SpringMass.x"/></signal>
    <signal name="velocity"><target scope="SpringMass.v"/></signal>
  </signals>
</experiment>
```

### 5.3 Example with *SSP* format and *OMSimulator* runner

```xml
<experiment schemaVersion="1.0.0" name="engine">
  <environment format="ssp" version="2.0">
    <model path="models/engine.ssp"/>
    <parameterSet name="nominalParameters.ssv"/>
  </environment>
  <simulation>
    <runner name="OMSimulator"/>
    <setting name="startTime" value="0"/>
    <setting name="stopTime" value="10"/>
    <setting name="fixedStepSize" value="0.001"/>
  </simulation>
  <design method="file" samples="8" path="designs/unit8.csv"/>
  <recording interval="0.02"/>
  <syntax separator=":"/>
  <parameters>
    <stochastic name="mu">
      <distribution name="normal" mean="1.2" sigma="0.1">
        <coverage probability="0.997"/>
      </distribution>
      <target scope="oscillator:mu"/>
      <target scope="sub.osc2:mu"/>
    </stochastic>
    <stochastic name="restitution">
      <distribution name="normal" mean="0.8" sigma="0.05">
        <coverage sigma="3"/>
      </distribution>
      <target scope="ball:e"/>
    </stochastic>
    <stochastic name="gravity">
      <distribution name="uniform" min="-10.3005" max="-9.3195"/>
      <target scope="ball:g"/>
    </stochastic>
  </parameters>
  <signals>
    <signal name="height"><target scope="ball:h"/></signal>
    <signal name="oscillator_x0"><target scope="oscillator:x0"/></signal>
  </signals>
</experiment>
```

## 6. Limitations and further development

- Correlated parameters are not supported.
- Derived parameters are not supported. This may needed for parameters that need to be defined with offsets e.g., same temperature with Kelvin and Celsius.
- Array elements for general modeling formats. Note that in *Modelica* arrays are addressed as scalars with its own syntax e.g., as `k[2]`.
- Distributions beyond `uniform` and `normal` are not supported.

