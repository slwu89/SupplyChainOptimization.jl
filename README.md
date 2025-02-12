# SupplyChainOptimization.jl

[![Build status (Github Actions)](https://github.com/SupplyChef/SupplyChainOptimization.jl/workflows/CI/badge.svg)](https://github.com/SupplyChef/SupplyChainOptimization.jl/actions)
[![codecov.io](http://codecov.io/github/SupplyChef/SupplyChainOptimization.jl/coverage.svg?branch=main)](http://app.codecov.io/github/SupplyChef/SupplyChainOptimization.jl?branch=main)
[![](https://img.shields.io/badge/docs-latest-blue.svg)](https://SupplyChef.github.io/SupplyChainOptimization.jl/dev)

SupplyChainOptimization.jl is a package for modeling and optimizing supply chains. 

Use built-in constructs to easily model your supply chain and use powerful solvers to optimize it.

Learn more by reading the [documentation](https://SupplyChef.github.io/SupplyChainOptimization.jl/dev).

## Mathematical problem

We describe the problem below.

### Variables

We list variables below. Subscripts are as follows, $p$ stands for plants, $s$ for storages, $t$ for times, $pr$ for products, $l$ for lanes, $c$ for customers, $sp$ for suppliers. If a particular subscript deviates from this definition, we will explicitly denote it (e.g. $s\in \text{plants}\cup\text{storages}$) If bounds or conditions on subscripts are not explicitly given, assume that it is for all values in that set.

$$
\begin{align*}
\text{total costs} &\geq 0 \\
\text{total transportation costs} &\geq 0 \\
\text{total fixed costs} &\geq 0 \\
\text{total holding costs} &\geq 0 \\
\text{total costs per period}_{t} &\geq 0 \\
\text{total transportation costs per period}_{t} &\geq 0 \\
\text{total fixed costs per period}_{t} &\geq 0 \\
\text{total holding costs per period}_{t} &\geq 0 \\
\text{opened}_{s,t} & \in \\{0,1\\}, \\; s\in \text{plants}\cup\text{storages} \\
\text{opening}_{s,t} & \in \\{0,1\\}, \\; s\in \text{plants}\cup\text{storages} \\
\text{closing}_{s,t} & \in \\{0,1\\}, \\; s\in \text{plants}\cup\text{storages} \\
\text{lost sales}_{pr,c,t} &\geq 0 \\
\text{bought}_{pr,sp,t} &\geq 0 \\
\text{produced}_{pr,p,t} &\geq 0 \\
\text{stored at start}_{pr,s,t} &\geq 0 \\
\text{stored at end}_{pr,s,t} &\geq 0 \\
\text{used}_{l,t} & \in \\{0,1\\} \\
\text{sent}_{pr,l,t} &\geq 0 \\
\text{received}_{pr,l,t} &\geq 0
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

### Constraints

The first set of constraints is related to ensuring that there is a logically consistent relationship between product storage, products sent and recieved, and sent and recieved products respect the limitations on the lanes to and from storage areas.

  1. Storage sites that have initial inventory have it added to stored at start.
  2. Storage sites without initial inventory (i.e. 0) have less than or equal to the amount at the end (consistency).
  3. Products recieved through a shipping lane at $t$ are equal to those sent through that lane at time $t-l_{\text{time}}$.
  4. No products can be recieved through a shipping lane at time 0.
  5. For each used lane, the sent products are less than some maximum.
  6. For each used lane with a minimum quantity per time unit that the send products are equal or greater than it.
  7. For opened storages, products sent along lanes out are less than some maximum.
  8. Products sent along lanes out from an opened storage site to a customer should equal the customer's deman for that product at that time.
  9. Sum of all products sent out on lanes connected to a storage site should be less than the max throughput of the storage site.
  10. All products recieved through lanes into an open storage site should be less than some maximum.

Mathematically they are given below:

$$
\begin{align*}
\text{stored at start}_{pr,s,t=1} &= \text{initial inventory}_{pr,s}, \\; s \\; \text{has initial inventory} \\; pr \\
\text{stored at start}_{pr,s,t=1} &\leq \text{stored at end}_{pr,s,t=t_{end}}, \\; s \\; \text{does not have initial inventory} \\; pr \\
\text{received}_{pr,l,t} &= \text{sent}_{p,l,t-l_{\text{time}}}, \\; t \geq l_{\text{time}} \\
\text{received}_{pr,l,t} &= 0, \\; t = 0 \\
\sum_{pr} \text{sent}_{pr,l,t} &\leq \text{bigM} * \text{used}_{l,t} \\
\sum_{pr} \text{sent}_{pr,l,t} &\geq l_{\text{minimum quantity}} * \text{used}_{l,t} \\
\sum_{pr,l\in s_{\text{lanes out}}} \text{sent}_{pr,l,t}  &\leq \text{bigM} * \text{opened}_{s,t} \\
\sum_{l\in s_{\text{lanes out}}; \\; l_{\text{destination}}=c} \text{sent}_{pr,l,t} &\leq \text{demand}_{pr,c,t} * \text{opened}_{s,t} \\
\sum_{l\in s_{\text{lanes out}}} \text{sent}_{pr,l,t}  &\leq \text{max throughput}_{s,pr} \\
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
@constraint(m, [p=products, s=storages, c=customers, t=times], sum(sent[p, l, t] for l in filter(l -> l.destination == c, get_lanes_out(supply_chain, s))) <= get_demand(supply_chain, c, p, t) * opened[s, t])
@constraint(m, [p=products, s=storages, t=times; !isinf(get_maximum_throughput(s, p))], sum(sent[p, l, t] for l in get_lanes_out(supply_chain, s)) <= get_maximum_throughput(s, p))
@constraint(m, [s=storages, t=times], sum(received[p, l, t] for p in products, l in get_lanes_in(supply_chain, s)) <= bigM * opened[s, t])
```

The next set of constraints ensures that there is consistency between opening/closing of various sites with their initial opening status. In the following set of constraints $s$ always referrs to an element of $\text{plants} \cup \text{storages}$, not $\text{storages}$.

  11. Makes sure there is consistency in initial opening, opened, and initial opening status at time 1.
  12. A site cannot be opening at time 1 and not be opened at time 1.
  13. A site cannot be opening if it was already initially opened.
  14. Makes sure there is consistency between opening and opened status for times t>1.
  15. A site cannot be opening and not opened at time t>1.
  16. A site opening at time $t$ should have been closed at time $t-1$ for times t>1.
  17. Makes sure there is consistency between closing, opened, and initial opening status at time 1.
  18. A site cannot be closing and opened at time 1.
  19. A site closing at time 1 must have been initially opened.
  20. Makes sure there is consistency between closing, opened, and initial opening status at t>1.
  21. A site cannot be closing and opened at t>1.
  22. A site closing at time t>1 must have been opened at time t-1.
  23. Sites with infinite opening costs must never open.
  24. Sites with infinite closing costs must never close.

Mathematically they are given below:

$$
\begin{align*}
\text{opening}_{s,t=1} &\geq \text{opened}_{s,t=1} + (1 - s_{\text{initial opened}}) - 1 \\
\text{opening}_{s,t=1} &\leq \text{opened}_{s,t=1} \\
\text{opening}_{s,t=1} &\leq 1 - s_{\text{initial opened}} \\
\text{opening}_{s,t} &\geq  \text{opened}_{s,t} + (1 - \text{opened}_{s,t-1}), \\; t>1 \\
\text{opening}_{s,t} &\leq  \text{opened}_{s,t}, \\; t>1 \\
\text{opening}_{s,t} &\leq  1-\text{opened}_{s,t-1}, \\; t>1 \\
\text{closing}_{s,t=1} &\geq  (1 - \text{opened}_{s,t=1}) + s_{\text{initial opened}} - 1 \\
\text{closing}_{s,t=1} &\leq  1 - \text{opened}_{s,t=1} \\
\text{closing}_{s,t=1} &\leq  1 - s_{\text{initial opened}} \\
\text{closing}_{s,t} &\geq  (1 - \text{opened}_{s, t}) + \text{opened}_{s, t-1} - 1, \\; t>1 \\
\text{closing}_{s,t} &\leq  1 - \text{opened}_{s, t}, \\; t>1 \\
\text{closing}_{s,t} &\leq \text{opened}_{s, t-1}, \\; t>1 \\
\text{opening}_{s,t} &= 0, \\; s_{\text{opening cost}} = \infty \\
\text{closing}_{s,t} &= 0, \\; s_{\text{closing cost}} = \infty
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

@constraint(m, [s=plants_storages, t=times; isinf(s.opening_cost)], opening[s, t] == 0)
@constraint(m, [s=plants_storages, t=times; isinf(s.closing_cost)], closing[s, t] == 0)
```

The next set of constraints controls bookkeeping for items sent and recieved.

  25. The amount of product stored at the end of a time step at a storage site must equal the amount stored at the start, plus amount recieved, minus amount sent.
  26. For t>1, the amount stored at start of time t must equal the amount stored at end of t-1.
  27. The amount stored at end should be greater that the sum of all product sent multiplied by the required addditional stock cover for each unit.
  28. Amount of product bought from a certain supplier at a time equals the amount that it has sent out at that time.
  29. For suppliers with finite maximum throughput, the total amount sent must be less than that maximum.

Mathematically they are:

$$
\begin{align*}
\text{stored at end}_{pr,s,t} &= \text{stored at start}_{pr,s,t} + \sum_{l\in s_{\text{lanes in}}} \text{received}_{pr,l,t} - \sum_{l\in s_{\text{lanes out}}}\text{sent}_{pr,l,t} \\
\text{stored at start}_{pr,s,t} &= \text{stored at end}_{pr,s,t-1}, \\; t>1 \\
\text{stored at end}_{pr,s,t} &\geq \text{additional stock cover}_{pr,s} * \sum_{l\in s_{\text{lanes out}}} \text{sent}_{pr,l,t} \\
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

  30. Production can only occur at opened plants, and the time $t_{i}$ accounts for lead time (time required for a plant to produce a product).
  31. The amount of product produced by a plant at time t should be equal to the amount shipped out after the amount of time equal to production lead time has elapsed.
  32. The amount of product sent out by a plant should be less than some maximum (for finite maximums).
  33. The amount of materials recieved pr to produce the amount of product pr2, should equal the amount of pr2 produced multiplied by the amount of pr needed to make one unit of pr2.

Mathematically they are:

$$
\begin{align*}
\text{produced}_{pr,p,t} &\leq \text{bigM}*\text{opened}_{p,t_{i}}, \\; t_{i} = \min(t+p_{\text{time}_{pr}}, \\; \text{time horizon}) \\
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

The final set of constraints are all about demand and costs in the system. As these are relatively straightforward we do not provide detailed explanations, only remarking that in the first `lost_sales` are demands from customers which cannot be fulfilled.

$$
\begin{align*}
\sum_{l\in c_{\text{lanes in}}} \text{received}_{pr,l,t} &= \text{demand}_{c,pr,t} - \text{lost sales}_{c,pr,t} \\
\sum_{t\in\text{times}} \text{lost sales}_{pr,c,t} &\leq 1-\text{service level}_{c,pr} * \sum_{t} \text{demand}_{c,pr,t} \\
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

### Objective

The final objective function is simply to minimize the total costs. In JuMP it is given as:

```julia
@objective(m, Min, 1.0 * total_costs)
```