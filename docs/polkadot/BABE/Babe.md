
\(
   \def\skvrf{\mathsf{sk}^v}
   \def\pkvrf{\mathsf{pk}^v}
   \def\sksgn{\mathsf{sk}^s}
   \def\pksgn{\mathsf{pk}^s}
   \def\skac{\mathsf{sk}^a}
   \def\pkac{\mathsf{pk}^a} 
   \def\D{\Delta}
   \def\A{\mathcal{A}}
   \def\vrf{\mathsf{VRF}}
   \def\sgn{\mathsf{Sign}}
\)

# BABE


## Overview

BABE stands for '**B**lind **A**ssignment for **B**lockchain **E**xtension'. 
In BABE, we deploy Ouroboros Praos [2] style block production. 

In Ouroboros [1] and Ouroboros Praos [2], the best chain (valid chain) is the longest chain. In Ouroboros Genesis, the best chain can be the longest chain or the chain which is forked long enough and denser than the other chains in some interval. We have a different approach for the best chain selection based on GRANDPA and longest chain. In addition, we do not assume that all parties can access the current slot number which is more realistic assumption.

## BABE 
In BABE, we have sequential non-overlaping epochs \((e_1, e_2,...)\), each of which contains a number of sequential slots (\(e_i = \{sl^i_{1}, sl^i_{2},...,sl^i_{t}\}\)) up to some bound \(t\).  We randomly assign each slot to a party, more than one parties, or no party at the beginning of the epoch.  These parties are called a slot leader.  We note that these assignments are private.  It is public after the assigned party (slot leader) produces the block in his slot.

Each party \(P_j\) has at least two type of secret/public key pair:

*    Account keys \((\skac_{j}, pk^a_{j})\) which are used to sign transactions.
*    Session keys consists of two keys: Verifiable random function (VRF) keys \((\skvrf_{j}, \pkvrf_{j})\) and the signing keys for blocks \((\sksgn_j,\pksgn_j)\). 

We favor VRF keys being relatively long lived, but parties should update their associated signing keys from time to time for forward security against attackers causing slashing.  More details related to these key are [here](https://github.com/w3f/research/tree/master/docs/polkadot/keys).

Each party \(P_j\) keeps a local set of blockchains \(\mathbb{C}_j =\{C_1, C_2,..., C_l\}\). These chains have some common blocks (at least the genesis block) until some height.

We assume that each party has a local buffer that contains the transactions to be added to blocks. All transactions are signed with the account keys. 


### BABE with GRANDPA Validators \(\approx\) Ouroboros Praos

In this version, we do not worry about offline parties because GRANDPA validators are online by design. Therefore, we do not consider the security problems such as speed up attacks. It is almost the same as Ouroboros Praos except chain selection rule and the slot time adjustment.

We give some parameters probability related to be selected as a slot leader in this version before giving the protocol details. As in Ouroboros Praos, we define probability of being selected as

$$p_i = \phi_c(\alpha_i) = 1-(1-c)^{\alpha_i}$$

where \(\alpha_i\) is the relative stake of the party \(P_i\) and \(c\) is a constant. Improtantly, the function \(\phi\) is that it has the '**independent aggregation**' property, which informally means the probability of being selected as a slot leader does not increase as a party splits his stakes across virtual parties.

We use \(\phi\) to set a threshold \(\tau_i\) for each party \(P_i\): 

$$\tau_i = 2^{\ell_{vrf}}\phi_c(\alpha_i)$$

where \(\ell_{vrf}\) is the length of the VRF's first output.

BABE with GRANDPA validators consists of three phases:

1. #### Genesis Phase

In this phase, we manually produce the unique genesis block.

The genesis block contain a random number \(r_1\) for use during the first epoch for slot leader assignments, the initial stake's of the stake holders (\(st_1, st_2,..., st_n\)) and their corresponding session public keys (\(\pkvrf_{1}, \pkvrf_{2},..., \pkvrf_{n}\)), \((\pksgn_{1}, \pksgn_{2},..., \pksgn_{n}\)) and account public keys (\(\pkac_{1}, \pkac_{2},..., \pkac_{n}\)).

We might reasonably set \(r_1 = 0\) for the initial chain randomness, by assuming honesty of all validators listed in the genesis block.  We could use public random number from the Tor network instead however.

TODO: In the delay variant, there is an implicit commit and reveal phase provided some suffix of our genesis epoch consists of *every* validator producing a block and *all* produced blocks being included on-chain, which one could achieve by adjusting paramaters.

2. #### Normal Phase

In normal operation, each slot leader should produce and publish a block.  All other nodes attempt to update their chain by extending with new valid blocks they observe.

We suppose each party \(P_j\) has a set of chains \(\mathbb{C}_j\) in the current slot \(sl_k\) in the epoch \(e_m\).  We have a best chain \(C\) selected in \(sl_{k-1}\) by our selection scheme, and the length of \(C\) is \(\ell\text{-}1\). 

Each party \(P_j\) produces a block if he is the slot leader of \(sl_k\).  If the first output (\(d\)) of the following VRF is less than the threshold \(\tau_j\) then he is the slot leader.

$$\vrf_{\skvrf_{j}}(r_m||sl_{k}) \rightarrow (d, \pi)$$

Remark that the more \(P_j\) has stake, the more he has a chance to be selected as a slot leader. 


If \(P_j\) is the slot leader, \(P_j\) generates a block to be added on \(C\) in \(sk_k\). The block \(B_\ell\) should contain the slot number \(sl_{k}\), the hash of the previous block \(H_{\ell\text{-}1}\), the VRF output  \(d, \pi\), transactions \(tx\), and the signature \(\sigma = \sgn_{\sksgn_j}(sl_{k}||H_{\ell\text{-}1}||d||pi||tx))\). \(P_i\) updates \(C\) with the new block and sends \(B_\ell\).


![ss](https://i.imgur.com/Yb0LTJN.png =250x )



In any case (being a slot leader or not being a slot leader), when \(P_j\) receives a block \(B = (sl, H, d', \pi', tx', \sigma')\) produced by a party \(P_t\), it validates the block  with \(\mathsf{Validate}(B)\). \(\mathsf{Validate}(B)\) should check the followings in order to validate the block:

* if \(\mathsf{Verify}_{\pksgn_t}(\sigma')\rightarrow \mathsf{valid}\),

* if the party is the slot leader: \(\mathsf{Verify}_{\pkvrf_t}(\pi', r_m||sl) \rightarrow \mathsf{valid}\) and \(d' < \tau_t\). 

* if \(P_t\) did not produce another block for another chain in slot \(sl\) (no double signature),

* if there exists a chain \(C'\) with the header \(H\).

If the validation process goes well, \(P_j\) adds \(B\) to \(C'\). Otherwise, it ignores the block.


At the end of the slot, \(P_j\) decides the best chain with the function 'BestChain' (described below).




3. #### Epoch Update

Before starting a new epoch, there are certain things to be updated in the chain.
* Stakes of the parties
* (Session keys)
* Epoch randomness

If a party wants to update his stake for epoch \(e_{m+1}\), it should be updated until the beginning of epoch \(e_m\) (until the end of epoch \(e_{m-1}\)) for epoch \(e_{m+1}\). Otherwise, the update is not possible. We want the stake update one epoch before because we do not want parties to adjust their stake after seeing the randomness for the epoch \(e_{m+1}\). 

The new randomness for the new epoch is computed as in Ouroboros Praos [2]: Concatenate all the VRF outputs in blocks starting from the first slot of the epoch to the \(R/2^{th}\) slot of \(e_m\) (\(R\) is the epoch size). Assume that the concatenation is \(\rho\). Then the randomness in the next epoch:

$$r_{m+1} = H(r_{m}||m+1||\rho)$$

This also can be combined with VDF output to prevent little bias by the adversaries for better security bounds.

## Best Chain Selection

Given a chain set \\(\mathbb{C}_j\\) an the parties current local chain \(C_{loc}\), the best chain algorithm eliminates all chains which do not include the finalized block \(B\) by GRANDPA. Let's denote the remaining chains by the set \(\mathbb{C}'_j\).
Then the algorithm outputs the longest chain which do not for from \(C_{loc}\)  more than \(k\) blocks. It discards the chains which forks from \(C_{loc}\) more than \(k\) slots.


We do not use the chain selection rule as in Ouroboros Genesis [3] because this rule is useful for parties who become online after a period of time and do not have any  information related to current valid chain (for parties always online the Genesis rule and Praos is indistinguishable with a negligible probability). Thanks to Grandpa finality, the new comers have a reference point to build their chain so we do not need the Genesis rule.


## Relative Time

It is important for parties to know the current slot  for the security and completeness of BABE. Therefore, we show how a party realizes the notion of slots. Here, we assume partial synchronous channel meaning that any message sent by a party arrives at most \(\D\)-slots later. \(\D\) is not an unknown parameter.


Each party has a local clock and this clock does not have to be synchronized with the network. When a party receives the genesis block, it stores the arrival time as \(t_0\) as a reference point of the beginning of the first slot. We are aware of the beginning of the first slot is not same for everyone. We assume that this difference is negligible comparing to \(T\). Then each party divides their timeline in slots. 


**Obtaining Slot Number:** Parties who join BABE after the genesis block released or who lose notion of slot run the following protocol r obtain the current slot number with one of the following protocols. 


If a party \(P_j\) is a newly joining party, he downloads chains and receives blocks at the same time. After chains' download completed, he adds the valid blocks to the corresponding chains. Assuming that a slot number $sl$ is executed in a (local) time interval $[t_{start}, t_{end}]$ of party $P_j$, we have the following protocols for $P_j$ to output $sl$ and $t \in [t_{start}, t_{time}]$.


**- Median Algorithm:**
The party $P_j$ stores the arrival time $t_i$ of $n$ blocks with their corresponding slot time $sl_i$. Let us denote the stored arrival times of blocks by \(t_1,t_2,...,t_n\) whose slot numbers are \(sl_1,sl_2,...,sl_n\), respectively. Remark that these slot numbers do not have to be consecutive since some slots may be empty, with multiple slot leaders or the slot leader is offline, late or early. After storing $n$ arrival times, $P_j$ sorts the following list \(\{t_1+a_1T, t_2+a_2T,..., t_n+a_nT_\}\) where $a_i = sl - sl_i$. Here, $sl$ is a slot number that $P_j$ wants to learn at what time it corresponds in his local time. At the end. $P_j$  outputs the median of the ordered list as ($t$) and $sl$. 

![](https://i.imgur.com/yGYw9CL.png)

**Lemma 1:** Asuming that $\D$ is the maximum network delay in terms of slot number and \(\alpha\gamma(1-c)^\D \geq (1+\epsilon)/2\)  where \(\alpha\) is the honest stake and $\gamma\alpha$ is the honest and synchronized parties' stake and $\epsilon \in (0,1)$,  \(sl' - sl \leq \D\) with the median algorithm where $sl'$ the correct slot number of time $t$ with probability 1 - \exp(\frac{\delta^2\mu}{2} where $0 < \delta \leq \frac{\epsilon}{1+\epsilon}$ and $\mu = n(1+\epsilon)/2$.

**Proof:** Let us first assume that more than half of the blocks among $n$ blocks are sent by the honest and synchronized parties and  $t = t_i + a_iT$. Then, it means that more than half of the blocks sent on time. If the block of $sl_i$ is sent by an honest and synchronized party, we can conclude it is sent at earliest at $t_i' \leq t_i - \D T$. In this case, the correct slot number $sl'$ at time $t$ is $sl_i + \lceil\frac{t-t_i'}{T}\rfloor = sl_i + \lceil\frac{t_i + a_iT - t_i'}{T}\rfloor$. If $\D T = 0$, sl' = sl, otherwise $sl' \geq sl_i + \lceil\frac{a_iT + \DT}{T}\rfloor = sl+\D$.

If the median does not corresponds to time derived from an honest and synchronized parties' block, we can say that there is at least one honest and synchronized time after the median because more than half of the times are honest and synchronized.  Let's denote this time by $t_u + a_uT$.  Let's assume that the latest honest one in the ordered list is delayed $\D' \leq \D$ slots. It means that if the median was this one, $sl_u' - sl \leq \D'$ as shown above where $sl_u'$ is the correct slot number of time $t_u + a_uT$. Clearly, $sl \leq sl_u'$. Then, we can conclude that $sl' - sl \leq sl_u' - sl \leq \D' \leq \D$.

Now, we show the probability of having more than half honest and synchronized blocks in $n$ blocks. If \(\alpha\gamma(1-c)^\D \geq (1+\epsilon)/2\), then the blocks of honest and synchronized parties are added to the best chain even if there are $\D$ slots delay (it is discussed in the proof of Theorem 2) with the probability more than $(1+\epsilon)/2$. We define a random variable $X_v \in \{0,1\}$ which is 1 if $t_v$ is the arrival time of an honest and synchronized block. Then the expected number of honest and synchronized blocks among $n$ blocks is $\mu = n(1+\epsilon)/2$. We bound this with the Chernoff bound:

$$\mathsf{Pr}[ \sum_{v = 1}^n X_v \leq \mu(1-\delta)] \leq \exp(\frac{\delta^2\mu}{2}) $$

If 0 < \delta \leq \frac{\epsilon}{1+\epsilon}$, \mu(1-\delta) \geq n/2. This probability should be negligibly small with a $\delta \approx 1$ in order to have more than half honest and synchronized blocks in $n$ slots.
$$\tag*{\(\blacksquare\)}$$

If $\epsilon \geq 0.1$ and $\delta = 0.09$, the probability of having less than half is less than $0.06$ if $n \geq 1200$.


We give another algorithm called consistency algorithm below. This can be run after the median algorithm to verify or update $t$ later on.

**- Consistency Algorithm:** Let us first define *lower consistent blocks*. Given consecutive blocks \(\{B'_1, B'_2,...,B'_n \in C\) if for each block pair \(B'_u\) and \(B'_v\) which belong to the slots \(sl_u\) and \(sl_v\) (\(sl_u < sl_v\)), respectively are lower consistent for a party \(P_j\), if they arrive on \(t_u\) and \(t_v\) such that \(sl_v - sl_u = \lfloor\frac{t_v - t_u}{T}\rfloor\). We call *upper consistent* if for all blocks \(sl_v - sl_u = \lceil\frac{t_v - t_u}{T}\rceil\). Whenever \(P_j\) receives at least \(k\) either upper or lower consistent blocks, it outputs $t$ and $sl = sl_u + \lfloor\frac{t}-t_u}{T}\rfloor$ where \(sl_u\) is the slot of one of the blocks in the block set.


**Lemma 2:** Assuming that the network delay is at most \(\D\) and the honest parties' stake satisfies the condion in Theorem 2, \(P_j\)'s current slot is at most \(\D\)-behind or $2\D$ -behind of the correct slot $sl'$ at time $t$ (i.e., \(sl' - sl \leq \D\)).

**Proof:** According to Theorem 2, there is at least one block honestly generated by an honest party in \(k\) slot with probability \(1 - e^{-\Omega(k)}\). Therefore, one of the blocks in the lower oe upper consistent blocks belong to an honest party. We do our proof with lower consistent block. The upper consistent one is similar. Let's denote $\hat{\D} = \D$ or $\hat{\D} = 2\D$ 

If \(k\) blocks are lower consistent, then it means that all blocks are lower consistent with the honest block. 

If \(P_j\) chooses the arrival time and slot number of this honest block, then \(sl \leq sl' - \hat{\D}\) because the honest parties' block must arrive to \(P_j\) at most \(\hat{\D}\)-slots later. Now, we need to show that if \(P_j\) chooses the arrival time of a different block which does not have to be produced by an honest and synchronized party, then he is still at most \(\D\)-behind.

Assume that \(P_j\) picks \(sl_v > sl_u\) to compute $sl$ for $t$. We show that this computation is equal to \(sl = sl_u +\lfloor\frac{t - t_u}{T}\rfloor\). We know because of the lower consistency \(sl_v- sl_u = \lfloor\frac{t_v - t_u}{T}\rfloor\). $$sl = sl_v - \lfloor\frac{t_v - t_u}{T}\rfloor + \lfloor\frac{t - t_u}{T}\rfloor = sl_v + \lfloor\frac{t-t_v}{T}\rfloor$$

So \(P_i\) is going to obtain the same \(sl\) and $t$ with all blocks. Similarly, if \(P_i\) picks \(sl_v < sl_u\), he obtains \(sl\)

$$\tag*{\(\blacksquare\)}$$


There are two drawbacks of this protocol. One of drawbacks is that a party may never have \(k\) consistent blocks if an adversary randomly delays some blocks. In this case, \(P_i\) may never has consistent blocks. The other drawback is that if the honest block in $k$-consistent block is not a synchronized party then consistency algorithm performs worse than the median. However, this protocol can be used after the median protocol to update or verify the slot number with the consistency algorithm.  If this party sees $k$-consistent blocks and the slot number $sl$ obtained with the 
the consistency algorithm is less than slot number obtained from the median protocol, he updates it with $sl'$.



## Security Analysis

BABE with Grandpa validators is the same as Ouroboros Praos except the chain selection rule and  slot time extraction. Therefore, we need a new security analysis.




### Definitions
We give the definitions of  security properties before jumping to proofs.

**Definition 1 (Chain Growth (CG)) [1,2]:** Chain growth with parameters \(\tau \in (0,1]\) and \(s \in \mathbb{N}\) ensures that if the best chain owned by an honest party at the onset of some slot \(sl_u\) is \(C_u\), and the best chain owned by a honest party at the onset of slot \(sl_v \geq sl_v+s\) is \(C_v\), then the difference between the length of \(C_v\) and \(C_u\) is greater or equal than/to \(\tau s\).

**Definition 2 (Chain Quality (CQ)) [1,2]:** Chain quality with parameters \(\mu \in (0,1]\) and \(k \in \mathbb{N}\) ensures that the ratio of honest blocks in any \(k\) length portion of an honest chain is \(\mu\). 

**Definition 3 (Common Prefix)** Common prefix with parameters \(k \in \mathbb{N}\) ensures that any chains \(C_1, C_2\) possessed by two honest parties at the onset of the slots \(sl_1 < sl_2\) are such satisfies \(C_1^{\ulcorner k} \leq C_2\) where  \(C_1^{\ulcorner k}\) denotes the chain obtained by removing the last \(k'\) blocks from \(C_1\), and \(\leq\) denotes the prefix relation.


We define a new and stronger conmmon prefix property since we have a chance to finalize blocks earlier (smaller \(k\)) than the probabilistic finality that Ouroboros Praos [2] provides thanks to GRANDPA.

**Definition 4: (Strong Common Prefix (SCP))** Assuming that the common prefix property is satisfied with parameter \(k\), strong common prefix  with parameter \(k \in \mathbb{N}\) ensures that there exists \(k' < k\) and a slot number \(sl_1\) such that for any two chain \(C_1,C_2\) possessed by two honest parties at the onset of \(sl_1\) and \(sl_2\) where \(sl_1 < sl_2\), \(C_1^{\ulcorner k'} \leq C_2\).

In a nutshell, strong common prefix property ensures that there is a least one block which is finalized earlier than other blocks. 

It has been shown [4] that the persistence and liveness is satisfied if the block production ensure chain growth, chain quality and common prefix proerties. **Persistence** ensures that, if a transaction is seen in a block deep enough in the chain, it will stay there and **liveness** ensures that if a transaction is given as input to all honest players, it will eventually be inserted in a block, deep enough in the chain, of an honest player.

### Security Proof of BABE

We first prove that BABE satisfies chain growth, chain quality and strong common prefix properties in one epoch. Second, we prove that BABE's secure by showing that BABE satisfies persistence and liveness in multiple epochs. 

Before starting the security analysis, we give probabilities of being selected as a slot leader [2] or noone selected. We use the notations \(sl = \bot\) if a slot \(sl\) is empty,  \(sl = 0_{L}\) if \(sl\) is given to only one late honest party (\(\D\) behind the current slot) and \(sl = 0_S\)  if \(sl\) is given to only one synchronized honest party.

$$p_\bot=\mathsf{Pr}[sl = \bot] = \prod_{i\in \mathcal{P}}1-\phi(\alpha_i) = \prod_{i \in \mathcal{P}} (1-c)^{\alpha_i} = 1-c$$

$$p_{0_L} = \sum_{i\in\mathcal{H}_L}\phi(\alpha_i)(1-\phi(1-\alpha_i)) = \sum_{i\in\mathcal{H}_L}(1-(1-c)^{\alpha_i})(1-c)^{1-\alpha_i}$$

similarly,
$$p_{0_S} = \sum_{i\in\mathcal{H}_S}(1-(1-c)^{\alpha_i})(1-c)^{1-\alpha_i}$$

where \(\mathcal{P}\) is the set of indexes of all parties, \(\mathcal{H}_L\) is the set of indexes of all late and honest parties, \(\mathcal{H}_S\) is the set of indexes of all honest and synchronized parties with using Proposition 1 in [2]. 

We can bound \(p_{0_S}\) and \(p_{0_L}\) as \(p_{0_S} \geq \phi(\alpha_S)(1-c) \geq \alpha_Sc(1-c)\) and \(p_{0_L} \geq \phi(\alpha_L)c(1-c)\geq \alpha_L(1-c)\) where \(\alpha_S\) denotes the total relative stake of synchronized and honest parties and \(\alpha_L\) denotes the total relative stake of honest and late parties. For the rest, we denote \(\alpha = \alpha_S + \alpha_L = \gamma\alpha + \beta\alpha\) where \(\gamma + \beta = 1\) and \(\alpha\) is the relative stakes of honest parties.





In Lemma 1 and Lemma 2, we prove that a late party can be at most \(\D\) behind of the current slot. If a late party is a slot leader then his block is added to the best chain if there are at least \(2\D\) consecutive empty slots because he sends his block \(\D\) times later and his block may be received \(\D\) times later by other honest parties becuase of the network delay. Having late parties in BABE influences chain growth. 

**Theorem 1 (CG):** Let \(k, R, \D \in \mathbb{N}\) and let \(\alpha = \alpha_S + \alpha_L = \gamma\alpha + \beta\alpha\) is the total relative stake of honest parties. Then, the probability that an adversary \(\A\) makes BABE violate the chain growth property (Definition 1) with parameters \(s \geq 6 \D\) and \(\tau = \frac{\lambda c\alpha(\gamma+ \lambda \beta)}{6}\) throughout a period of \(R\) slots, is no more than \(2\D Rc  \exp({-\frac{(s-5\D)\lambda c\alpha(\gamma+ \lambda \beta)}{16\D}})\), where c denotes the constant \(\lambda = (1-c)^{\D}\).

**Proof:** We define two types of slot. We call a slot *\(2\D\)-right isolated* if the slot leader is one late party and the next \(2\D - 1\) slots are empty (no party is assigned). We call a slot *\(\D\)-right isolated* if the slot leader is only one synchronized honest party (not late party) and the next consecutive \(\D-1\) slots are empty. 

Now consider a chain owned by an honest party in \(sl_u\) and a chain owned by an honest party in \(sl_v \geq sl_u + s\). We need to show that honest parties' blocks are added most of times between \(sl_u\) and \(sl_v\). Therefore, we need to find the expected number of \(2\D\)-right isolated slots between \(sl_u\) and \(sl_v\)  given that the relative stake of late parties is \(\alpha_L = \beta \alpha\) and expected number of \(\D\)-right isolated slots given that the relative stake of synchronized honest parties is \(\alpha_S = \gamma\alpha\). Remark that a slot can be either \(2\D\)-right isolated or \(\D\)-right isolated or neither of them.

Consider the chains \(C_u\) and \(C_v\) in slots \(sl_u\) and \(sl_v\) owned by the honest parties, respectively where \(sl_u\) is the first slot of the epoch. We can guarantee that \(C_u\) is one of the chains of everyone in \(sl_u + 2\D\) and the chain \(C_v\) is one of the chains of everyone if it is sent in slot \(sl_v - 2\D\). Therefore, we are interested in slots between \(sl_u + 2\D\) and \(sl_v - 2\D\). Let us denote the set of these slots by \(S = \{sl_u + 2\D, sl_u+2\D+1,...,sl_v-2\D\}\). Remark that \(|S| = s-4\D\).

Now, we define a random variable \(X_t \in \{0,1\}\) where \(t\in S\). \(X_t = 1\) if \(t\) is \(2\D\) or \(\D\)-right isolated with respect to the probabilities \(p_\bot, p_{0_L}, p_{0_S}\). Then 
$$\mu = \mathbb{E}[X_t] = p_{0_S}p_\bot^{\D-1}+p_{0_L}p_\bot^{2\D-1} \geq \alpha_Sc(1-c)^{\D}+\alpha_Lc(1-c)^{2\D}.$$

With \( \lambda = (1-c)^{\D}\), \(\alpha = \alpha_L+\alpha_S = \beta\alpha+ \gamma \alpha\),

$$\mu \geq \lambda c\alpha(\gamma+ \lambda  \beta)$$

Remark that \(X_t\) and \(X_{t'}\) are independent if \(|t-t'| \geq 2\D\). Therefore, we define \(S_z = \{t\in S: t \equiv z \text{ mod }2\D\}\) where all \(X_t\) indexed by \(S_z\) are independent and \(|S_z| >  \frac{s-5\D}{2\D}\).

We apply a [Chernoff Bound](http://math.mit.edu/~goemans/18310S15/chernoff-notes.pdf) to each \(S_z\) with \(\delta = 1/2\).

$$\mathsf{Pr}[\sum_{t \in S_z}X_t < |S_z|\mu/2] \leq e^{-\frac{|S_z|\mu}{8}}\leq e^{-\frac{(s-4\D)\mu}{16\D}}$$


Recall that we want to bound the number of \(2\D\) and \(\D\)-right isolated slots. Let's call this number \(H\). If for all \(z\), \(\sum_{t \in S_z}X_t \geq |S_z|\mu/2\), then \(H = \sum_{t\in S} X_t \geq |S|\mu/2\). With union bound

$$\mathsf{Pr}[H < |S|\mu/2] \leq 2\D e^{-\frac{(s-5\D)\mu}{16\D}}$$

since \(\mu \geq \lambda c\alpha(\gamma+  \beta)\)

$$\mathsf{Pr}[H < |S|\frac{\lambda c\alpha(\gamma+ \lambda \beta)}{2}] \leq \mathsf{Pr}[H < |S|\mu/2]\leq 2\D \exp({-\frac{(s-5\D)\lambda c\alpha(\gamma+ \lambda \beta)}{16\D}})\space\space\space (2)$$


We find that in the first \(s\) slot of an epoch the chain grows \(\tau s\) block with the probability given in (2). Now consider the chain growth from slot \(sl_{u+1}\) to \(sl_{v+1}\). We know that the chain grows at least \(\tau s -1\) blocks between \(sl_{u+1}\) to \(sl_v\). So, the chain grows one block for sure if \(sl_{v+1}\) is \(\D\) or \(2\D\)-right isolated which with probability \(\alpha f c(\gamma+c\beta)\).    
If we apply the same for each \(sl > sl_u\) we obtain

$$2\D R \alpha \lambda  c(\gamma+\lambda\beta) \exp({-\frac{(s-5\D)\lambda c\alpha(\gamma+ \lambda \beta)}{16\D}})$$


given \(|S| = s-4\D\) and if \(s \geq 6\D\), \(|S| \geq \frac{s}{3}\) (\(\tau s = \frac{\lambda c\alpha(\gamma+ \lambda \beta)}{6}s \geq \frac{\lambda c\alpha(\gamma+ \lambda \beta)}{2}|S|\)). 

$$\tag*{\(\blacksquare\)}$$


**Theorem 2 (CQ):** Let \(k,\D \in \mathbb{N}\) and \(\epsilon \in (0,1)\). Let \(\alpha(\gamma+(1-c)^\D\beta)(1-c)^\D \geq (1+\epsilon)/2\) where \(\alpha = \alpha_S+\alpha_L = \gamma\alpha + \beta\alpha\) is the relative stake of honest parties. Then, the probability of an adversary \(\A\) whose relative stake is at most \(1-\alpha\) violate the chain growth property (Definition 2) with parammeters \(k\) and \(\mu = 1/k\) in \(R\) slots with probability at most \(Re^{-\Omega(k)}\).

**Proof (sketch):** The proof is very similar to the proof in [2]. It is based on the fact that the number of \(2\D\) and \(\D\) isolated slots are more than normal slots because of the assumption \((\alpha(\gamma+(1-c)^\D\beta)(1-c)^\D \geq (1+\epsilon)/2\). Remark that probability of having \(2\D\)-right isolated slot is \(\alpha\beta(1-c)^{2\D}\), having \(\D\)-right isolated slot is \(\alpha\gamma(1-c)^{\D}\) and sum of them are greater than 1/2 because of the assumption. 

$$\tag*{\(\blacksquare\)}$$


**Theorem 3 (SCP):** Let \(k,\D \in \mathbb{N}\) and \(\epsilon \in (0,1)\). Let \(\alpha(\gamma+(1-c)^\D\beta)(1-c)^\D \geq (1+\epsilon)/2\) where \(\alpha = \alpha_S+\alpha_L = \gamma\alpha + \beta\alpha\) is the relative stake of honest parties. Assuming that the GRANDPA finality gadget finalizes a block at most \(\kappa\) slots later with the probability \(\theta\), then the probability of an adversary \(\A\) whose relative stake is at most \(1-\alpha_L+\alpha_S\) violate the **strong** common prefix property with parammeter \(k\) in \(R\) slots with probability at most \((\theta Rc^{k+1}(1-c)^{\kappa - k} + (1-\theta))\exp(\ln R +  − \Omega(k-2\D))\).


**Proof Sketch:** First of all, we need to show that common prefix prefix property is satisfied with the honest relative-stake assumption. With a similar proof in [2] in Theorem 5, we can conclude that the common prefix property can be violated with the probability at most \(\exp(\ln R +  − \Omega(k-2\D))\). 

SCP property is violated if there is no  two chain \(C_1,C_2\) at any slot number \(sl_1\) such that \(C_1^{\ulcorner k'} \leq C_2\) and \(k'\leq k\) where \(C_2\) is a chain of an honest party in slot \(sl_2>sl_1\). If \(\kappa\) slots later, the chain grows more than \(k\), then the probabilistic finality passes the GRANDPA finality gadget. So, if for all \(\kappa\) slots after a non-empty slot, the chains grows more than \(k\) or the GRANDPA finality gadget finalize a block after \(\kappa\) slots then strong common prefix proerty is violated given that the GRANDPA finality gadget finalizes a block at most    \(\kappa\) slots later with the probability \(\theta\). This happens with at least the probability \(\theta Rc^{k+1}(1-c)^{\kappa - k} + (1-\theta)\) in \(R\) slots. 

$$\tag*{\(\blacksquare\)}$$

Remark than even if \(\theta = 0\), we still have the common prefix property as in Ouroboros Praos [2].


**Theorem 4 (Persistence and Liveness):** Fix parameters \(k, R, \D, L \in \mathbb{N}\), \(\epsilon \in (0,1)\) and \(r\). Let \(R \geq 24k/c(1+\epsilon)\) be the epoch length, \(L\) is the total lifetime of the system and $$\alpha(\gamma+(1-c)^\D\beta)(1-c)^\D \geq (1+\epsilon)/2$$
BABE satisfies persitence [2] with parameters \(k\) and liveness with parameters \(s \geq  12k/c(1+\epsilon)\) with probability \(1-\exp({\ln L\D c-\Omega(k-\ln tqk)})\) where \(r= 8tqk/(1+\epsilon)\) is the resetting power of the adversary during the randomness generation.


**Proof (Sketch):** The proof is very similar to Theorem 9 in [2]. The idea is as follows: The randomness for the next epoch is resettable until  the slot number \(R/2(1+\epsilon) > 12k/c(1+\epsilon)\). Now let's check the chain growth in \(s = 12k/c(1+\epsilon)\) with \(\tau= \frac{\lambda c\alpha(\gamma+ \lambda \beta)}{6}\) where \(\lambda = (1-c)^\D\).

$$\tau s = \frac{(1-c)^\D c\alpha(\gamma+ (1-c)^{\D} \beta)}{6}\frac{12k}{c(1+\epsilon)} \geq k $$

The stake distribution (for epoch $e_{j+2}$) which is updated until the end of epoch $e_j$ is finalized at latest in the slot number $12k/c(1+\epsilon)$ of epoch $e_{j+1}$. So it is finalized before the randomness of the next epoch's ($e_{j+2$) generated. In addition to this, chain growth property shows that there will be at least one honest block in the first $12k/c(1+\epsilon)$ slots. These two imply that the adversary cannot adapt his stake according to the random number for the next epoch \(e_{j+1}\) and this random number provides good randomness for the next epoch even though the adversary has capability of resetting \(r = 8tkq/(1+\epsilon)\) times (\(t\) is the number of corrupted parties and \(q\) is the maximum number of random-oracle queries for a party). So, the common prefix property still preserved with the dynamic staking. Therefore, we can conclude that persistence is satisfied thanks to the common prefix property of dynamic stake with the probability (comes from Theorem 1) $$2r\D Lf \exp({-\frac{(s-5\D)(1-c)^\D c\alpha(\gamma+ (1-c)^\D\beta)}{16\D}}) \space\space\space (3).$$ If we use the assumptions we can simplify this probability as \(\exp({\ln L\D c-\Omega(k-\ln tqk)}\).
Liveness is the result of the chain growth and chain quality properties.
$$\tag*{\(\blacksquare\)}$$



**These results are valid assuming that the signature scheme with account key is  EUF-CMA (Existentially Unforgible Chosen Message Attack) secure, the signature scheme with the session key is forward secure, and VRF realizing is realizing the functionality defined in [2].**


**Analysis With VDF:** TODO

If we use VDF in the randomness update for the next epoch, \(r = \mathsf{log}tkq\) disappers in \(p_{sec}\) because we have completely random value which do not depend on hashing power of the adversary.



## Practical Results

In this section, we find parameters of BABE in order to achieve the security in BABE. In addition to this, we show block time of BABE in worst cases (big network delays, many malicious parties) and in average case.

We fix the life time of the protocol as \(\mathcal{L}=2.5 \text{ years}  = 15768000\) seconds. Then we find the life time of the protocol  \(L = \frac{\mathcal{L}}{T}\). We find the network delay in terms of slot number with $\lfloor \frac{D}{T}\rfloor$ where $D$ is the network delay in seconds. Assuming that parties send their block in the beginning of their slots, $\lfloor\rfloor$ operation is the enough to compute the delay in terms of slots. 


The parameter $c$ is very critical because it specifies the number of empty slots because probability of having empty slot is $1-c$. If $c$ is very small, we have a lot of empty slots and so we have longer block time. If $c$ is big, we may not satisfy the the condition \(\alpha(\gamma+(1-c)^\D\beta)(1-c)^\D \geq (1+\epsilon)/2\) to apply the result of Theorem 4. So, we need to have a tradeoff between security and practicality. Therefore, we fix $c = 0.5$.

We need to satisfy two conditions $$\frac{1}{c}(\phi(\alpha\gamma)(1-c)^{\D-\alpha}(1-c)^{D-1}+ \phi(alpha\beta)(1-c)^{2\D-\alpha\beta} \geq \(\alpha(\gamma+(1-c)^\D\beta)(1-c)^\D \geq (1+\epsilon)/2\)\geq (1+\epsilon/2)$$ to apply the result of Theorem 4 and $$\frac{1}{c}(\phi(\alpha\gamma)(1-c)^{\D-\alpha}(1-c)^{D-1} \geq \(\alpha\gamma(1-c)^\D \geq (1+\epsilon)/2\) $$ to apply the result of Lemma 1 . Second condition implies the first one.  These conditions are satisfied given $c = 0.5$, $\D = 1$ and $\alpha = 0.65$ and $\gamma = 0.7$.




In order to find the average block time (i.e., the required time to add one block to the best chain), we need the expected number of $\D$ and $2\D$-right isolated slots in $L$ slots. However, we use a different definition of $\D$ and $2\D$-right isolated slots in this analysis.  A slot is $\D$ (resp. $2\D$-right ) isolated slot if the slot leaders are all honest and at least one syncronized (resp. all honest and late) and the next $\D-1$ (resp. $2\D-1$) slots are empty. If all parties honest and at least one of them is synchronized, then the synchronized party's blcok will be added because, he relaeases the block earlier than the other selected parties. Therefore, we need at least one synchronized and honest slot leader in the definition of $\D$-isolated slot. Remark that the definitions of $\D$ and $2\D$ right isolated slots are more relaxed than the definitions in the proof of Theorem 1 because we do not care the growth of other chains as we care in the security analysis. 


The probability of $2\D$-right isolated slot is  
$$p_{2\D} \geq \frac{\phi(\alpha\beta)(1-c)^{1-\alpha\beta}}{c}(1-c)^{2\D-1}.$ 
The probability of $\D$-right isolated slot is
$$p_{\D}\geq \frac{\phi(\alpha\gamma)(1-c)^{1-\alpha}}{c}(1-c)^{\D-1}.$$
The expected number of non-empty slot in $L$. The expected number of $\D$ and $2\D$-right isolated slot is  
$$\mathbb{E} = cL(p_{H_L}+p_{H_S})$$. Then, the block time $T_{block} \leq \frac{LT}{\mathbb{E}}$. Remark that we have already had the condition that $p_{H_L} + p{H_S} \geq (1+ \epsilon)/2$ from the security even if we have maximumum network delay.  In order to find the average block time, we do not consider the maximum network delay here. We consider the average network delay which 1 seconds. In the graph below, we have the average block time for each slot time in blue. In addition, the graph shows the maximum network delay resistance for each slot time in red.

![](https://i.imgur.com/rePtRjL.png)


If we choose \(\mathbf{c = 0.5}\), \(\gamma = 0.7, \alpha = 0.65\), $T \geq 1$ second and $D = 1$, we find that \(k > 150\)  to have a good security level in 2.5 years according to Theorem 4.  

Remark that \(k\) is the finality that is provided by BABE. Since we have GRANDPA on top of BABE, we expect much earlier finalization. This \(k\) value is valid when GRANDPA does not work properly. If \(k = 150\), the minimum **epoch length must be 7200 slots** (e.g., for $T = 1$ sec, one epoch should take 2 hours.) according to Theorem 4. However, if GRANDPA finalizes a block earlier than $k$, we do not need to wait for $R/2$ slots more after randomness leaked to finalize the stake distribution so we need less slot number in an epoch. 




## References

[1] Kiayias, Aggelos, et al. "Ouroboros: A provably secure proof-of-stake blockchain protocol." Annual International Cryptology Conference. Springer, Cham, 2017.

[2] David, Bernardo, et al. "Ouroboros praos: An adaptively-secure, semi-synchronous proof-of-stake blockchain." Annual International Conference on the Theory and Applications of Cryptographic Techniques. Springer, Cham, 2018.

[3] Badertscher, Christian, et al. "Ouroboros genesis: Composable proof-of-stake blockchains with dynamic availability." Proceedings of the 2018 ACM SIGSAC Conference on Computer and Communications Security. ACM, 2018.

[4] Aggelos Kiayias and Giorgos Panagiotakos. Speed-security tradeoffs in blockchain protocols. Cryptology ePrint Archive, Report 2015/1019, 2015. http://eprint.iacr.org/2015/1019