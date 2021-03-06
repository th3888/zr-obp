================================================
Overview of Off-Policy Evaluation
================================================


Setup
------

We consider a general contextual bandit setting.
Let :math:`{\cal A}=\{0,\ldots,m\}` be a finite set of :math:`m+1` *actions*, that the decision maker can choose from.
Let :math:`Y(\cdot): {\cal A}\rightarrow \mathbb{R}` denote a potential reward function that maps actions into rewards or outcomes, where :math:`Y(a)` is the reward when action :math:`a` is chosen (e.g., whether a fashion item as an action results in a click).
Let :math:`X` denote a *context* vector (e.g., the user's demographic profile) that the decision maker observes when picking an action.
We denote the finite set of possible contexts by :math:`{\cal X}`.
We think of :math:`(Y(\cdot),X)` as a random vector with unknown distribution :math:`G`.
Given a vector of :math:`(Y(\cdot),X)`, we define the mean reward function :math:`\mu: {\cal X} \times {\cal A} \rightarrow \mathbb{R}` as :math:`\mu (x, a) := \mathbb{E} [Y (a) | X=x ]`.

Let :math:`\mathcal{D} := \{(X_t,A_t,Y_t)\}_{t=1}^T` be historical logged bandit feedback with :math:`T` rounds of observations.
:math:`A_{t}` is a discrete variable indicating which action in :math:`\mathcal{A}` is chosen in round :math:`t`.
:math:`Y_t := Y_t(A_t)` and :math:`X_t` denote the reward and the context observed in round :math:`t`, respectively.

We call a function :math:`\pi: {\cal X} \rightarrow \Delta({\cal A})` a *policy*.
It maps each context :math:`x \in {\cal X}` into a distribution over actions, where :math:`\pi (a | x)` is the probability of taking action :math:`a` given :math:`x`.
We assume that a logged bandit feedback is generated by a *behavior policy* :math:`\pi_b` as follows:

* In each round :math:`t=1,...,T`, :math:`(Y_t(\cdot),X_t)` is i.i.d. drawn from distribution :math:`G`.
* Given :math:`X_t`, an action is randomly chosen based on :math:`\pi_b(\cdot|X_t)`, creating the action choice :math:`A_{t}` and the associated reward :math:`Y_t`.

Suppose that :math:`\pi_b` is fixed for all rounds, and thus :math:`A_t` is i.i.d. across rounds.
Because :math:`(Y_t(\cdot),X_t)` is i.i.d. across rounds and :math:`Y_t=Y(A_t)`, each observation :math:`(X_t,A_t,Y_t)` is i.i.d. across rounds.
Note that :math:`A_t` is independent of :math:`Y_t(\cdot)` conditional on :math:`X_t`.


Examples
---------
Our setup allows for many popular multi-armed bandit algorithms, as the following examples illustrate.

Random A/B testing
~~~~~~~~~~~~~~~~~~~~~
We always choose each action uniformly at random: :math:`\pi_b(\cdot|X) =\frac{1}{m+1}` always holds for any :math:`a \in {\cal A}` and :math:`X \in {\cal X}`.


Bernoulli Thompson Sampling
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
When the context :math:`X_t` is given, we sample the potential reward :math:`\tilde{Y}(a)` from the beta distribution :math:`Beta (S_{ta} + \alpha, F_{ta} + \beta) ` for each action, where :math:`S_{ta} = \sum_{t'=1}^{t-1} Y_{t'}D_{t'a}, F_{ta} = (t-1) - S_{ta}`.
:math:`\alpha, \beta` are the hyperparameters of the prior Beta distribution.
We then choose the action with the highest sampled potential reward, :math:`\argmax_{a^{\prime} \in {\cal A}}\tilde{Y}(a^{\prime})`.
As a result, this algorithm chooses actions with the following probabilities:

.. math::
    \pi(a|X_t) = \Pr\{a=\argmax_{a'\in {\cal A}}\tilde{Y}(a^{\prime})\}.


Prediction Target
-------------------------
We are interested in using the historical logged bandit data to estimate the following *policy value* of any given *evaluation policy* :math:`\pi_e` which might be different from :math:`\pi_b`:

.. math::
    V (\pi_e) := \mathbb{E}_{(Y(\cdot),X)\sim G}[\sum_{a=0}^mY(a)\pi_e(a|X)].

where the last equality uses the independence of :math:`A` and :math:`Y(\cdot)` conditional on :math:`X` and the definition of :math:`\pi_b(\cdot|X)`.
We allow the evaluation policy :math:`\pi_e` to be degenerate, i.e., it may choose a particular action with probability 1.
Estimating :math:`V(\pi_e)` before implementing :math:`\pi_e` in an online environment is valuable because :math:`\pi_e` may perform poorly and damage user satisfaction.
Additionally, it is possible to select an evaluation policy that maximizes the policy value by comparing their estimated performances.


Benchmark Estimators
-----------------------
There are several approaches to estimate the policy value of the evaluation policy.

Direct Method (DM)
~~~~~~~~~~~~~~~~~~~~
A widely-used method, DM, first learns a supervised machine learning model, such as random forest, ridge regression, and gradient boosting, to estimate the mean reward function.
DM then uses it to estimate the policy value as

.. math::
    \hat{V}_{DM} (\pi_e; \mathcal{D}, \hat{\mu})=\frac{1}{T}\sum_{t=1}^T\sum_{a=0}^m\hat{\mu}(X_t, a) \pi_e(a|X_t).

where :math:`\hat{\mu}(a| x)` is the estimated reward function.
If :math:`\hat{\mu}(a| x)` is a good approximation to the mean reward function, this estimator accurately estimates the policy value of the evaluation policy :math:`V^{\pi}`.
If :math:`\hat{\mu}(a| x)` fails to approximate the mean reward function well, however, the final estimator is no longer consistent.
The model misspecification issue is problematic because the extent of misspecification cannot be easily quantified from data :cite:`Farajtabar2018`.


Inverse Probability Weighting (IPW)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
To alleviate the issue with DM, researchers often use another estimator called IPW :cite:`Precup2000` :cite:`Strehl2010`.
IPW re-weights the rewards by the ratio of the evaluation policy and behavior policy as

.. math::
    \hat{V}_{IPW} (\pi_e; \mathcal{D})=\frac{1}{T}\sum_{t=1}^T  Y_t \frac{\pi_e(A_t|X_t)}{\pi_b(A_t | X_t)}.

When the behavior policy is known, the IPW estimator is unbiased and consistent for the policy value.
However, it can have a large variance, especially when the evaluation policy significantly deviates from the behavior policy.


Doubly Robust (DR)
~~~~~~~~~~~~~~~~~~~

The final approach is DR :cite:`Dudik2014`, which combines the above two estimators as

.. math::
    \hat{V}_{DR} (\pi_e; \mathcal{D}, \hat{\mu})
    =\hat{V}_{DM} (\pi_e; \mathcal{D}, \hat{\mu})+\frac{1}{T}\sum_{t=1}^T  (Y_t-\hat{\mu}(X_t, A_t) ) \frac{\pi_e(A_t|X_t)}{\pi_b(A_t|X_t)}.

DR mimics IPW to use a weighted version of rewards, but DR also uses the estimated mean reward function as a control variate to decrease the variance.
It preserves the consistency of IPW if either the importance weight or the mean reward estimator is accurate (a property called *double robustness*).
Moreover, DR is *semiparametric efficient* :cite:`Narita2019` when the mean reward estimator is correctly specified.
On the other hand, when it is wrong, this estimator can have larger asymptotic mean-squared-error than IPW :cite:`Kallus2019` and perform poorly in practice :cite:`Kang2007`.

