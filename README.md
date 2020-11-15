# py2pddl (Python to PDDL)

Write your planning task as Python classes, then translate
them to PDDL files. We will use PDDL with `(:requirements :strips :typing)`, i.e.
the actions only use positive preconditions and deterministic effects, and
we use 'types' like in OOP to represent sets of objects.

The library is written with these considerations:

* To quickly define the domain and problem using some **boilerplate**
* To define the domain and problem using a **familiar syntax**, prefereably similar to PDDL
* To be informed of wrong **types**
* To define ground predicates **cleanly**
* To **reduce the use of strings** while defining ground predicates

---

* [Requirements](##requirements)
* [Installation](##installation)
* [Quick start in 5 steps](##quick-start-in-5-steps)
* [Resources](##resources)

## Requirements

* Python 3.6
* [python-fire](https://github.com/google/python-fire) (`pip install fire`)

## Installation

```bash
pip install git+https://github.com/remykarem/py2pddl#egg=py2pddl
```

## Quick start in 5 steps

We will use the following problem:

![aircargoproblem.png](aircargoproblem.png)

### 1. Set up boilerplate

Run

```text
python -m py2pddl.init aircargo.py
```

and enter the following:

```text
Name: AirCargo
Types (separated by space): cargo airport plane
Predicates (separated by space): plane_at cargo_at in_
Actions (separated by space): load unload fly
```

### 2. Define the domain

In the `aircargo.py` source file, there is a class called `AirCargoDomain`.
The structure of the class is similar to how a PDDL domain should be defined.

* Name of the domain is the name of the Python class (`AirCargoDomain`).
* Types are defined as class variables at the top (`Plane`, `Cargo`, `Airport`).
* Predicates are defined as instance methods decorated with `@predicate`.
* Actions are defined as instance methods decorated with `@action`.

Now, complete the class definition such that it looks like this:

```python
from py2pddl import Domain, create_type
from py2pddl import predicate, action

class AirCargoDomain(Domain):

    Plane = create_type("Plane")
    Cargo = create_type("Cargo")
    Airport = create_type("Airport")

    @predicate(Cargo, Airport)
    def cargo_at(self, c, a):
        pass

    @predicate(Plane, Airport)
    def plane_at(self, p, a):
        pass

    @predicate(Cargo, Plane)
    def in_(self, c, p):
        pass

    @action(Cargo, Plane, Airport)
    def load(self, c, p, a):
        precond = [self.cargo_at(c, a), self.plane_at(p, a)]
        effect = [~self.cargo_at(c, a), self.in_(c, p)]
        return precond, effect

    @action(Cargo, Plane, Airport)
    def unload(self, c, p, a):
        precond = [self.in_(c, p), self.plane_at(p, a)]
        effect = [self.cargo_at(c, a), ~self.in_(c, p)]
        return precond, effect

    @action(Plane, Airport, Airport)
    def fly(self, p, orig, dest):
        precond = [self.plane_at(p, orig)]
        effect = [~self.plane_at(p, orig), self.plane_at(p, dest)]
        return precond, effect
```

Note:

* To create a new type `Car`, simply add `Car = create_type("Car")` at the top
of the class definition.
* The positional arguments of `@predicate` and `@action` decorators
are the types of the respective arguments.
* Methods decorated with `@predicate` should have empty bodies.
* Methods decorated with `@action` return a tuple of two lists.

### 3. Define the problem

In the bottom part of `aircargo.py`, there is another class called `AirCargoProblem`.
Again, the structure of the class is similar to how a PDDL problem should be defined.

* Name of the domain is the name of the Python class (`AirCargoProblem`).
* Objects are defined as the instance attributes in the `__init__` method.
* Initial states are defined as a methods decorated with `@init`.
* Goal is defined as an instance methods decorated with `@goal`.

Complete the class definition as follows:

```python
from py2pddl import create_objs
from py2pddl import goal, init

class AirCargoProblem(AirCargoDomain):

    def __init__(self):
        super().__init__()
        self.cargos = create_objs(AirCargoDomain.Cargo, [1, 2], None, "c")
        self.planes = create_objs(AirCargoDomain.Plane, [1, 2], None, "p")
        self.airports = create_objs(
            AirCargoDomain.Airport, ["sfo", "jfk"], False)

    @init
    def init(self):
        at = [self.cargo_at(self.cargos[1], self.airports["sfo"]),
              self.cargo_at(self.cargos[2], self.airports["jfk"]),
              self.plane_at(self.planes[1], self.airports["sfo"]),
              self.plane_at(self.planes[2], self.airports["jfk"]),]
        return at

    @goal
    def goal(self):
        return [self.cargo_at(self.cargos[1], self.airports["jfk"]),
                self.cargo_at(self.cargos[2], self.airports["sfo"])]
```

Note:

* `create_objs` return a dictionary whose keys are `[1,2]` (for cargos and planes)
and `["sfo", "jfk"]` for airports. This allows cleaner access to these objects
while defining ground predicates, which usually can get pretty messy.
* The PDDL objects defined in the `__init__` are meant to be used across
the 2 instance methods.

### 4. Parse

* To generate the PDDL files from the command line, run

    ```text
    python -m py2pddl.parse aircargo.py
    ```

* You can also import the parsing function from the module

    ```python
    from py2pddl import parse
    parse("aircargo.py")
    ```

* The class itself also contains methods to generate the domain
and problem PDDL files separately. These methods were inherited from
`Domain`.

    ```python
    from flying import GridCarProblem

    p = AirCargoProblem()
    p.generate_domain_pddl()
    p.generate_problem_pddl()
    ```

Here is the generated `domain.pddl` file.

```text
(define
	(domain somedomain)
	(:requirements :strips :typing)
	(:types
		airport
		cargo
		plane
	)
	(:predicates
		(cargo_at ?c - cargo ?a - airport)
		(in_ ?c - cargo ?p - plane)
		(plane_at ?p - plane ?a - airport)
	)
	(:action fly
		:parameters (?p - plane ?orig - airport ?dest - airport)
		:precondition (plane_at ?p ?orig)
		:effect (and (not (plane_at ?p ?orig)) (plane_at ?p ?dest))
	)
	(:action load
		:parameters (?c - cargo ?p - plane ?a - airport)
		:precondition (and (cargo_at ?c ?a) (plane_at ?p ?a))
		:effect (and (not (cargo_at ?c ?a)) (in_ ?c ?p))
	)
	(:action unload
		:parameters (?c - cargo ?p - plane ?a - airport)
		:precondition (and (in_ ?c ?p) (plane_at ?p ?a))
		:effect (and (cargo_at ?c ?a) (not (in_ ?c ?p)))
	)
)
```

And here is the generated `problem.pddl` file.

```text
(define
	(problem someproblem)
	(:domain somedomain)
	(:objects
		sfo jfk - airport
		c1 c2 - cargo
		p1 p2 - plane
	)
	(:init (cargo_at c1 sfo) (cargo_at c2 jfk) (plane_at p1 sfo) (plane_at p2 jfk))
	(:goal (and (cargo_at c1 jfk) (cargo_at c2 sfo)))
)
```

Then use your favourite planner like [Fast Downward](https://github.com/aibasel/downward).
To output a plan. Here's the plan generated from the above PDDL:

```text
(load c1 p1 sfo)
(fly p1 sfo jfk)
(load c2 p1 jfk)
(unload c1 p1 jfk)
(fly p1 jfk sfo)
(unload c2 p1 sfo)
; cost = 6 (unit cost)
```

See more examples in the `pddl/` folder.

### 5. Generate the problems dynamically

If you want the problem PDDL to be more dynamic if you have
changing inits and goals, you could use dictionaries and
specify in the `init` or `goal` keyword argument.

```python
p.generate_problem_pddl(
    goal={"cargo": "C2"})
```

## Resources

* [PDDL4J](https://github.com/pellierd/pddl4j)
* [Fast Downward](https://github.com/aibasel/downward)
