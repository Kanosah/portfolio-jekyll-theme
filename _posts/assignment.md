  
---
layout: post
title: "Using github pages"
---
## Problem Description 

There are six end landfills, each with a known demand for a waste material.  Landfill demand can be satisfied from a set of four transfer station depots, or directly from a set of two waste generating centers.  Each transfer depot can support a maximum volume of waste moving through it, and each waste generating center can generate a maximum amount of waste.  There are known costs associated with transporting the waste, from a center to a depot, from a depot to a landfill, or from a center directly to a landfill.

The waste network has two waste generating centers, in NewYork and NewJersey, which generate the waste.  Each has a maximum waste generating volume:

| Center | Waste (tons) |
| --- | --- |
| NewYork | 300,000 |
| NewJersey |  400,000 |

The waste can be shipped from a center to a set of four depots. Each depot has a maximum throughput.

| Depot | Throughput (tons) |
| --- | --- |
| Bronx | 140,000 |
| Brooklyn | 100,000 |
| Queens | 200,000 |
| StatenIsland | 80,000 |

Our network has six landfills, each with a given maxiumum demand.

| Landfill | Demand (tons) |
| --- | --- |
| C1 | 100,000 |
| C2 | 20,000 |
| C3 | 80,000 |
| C4 | 70,000 |
| C5 | 120,000 |
| C6 | 40,000 |

Transporation costs are given in the following table (in dollars per ton).  Columns are source cities and rows are destination cities.  

| To | NewYork | NewJersey | Bronx | Brooklyn | Queens | StatenIsland |
| --- | --- | --- | --- | --- | --- | --- |
| Depots |
| Bronx  | 0.7 |   - |
| Brooklyn | 0.7 | 0.5 |
| Queens     | 1.2 | 0.7 |
| StatenIsland     | 0.4 | 0.4 |
| Landfills |
| C1 | 1.2 | 2.2 |   - | 1.2 |   - |   - |
| C2 |   - |   - | 1.7 | 0.7 | 1.7 |   - |
| C3 | 1.7 |   - | 0.7 | 0.75 | 2.2 | 0.4 |
| C4 | 2.2 |   - | 1.7 | 1.2|   - | 1.7 |
| C5 |   - |   - |   - | 0.7 | 0.7 | 0.7 |
| C6 | 1.2 |   - | 1.2 |   - | 1.7 | 1.7 |

Question: How to satisfy the demands of the end Landfills while minimizing shipping costs.

---
## Model Formulation

### Sets and Indices

$f \in \text{Centers}=\{\text{NewYork}, \text{NewJersey}\}$

$d \in \text{Depots}=\{\text{Bronx}, \text{Brooklyn}, \text{Queens}, \text{StatenIsland}\}$

$c \in \text{Landfills}=\{\text{C1}, \text{C2}, \text{C3}, \text{C4}, \text{C5}, \text{C6}\}$

$\text{Locations} = \text{Centers} \cup \text{Depots} \cup \text{Landfills}$

### Parameters

$\text{cost}_{s,t} \in \mathbb{R}^+$: Cost of shipping one ton from source $s$ to destination $t$.

$\text{supply}_f \in \mathbb{R}^+$: Maximum possible supply from center $f$ (in tons).

$\text{through}_d \in \mathbb{R}^+$: Maximum possible flow through depot $d$ (in tons).

$\text{demand}_c \in \mathbb{R}^+$: Demand for waste at landfill $c$ (in tons).

### Decision Variables

$\text{flow}_{s,t} \in \mathbb{N}^+$: Quantity of waste (in tons) that is shipped from source $s$ to destionation $t$.


### Objective Function

- **Cost**: Minimize total shipping costs.

\begin{equation}
\text{Minimize} \quad Z = \sum_{(s,t) \in \text{locations} \times \text{locations}}{\text{cost}_{s,t}*\text{flow}_{s,t}}
\end{equation}

### Constraints

- **Center output**: Flow of goods from a factory must respect maximum capacity.

\begin{equation}
\sum_{t \in \text{locations}}{\text{flow}_{f,t}} \leq \text{supply}_{f} \quad \forall f \in \text{Centers}
\end{equation}

- **Landfill demand**: Flow of goods must meet customer demand.

\begin{equation}
\sum_{s \in \text{locations}}{\text{flow}_{s,c}} = \text{demand}_{c} \quad \forall c \in \text{Landfills}
\end{equation}

- **Depot flow**: Flow into a depot equals flow out of the depot.

\begin{equation}
\sum_{s \in \text{locations}}{\text{flow}_{s,d}} = 
\sum_{t \in \text{locations}}{\text{flow}_{d,t}}
\quad \forall d \in \text{Depots}
\end{equation}

- **Depot capacity**: Flow into a depot must respect depot capacity.

\begin{equation}
\sum_{s \in \text{locations}}{\text{flow}_{s,d}} \leq \text{through}_{d}
\quad \forall d \in \text{Depots}
\end{equation}

---
## Python Implementation

We import the Gurobi Python Module and other Python libraries.


```python
%pip install gurobipy
```

    Looking in indexes: https://pypi.org/simple, https://us-python.pkg.dev/colab-wheels/public/simple/
    Requirement already satisfied: gurobipy in /usr/local/lib/python3.8/dist-packages (10.0.0)
    


```python
import pandas as pd

import gurobipy as gp
from gurobipy import GRB

# tested with Python 3.7.0 & Gurobi 9.0
```

## Input Data
We define all the input data for the model.


```python
# Create dictionaries to capture center supply limits, depot throughput limits, and landfill demand.

supply = dict({'NewYork': 300000,
               'NewJersey': 400000})

through = dict({'Bronx': 140000,
                'Brooklyn': 100000,
                'Queens': 200000,
                'StatenIsland': 80000})

demand = dict({'C1': 100000,
               'C2': 20000,
               'C3': 80000,
               'C4': 70000,
               'C5': 120000,
               'C6': 40000})

# Create a dictionary to capture shipping costs.

arcs, cost = gp.multidict({
    ('NewYork', 'Bronx'): 0.7,
    ('NewYork', 'Brooklyn'): 0.7,
    ('NewYork', 'Queens'): 1.2,
    ('NewYork', 'StatenIsland'): 0.4,
    ('NewYork', 'C1'): 1.2,
    ('NewYork', 'C3'): 1.7,
    ('NewYork', 'C4'): 2.2,
    ('NewYork', 'C6'): 1.2,
    ('NewJersey', 'Brooklyn'): 0.5,
    ('NewJersey', 'Queens'): 0.7,
    ('NewJersey', 'StatenIsland'): 0.4,
    ('NewJersey', 'C1'): 2.2,
    ('Bronx', 'C2'): 1.7,
    ('Bronx', 'C3'): 0.7,
    ('Bronx', 'C4'): 1.7,
    ('Bronx', 'C6'): 1.2,
    ('Brooklyn', 'C1'): 1.2,
    ('Brooklyn', 'C2'): 0.7,
    ('Brooklyn', 'C3'): 0.7,
    ('Brooklyn', 'C4'): 1.2,
    ('Brooklyn', 'C5'): 0.7,
    ('Queens', 'C2'): 1.7,
    ('Queens', 'C3'): 2.2,
    ('Queens', 'C5'): 0.7,
    ('Queens', 'C6'): 1.7,
    ('StatenIsland', 'C3'): 0.4,
    ('StatenIsland', 'C4'): 1.7,
    ('StatenIsland', 'C5'): 0.7,
    ('StatenIsland', 'C6'): 1.7
})
```

## Model Deployment

Create a model and  variables. The variables simply capture the amount of waste that flows along each allowed path between a source and destination.  Objective coefficients are provided here (in $\text{cost}$) , so we don't need to provide an optimization objective later.


```python
model = gp.Model('SupplyNetworkDesign')
flow = model.addVars(arcs, obj=cost, name="flow")
```

    Restricted license - for non-production use only - expires 2024-10-28
    



First constraints require the total flow along arcs leaving a center to be at most as large as the supply capacity of that center.


```python
# Center capacity limits

centers = supply.keys()
center_flow = model.addConstrs((gp.quicksum(flow.select(center, '*')) <= supply[center]
                                 for center in centers), name="center")
```

Next constraints require the total flow along arcs entering a landfill to be equal to the demand from that landfill.


```python
# landfill demand

landfills = demand.keys()
landfill_flow = model.addConstrs((gp.quicksum(flow.select('*', landfill)) == demand[landfill]
                                  for landfill in landfills), name="landfill")
```

Final constraints relate to depots.  The first constraints require that the total amount of waste entering the depot must equal the total amount leaving.


```python
# Depot flow conservation

depots = through.keys()
depot_flow = model.addConstrs((gp.quicksum(flow.select(depot, '*')) == gp.quicksum(flow.select('*', depot))
                               for depot in depots), name="depot")
```

The second set limits the waste passing through the depot to be at most equal the throughput of that depot.


```python
# Depot throughput

depot_capacity = model.addConstrs((gp.quicksum(flow.select('*', depot)) <= through[depot]
                                   for depot in depots), name="depot_capacity")
```

Optimize the model


```python
model.optimize()
```

    Gurobi Optimizer version 10.0.0 build v10.0.0rc2 (linux64)
    
    CPU model: Intel(R) Xeon(R) CPU @ 2.20GHz, instruction set [SSE2|AVX|AVX2]
    Thread count: 1 physical cores, 2 logical processors, using up to 2 threads
    
    Optimize a model with 16 rows, 29 columns and 65 nonzeros
    Model fingerprint: 0x6a0e70a2
    Coefficient statistics:
      Matrix range     [1e+00, 1e+00]
      Objective range  [4e-01, 2e+00]
      Bounds range     [0e+00, 0e+00]
      RHS range        [2e+04, 4e+05]
    Presolve removed 1 rows and 0 columns
    Presolve time: 0.03s
    Presolved: 15 rows, 29 columns, 64 nonzeros
    
    Iteration    Objective       Primal Inf.    Dual Inf.      Time
           0    3.8200000e+05   3.625000e+04   0.000000e+00      0s
           7    5.4100000e+05   0.000000e+00   0.000000e+00      0s
    
    Solved in 7 iterations and 0.04 seconds (0.00 work units)
    Optimal objective  5.410000000e+05
    

---
## Analysis

Waste demand from all of our landfillss can be satisfied for a total cost of $\$541,000$. The optimal plan is as follows.


```python
product_flow = pd.DataFrame(columns=["From", "To", "Flow"])
for arc in arcs:
    if flow[arc].x > 1e-6:
        product_flow = product_flow.append({"From": arc[0], "To": arc[1], "Flow": flow[arc].x}, ignore_index=True)  
product_flow.index=[''] * len(product_flow)
product_flow
```





  <div id="df-b2850399-d306-46cd-a3f1-f1e65d7f9867">
    <div class="colab-df-container">
      <div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>From</th>
      <th>To</th>
      <th>Flow</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th></th>
      <td>NewYork</td>
      <td>C1</td>
      <td>100000.0</td>
    </tr>
    <tr>
      <th></th>
      <td>NewYork</td>
      <td>C6</td>
      <td>40000.0</td>
    </tr>
    <tr>
      <th></th>
      <td>NewJersey</td>
      <td>Brooklyn</td>
      <td>100000.0</td>
    </tr>
    <tr>
      <th></th>
      <td>NewJersey</td>
      <td>Queens</td>
      <td>110000.0</td>
    </tr>
    <tr>
      <th></th>
      <td>NewJersey</td>
      <td>StatenIsland</td>
      <td>80000.0</td>
    </tr>
    <tr>
      <th></th>
      <td>Brooklyn</td>
      <td>C2</td>
      <td>20000.0</td>
    </tr>
    <tr>
      <th></th>
      <td>Brooklyn</td>
      <td>C4</td>
      <td>70000.0</td>
    </tr>
    <tr>
      <th></th>
      <td>Brooklyn</td>
      <td>C5</td>
      <td>10000.0</td>
    </tr>
    <tr>
      <th></th>
      <td>Queens</td>
      <td>C5</td>
      <td>110000.0</td>
    </tr>
    <tr>
      <th></th>
      <td>StatenIsland</td>
      <td>C3</td>
      <td>80000.0</td>
    </tr>
  </tbody>
</table>
</div>
      <button class="colab-df-convert" onclick="convertToInteractive('df-b2850399-d306-46cd-a3f1-f1e65d7f9867')"
              title="Convert this dataframe to an interactive table."
              style="display:none;">

  <svg xmlns="http://www.w3.org/2000/svg" height="24px"viewBox="0 0 24 24"
       width="24px">
    <path d="M0 0h24v24H0V0z" fill="none"/>
    <path d="M18.56 5.44l.94 2.06.94-2.06 2.06-.94-2.06-.94-.94-2.06-.94 2.06-2.06.94zm-11 1L8.5 8.5l.94-2.06 2.06-.94-2.06-.94L8.5 2.5l-.94 2.06-2.06.94zm10 10l.94 2.06.94-2.06 2.06-.94-2.06-.94-.94-2.06-.94 2.06-2.06.94z"/><path d="M17.41 7.96l-1.37-1.37c-.4-.4-.92-.59-1.43-.59-.52 0-1.04.2-1.43.59L10.3 9.45l-7.72 7.72c-.78.78-.78 2.05 0 2.83L4 21.41c.39.39.9.59 1.41.59.51 0 1.02-.2 1.41-.59l7.78-7.78 2.81-2.81c.8-.78.8-2.07 0-2.86zM5.41 20L4 18.59l7.72-7.72 1.47 1.35L5.41 20z"/>
  </svg>
      </button>

  <style>
    .colab-df-container {
      display:flex;
      flex-wrap:wrap;
      gap: 12px;
    }

    .colab-df-convert {
      background-color: #E8F0FE;
      border: none;
      border-radius: 50%;
      cursor: pointer;
      display: none;
      fill: #1967D2;
      height: 32px;
      padding: 0 0 0 0;
      width: 32px;
    }

    .colab-df-convert:hover {
      background-color: #E2EBFA;
      box-shadow: 0px 1px 2px rgba(60, 64, 67, 0.3), 0px 1px 3px 1px rgba(60, 64, 67, 0.15);
      fill: #174EA6;
    }

    [theme=dark] .colab-df-convert {
      background-color: #3B4455;
      fill: #D2E3FC;
    }

    [theme=dark] .colab-df-convert:hover {
      background-color: #434B5C;
      box-shadow: 0px 1px 3px 1px rgba(0, 0, 0, 0.15);
      filter: drop-shadow(0px 1px 2px rgba(0, 0, 0, 0.3));
      fill: #FFFFFF;
    }
  </style>

      <script>
        const buttonEl =
          document.querySelector('#df-b2850399-d306-46cd-a3f1-f1e65d7f9867 button.colab-df-convert');
        buttonEl.style.display =
          google.colab.kernel.accessAllowed ? 'block' : 'none';

        async function convertToInteractive(key) {
          const element = document.querySelector('#df-b2850399-d306-46cd-a3f1-f1e65d7f9867');
          const dataTable =
            await google.colab.kernel.invokeFunction('convertToInteractive',
                                                     [key], {});
          if (!dataTable) return;

          const docLinkHtml = 'Like what you see? Visit the ' +
            '<a target="_blank" href=https://colab.research.google.com/notebooks/data_table.ipynb>data table notebook</a>'
            + ' to learn more about interactive tables.';
          element.innerHTML = '';
          dataTable['output_type'] = 'display_data';
          await google.colab.output.renderOutput(dataTable, element);
          const docLink = document.createElement('div');
          docLink.innerHTML = docLinkHtml;
          element.appendChild(docLink);
        }
      </script>
    </div>
  </div>



