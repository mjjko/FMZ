// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © Mjjka
//@version=5
//
strategy(title='SAIYAN OCC Strategy R5.33', overlay=true, pyramiding=0, default_qty_type=strategy.percent_of_equity, default_qty_value=10, calc_on_every_tick=false)
// [1]
// === INPUTS ===
useRes = input(defval=true, title='Use Alternate Signals')
intRes = input(defval=8, title='Multiplier for Alernate Signals')
stratRes = timeframe.ismonthly ? str.tostring(timeframe.multiplier * intRes, '###M') : timeframe.isweekly ? str.tostring(timeframe.multiplier * intRes, '###W') : timeframe.isdaily ? str.tostring(timeframe.multiplier * intRes, '###D') : timeframe.isintraday ? str.tostring(timeframe.multiplier * intRes, '####') : '60'
basisType = input.string(defval='ALMA', title='MA Type: ', options=['TEMA', 'HullMA', 'ALMA'])
basisLen = input.int(defval=2, title='MA Period', minval=1)
offsetSigma = input.int(defval=5, title='Offset for LSMA / Sigma for ALMA', minval=0)
offsetALMA = input.float(defval=0.85, title='Offset for ALMA', minval=0, step=0.01)
scolor = input(true, title='Show coloured Bars to indicate Trend?')
delayOffset = input.int(defval=0, title='Delay Open/Close MA (Forces Non-Repainting)', minval=0, step=1)
tradeType = input.string('BOTH', title='What trades should be taken : ', options=['LONG', 'SHORT', 'BOTH', 'NONE'])
// === /INPUTS ===
// [2]
h = input(false, title='Signals for Heikin Ashi Candles')
src = h ? request.security(ticker.heikinashi(syminfo.tickerid), timeframe.period, close, lookahead=barmerge.lookahead_off) : close

noRepaint = request.security(syminfo.tickerid, "D", close[1], lookahead = barmerge.lookahead_on)
// Constants colours that include fully non-transparent option.
green100 = #008000FF
lime100 = #00FF00FF
red100 = #FF0000FF
blue100 = #0000FFFF
aqua100 = #00FFFFFF
darkred100 = #8B0000FF
gray100 = #808080FF

// Main Indicator
// [3]
// Functions
smoothrng(x, t, m) =>
	wper = t * 2 - 1
	avrng = ta.ema(math.abs(x - x[1]), t)
	smoothrng = ta.ema(avrng, wper) * m
rngfilt(x, r) =>
	rngfilt = x
	rngfilt := x > nz(rngfilt[1]) ? x - r < nz(rngfilt[1]) ? nz(rngfilt[1]) : x - r : x + r > nz(rngfilt[1]) ? nz(rngfilt[1]) : x + r
percWidth(len, perc) => (ta.highest(len) - ta.lowest(len)) * perc / 100
securityNoRep(sym, res, src) => request.security(sym, res, src, barmerge.gaps_off, barmerge.lookahead_on)
swingPoints(prd) =>
	pivHi = ta.pivothigh(prd, prd)
	pivLo = ta.pivotlow (prd, prd)
	last_pivHi = ta.valuewhen(pivHi, pivHi, 1)
	last_pivLo = ta.valuewhen(pivLo, pivLo, 1)
	hh = pivHi and pivHi > last_pivHi ? pivHi : na
	lh = pivHi and pivHi < last_pivHi ? pivHi : na
	hl = pivLo and pivLo > last_pivLo ? pivLo : na
	ll = pivLo and pivLo < last_pivLo ? pivLo : na
	[hh, lh, hl, ll]
f_chartTfInMinutes() =>
    float _resInMinutes = timeframe.multiplier * (
      timeframe.isseconds ? 1                   :
      timeframe.isminutes ? 1.                  :
      timeframe.isdaily   ? 60. * 24            :
      timeframe.isweekly  ? 60. * 24 * 7        :
      timeframe.ismonthly ? 60. * 24 * 30.4375  : na)
f_kc(src, len, sensitivity) =>
    basis = ta.sma(src, len)
    span  = ta.atr(len)
    [basis + span * sensitivity, basis - span * sensitivity]
wavetrend(src, chlLen, avgLen) =>
    esa = ta.ema(src, chlLen)
    d = ta.ema(math.abs(src - esa), chlLen)
    ci = (src - esa) / (0.015 * d)
    wt1 = ta.ema(ci, avgLen)
    wt2 = ta.sma(wt1, 3)
    [wt1, wt2]
f_top_fractal(src) => src[4] < src[2] and src[3] < src[2] and src[2] > src[1] and src[2] > src[0]
f_bot_fractal(src) => src[4] > src[2] and src[3] > src[2] and src[2] < src[1] and src[2] < src[0]
f_fractalize (src) => f_top_fractal(src) ? 1 : f_bot_fractal(src) ? -1 : 0
f_findDivs(src, topLimit, botLimit) =>
    fractalTop = f_fractalize(src) > 0 and src[2] >= topLimit ? src[2] : na
    fractalBot = f_fractalize(src) < 0 and src[2] <= botLimit ? src[2] : na
    highPrev = ta.valuewhen(fractalTop, src[2], 0)[2]
    highPrice = ta.valuewhen(fractalTop, high[2], 0)[2]
    lowPrev = ta.valuewhen(fractalBot, src[2], 0)[2]
    lowPrice = ta.valuewhen(fractalBot, low[2], 0)[2]
    bearSignal = fractalTop and high[1] > highPrice and src[1] < highPrev
    bullSignal = fractalBot and low[1] < lowPrice and src[1] > lowPrev
    [bearSignal, bullSignal]

// Get components
// [4]
rsi       = ta.rsi(close, 28)

rsiOb     = rsi > 65 and rsi > ta.ema(rsi, 10)
rsiOs     = rsi < 35 and rsi < ta.ema(rsi, 10)

dHigh     = securityNoRep(syminfo.tickerid, "D", high [1])
dLow      = securityNoRep(syminfo.tickerid, "D", low  [1])
dClose    = securityNoRep(syminfo.tickerid, "D", close[1])

ema = ta.ema(close, 144)
emaBull = close > ema

security_lower_tf(_symbol, _timeframe, _expression) =>
    var lower_tf = ""
    _timeframe_minutes = _timeframe == 0 ? 0 : str.tonumber(_timeframe) * (timeframe.isseconds ? 1.0/60 : 1)
    request.security(_symbol, str.tostring(_timeframe_minutes) + "m", _expression, barmerge.gaps_off, barmerge.lookahead_on)

    if timeframe.isseconds
        lower_tf := str.tostring(f_chartTfInMinutes()) + "S"
    else
        lower_tf := str.tostring(timeframe.multiplier * str.tonumber(timeframe.period))
    if lower_tf == "0"
        lower_tf := "1"
    if lower_tf == "1" and timeframe.ismonthly
        lower_tf := "10"
    if lower_tf == "1" and timeframe.isweekly
        lower_tf := "3"
    request.security(_symbol, lower_tf, _expression, syminfo.tickerid)


equal_tf(res) => str.tonumber(res) == f_chartTfInMinutes() and not timeframe.isseconds
higher_tf(res) => str.tonumber(res) > f_chartTfInMinutes() or timeframe.isseconds
too_small_tf(res) => (timeframe.isweekly and res=="1") or (timeframe.ismonthly and str.tonumber(res) < 10)
securityNoRep1(sym, res, src) =>
    bool bull_ = na
    bull_ := equal_tf(res) ? src : bull_
    bull_ := higher_tf(res) ? request.security(sym, res, src, barmerge.gaps_off, barmerge.lookahead_on) : bull_
    bull_array = request.security_lower_tf(syminfo.tickerid, higher_tf(res) ? str.tostring(f_chartTfInMinutes()) + (timeframe.isseconds ? "S" : "") : too_small_tf(res) ? (timeframe.isweekly ? "3" : "10") : res, src)
    if array.size(bull_array) > 1 and not equal_tf(res) and not higher_tf(res)
        bull_ := array.pop(bull_array)
    array.clear(bull_array)
    bull_


TF1Bull   = securityNoRep1(syminfo.tickerid, "1"   , emaBull)
TF3Bull   = securityNoRep1(syminfo.tickerid, "3"   , emaBull)
TF5Bull   = securityNoRep1(syminfo.tickerid, "5"   , emaBull)
TF15Bull  = securityNoRep1(syminfo.tickerid, "15"  , emaBull)
TF30Bull  = securityNoRep1(syminfo.tickerid, "30"  , emaBull)
TF60Bull  = securityNoRep1(syminfo.tickerid, "60"  , emaBull)
TF120Bull = securityNoRep1(syminfo.tickerid, "120" , emaBull)
TF240Bull = securityNoRep1(syminfo.tickerid, "240" , emaBull)
TF480Bull = securityNoRep1(syminfo.tickerid, "480" , emaBull)
TFDBull   = securityNoRep1(syminfo.tickerid, "1440", emaBull)

[wt1, wt2] = wavetrend(close, 5, 10)
[wtDivBear1, wtDivBull1] = f_findDivs(wt2, 15, -40)
[wtDivBear2, wtDivBull2] = f_findDivs(wt2, 45, -65)
wtDivBull = wtDivBull1 or wtDivBull2
wtDivBear = wtDivBear1 or wtDivBear2
////////////////////////////////////////////////////////
// === BASE FUNCTIONS ===
// [5]
// Returns MA input selection variant, default to SMA if blank or typo.
variant(type, src, len, offSig, offALMA) =>
    v1 = ta.sma(src, len)  // Simple
    v2 = ta.ema(src, len)  // Exponential
    v3 = 2 * v2 - ta.ema(v2, len)  // Double Exponential
    v4 = 3 * (v2 - ta.ema(v2, len)) + ta.ema(ta.ema(v2, len), len)  // Triple Exponential
    v5 = ta.wma(src, len)  // Weighted
    v6 = ta.vwma(src, len)  // Volume Weighted
    v7 = 0.0
    sma_1 = ta.sma(src, len)  // Smoothed
    v7 := na(v7[1]) ? sma_1 : (v7[1] * (len - 1) + src) / len
    v8 = ta.wma(2 * ta.wma(src, len / 2) - ta.wma(src, len), math.round(math.sqrt(len)))  // Hull
    v9 = ta.linreg(src, len, offSig)  // Least Squares
    v10 = ta.alma(src, len, offALMA, offSig)  // Arnaud Legoux
    v11 = ta.sma(v1, len)  // Triangular (extreme smooth)
    // SuperSmoother filter
    // © 2013  John F. Ehlers
    a1 = math.exp(-1.414 * 3.14159 / len)
    b1 = 2 * a1 * math.cos(1.414 * 3.14159 / len)
    c2 = b1
    c3 = -a1 * a1
    c1 = 1 - c2 - c3
    v12 = 0.0
    v12 := c1 * (src + nz(src[1])) / 2 + c2 * nz(v12[1]) + c3 * nz(v12[2])
    type == 'EMA' ? v2 : type == 'DEMA' ? v3 : type == 'TEMA' ? v4 : type == 'WMA' ? v5 : type == 'VWMA' ? v6 : type == 'SMMA' ? v7 : type == 'HullMA' ? v8 : type == 'LSMA' ? v9 : type == 'ALMA' ? v10 : type == 'TMA' ? v11 : type == 'SSMA' ? v12 : v1

// security wrapper for repeat calls
reso(exp, use, res) =>
    security_1 = request.security(syminfo.tickerid, res, exp, gaps=barmerge.gaps_off, lookahead=barmerge.lookahead_on)
    use ? security_1 : exp

// === SERIES SETUP ===
// [6]
closeSeries = variant(basisType, close[delayOffset], basisLen, offsetSigma, offsetALMA)
openSeries = variant(basisType, open[delayOffset], basisLen, offsetSigma, offsetALMA)
// === /SERIES ===
// Get Alternate resolution Series if selected.
closeSeriesAlt = reso(closeSeries, useRes, stratRes)
openSeriesAlt = reso(openSeries, useRes, stratRes)
//
trendColour = closeSeriesAlt > openSeriesAlt ? color.green : color.red
bcolour = closeSeries > openSeriesAlt ? lime100 : red100
barcolor(scolor ? bcolour : na, title='Bar Colours')
closeP = plot(closeSeriesAlt, title='Close Series', color=trendColour, linewidth=2, style=plot.style_line, transp=20)
openP = plot(openSeriesAlt, title='Open Series', color=trendColour, linewidth=2, style=plot.style_line, transp=20)
fill(closeP, openP, color=trendColour, transp=80)
//
// === /ALERT conditions.
// [7]
buy = ta.crossover(closeSeriesAlt, openSeriesAlt) and barstate.isconfirmed
sell = ta.crossunder(closeSeriesAlt, openSeriesAlt) and barstate.isconfirmed

longCond = buy
shortCond = sell

plotshape(buy,  title = "Buy",  text = 'Buy',  style = shape.labelup,   location = location.belowbar, color= #39ff14, textcolor = #FFFFFF, transp = 0, size = size.tiny)
plotshape(sell, title = "Sell", text = 'Sell', style = shape.labeldown, location = location.abovebar, color= #ff1100, textcolor = #FFFFFF, transp = 0, size = size.tiny)

// === STRATEGY ===
// [8]
// stop loss
slPoints = input.int(defval=0, title='Initial Stop Loss Points (zero to disable)', minval=0)
tpPoints = input.int(defval=0, title='Initial Target Profit Points (zero for disable)', minval=0)
// Include bar limiting algorithm
ebar = input.int(defval=4000, title='Number of Bars for Back Testing', minval=0)
dummy = input(false, title='- SET to ZERO for Daily or Longer Timeframes')
//
// Calculate how many mars since last bar
tdays = (timenow - time) / 60000.0  // number of minutes since last bar
tdays := timeframe.ismonthly ? tdays / 1440.0 / 5.0 / 4.3 / timeframe.multiplier : timeframe.isweekly ? tdays / 1440.0 / 5.0 / timeframe.multiplier : timeframe.isdaily ? tdays / 1440.0 / timeframe.multiplier : tdays / timeframe.multiplier  // number of bars since last bar
//
//set up exit parameters
TP = tpPoints > 0 ? tpPoints : na
SL = slPoints > 0 ? slPoints : na

// === /STRATEGY ===
// [9]
////////////////////////////////////////////////////////////////////////////////
// to automate put this in trendinview message:     {{strategy.order.alert_message}}
i_alert_txt_entry_long = input.text_area(defval = "", title = "Long Entry Message", group = "Alerts")
i_alert_txt_entry_short = input.text_area(defval = "", title = "Short Entry Message", group = "Alerts")

// Make sure we are within the bar range, Set up entries and exit conditions
if (ebar == 0 or tdays <= ebar) and tradeType != 'NONE'
    strategy.entry('long', strategy.long, when=longCond == true and tradeType != 'SHORT')
    strategy.entry('short', strategy.short, when=shortCond == true and tradeType != 'LONG')
    strategy.close('long', when=shortCond == true and tradeType == 'LONG')
    strategy.close('short', when=longCond == true and tradeType == 'SHORT')
