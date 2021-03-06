class: middle, center, title-slide

# Large-scale Data Systems

Lecture 5: Consensus

<br><br>
Prof. Gilles Louppe<br>
[g.louppe@uliege.be](g.louppe@uliege.be)

---

# Today

- *Most important* abstraction in distributed systems: **consensus**.
- Builds upon broadcast and failure detectors.
- From consensus, we will build:
    - total order broadcast
    - replicated state machines
    - ... and almost all higher level distributed fault-tolerant applications!

---

class: middle, center, black-slide

.width-80[![](figures/lec5/iceberg.png)]

???

Bump on Francois' question: is reality as complicated as the lasagna of abstractions we have been defining?
- Not necessarily, and we could certainly implement services in a more compact and optimized way.
- However, what our theoretical framework allows us to do properly is to ensure correctness.
- In all previous lectures, we have defined self-contained components with a limited number of features, which allowed to properly analyze them and prove their correctness.
- We also went from simple to more and more complicated components by building in a bottom-up fashion and relying on things for which correctness was already ensured.
- Directly starting from a complicated set of requirements and implementing everything in one go is possible, but would be much more difficult to achieve, debug and analyze.

Also, this core set of abstractions is here to stay. We will study some applications that make use of those principles, but they likely to be obsolete in 5 to 10 years. In fact, some of them already are. The theory is not.

---

class: middle

# Consensus

---

class: middle

**Consensus** is the problem of making processes all *agree* on one of the values they propose.

.center.width-100[![](figures/lec5/consensus.png)]
.caption[How do we reach an agreement?]

---

class: middle

## Motivation

Solving consensus is **key** to solving many problems in distributed computing:
- synchronizing replicated state machines;
- electing a leader;
- managing group membership;
- deciding to commit or abort distributed transactions.

Any algorithm that helps multiple processes *maintain common state* or to *decide on a future action*, in a model where processes may fail, involves **solving a consensus problem**.

---

# Consensus

.width-90[![](figures/lec5/consensus-interface.png)]

.exercice[Which is safety, which is liveness?]

???

- Termination: liveness
- Validity: safety
- Integrity: safety
- Agreement: safety

---

class: middle

## Sample execution

.center.width-100[![](figures/lec5/consensus-exec.png)]

.exercice[Does this satisfy consensus?]

???

Yes

---

# Uniform consensus

.width-90[![](figures/lec5/uniform-consensus-interface.png)]

---

class: middle

## Sample execution

.center.width-100[![](figures/lec5/consensus-exec.png)]

.exercice[Does this satisfy uniform consensus?]

???

No

---

class: center, middle

.width-100[![](figures/lec5/impossibility.png)]

---

class: middle

Are we done then? **No!**
- The FLP impossibility result holds for *asynchronous systems* only.
- Consensus can be implemented in **synchronous** and **partially synchronous** systems. (We will prove it!)
- The result only states that termination cannot be guaranteed.
    - Can we have other guarantees while maintaining a high probability of termination?

---

class: middle

# Consensus in fail-stop

---

# Hierarchical consensus

## Asumptions

- Assume a **perfect failure detector** (synchronous system).
- Assume processes $1, ..., N$ form an ordered **hierarchy** as given by $\text{rank}(p)$.
    - $\text{rank}(p)$ is a *unique* number between $1$ and $N$ (e.g., the pid).

---

class: middle

## Algorithm

- Hierarchical consensus ensures that the correct process with **the lowest rank imposes its value** on all the other processes.
    - If $p$ is correct and rank $1$, it imposes its values on all other processes by broadcasting its proposal.
    - If $p$ crashes immediately and $q$ is correct and rank 2, then it ensures that $q$'s proposal is decided.
    - The core of the algorithm addresses the case where $p$ is faulty but crashes after sending some of its proposal messages and $q$ is correct.
- Hierarchical consensus works *in rounds*, with a rotating *leader*.
    - At round $i$, process $p$ with rank $i$ is the leader.
    - It decides its proposal and broadcasts it to all processes.
    - All other processes that reach round $i$ wait before taking any actions, until they deliver this message or until they detect the crash of $p$.
        - upon which processes move to the next round.

---

class: middle

.width-70[![](figures/lec5/hierarchical-consensus.png)]

???

- What if a process does not propose?

---

class: middle

## Execution without failure

.center.width-100[![](figures/lec5/hierarchical-consensus-exec1.png)]

---

class: middle

## Execution with failure (1)

.center.width-100[![](figures/lec5/hierarchical-consensus-exec2.png)]

.exercice[- Uniform consensus?
- How many failures can be tolerated?]

???

- Not uniform consensus.
- $N-1$ failures at most.

---

class: middle

## Execution with failure (2)

.center.width-100[![](figures/lec5/hierarchical-consensus-exec3.png)]

---

class: middle

## Correctness

- *Termination*: Every correct process eventually decides some value.
    - Every correct node makes it to the round it is leader in.
        - If some leader fails, completeness of the FD ensures progress.
        - If leader correct, validity of BEB ensures delivery.
- *Validity*: If a process decides $v$, then $v$ was proposed by some process.
    - Always decide own proposal or adopted value.
- *Integrity*: No process decides twice.
    - Rounds increase monotonically.
    - A node only decides once in the round it is leader.
- *Agreement*: No two correct processes decide differently.
    - Take correct leader with minimum rank $i$.
        - By termination, it will decide $v$.
        - It will BEB $v$:
            - Every correct node gets $v$ and adopts it.
            - No older proposals can override the adoption.
            - All future proposals and decisions will be $v$.

---

# Hierarchical uniform consensus

- Same as *hierarchical consensus*, but must ensure uniform agreement.
- A round consists of **two communication steps**:
    - The leader BEB broadcasts its proposal
    - The leader collects acknowledgements
- Upon reception of all acknowledgements, **RB** broadcast the decision and decide at delivery.
    - This ensures that if a decision is made (at a faulty or correct process), then this decision will be made at all correct processes.
    - Processes proceed to the next round only if the current leader fails.

???

Different from Hierarchical consensus in the sense that we dont necessarily have to wait N rounds.

---

class: middle

.width-70[![](figures/lec5/uniform-hierarchical-consensus1.png)]

---

class: middle

.width-70[![](figures/lec5/uniform-hierarchical-consensus2.png)]

???

<span class="Q">[Q]</span> Why is reliable non-uniform broadcast sufficient to have uniform consensus?

---

class: middle

# Consensus in fail-noisy

---

class: middle

Would hierarchical consensus work with an *eventually perfect failure detector*?
- A false suspicion (i.e., a violation of strong accuracy) might lead to the **violation of agreement**.
- Not suspecting a crashed process (i.e., a violation of strong completeness) might lead to the **violation of termination**.

.center.width-100[![](figures/lec5/hierarchical-consensus-false.png)]

---

class: middle

## Towards consensus...

We will build a consensus component in **fail-noisy** by combining three abstractions:

1. an eventual leader detector
2. an epoch-change abstraction
3. an epoch consensus abstraction

---

# Eventual leader detector ($\Omega$)

.center[![](figures/lec5/ele.png)]

.exercice[This abstraction can be implemented from an eventually perfect failure detector. How?]

???

Elect as leader the correct process with the minimal rank. Eventually the set of correct processes will be accurate at all correct processes, resulting in eventual agreement.

---

# Epoch-Change ($ec$)

- Let us define an **Epoch-Change** abstraction, whose purpose it is to signal a change of epoch corresponding to the election of a leader.
- An indication event `StartEpoch` contains:
    - an epoch timestamp $ts$
    - a leader process $l$.

???

This component is emitting only.

---

class: middle

.center[![](figures/lec5/ec-interface.png)]

---

# Leader-based Epoch-Change

- Every process $p$ maintains two timestamps:
    - a timestamp $lastts$ of the last epoch that it locally started;
    - a timestamp $ts$ of the last epoch it attempted to start as a leader.
- When the leader detector makes $p$ trust itself, $p$ adds $N$ to $ts$ and broadcasts a `NewEpoch` message with $ts$.
- When $p$ receives a `NewEpoch` message with parameter $newts > lastts$ from $l$ and $p$ most recently trusted $l$, then $p$ triggers a `StartEpoch` message.
- Otherwise, $p$ informs the aspiring leader $l$ with a `NACK` that a new epoch could not be started.
- When $l$ receives a `NACK` and still trusts itself, it increments $ts$ by $N$ and tries again to start a new epoch.

---

class: middle

.width-60[![](figures/lec5/ec-impl.png)]

???

<span class="Q">[Q]</span> How many failures can be tolerated? $N-1$

---

class: middle

## Sample execution (1)

.center.width-100[![](figures/lec5/ec-exec1.png)]

---

class: middle

## Sample execution (2)

.center.width-100[![](figures/lec5/ec-exec2.png)]

---

class: middle

.exercice[
- What if $p_1$ fails only later, some time after the second `bebDeliver` event?
- What if instead of crashing, $p_1$ eventually trusts $p_2$?
- Could $p_1$ and $p_2$ keep bouncing NACKs to each other?]

---

# Epoch consensus ($ep$)

- Let us  define an **epoch consensus** abstraction, whose purpose is similar to consensus, but with the following simplifications:
    - Epoch consensus represents an *attempt* to reach consensus.
        - The procedure can be aborted when it does not decide or when the next epoch should already be started by the higher-level algorithm.
    - Every epoch consensus instance is identified by an epoch timestamp $ts$ and a designated leader $l$.
    - **Only the leader** proposes a value. Epoch consensus is required to decide *only when the leader is correct*.
- An instance **must terminate** when the application locally triggers an `Abort` event.
- The state of the component is initialized
    - with a higher timestamp than that of all instances it initialized previously;
    - with the state of the most recently locally aborted epoch consensus instance.

---

class: middle

.center[![](figures/lec5/econs-interface1.png)]

---

class: middle

.center[![](figures/lec5/econs-interface2.png)]

---

# Read/Write Epoch consensus

- Let us **initialize** the *Read/Write Epoch consensus* algorithm with the state of the most recently aborted epoch consensus instance.
    - The state contains a proposal $val$ and its associated timestamp $valts$.
    - Passing the state to the next epoch consensus serves the *validity* and *lock-in* properties.
- The algorithm involves *two rounds of messages* from the leader to all processes.
    - The leader **writes** its proposal value to all processes, who store the epoch timestamp and the value in their state, and acknowledge this to the leader.
    - When the leader receives enough acknowledgements, it decides this value.
    - However, if the leader of some previous epoch already decided some value $val$, then no other value should be decided (to not violate *lock-in*).
    - To prevent this, the leader first **reads** the state of the processes, which return `State` messages.
    - The leader receives a quorum of `State` messages and choses the value that comes with the highest timestamp, if one exists.
    - The leader *decides* and broadcasts its decision to all processes, which then decide too.

---

class: middle

![](figures/lec5/rwec-impl1.png)

---

class: middle

![](figures/lec5/rwec-impl2.png)

---

class: middle

## Sample execution (1)

.center.width-100[![](figures/lec5/econs-exec1.png)]

---

class: middle

## Sample execution (2)

.center.width-100[![](figures/lec5/econs-exec2.png)]

???

Note how in the read phase p1 could proceed as soon as it obtains a quorum.

Quorum: minimum number of votes that a distributed transaction must obtain in order to perform an operation.


---

class: middle

## Sample execution (3)

.center.width-100[![](figures/lec5/econs-exec3.png)]

.exercice[What is wrong in this execution?]

???

$p1$ should not proceed since 2 is not > 2.

---

class: middle

## Sample execution (4a)

.center.width-100[![](figures/lec5/econs-exec4a.png)]

---

class: middle

## Sample execution (4b)

.center.width-100[![](figures/lec5/econs-exec4b.png)]

---

class: middle

## Correctness

Assume a **majority of correct processes**, i.e. $N > 2f$, where $f$ is the number of crash faults.

- *Lock-in*: If a correct process has ep-decided $v$ in an epoch consensus with timestamp $ts' \leq ts$, then no correct process ep-decides a value different from $v$.
    - If some process ep-decided $v$ at $ts' < ts$, then it decided after receiving a `Decided` message with $v$ from leader $l'$ of epoch $ts'$.
    - Before sending this message, $l'$ had broadcast a `Write` containing $v$ and collected `Accept` messages.
    - These responding processes set their variables $val$ to $v$ and $valts$ to $ts'$.
    - At the next epoch, the leader sent a `Write` message with the previous $(ts', v)$ pair and collected `Accept` messages.
    - This pair has the highest timestamp with a non-null value.
    - This implies that the leader of this epoch can only ep-decides $v$.
    - This argument can be continued until $ts$, establishing lock-in.


---

class: middle

- *Validity*: If a correct process ep-decides $v$, then $v$ was ep-proposed by the leader $l'$ of some epoch consensus with timestamp $ts' \leq ts$ and leader $l'$.
    - If some process ep-decides $v$, it is because this value was delivered from a `Decided` message.
    - Furthermore, every process stores in $val$ only the value received in a `Write` message from the leader.
    - In both cases, this value comes from `tmpval` of the leader.
    - In any epoch, the leader sets `tmpval` only to the value it ep-proposed or to some value it received in a `State` message from another process.
    - By backward induction, $v$ was ep-proposed by the leader in some epoch $ts' \leq ts$.
- *Uniform agreement* + *integrity*: No two processes ep-decide differently + Every correct process ep-decides at most once.
    - $l$ sends the same value to all processes in the `Decided` message.
- *Termination*: If the leader $l$ is correct, has ep-proposed a value, and no correct process aborts this epoch consensus, then every correct process eventually ep-decides some value.
    - When $l$ is correct and no process aborts the epoch, then every process eventually receives a `Decide` message and ep-decides.

???

<span class="Q">[Q]</span> What may go wrong if we do not assume a majority of correct processes?

---

# Leader-Driven consensus

- Let us now combine the epoch-change and the epoch consensus abstractions to
form the **leader-driven consensus** algorithm.
- We will write the *glue* to repeatedly run epoch consensus until epoch changes stabilize and all decisions are taken.
- The algorithm provides *uniform consensus* in *fail-noisy*.

---

class: middle

Leader-driven consensus runs through a **sequence of epochs**, triggered by `StartEpoch` events from
the epoch-change primitive:

- The current epoch timestamp is $ets$ and the associated leader is $l$.
- The `StartEpoch` events determine the timestamp $newts$ and the leader $newl$ of the next epoch consensus instance to start.
- To switch from one epoch consensus to the next, the algorithm aborts the running epoch consensus instance, obtains its state and initializes the next epoch consensus instance with it.
- As soon as a process has obtained a proposal value $v$ for consensus and is the leader of the current epoch, it ep-proposes this value for epoch consensus.
- When the current epoch ep-decides a value, the process also decides this value for consensus.
- The process continue to participate in the consensus to help other processes decide.

---

class: middle

![](figures/lec5/ld-consensus-impl1.png)

---

class: middle

![](figures/lec5/ld-consensus-impl2.png)

---

class: middle

## Sample execution

.center.width-100[![](figures/lec5/ld-consensus-exec.png)]

???

Every process proposes a different value and then starts epoch 6 with leader q.
The open circle depicts that process q is the leader and the arrows show that it
drives the message exchange in the epoch consensus instance, which is
implemented by Algorithm 5.6.

Process q writes its proposal x, but only process r receives it
and sends an ACCEPT message before the epoch ends; hence, process r updates its
state to (6, x). Epoch 8 with leader s starts subsequently and the processes
abort the epoch consensus instance with timestamp 6.

At process r, epoch 8 starts much later, and r neither receives nor sends any
message before the epoch is aborted again. Note that the specification of
epoch-change would also permit that process r never starts epoch 8 and moves
from epoch 6 to 11 directly. Process s is the leader of epoch 8. It finds no
highest value different from ⊥ and writes its own proposal z. Subsequently, it
sends a DECIDED message and process p ep-decides and uc-decides z in epoch 8.

Note that p, q, and s now have state (8, z). Then process s crashes and epoch 11
with leader r starts. The state of r is still (6, x); but it reads value z from
p or q, and therefore writes z. As r is correct, all remaining processes
ep-decide z in this epoch consensus instance and, consequently, also q and r
uc-decide z.

It could also have been that process p crashed immediately after
deciding z; in this case, the remaining processes would also have decided z due
to the lock-in and agreement properties of epoch consensus. This illustrates
that the algorithm provides uniform consensus.

---

class: middle

## Correctness

- *Validity*: If a process decides $v$, then $v$ was proposed by some process.
    - A process uc-decides $v$ only when it has ep-decided $v$ in the current epoch consensus.
    - Every decision can be attributed to a unique epoch and to a unique instance of epoch consensus.
    - Let $ts^\*$ be the smallest timestamp of an epoch consensus in which some process decides $v$.
    - According to the validity property of epoch consensus, this means $v$ was ep-proposed by the leader of some epoch whose timestamp is a most $ts^\*$.
    - Since a process only ep-proposes $val$ when $val$ has been uc-proposed for consensus, the *validity* property follows for processes that uc-decide in epoch $ts^\*$.
    - The argument extends to $ts > ts^\*$ because the lock-in property of epoch consensus forces processes to ep-decide $v$ only, which in turn make them uc-decide.

---

class: middle

- *Uniform agreement*: No two processes decide differently.
    - Every decision attributed to an ep-decision of some epoch consensus instance.
    - If two correct processes decide when they are in the same epoch, then the uniform agreement of epoch consensus ensures the decisions are the same.
    - If they decide in different epoch, the lock-in property establishes uniform agreement.

---

class: middle

- *Integrity*: No process decides twice.
    - The `decided` flag in the algorithm prevents multiple decisions.
- *Termination*: Every correct process eventually decides some value.
    - Because of *eventual leadership* of the epoch-change primitive, there is some epoch with timestamp $ts$ and leader $l$ such that no further epoch starts and $l$ is correct.
    - From that instant, no further abortions are triggered.
    - The *termination* property of epoch consensus ensures that every correct process eventually ep-decides, and therefore uc-decides.

---

.grid[
.kol-3-4[
# Paxos

Leader-driven consensus is a modular formulation  of the **Paxos** consensus algorithm by Leslie Lamport.
]
.kol-1-4[.circle.width-100[![](figures/lec5/lamport.jpg)]]
]

.center.width-80[![](figures/lec5/paxos1.png)]

---

class: center, black-slide, middle

<iframe width="640" height="400" src="https://www.youtube.com/embed/s8JqcZtvnsM?cc_load_policy=1&hl=en&version=3" frameborder="0" allowfullscreen></iframe>

???

Formulation is different but equivalent.
- Leader election, epoch change =p Promise phase
- Lock-in = commitment

---

class: middle

# Total order broadcast

---

# Total order broadcast ($tob$)

- The **total-order (reliable) broadcast** (also known as *atomic broadcast*) abstraction
 ensures that all processes deliver the same messages in a *common global order*.
- Total-order broadcast is the key abstraction for maintaining consistency among multiple replicas
that implement one logical service.

---

class: middle, center

.width-100[![](figures/lec5/tob-interface.png)]

---

# Consensus-based TOB

- Messages are first disseminated using a *reliable broadcast instance*.
    - No particular order is imposed on the messages.
    - At any point in time, it may be that no two processes have the same sets of unordered messages.
- The processes use **consensus to decide on one set of messages** to be delivered, order the messages in this set, and finally deliver them.

---

class: middle

.width-80[![](figures/lec5/tob-impl.png)]

---

class: middle

## Sample execution

.center.width-90[![](figures/lec5/tob-exec.png)]

---

# Replicated state machines ($rsm$)

- A **state machine** consists of variables and commands that transform its state and produce some output.
- Commands are *deterministic* programs, such that the outputs are solely determined by the initial state and the sequence of commands.
- A state machine can be made *fault-tolerant* by replicating it on different processes.
- This can now be easily implemented simply by disseminating all commands to execute using a uniform total-order broadcast primitive.
- This gives a **generic recipe** to make any deterministic program distributed, consistent and fault-tolerant!

---

class: middle, center

![](figures/lec5/rsm.jpg)

---

class: middle

.center[![](figures/lec5/rsm-interface.png)]

---

# TOB-based Replicated state machines

![](figures/lec5/rsm-impl.png)

---

# Summary

- **Consensus** is the problem of making processes all *agree* on one of the values they propose.
- The **FLP impossibility result** states that no consensus protocol can be proven
to always terminate in an asynchronous system.
- In fail-stop, *Hierarchical Consensus* provides an implementation based on broadcast and failure detection.
- In fail-noisy, *Leader-Driven Consensus* achieves consensus by repeatedly running epoch consensus until all decisions are taken.
- The consensus primitive **greatly simplifies** the implementation of any fault-tolerant consistent distributed system.
    - Total-order broadcast
    - Replicated state machines

---

class: end-slide, center
count: false

The end.

---

# References

- Fischer, Michael J., Nancy A. Lynch, and Michael S. Paterson. "Impossibility of distributed consensus with one faulty process." Journal of the ACM (JACM) 32.2 (1985): 374-382.
- Lamport, Leslie. "The part-time parliament." ACM Transactions on Computer Systems (TOCS) 16.2 (1998): 133-169.
- Lamport, Leslie. "Paxos made simple." ACM Sigact News 32.4 (2001): 18-25.
