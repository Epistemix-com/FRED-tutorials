# Flu with Vaccine Model

## Introduction

This example builds on the the Flu with Behavior model (see `../flu-with-behavior`) by adding an `INFLUENZA_VACCINE` condition that allows agents to be exposed to a vaccine that protects the agent from influenza with a probability equal to `vaccine_effectiveness` by setting `INFLUENZA.sus = 0`.

## Review of code implementing the model

The code that implements the Flu with Vaccine model is contained in five `.fred` files:

- `main.fred`
- `simpleflu.fred`
- `stayhome.fred`
- `vaccine.fred`
- `parameters.fred`

### `main.fred`

This `main.fred` contains four basic components.
The first two will be familiar from the Simple Flu and Flu with Behavior models.
The simulation block defines the dates and physical location of the simulation.
A series of `include` statement refer to other `.fred` files to reference conditions, states, and variables necessary to run the simulation.

The Flu with Vaccine `main.fred` also directly defines a global variable.

```fred
variables {
    shared numeric flu_delay
    flu_delay = 90
}
```

In this case, it is the number of days to wait before vaccinating the first agents.
The next code chunk modifies the `INFLUENZA` condition (which was previously defined in `simpleflu.fred`)

```fred
condition INFLUENZA {
    meta_start_state = ImportDelay

    state ImportDelay {
        wait(24 * flu_delay)
        next(Import)
    }
}
```

Specifically, it directs the meta-agent to start in the `ImportDelay` state and transition to `Import` after the delay.
Looking back to `simpleflu.fred`, we can see that `meta_start_state = Import`.
Calling `include simpleflu.fred` prior to this new `condition` block in the `main.fred` script causes the settings in the initial file to be overwritten.
Now, rather than initializing in `Import` and immediately infecting some of the agents with `INFLUENZA`, the meta-agent will begin in `ImportDelay` and wait the parameterized delay before advancing to `Import` and assigning infections.


### `simpleflu.fred`

`simpleflu.fred` is identical to the file of the same name in the Simple Flu model.
It specifies the state update rules for the `INFLUENZA` Condition that models agents' status with respect to influenza infection.
Specifically, each agent is in one of the following five states:

1. Susceptible to infection with influenza
2. Exposed to influenza
3. Infectious and symptomatic with influenza
4. Infectious and asymptomatic with influenza
5. Recovered following infection with influenza and assumed to be immune

Refer to the tutorial on the Simple Flu Model for a complete explanation of the code in `simpleflu.fred`.
For the purpose of this tutorial, consider the following code snippet which specifies the State rules for the `InfectiousSymptomatic` and `Recovered` States belonging to the `INFLUENZA` condition:

```fred
    state InfectiousSymptomatic {
        INFLUENZA.trans = 1
        wait(24* lognormal(5.0,1.5))
        next(Recovered)
    }

    ...

    state Recovered {
        INFLUENZA.trans = 0
        wait()
        next()
    }
```

Here `INFLUENZA.trans = 1` and `INFLUENZA.trans = 0` are action rules executed by agents entering the `INFLUENZA.InfectiousSymptomatic` state to change their [transmissibility](https://docs.epistemix.com/projects/lang-guide/en/latest/chapter13.html#the-transmissibility-of-an-agent) of influenza to 1, and agents entering the `INFLUENZA.Recovered` state to change their transmissibility of influenza to 0 (i.e. they are non-infectious).
This change is accomplished by multiplying condition transmissiblity by the agent's transmissibility, but in this case `INFLUENZA` transmissibility is already equal to 1.
The wait rule `wait(24* lognormal(5.0,1.5))` causes each agent that becomes infectious and symptomatic to remain so for a number of days determined by sampling from a [lognormal distribution](https://docs.epistemix.com/projects/lang-guide/en/latest/chapter7.html#statistical-distribution-functions) with median=5.0 and dispersion=1.5. Finally the transition rule `next(Recovered)` causes agents to transition, deterministically, to the `Recovered` state once their period of infection has elapsed.
Once in `Recovered`, transmissibility is set to 0 and the agent waits indefinitely (`wait()`).
The `next()` transition has no effect, but is required to have any following rule.

### `stayhome.fred`

`stayhome.fred` is identical to the file of the same name in the Flu with Behavior model.
This file defines new behavior for agents: while symptomatic with an influenza infection (specifically, while in the `InfectedSymptomatic` state of the `INFLUENZA` condition), agents will probabilistically stay at home all day rather than attending work or school.


### `vaccine.fred`

`vaccine.fred` defines the variables, condition, and states (for both agents and the meta-agent) required to simulate basic vaccination behavior in our population.

The primary change to this simulation, and the entirety of this file, is to create an `INFLUENZA_VACCINE` condition with states that influence both this condition and the `INFLUENZA` condition.
Agents begin in `Start` and advance to `Considering` with the probability `willing_to_consider`, defined in the `variables` block of the current file.
Any agent that does not advance to `Considering` goes instead to the `Excluded` state for `INFLUENZA_VACCINE` and stays there permanently.
These agents will not gain `INFLUENZA` immunity except by contracting the illness.

```fred
    state Considering {
        INFLUENZA_VACCINE.sus = 1
        wait()
        next()
    }
```

Agents that move to `Considering` have their susceptibility to `INFLUENZA_VACCINE` set to 1 and then wait to have the vaccine transmitted to the by the meta-agent.

```fred
    state Decide {
        INFLUENZA_VACCINE.sus = 0
        wait(24)
        next(Taker)
    }

    state Taker {
        wait(24 * days_until_effective)
        next(Immune) with prob(vaccine_effectiveness)
        default(Failed)
    }
```

Once an agent in `Considering` is exposed by the meta-agent, they move to the `Decide` state.
These agents change their susceptibility to the vaccine to zero, wait 24 hours, and then "take" the vaccine, moving to the `Taker` state.
Here, agents wait until the vaccine becomes effective, defined in the `variables` block as `days_until_effective`, and then move to either `Immune` with the `vaccine_effectiveness` probability or `Failed` otherwise.
Agents in the `Immune` state modify their `INFLUENZA` susceptibility to zero and can no longer be infected with that condition.
The `Failed` state does not modify `INFLUENZA` susceptibility, mimicking a vaccination failure.
Agents in both of these states remain there indefinitely.

``` fred
    meta_start_state = ImportStart
    start_state = Start
    transmission_mode = proximity
    transmissibility = 1
    exposed_state = Decide

    ...

   state ImportStart {
        wait(24 * vaccine_delay)
        next(ImportVaccine)
    }

    state ImportVaccine {
        import_per_capita(initial_vaccines)
        wait()
        next()
    }
```

The remaining two states apply only to the meta-agent.
At the top of the `INFLUENZA_VACCINE` condition, the meta-agent is started in `ImportStart`.
We also define that the condition will be transmitted via proximity and that agents exposed to the condition will advance to the `Decide` state.

When the meta-agent begins in `ImportStart`, it waits the number of days defined by `vaccine_delay` (plus 1 hour) and then advances to `ImportVaccine`.
During each time step, the meta-agent changes state before the ordinary agents, so it is common to have the meta-agent wait 1 time step at the start, so that ordinary agent have time to execute the rules in their start state. In this case, some ordinary agents end up on the `Considering` state at time `0`, where they become susceptible to vaccination.
In this `ImportVaccine` state, the meta-agent exposes some proportion of the `Considering` agents to the vaccine (moving them to `Decide`).
This proportion is defined in the `variables` block as `initial_vaccines`, though we will alter this value in the `parameters.fred` file.

### `parameters.fred`

A `parameters.fred` file is also introduced in this model.
This is largely a convenience for programmatically modifying the simulation parameters with a bash script.
`parameters.fred` is the last file `included` in `main.fred`, allowing the contained values to overwrite any that were specified in earlier-imported files.
The `METHODS` file in this directory defines two different FRED jobs by sequentially creating two different `parameters.fred` files and running `fred_job` with each of those files.

## Sample Model Outputs

This model is run using the `METHODS` file in a
[FRED Local](https://docs.epistemix.com/projects/fred-local) container.
Running this file produces two sets of outputs that differ only in the proportion of the population that is originally vaccinated, 0% or 30%.
Ideally, there would be a testing step in development to make sure that vaccines are being transmitted and imparting immunity (or reducing susceptibility) in the way that is expected.
We can see that initially vaccinating 30% of the population reduces the cumulative infections substantially, providing evidence that
the `INFLUENZA_VACCINE` condition is functioning as expected.

![](figures/vaccine-tot.png)
<p class="caption"><span class="caption-text">Cumulative cases over time</span></p>


## Summary

In addition to the features from previous tutorial modules, this model also includes:

- A second, transmissible `condition` can alter the behavior of the original `condition` by directly changing susceptibility.
- The use of ordered imports overwrites earlier parameter values.
- An example of modifying parameters in successive simulations by rewriting `parameters.fred`.
