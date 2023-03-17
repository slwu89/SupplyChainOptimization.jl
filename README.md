# SupplyChainOptimization.jl

[![Build status (Github Actions)](https://github.com/SupplyChef/SupplyChainOptimization.jl/workflows/CI/badge.svg)](https://github.com/SupplyChef/SupplyChainOptimization.jl/actions)
[![codecov.io](http://codecov.io/github/SupplyChef/SupplyChainOptimization.jl/coverage.svg?branch=main)](http://codecov.io/github/SupplyChef/SupplyChainOptimization.jl?branch=main)
[![](https://img.shields.io/badge/docs-latest-blue.svg)](https://SupplyChef.github.io/SupplyChainOptimization.jl/dev)

SupplyChainOptimization.jl is a package for modeling and optimizing supply chains. 

Use built-in constructs to easily model your supply chain and use powerful solvers to optimize it.

Learn more by reading the [documentation](https://SupplyChef.github.io/SupplyChainOptimization.jl/dev).

## Mathematical problem

The problem is structured as follows:

We list variables below. Subscripts are as follows, $p$ stands for plants, $s$ for storages, $t$ for times, $pr$ for products, $l$ for lanes, $c$ for customers, $sp$ for suppliers. If bounds or conditions on subscripts are not explicitly given, assume that it "for all" values in that set.

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
\text{received}_{pr,l,t} &\geq 0, \\; pr \in \text{products}, \\; l \in \text{lanes}, \\; t \in \text{times}
\end{align*}
$$

In JuMP they appear as:

```julia
@variable(m, total_costs >= 0)
@variable(m, total_transportation_costs >= 0)
@variable(m, total_fixed_costs >= 0)
@variable(m, total_holding_costs >= 0)

@variable(m, total_costs_per_period[times] >= 0)
@variable(m, total_transportation_costs_per_period[times] >= 0)
@variable(m, total_fixed_costs_per_period[times] >= 0)
@variable(m, total_holding_costs_per_period[times] >= 0)

@variable(m, opened[plants_storages, times], Bin)
@variable(m, opening[plants_storages, times], Bin)
@variable(m, closing[plants_storages, times], Bin)


@variable(m, lost_sales[products, customers, times] >= 0)

@variable(m, bought[products, suppliers, times] >= 0)

@variable(m, produced[products, plants, times] >= 0)

@variable(m, stored_at_start[products, storages, times] >= 0)
@variable(m, stored_at_end[products, storages, times] >= 0)

@variable(m, used[lanes, times], Bin)
@variable(m, sent[products, lanes, times] >= 0)
@variable(m, received[products, lanes, times] >= 0)
```

Constraints are below.

The first set of constraints is related to ensuring that there is a logically consistent relationship between product storage, products sent and recieved, and sent and recieved products respect the limitations on the lanes to and from storage areas.

$$
\begin{align*}
\text{stored at start}_{pr,s,t=1} &= \text{initial inventory}_{pr,s}, \\; s \\; \text{has initial inventory} \\; pr \\
\text{stored at start}_{pr,s,t=1} &\leq \text{stored at end}_{pr,s,t=t_{end}}, \\; s \\; \text{does not have initial inventory} \\; pr \\
\text{received}_{pr,l,t} &= \text{sent}_{p,l,t-l_{time}}, \\; t \geq l_{time} \\
\text{received}_{pr,l,t} &= 0, \\; t = 0 \\
\sum_{pr} \text{sent}_{pr,l,t} &\leq \text{bigM} * \text{used}_{l,t}, \\; l_{minimum quantity} > 0 \\
\sum_{pr} \text{sent}_{pr,l,t} &\geq l_{minimum quantity} * \text{used}_{l,t}, \\; l_{minimum quantity} > 0 \\
\sum_{pr,l\in s_{\text{lanes out}}} \text{sent}_{pr,l,t}  &\leq \text{bigM} * \text{opened}_{s,t} \\
\sum_{l\in s_{\text{lanes out}}} \text{sent}_{pr,l,t}  &\leq \text{max throughput}(s,pr), \\; \text{max throughput}(s,pr) \neq \infty \\
\sum_{l\in s_{\text{lanes in}}} \text{received}_{pr,l,t} &\leq \text{bigM} * \text{opened}_{s,t} \\
\end{align*}
$$

In the JuMP language they appear as:

```julia
@constraint(m, [p=products, s=storages; haskey(s.initial_inventory, p)], stored_at_start[p, s, 1] == s.initial_inventory[p])
@constraint(m, [p=products, s=storages; !haskey(s.initial_inventory, p)], stored_at_start[p, s, 1] <= stored_at_end[p, s, supply_chain.horizon])

@constraint(m, [p=products, l=lanes, t=times; t > l.time], received[p, l, t] == sent[p, l, t - l.time])
@constraint(m, [p=products, l=lanes, t=times; t <= l.time], received[p, l, t] == 0)

@constraint(m, [l=lanes, t=times; l.minimum_quantity > 0], sum(sent[p, l, t] for p in products) <= bigM * used[l, t])
@constraint(m, [l=lanes, t=times; l.minimum_quantity > 0], sum(sent[p, l, t] for p in products) >= l.minimum_quantity * used[l, t])

@constraint(m, [s=storages, t=times], sum(sent[p, l, t] for p in products, l in get_lanes_out(supply_chain, s)) <= bigM * opened[s, t])
@constraint(m, [p=products, s=storages, t=times; !isinf(get_maximum_throughput(s, p))], sum(sent[p, l, t] for l in get_lanes_out(supply_chain, s)) <= get_maximum_throughput(s, p))
@constraint(m, [s=storages, t=times], sum(received[p, l, t] for p in products, l in get_lanes_in(supply_chain, s)) <= bigM * opened[s, t])
```

The next set of constraints ensures that there is consistency between opening/closing of various sites with their initial opening status.

$$
\begin{align*}
\text{opening}_{s,t=1} &\geq \text{opened}_{s,t=1} + (1 - s_{\text{initial opened}}) - 1, \\; s \in \text{plants} \cup \text{storages} \\
\text{opening}_{s,t=1} &\leq \text{opened}_{s,t=1}, \\; s \in \text{plants} \cup \text{storages} \\
\text{opening}_{s,t=1} &\leq 1 - s_{\text{initial opened}}, \\; s \in \text{plants} \cup \text{storages} \\
\text{opening}_{s,t} &\geq  \text{opened}_{s,t} + (1 - \text{opened}_{s,t-1}), \\; s \in \text{plants} \cup \text{storages} , \\; t>1 \\
\text{opening}_{s,t} &\leq  \text{opened}_{s,t}, \\; s \in \text{plants} \cup \text{storages} , \\; t>1 \\
\text{opening}_{s,t} &\leq  1-\text{opened}_{s,t-1}, \\; s \in \text{plants} \cup \text{storages} , \\; t>1 \\
\text{closing}_{s,t=1} &\geq  (1 - \text{opened}_{s,t=1}) + s_{\text{initial opened}} - 1, \\; s \in \text{plants} \cup \text{storages} \\
\text{closing}_{s,t=1} &\leq  1 - \text{opened}_{s,t=1}, \\; s \in \text{plants} \cup \text{storages} \\
\text{closing}_{s,t=1} &\leq  1 - s_{\text{initial opened}}, \\; s \in \text{plants} \cup \text{storages} \\
\text{closing}_{s,t} &\geq  (1 - \text{opened}_{s, t}) + \text{opened}_{s, t-1} - 1, \\; s \in \text{plants} \cup \text{storages}, \\; t>1 \\
\text{closing}_{s,t} &\leq  1 - \text{opened}_{s, t}, \\; s \in \text{plants} \cup \text{storages}, \\; t>1 \\
\text{closing}_{s,t} &\leq \text{opened}_{s, t-1}, \\; s \in \text{plants} \cup \text{storages}, \\; t>1 \\
\text{opening}_{s,t} &= 0, \\; s \in \text{plants} \cup \text{storages}, \\; s_{\text{opening cost}} = \infty \\
\text{closing}_{s,t} &= 0, \\; s \in \text{plants} \cup \text{storages}, \\; s_{\text{closing cost}} = \infty
\end{align*}
$$

In the JuMP language they appear as:

```julia
@constraint(m, [s=plants_storages], opening[s, 1] >= opened[s, 1] + (1 - s.initial_opened) - 1)
@constraint(m, [s=plants_storages], opening[s, 1] <= opened[s, 1])
@constraint(m, [s=plants_storages], opening[s, 1] <= 1 - s.initial_opened)

@constraint(m, [s=plants_storages, t=times; t > 1], opening[s, t] >= opened[s, t] + (1 - opened[s, t-1]) - 1)
@constraint(m, [s=plants_storages, t=times; t > 1], opening[s, t] <= opened[s, t])
@constraint(m, [s=plants_storages, t=times; t > 1], opening[s, t] <= 1 - opened[s, t-1])

@constraint(m, [s=plants_storages], closing[s, 1] >= (1 - opened[s, 1]) + s.initial_opened - 1)
@constraint(m, [s=plants_storages], closing[s, 1] <= 1 - opened[s, 1])
@constraint(m, [s=plants_storages], closing[s, 1] <= s.initial_opened)

@constraint(m, [s=plants_storages, t=times; t > 1], closing[s, t] >= (1 - opened[s, t]) + opened[s, t-1] - 1)
@constraint(m, [s=plants_storages, t=times; t > 1], closing[s, t] <= 1 - opened[s, t])
@constraint(m, [s=plants_storages, t=times; t > 1], closing[s, t] <= opened[s, t-1])

@constraint(m, [s=plants_storages, t=times; isinf(s.opening_cost)], opending[s, t] == 0)
@constraint(m, [s=plants_storages, t=times; isinf(s.closing_cost)], closing[s, t] == 0)
```

## Questions

  1. `stored_at_start` and `stored_at_end`; should these just only look at the initial and final time periods?