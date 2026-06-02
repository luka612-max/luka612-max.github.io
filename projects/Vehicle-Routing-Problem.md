---
layout: single
title: ""
permalink: /projects/Vehicle-Routing-Problem/
author_profile: false
classes: wide-project-layout
mathjax: true
---

# Introduction & Motivation

Our team set out to solve the Vehicle Routing Problem (VRP)—a computationally intractable challenge in operations research—using advanced parallel processing in C++. Rather than relying on a single optimization algorithm (which gets trapped in local optima), we designed KRITI-Optimization to orchestrate multiple competing metaheuristics simultaneously: ALNS, Branch-And-Cut, Heterogeneous DARP, and Memetic algorithms. Each algorithm explores the solution space differently, and their complementary strengths combine to deliver near-optimal routes and are run parallely by using multithreading. 

I really wanted to see what it takes to build software in the real world, not just for a class assignment. Working with a team on something as heavy as the Vehicle Routing Problem was the perfect excuse to geek out on low-level C++. It was an awesome experience figuring out how to use multithreading to run competing algorithms at the same time and actually build a solver that works at scale

The implementation of the whole project can be found here: [Github Repository](https://github.com/luka612-max/KRITI-Optimization)

## Problem Overview

The Vehicle Routing Problem is NP-Hard in complexity. Traditional approaches fail: exact algorithms take hours for 100+ nodes, single heuristics get trapped in local optima, and greedy methods produce poor-quality solutions. Our insight was simple but powerful: instead of betting on one algorithm, orchestrate multiple competing metaheuristics simultaneously, each exploring the solution space differently, so their complementary strengths combine to deliver superior routes.

## System Architecture

Our implementation included the following:
- Multithreaded geographic matrix generation (parallelized OSRM API calls with Haversine fallback)
- Multi-algorithm orchestration (ALNS, Branch-And-Cut, Heterogeneous DARP, GOD, Memetic Algorithm running in parallel)
- Request isolation and concurrency safety (UUID-based sandboxing with mutex protection)
- Metadata-driven configuration (flexible optimization goals without recompilation)
- REST API backend (Crow framework for high-performance HTTP handling)
- Docker deployment (production-ready containerization)

## Matrix Generation: Parallel I/O Architecture

The first bottleneck we identified was distance matrix computation. OSRM API has a 100-coordinate limit per request, meaning N nodes require O(N²/100) sequential API calls—prohibitively slow.

Our solution: partition N nodes into blocks of 99 coordinates and spawn a thread for every (srcBlock, dstBlock) pair:

```cpp
std::vector<std::vector<int>> blocks;
for (int i = 0; i < N; i += LIMIT) {
    std::vector<int> block;
    for (int j = i; j < std::min(i + LIMIT, N); ++j)
        block.push_back(j);
    blocks.push_back(block);
}

std::vector<std::thread> threads;
for (const auto &srcBlock : blocks) {
    for (const auto &dstBlock : blocks) {
        threads.emplace_back(fetch_block, srcBlock, dstBlock);
    }
}

for (auto &t : threads)
    if (t.joinable()) t.join();
```
**Performance Impact:** 200 nodes with sequential fetching = 200 seconds. With parallel threads (8 cores) = ~25 seconds. 8x speedup.

## Graceful Fallback: OSRM to Haversine

If OSRM network requests fail, we automatically fall back to Haversine great-circle distance calculations using an atomic flag:

```cpp
std::atomic<bool> osrm_failed{false};

auto fetch_block = [&](const std::vector<int> &srcBlock, 
                       const std::vector<int> &dstBlock) {
    bool osrm_success = false;
    // Attempt OSRM fetch...
    if (osrm_success) {
        // Use road network distances
        outMatrix[i][j] = osrm_result / 1000.0;
    } else {
        // Fallback: Great-circle distance
        outMatrix[i][j] = haversine(lat1, lon1, lat2, lon2);
    }
};
```
No restart needed. Threads adapt mid-execution. This ensures offline functionality and robustness against network failures.

## Multi-Algorithm Orchestration

Rather than choosing a single optimization algorithm, we run five competing approaches in parallel:

| Algorithm | Strategy | Strength |
| :--- | :--- | :--- |
| **ALNS** | Destroy-repair cycles with adaptive operator selection | Escapes local optima through large neighborhoods |
| **Branch-And-Cut** | Exact LP relaxation + cutting planes | Provides mathematical upper bounds on solution quality |
| **Heterogeneous DARP** | Graph clustering + dynamic programming | Handles diverse vehicle types natively |
| **GOD** | Heuristic-specific optimization | Fast convergence on benchmark instances |
| **Memetic Algorithm** | Population-based + local search (seeded) | Evolutionary refinement of best solutions |

```cpp
std::thread t1([&](){ alns_res = run_solver("ALNS", "main_ALNS", reqDir); });
std::thread t2([&](){ bac_res = run_solver("Branch-And-Cut", "main_BAC", reqDir); });
std::thread t4([&](){ hd_res = run_solver("Heterogeneous_DARP", "hetero", reqDir); });
std::thread t6([&](){ god = run_solver("god", "god", reqDir); });

t1.join();
t2.join();
t4.join();
t6.join();

// All algorithms complete. Memetic runs after with seeded population.
mem = run_solver("memetic_algorithm", "main_Memetic", reqDir);
```
Each algorithm explores differently. If ALNS gets stuck, Branch-And-Cut escapes it. Memetic seeds from their outputs instead of random initialization, amplifying discoveries.

## ALNS: Adaptive Operator Selection

The ALNS algorithm uses destroy-repair cycles with adaptive operator weighting:

```cpp
for (int iteration = 1; iteration < total_iterations; iteration++) {
    
    // 1. Adaptively select destroy operator by weighted probability
    int destroy_op = selectOperator({
        dStats[0].weight, dStats[1].weight,
        dStats[2].weight, dStats[3].weight
    }, rng);
    
    // 2. Destroy: Remove k customers (k adapts throughout search)
    if (destroy_op == RANDOM)
        randomDestroy(solution, k);
    else if (destroy_op == LONGEST_TAIL)
        LongestRouteTailRemoval(solution, k);
    else if (destroy_op == WORST_TAIL)
        WorstRouteTailRemoval(solution, k);
    
    // 3. Repair: Re-insert customers using different heuristics
    int repair_op = selectOperator({
        rStats[0].weight, rStats[1].weight, rStats[2].weight
    }, rng);
    
    if (repair_op == GREEDY)
        greedyRepair(solution, employees, vehicles, metadata);
    else if (repair_op == REGRET)
        regretRepair(solution, employees, vehicles, metadata, k);
    
    // 4. Reward feedback: Update operator weights
    if (newCost < bestCost)
        destroy_stats[destroy_op].score += 5.0;  // Excellent
    else if (newCost < currentCost)
        destroy_stats[destroy_op].score += 2.0;  // Improvement
    else
        destroy_stats[destroy_op].score += 0.5;  // Accepted
    
    // 5. Simulated annealing: Accept worse solutions probabilistically
    double acceptance_prob = exp(-cost_delta / temperature);
    if (acceptance_prob > random(0, 1))
        pool.replace(worst_solution, new_solution);
}
```
Operators that work well get higher selection probability. Multiple destroy/repair combinations prevent cycling. Simulated annealing provides thermal kicks to escape local optima.

## Request Isolation & Concurrency Safety

**The key challenge:** multiple concurrent API requests must not interfere with each other. Our solution: UUID-based sandboxing + mutex protection.

```cpp
fs::path create_request_tmp_dir() {
    fs::path base = fs::absolute("tmp");
    fs::create_directories(base);
    
    std::string id = "req_" + generate_uuid();  // Unique per request
    fs::path reqDir = base / id;
    
    fs::create_directory(reqDir);
    return reqDir;
}

// Each algorithm gets its own subdirectory
fs::path runDir = reqDir / "ALNS";           // tmp/req_abc123/ALNS/
fs::path runDir = reqDir / "Branch-And-Cut"; // tmp/req_abc123/Branch-And-Cut/

// Global mutex protects console output
std::mutex log_mutex;
{
    std::lock_guard<std::mutex> lock(log_mutex);
    std::cout << "[ALNS] Iteration 100..." << std::endl;  // No garbled output
}
```
Each request gets isolated temp storage. Algorithms run in separate processes. No data races. Zero interference between concurrent requests.

## Metadata-Driven Weighted Optimization

Routes must optimize two competing metrics simultaneously: Cost (fuel, vehicle wear) and Time (employee wait time, punctuality).

<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.7/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

We use a weighted objective function: 

{% raw %}
$$ Z = (W_{cost} \times \text{TripCost}) + (W_{time} \times \text{TotalPassengerTime}) $$
{% endraw %}

```cpp
key,value
objective_cost_weight,0.6
objective_time_weight,0.4
priority_1_max_delay_min,5
priority_2_max_delay_min,10
allow_external_maps,TRUE
```
Users adjust weights without recompilation:

* `W_cost=0.9, W_time=0.1` : Cost-optimized (cheaper vehicles, longer waits)
* `W_cost=0.3, W_time=0.7` : Time-optimized (express service, higher fuel cost)
* `W_cost=0.5, W_time=0.5` : Balanced approach

## Benchmark Results

Performance characteristics on a standard benchmark:

| Metric | Performance |
| :--- | :--- |
| **Employees Handled** | 50-1000+ |
| **Vehicles in Fleet** | 5-200 |
| **Time to Optimize** | 10-60 seconds |
| **Matrix Generation (200 nodes)** | ~3 seconds (parallel OSRM) |
| **Parallel Solver Speedup** | 4-6x vs. single-algorithm baseline |
| **Memory per Request** | ~50MB (sandboxed temp dir) |
| **API Throughput** | 8+ concurrent requests |

Example: 500 employees, 20 vehicles

* Sequential ALNS: Cost = 5,200 units
* Branch-And-Cut: Cost = 4,800 units
* Memetic (seeded): Cost = 4,700 units
* 9% improvement over naive approach

## Conclusion and Learnings

This project was excellent for learning parallel multithreading in C++, distributed algorithm orchestration, and production-grade systems design.Overall, a very positive learning outcome and a deployable solution for large-scale vehicle routing optimization.