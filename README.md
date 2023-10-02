# Concurrent Scalping Algo

This python script is a working example to execute 
[scalping trading](https://www.investopedia.com/articles/trading/05/scalping.asp) 
algorithm for [Alpaca API](https://alpaca.markets). This algorithm uses real time order updates
as well as minute level bar streaming from Polygon via Websockets (see the
[document](https://docs.alpaca.markets/market-data/#consolidated-market-data) for
Polygon data access).
One of the contributions of this example is to demonstrate how to handle
multiple stocks concurrently as independent routine using Python's
[asyncio](https://docs.python.org/3/library/asyncio.html).

The strategy holds positions for very short period and exits positions quickly, so
you have to have more than $25k equity in your account due to the Pattern Day Trader rule,
to run this example. For more information about PDT rule, please read the
[document](https://support.alpaca.markets/hc/en-us/articles/360012203032-Pattern-Day-Trader).

## Table of Contents
1. [Dependency](#dependency)
2. [Usage](#usage)
3. [Strategy](#strategy)
4. [Implementation](#implementation)
5. [Algo Instance and State Management](#algo-instance-and-state-management)
6. [Event Handlers](#event-handlers)
7. [Note](#note)
8. [Customization](#customization)

## Dependency
This script needs latest [Alpaca Python SDK](https://github.com/alpacahq/alpaca-trade-api-python).
Please install it using pip

```sh
$ pip3 install alpaca-trade-api
```

or use [pipenv](https://github.com/pypa/pipenv) using `Pipfile` in this directory.

```sh
$ pipenv install
```

Afterwards, make sure that all python dependencies are met. 
These can be installed with 
```sh
$ pip install -r requirements.txt
```
## Usage

```sh
$ python main.py --lot=2000 TSLA FB AAPL
```

You can specify as many symbols as you want. The script is designed to kick off while market
is open. Nothing would happen until 21 minutes from the market open as it relies on the
simple moving average as the buy signal.

For safety purposes, we strongly recommend that your instance of this script does not include your API keys in plain text.
That is, they should be stored seperately in a .env file and accessed using the `os` module.
These are the lines that should be replaced:
    
```py
    ALPACA_API_KEY = "<key_id>"
    ALPACA_SECRET_KEY = "<secret_key>"
```
However, if you are using paper trading keys, they are relatively safe to use live. 

## Strategy
The algorithm idea is to buy the stock upon the buy signal (20 minute
[moving average crossover](https://www.investopedia.com/articles/active-trading/052014/how-use-moving-average-buy-stocks.asp)) 
as much as `lot` amount of dollar, then immediately sell the position at or above the entry price.
The assumption is that the market is bouncing upward when this signal occurs in a short period of time.
The buy signal is extremely simple, but what this strategy achieves is the quick reaction to
exit the position as soon as the buy order fills. There are reasonable probabilities that you can sell
the positions at the better prices than or the same price as your entry within the small window. We send
limit order at the last trade or position entry price whichever the higher to avoid unnecessary slippage.

The buy order is canceled after 2 minutes if it does not fill, assuming the signal is not
effective anymore. This could happen in a fast-moving market situation. The sell order is left
indifinitely until it fills, but this may cause loss more than the accumulated profit depending
on the market situation. This is where you can improve the risk control beyond this example.

The buy signal is calculated as soon as a minute bar arrives, which typically happen about 4 seconds
after the top of every minute (this is Polygon's behavior for minute bar streaming).

If you want to customize any part of the algorithm, it is crucial that the 'limit' parameter of the buy order remains so. In such a fast moving market (which is what the scalping algorithm targets, a market order could be filled at a price that is far from the last trade price. This could result in a loss. This slippage, compounded over time, could result in a significant loss.)
```py
def _submit_buy(self):
        trade = self._api.get_last_trade(self._symbol)
        amount = int(self._lot / trade.price)
        try:
            order = self._api.submit_order(
                symbol=self._symbol,
                side='buy',
                type='limit',
                qty=amount,
                time_in_force='day',
                limit_price=trade.price,
            )
        except Exception as e:
            self._l.info(e)
            self._transition('TO_BUY')
            return

        self._order = order
        self._l.info(f'submitted buy {order}')
        self._transition('BUY_SUBMITTED')
```
The same applies to sell functions, in all scenarios except for the liquidation of positions at the end of the day, or in bailout situations.

This example liquidates all watching positions with market order at the end of market hours (03:55pm ET), as seen here:
```py
    def checkup(self):
        '''periodic job to check the status'''
        if self._state == 'TO_BUY':
            self._submit_buy()
        elif self._state == 'BUY_SUBMITTED':
            self._check_buy()
        elif self._state == 'TO_SELL':
            self._submit_sell()
        elif self._state == 'SELL_SUBMITTED':
            self._check_sell()
        elif self._state == 'DONE':
            return

        # liquidate all positions at the end of the day
        if self._clock.is_open and self._clock.next_close.hour == 15 and self._clock.next_close.minute == 55:
            self._liquidate_all()
            self._transition('DONE')
```


## Implementation
This example heavily relies on Python's asyncio. Although the thread is single, we handle
multiple symbols concurrently using this async loop.

We keep track of each symbol state in a separate `ScalpAlgo` class instance. That way,
everything stays simple without complex data structure and easy to read. The `main()`
function creates the algo instance for each symbol and creates streaming object
to listen the bar events. As soon as we receive a minute bar, we invoke event handler
for each symbol.

The main routine also starts a period check routine to do some work in background every 30 seconds.
In this background task, we check market state with the clock API and liquidate positions
before the market closes.

### Algo Instance and State Management
Each algo instance initializes its state by fetching day's bar data so far and position/order
from Alpaca API to synchronize, in case the script restarts after some trades. There are
four internal states and transitions as events happen.

- `TO_BUY`: no position, no order. Can transition to `BUY_SUBMITTED`
- `BUY_SUBMITTED`: buy order has been submitted. Can transition to `TO_BUY` or `TO_SELL`
- `TO_SELL`: buy is filled and holding position. Can transition to `SELL_SUBMITTED`
- `SELL_SUBMITTED`: sell order has been submitted. Can transition to `TO_SELL` or `TO_BUY`

### Event Handlers
`on_bar()` is an event handler for the bar data. Here we calculate signal that triggers
a buy order in the `TO_BUY` state. Once order is submitted, it goes to the `BUY_SUBMITTED`
state.

If order is filled, `on_order_update()` handler is called with `event=fill`. The state
transitions to `TO_SELL` and immediately submits a sell order, to transition to the
`SELL_SUBMITTED` state.

Orders may be canceled or rejected (caused by this script or you manually cancel them
from the dashboard). In these cases, the state transitions to `TO_BUY` (if not holding
a position) or `TO_SELL` (if holding a position) and wait for the next events.

`checkup()` method is the background periodic job to check several conditions, where
we cancel open orders and sends market sell order if there is an open position.

It exits once the market closes.

### Note
Each algo instance owns its child logger, prefixed by the symbol name. The console
log is also emitted to a file `console.log` under the same directory for your later review.

Again, the beautify of this code is that there is no multithread code but each
algo instance can focus on the bar/order/position data only for its own. It still
handles multiple symbols concurrently plus runs background periodic job in the
same async loop.

The trick to run additional async routine is as follows.

```py
    loop = stream.loop
    loop.run_until_complete(asyncio.gather(
        stream.subscribe(channels),
        periodic(),
    ))
    loop.close()
```

We use `asyncio.gather()` to run all bar handler, order update handler and periodic job
in one async loop indifinitely. You can kill it by `Ctrl+C`.

### Customization
Instead of using this buy signal of 20 minute simple moving average cross over, you can
use your own buy signal. To do so, extend the `ScalpAlgo` class and write your own
`_calc_buy_signal()` method.

```py
    class MyScalpAlgo(ScalpAlgo):
        def _calculate_buy_signal(self):
            '''self._bars has all minute bars in the session so far. Return True to
            trigger buy order'''
            pass
```

And use it instead of the original class.
