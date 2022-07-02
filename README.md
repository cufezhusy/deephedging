# Deep Hedging

This archive contains a sample implementation of of the Deep Hedging (http://deep-hedging.com) framework.
The notebook directory has a number of examples on how to use it. The framework relies on the pip package cdxbasics.

The Deep Hedging problem for a horizon $T$ is given as
$$
    \max_{a}: U\left[ \
        Z_T + \sum_{t=0}^{T-1} a(f_t) \cdot DH_t - | a(f_t)\gamma_t|
    \ \right]
$$
where  $DH_t:=H_T - H_t$ denotes the vector of returns of the hedging instruments to $T$. Cost are proportional.
The policy $a$ is a neural network which is fed both pre-computed and live features $f_t$ at each time step.
<ul>
<li>
<tt>data['martket']['payoff']</tt> (:,M)<br> 
The payoff $Z_T$ at maturity. Since this is at or part the expiry of the product, this can be computed off the path until $T$.
<br>&nbsp;
</li>
<li>
<ttdata['martket']['payoff']</tt> (:,M,N)<br>
Returns of the hedges, i.e. the vector $DH_t:=H_T - H_t$. That means $H_t$ is the model price at time $t$, and $H_T$ is the price at time $T$. In most applications $T$ is chosen such that $H_T$
is the actual payoff.<br>
    For example, if $S_t$ is spot, $\sigma_t$ is an implied volatility,  $\tau$ is the time-to-maturity, and
    $k$ a relative strike, then $H_t = \mathrm{BSCall}( S_t, \sigma_t; \tau, kS_t )$ and $H_T = ( S_{t+\tau} / S_t - k )^+$.
<br>&nbsp;
</li>
<li>
<tt>data['martket']['payoff']</tt> (:,M,N)<br>
Cost $\gamma_t$ of trading the hedges in $t$ for proportional cost $c_t(a) = \gamma_t\cdot |a|$. 
More advanced implementations allow to pass the cost function itself as a tensorflow model.<br>
    In the simple setting an example for the cost of trading a vanilla call could be $\gamma_t = \gamma^\mathrm{Delta} \mathrm{BSDelta}(t,\cdots) 
    + \gamma^\mathrm{Vega}  \mathrm{BSVega}(t,\cdots)$.
<br>&nbsp;
</li><li>
<tt>data['martket']['unbd_a'], data['martket']['lnbd_a']</tt> (:,M,N)<br>
Min/max allowed action per time step: $a^\min_t \leq a \leq a^\max_t$, componenwise.
<br>&nbsp;

</li><li>
<tt>data['features']['per_step']</tt> (:,M,N)<br>
Featues for feeding the action network per time step such as current spot, current implied volatilities, time of the day, etx
<br>&nbsp;
</li><li>
<tt>data['features']['per_sample']</tt> (:,M)<br>
Featues for feeding the action network which are constant along the path such as features of the payoff, risk aversion, 
</ul>

The code examples provided are fairly general and allow for a wide range of applications. 
An example world generator for simplistic model dynamics is provided, but in practise it is recommend to rely
on fully machine learned market simulators such as https://arxiv.org/abs/2112.06823

## Industrial Machine Learning Code Philosophy

We attempted to provide a base for industrial code development.
<ul>
    <li>
        <b>Notebook-free</b>: all code can, and is meant to run 
        outside a jupyter notebook. Notebooks are good for playing around but should not feature in any production environment.
        Notebooks are used for demonstration only.
        <br>&nbsp;
    </li>
    <li>    
        <b>Defensive programming</b>: validate as many inputs to functions as reasonable with clear, actionable, context-dependent 
        error messages. We use the <tt>cdxbasics.logger</tt> framework with a similar usage paradigm as C++ ASSSET/VERIFY.
        <br>&nbsp;
    </li>
    <li>
    <b>Robust configs</b>: all configurations of all objects are driven by dictionaries.
    However, the use of simple dictionaries leads to a number of inefficiencies which can slow down development.
    We therefore use <tt>cdxbasics.config.Config</tt> which provides:
    <ul>
        <li><b>Catch Spelling Errors</b>: ensures that any config parameter is understood by the receving code. 
            That means if the code expects <tt>config['nSamples']</tt> but we passed <tt>config['samples']</tt>
            an error is thrown.
        </li>
        <li><b>Self-documentation</b>: once parsed by receving code, the config is self-documenting and is able
            to print out any values used, including those which were not set by the users when calling the receiving code.
            </li>
        <li><b>Object notation</b>: it is a matter of tastes, but we prefer using <tt>config.nSamples = 2</tt> instead
            of the standard dictionary notation.
        </li>
        &nbsp;
    </ul>
    </li>
    <li>
        <b>Config-driven built</b>: 
        the config provided will create any models used within the framework. There is no need to know the specifics
        of the implementation, and which sub-models are instantiated.           
        <br>&nbsp;
    </li>
</ul>

## Key Objects and Functions

<ul>
    <li><b>world.SimpleWorld_Spot_ATM</b> class<br>
         Simple World with one asset and one floating ATM option.
    The asset has stochastic volatility, and a mean-reverting drift.
    The implied volatility of the asset is not the realized volatility, allowing to re-create some results from https://arxiv.org/abs/2103.11948
        <br>
    See <tt>notebooks/simpleWorld_Spot_ATM.ipynb</tt>
      <br>&nbsp;  
    </li>
    <li><b>gym.VanillaDeepHedgingGym</b> class<br>
        Main Deep Hedging training gym (the Monte Carlo). It will create internally the agent network and the monetary utility $U$.
        <br>
        To run the models for all samples of a given <tt>world</tt> use <tt>r = gym(world.tf_data)</tt>.<br>
        The returned dictionary contains the following members
        <ol>
                 <li><tt>utiliy:  </tt> (,) primary objective to maximize
            </li><li><tt>utiliy0: </tt> (,) objective without hedging
            </li><li><tt>loss:    </tt> (,) -utility-utility0
            </li><li><tt>payoff:  </tt> (,) terminal payoff 
            </li><li><tt>pnl:     </tt> (,) mid-price pnl of trading (e.g. ex cost)
            </li><li><tt>cost:    </tt> (,) cost of trading
            </li><li><tt>gains:   </tt> (,) total gains: payoff + pnl - cost 
            </li><li><tt>actions: </tt> (,M,N) actions, per step, per path
            </li><li><tt>deltas:  </tt> (,M,N) deltas, per step, per path
            </li>
        </ol>
        See <tt>notebooks/trainer.ipynb</tt>.
    </li>
      <br>&nbsp;  
    </li>
    <li><b>trainer.train</b> function<br>
        Main Deep Hedging training engine (stochastic gradient descent). <br>
        Trains the model using keras. Any optimizer supported by Keras might be used. When run in a Jupyer notebook the model will 
        dynamically plot progress in a number if graphs which will aid understanding on how training is progressing. 
    <br>
        See <tt>notebooks/trainer.ipynb</tt>.
    </li>
       
    
</ul>


## Running Deep Hedging

Copied from <tt>notebooks/trainer.ipynb</tt>:

<code>
from cdxbasics.config import Config
from deephedging.trainer import train
from deephedging.gym import VanillaDeepHedgingGym
from deephedging.world import SimpleWorld_Spot_ATM

\# see print of the config below for numerous options
config = Config()
\# world
config.world.samples = 10000
config.world.steps = 20
config.world.black_scholes = True
\# gym
config.gym.objective.utility = "exp2"
config.gym.objective.lmbda = 10.
config.gym.agent.network.depth = 3
config.gym.agent.network.activation = "softplus"
\# trainer
config.trainer.train.batch_size = None
config.trainer.train.epochs = 400
config.trainer.train.run_eagerly = False
config.trainer.visual.epoch_refresh = 1
config.trainer.visual.time_refresh = 10
config.trainer.visual.pcnt_lo = 0.25
config.trainer.visual.pcnt_hi = 0.75

\# create world
world  = SimpleWorld_Spot_ATM( config.world )
val_world  = world.clone(samples=1000)

\# create training environment
gym = VanillaDeepHedgingGym( config.gym )

\# create training environment
train( gym=gym, world=world, val_world=val_world, config=config.trainer )

\# print information on all available parameters and their usage
print("=========================================")
print("Config usage report")
print("=========================================")
print( config.usage_report() )
config.done()
</code>

## Config Parameters

This is the output of the <tt>print( config.usage_report() )</tt> call above. It provides a summary of all config values available, their defaults, and what values where used.

<code>
config.gym.agent.network['activation'] = softplus # Network activation function; default: relu
config.gym.agent.network['depth'] = 3 # Network depth; default: 3
config.gym.agent.network['width'] = 20 # Network width; default: 20
config.gym.agent['agent_type'] = feed_forward #  Default: feed_forward
config.gym.agent['features'] = ['price', 'delta', 'time_left'] # Named features the agent uses from the environment; default: ['price', 'delta', 'time_left']
    
config.gym.environment['softclip_hinge_softness'] = 1.0 # Specifies softness of bounding actions between lbnd_a and ubnd_a; default: 1.0
    
config.gym.objective['lmbda'] = 10.0 # Risk aversion; default: 1.0
config.gym.objective['utility'] = exp2 # Type of monetary utility: mean, exp, exp2, vicky, cvar, quad; default: entropy
    
config.trainer.train['batch_size'] = None # Batch size; default: None
config.trainer.train['epochs'] = 10 # Epochs; default: 100
config.trainer.train['optimizer'] = adam # Optimizer; default: adam
config.trainer.train['run_eagerly'] = False # Keras model run_eagerly; default: False
config.trainer.train['time_out'] = None # Timeout in seconds. None for no timeout; default: None
config.trainer.visual.fig['col_nums'] = 6 # Number of columbs; default: 6
config.trainer.visual.fig['col_size'] = 5 # Plot size of a column; default: 5
config.trainer.visual.fig['row_size'] = 5 # Plot size of a row; default: 5
config.trainer.visual['bins'] = 200 # How many x to plot; default: 200
config.trainer.visual['epoch_refresh'] = 1 # Epoch fefresh frequency for visualizations; default: 10
config.trainer.visual['err_dev'] = 1.0 # How many standard errors to add to loss to assess best performance; default: 1.0
config.trainer.visual['lookback_window'] = 30 # Lookback window for determining y min/max; default: 30
config.trainer.visual['pcnt_hi'] = 0.75 # Upper percentile lower end to compute average performance over for by step action analysis; default: 0.5
config.trainer.visual['pcnt_lo'] = 0.25 # Lower percentile upper end to compute average performance over for by step action analysis; default: 0.5
config.trainer.visual['show_epochs'] = 100 # Maximum epochs displayed; default: 100
config.trainer.visual['time_refresh'] = 10 # Time refresh interval for visualizations; default: 20

config.world['black_scholes'] = True # Hard overwrite to use a black & scholes model with vol 'rvol' and drift 'drift; default: False
config.world['corr_ms'] = 0.5 # Correlation between the asset and its mean; default: 0.5
config.world['corr_vi'] = 0.8 # Correlation between the implied vol and the asset volatility; default: 0.8
config.world['corr_vs'] = -0.7 # Correlation between the asset and its volatility; default: -0.7
config.world['cost_p'] = 0.0005 # Trading cost for the option on top of delta and vega cost; default: 0.0005
config.world['cost_s'] = 0.0002 # Trading cost spot; default: 0.0002
config.world['cost_v'] = 0.02 # Trading cost vega; default: 0.02
config.world['drift'] = 0.1 # Mean drift of the asset; default: 0.1
config.world['drift_vol'] = 0.1 # Vol of the drift; default: 0.1
config.world['dt'] = 0.02 # Time per timestep; default: One week (1/50)
config.world['invar_steps'] = 5 # Number of steps ahead to sample from invariant distribution; default: 5
config.world['ivol'] = 0.2 # Initial implied volatility; default: Same as realized vol
config.world['lbnd_as'] = -5.0 # Lower bound for the number of shares traded at each time step; default: -5.0
config.world['lbnd_av'] = -5.0 # Lower bound for the number of options traded at each time step; default: -5.0
config.world['meanrev_drift'] = 1.0 # Mean reversion of the drift of the asset; default: 1.0
config.world['meanrev_ivol'] = 0.1 # Mean reversion for implied vol vol vs initial level; default: 0.1
config.world['meanrev_rvol'] = 2.0 # Mean reversion for realized vol vs implied vol; default: 2.0
config.world['payoff'] = \<function SimpleWorld_Spot_ATM.__init__.\<locals\>.\<lambda\> at 0x0000022125590708\> # Payoff function. Parameters is spots[samples,steps+1]; default: Short ATM call function
config.world['rcorr_vs'] = -0.5 # Residual correlation between the asset and its implied volatility; default: -0.5
config.world['rvol'] = 0.2 # Initial realized volatility; default: 0.2
config.world['samples'] = 10000 # Number of samples; default: 1000
config.world['seed'] = 2312414312 # Random seed; default: 2312414312
config.world['steps'] = 20 # Number of time steps; default: 10
config.world['strike'] = 1.0 # Relative strike. Set to zero to turn off option; default: 1.0
config.world['ttm_steps'] = 4 # Time to maturity of the option; in steps; default: 4
config.world['ubnd_as'] = 5.0 # Upper bound for the number of shares traded at each time step; default: 5.0
config.world['ubnd_av'] = 5.0 # Upper bound for the number of options traded at each time step; default: 5.0
config.world['volvol_ivol'] = 0.5 # Vol of Vol for implied vol; default: 0.5
config.world['volvol_rvol'] = 0.5 # Vol of Vol for realized vol; default: 0.5
</code>