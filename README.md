# SupplyChainOptimization.jl

[![Build status (Github Actions)](https://github.com/SupplyChef/SupplyChainOptimization.jl/workflows/CI/badge.svg)](https://github.com/SupplyChef/SupplyChainOptimization.jl/actions)
[![codecov.io](http://codecov.io/github/SupplyChef/SupplyChainOptimization.jl/coverage.svg?branch=main)](http://codecov.io/github/SupplyChef/SupplyChainOptimization.jl?branch=main)
[![](https://img.shields.io/badge/docs-latest-blue.svg)](https://SupplyChef.github.io/SupplyChainOptimization.jl/dev)

SupplyChainOptimization.jl is a package for modeling and optimizing supply chains. 

Use built-in constructs to easily model your supply chain and use powerful solvers to optimize it.

Learn more by reading the [documentation](https://SupplyChef.github.io/SupplyChainOptimization.jl/dev).

## Mathematical problem

The problem is structured as follows:

$$
\begin{align*}
\text{total costs} &\geq 0 \\
\text{total transportation costs} &\geq 0 \\
\text{total fixed costs} &\geq 0 \\
\text{total holding costs} &\geq 0 \\
\text{total costs per period}_{t} &\geq 0, \\; t \in \text{times} \\
\text{total transportation costs per period}_{t} &\geq 0, \\; t \in \text{times} \\
\text{total fixed costs per period}_{t} &\geq 0, \\; t \in \text{times} \\
\text{total holding costs per period}_{t} &\geq 0, \\; t \in \text{times} \\
\text{opened}_{p,s,t} & \in \\{0,1\\}, \\; p \in \text{plants}, \\; s \in \text{storages}, \\; t \in \text{times} \\
\text{opening}_{p,s,t} & \in \\{0,1\\}, \\; p \in \text{plants}, \\; s \in \text{storages}, \\; t \in \text{times} \\
\text{closing}_{p,s,t} & \in \\{0,1\\}, \\; p \in \text{plants}, \\; s \in \text{storages}, \\; t \in \text{times} \\
\text{lost sales}_{pr,c,t} &\geq 0, \\; pr \in \text{products}, \\; c \in \text{customers}, \\; t \in \text{times} \\
\text{bought}_{pr,sp,t} &\geq 0, \\; pr \in \text{products}, \\; sp \in \text{suppliers}, \\; t \in \text{times} \\
\text{produced}_{pr,p,t} &\geq 0, \\; pr \in \text{products}, \\; p \in \text{plants}, \\; t \in \text{times} \\
\text{stored at start}_{pr,s,t} &\geq 0, \\; pr \in \text{products}, \\; s \in \text{storages}, \\; t \in \text{times} \\
\text{stored at end}_{pr,s,t} &\geq 0, \\; pr \in \text{products}, \\; s \in \text{storages}, \\; t \in \text{times} \\
\text{used}_{l,t} & \in \\{0,1\\}, \\; l \in \text{lanes}, \\; t \in \text{times} \\
\text{sent}_{pr,l,t} &\geq 0, \\; pr \in \text{products}, \\; l \in \text{lanes}, \\; t \in \text{times} \\
\text{received}_{pr,l,t} &\geq 0, \\; pr \in \text{products}, \\; l \in \text{lanes}, \\; t \in \text{times} \\
\text{s.t.} \\; \text{stored at start}_{pr,s,t=1} &= \text{initial inventory}_{pr,s}, \\; s \\; \text{has initial inventory} \\; pr \\
\text{stored at start}_{pr,s,t=1} &\leq \text{stored at end}_{pr,s}, \\; s \\; \text{does not have initial inventory} \\; pr 
\end{align*}
$$

## Questions

  1. `stored_at_start` and `stored_at_end`; should these just only look at the initial and final time periods?