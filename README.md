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

The next set of constraints controls bookkeeping for items sent and recieved.

$$
\begin{align*}
\text{stored at end}_{pr,s,t} &= \text{stored at start}_{pr,s,t} + \sum_{l\in s_{\text{lanes in}}} \text{received}_{pr,l,t} - \sum_{l\in s_{\text{lanes out}}}\text{sent}_{pr,l,t} \\
\text{stored at start}_{pr,s,t} &= \text{stored at end}_{pr,s,t-1}, \\; t>1 \\
\text{stored at end}_{pr,s,t} &= \text{additional stock cover}_{pr,s} * \sum_{l\in s_{\text{lanes out}}} \text{sent}_{pr,l,t} \\
\text{bought}_{pr,sp,t} &= \sum_{l\in sp_{\text{lanes out}}} \text{sent}_{pr,l,t} \\
\sum_{l\in sp_{\text{lanes out}}} \text{sent}_{pr,l,t} &\leq \text{maximum throughput}_{pr,sp}
\end{align*}
$$

In JuMP they are:

```julia
@constraint(m, [p=products, s=storages, t=times], stored_at_end[p, s, t] == stored_at_start[p, s, t] 
                                                                        + sum(received[p, l, t] for l in get_lanes_in(supply_chain, s))
                                                                        - sum(sent[p, l, t] for l in get_lanes_out(supply_chain, s))
                                                                        )
@constraint(m, [p=products, s=storages, t=times; t > 1], stored_at_start[p, s, t] == stored_at_end[p, s, t - 1])
@constraint(m, [p=products, s=storages, t=times], stored_at_end[p, s, t] >= get_additional_stock_cover(s, p) * sum(sent[p, l, t] for l in get_lanes_out(supply_chain, s)))

@constraint(m, [p=products, s=suppliers, t=times], bought[p, s, t] == sum(sent[p, l, t] for l in get_lanes_out(supply_chain, s)))
@constraint(m, [p=products, s=suppliers, t=times; !isinf(get_maximum_throughput(s, p))], sum(sent[p, l, t] for l in get_lanes_out(supply_chain, s)) <= get_maximum_throughput(s, p))
```

The next set of constraints also focus on shipping of products, as well as lead times from plants.

The first ensures that production can only occur at opened plants, and the time $t_{i}$ accounts for lead time (time required for a plant to produce a product).

$$
\begin{align*}
\text{produced}_{pr,p,t} &\leq \text{bigM}*\text{opened}_{p,t_{i}}, \; t_{i} &= \min(t+p_{\text{time}_{pr}}, \\; \text{time horizon}) \\
\text{produced}_{pr,p,t} &= \sum_{l\in p_{\text{lanes out}}} \text{sent}_{p,l,t+p_{\text{time}_{pr}}}, \; t + p_{\text{time}_{pr}} \leq \text{time horizon} \\
\sum_{l\in p_{\text{lanes out}}} \text{sent}_{pr,l,t} &\leq \text{maximum throughput}_{pr,p} \\
\sum_{pr2\in\text{products}} \text{produced}_{pr2,p,t} * \text{bill of materials}_{pr2,pr,p} &= \sum_{l\in p_{\text{lanes in}}} \text{received}_{pr,l,t}
\end{align*}
$$

```julia
for s in plants, p in products
    if haskey(s.time, p)
        @constraint(m, [t=times, ti=t:min(t+s.time[p], supply_chain.horizon)], produced[p, s, t] <= bigM * opened[s, ti])
    else
        @constraint(m, [t=times], produced[p, s, t] == 0)
    end
end
@constraint(m, [p=products, s=plants, t=times; haskey(s.time, p) && (t + s.time[p] <= supply_chain.horizon)], produced[p, s, t] == sum(sent[p, l, t + s.time[p]] for l in get_lanes_out(supply_chain, s)))
@constraint(m, [p=products, s=plants, t=times; !isinf(get_maximum_throughput(s, p))], sum(sent[p, l, t] for l in get_lanes_out(supply_chain, s)) <= get_maximum_throughput(s, p))
@constraint(m, [p=products, s=plants, t=times], sum(produced[p2, s, t] * get_bom(s, p2, p) for p2 in products) == sum(received[p, l, t] for l in get_lanes_in(supply_chain, s)))
```

The final set of constraints are all about demand and costs in the system.

$$
\begin{align*}
\sum_{l\in c_{\text{lanes in}}} \text{received}_{pr,l,t} &= \text{demand}_{c,pr,t} - \text{lost sales}_{c,pr,t} \\
\sum_{t\in\text{times}} \text{lost sales}_{pr,c} &\leq 1-\text{service level}_{c,pr} * \sum_{t} \text{demand}_{c,pr,t} \\
\text{total transportation costs per period}_{t} &= \sum_{pr, l} \text{sent}_{pr,l,t} * \text{unit cost}_{l} \\
\text{total transportation costs} &= \sum_{t}\text{total transportation costs per period}_{t} \\
\text{total fixed costs per period}_{t} &= \sum_{s \in \text{plants}\cup\text{storages}, t} \text{opened}_{s,t} * \text{fixed cost}_{s} \\
\text{total fixed costs} &= \sum_{t}\text{total fixed costs per period}_{t} \\
\text{total holding costs per period}_{t} &= \sum_{pr, s} \text{stored at end}_{pr,s,t} * \text{unit holding cost}_{pr} \\
\text{total holding costs} &= \sum_{t}\text{total holding costs per period}_{t} \\
\end{align*}
$$

```julia
@constraint(m, [p=products, c=customers, t=times], sum(received[p, l, t] for l in get_lanes_in(supply_chain, c)) == get_demand(supply_chain, c, p, t) - lost_sales[p, c, t])

@constraint(m, [p=products, c=customers], sum(lost_sales[p, c, t] for t in times) <= (1 - get_service_level(supply_chain, c, p)) * sum(get_demand(supply_chain, c, p, t) for t in times))

@constraint(m, [t=times], total_transportation_costs_per_period[t] == sum(sent[p, l, t] * l.unit_cost for p in products, l in lanes))
@constraint(m, total_transportation_costs == sum(total_transportation_costs_per_period[t] for t in times))

@constraint(m, [t=times], total_fixed_costs_per_period[t] == sum(opened[s, t] * s.fixed_cost for s in plants_storages))
@constraint(m, total_fixed_costs == sum(total_fixed_costs_per_period[t] for t in times))

@constraint(m, [t=times], total_holding_costs_per_period[t] == sum(stored_at_end[p, s, t] * p.unit_holding_cost for p in products, s in storages))
@constraint(m, total_holding_costs == sum(total_holding_costs_per_period[t] for t in times))
```

Calculating the total costs per period may look complex but its just bookkeeping.

$$
\begin{align*}
\text{total costs per period}_{t} &= \text{total costs per period}_{t} + \text{total transportation costs per period}_{t} + \text{total fixed costs per period}_{t} \\
&+ \sum_{s\in\text{plants}\cup\text{storages}} \text{opening}_{s,t} * \text{opening cost}_{s} \\
&+ \sum_{s\in\text{plants}\cup\text{storages}} \text{closing}_{s,t} * \text{closing cost}_{s} \\ 
&+ \sum_{pr,s}\left(\sum_{l\in s_{\text{lanes in}}} \text{received}_{pr,l,t} * \text{unit handling cost}_{s,pr} \right ) \\ 
&+ \sum_{pr,sp} \text{bought}_{pr,sp,t} * \text{unit cost}_{pr,sp}  \\ 
&+ \sum_{pr,p} \text{produced}_{pr,p,t} * \text{unit cost}_{pr,p} \\
&+ \text{total holding costs}
\end{align*}
$$

and the grant total cost:

$$
\text{total costs} = \sum_{t} \text{total costs per period}_{t}
$$

```julia
@constraint(m, [t=times], total_costs_per_period[t] == total_transportation_costs_per_period[t] + 
                    total_fixed_costs_per_period[t] + 
                    sum(opening[s, t] * s.opening_cost for s in plants_storages if !isinf(s.opening_cost)) + 
                    sum(closing[s, t] * s.closing_cost for s in plants_storages if !isinf(s.closing_cost)) + 
                    sum(sum(received[p, l, t] * s.unit_handling_cost[p] for l in get_lanes_in(supply_chain, s)) for p in products for s in storages if haskey(s.unit_handling_cost, p)) +
                    sum(bought[p, s, t] * s.unit_cost[p] for p in products, s in suppliers if haskey(s.unit_cost, p)) +
                    sum(produced[p, s, t] * s.unit_cost[p] for p in products, s in plants if haskey(s.unit_cost, p)) +
                    total_holding_costs)

@constraint(m, total_costs == sum(total_costs_per_period[t] for t in times))
```

## Questions

  1. `stored_at_start` and `stored_at_end`; should these just only look at the initial and final time periods?