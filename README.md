
//+------------------------------------------------------------------+
//|                                              AdvancedFibRobot.mq5|
//|                        Top-Down Analysis + Fibonacci + Patterns  |
//+------------------------------------------------------------------+
#property strict

// Input parameters
input double RiskRewardRatio = 3.0;       // Minimum Risk-Reward Ratio
input double RiskPerTrade = 1.0;          // Risk per trade (% of account)
input double TrailingStopPct = 2.0;       // Trailing Stop Loss (%)
input int MinPositionsPerTrade = 1;       // Minimum positions per trade
input int MaxPositionsPerTrade = 3;       // Maximum positions per trade
input int ATRPeriod = 14;                 // ATR period for volatility filter
input int SMAPeriod = 50;                 // SMA period for trend filter

// Global variables
double stopLossLevel = 0;
double takeProfitLevel1 = 0; // 30% Fib retracement
double takeProfitLevel2 = 0; // 40% Full retracement
double takeProfitLevel3 = 0; // 30% at 1.27 Fib extension
int positionsOpened = 0;

//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
int OnInit()
{
    return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
}

//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
{
    // Check for open positions
    if (PositionsTotal() > 0)
    {
        TrailingStop();
        CheckTakeProfitLevels();
        return;
    }

    // Reset positions opened counter
    positionsOpened = 0;

    // Top-down analysis: Check higher timeframe trend
    double smaValue = iMA(_Symbol, PERIOD_CURRENT, SMAPeriod, 0, MODE_SMA, PRICE_CLOSE, 0);
    bool isUptrend = Close[0] > smaValue;

    // Calculate Fibonacci levels
    double swingHigh = iHigh(_Symbol, PERIOD_CURRENT, iHighest(_Symbol, PERIOD_CURRENT, MODE_HIGH, 20, 1));
    double swingLow = iLow(_Symbol, PERIOD_CURRENT, iLowest(_Symbol, PERIOD_CURRENT, MODE_LOW, 20, 1));
    double fibLevel50 = swingHigh - (swingHigh - swingLow) * 0.5;
    double fibLevel618 = swingHigh - (swingHigh - swingLow) * 0.618;

    // Check for confluence (Fibonacci + Support/Resistance + Candlestick Patterns)
    if (IsConfluenceZone(fibLevel50, fibLevel618))
    {
        if (isUptrend && IsBullishEngulfing() && IsSupport())
        {
            // Buy signal
            OpenTrade(ORDER_TYPE_BUY, fibLevel50, fibLevel618);
        }
        else if (!isUptrend && IsBearishEngulfing() && IsResistance())
        {
            // Sell signal
            OpenTrade(ORDER_TYPE_SELL, fibLevel50, fibLevel618);
        }
    }

    // Check for consolidation patterns (counter-trendline break)
    if (IsConsolidationPattern())
    {
        if (IsCounterTrendlineBreak())
        {
            // Open trade based on counter-trendline break
            OpenTrade(ORDER_TYPE_BUY, fibLevel50, fibLevel618);
        }
    }
}

//+------------------------------------------------------------------+
//| Function to check confluence zone                                |
//+------------------------------------------------------------------+
bool IsConfluenceZone(double fibLevel50, double fibLevel618)
{
    // Check if price is within Fibonacci golden zone (50%-61.8%)
    return (Close[0] >= fibLevel50 && Close[0] <= fibLevel618);
}

//+------------------------------------------------------------------+
//| Function to check bullish engulfing pattern                      |
//+------------------------------------------------------------------+
bool IsBullishEngulfing()
{
    // Bullish engulfing pattern logic
    return (Close[1] < Open[1] && Close[0] > Open[0] && Close[0] > Open[1] && Close[1] < Open[0]);
}

//+------------------------------------------------------------------+
//| Function to check bearish engulfing pattern                      |
//+------------------------------------------------------------------+
bool IsBearishEngulfing()
{
    // Bearish engulfing pattern logic
    return (Close[1] > Open[1] && Close[0] < Open[0] && Close[0] < Open[1] && Close[1] > Open[0]);
}

//+------------------------------------------------------------------+
//| Function to check support level                                  |
//+------------------------------------------------------------------+
bool IsSupport()
{
    // Support level logic (e.g., previous swing low)
    return (Close[0] > iLow(_Symbol, PERIOD_CURRENT, iLowest(_Symbol, PERIOD_CURRENT, MODE_LOW, 20, 1)));
}

//+------------------------------------------------------------------+
//| Function to check resistance level                               |
//+------------------------------------------------------------------+
bool IsResistance()
{
    // Resistance level logic (e.g., previous swing high)
    return (Close[0] < iHigh(_Symbol, PERIOD_CURRENT, iHighest(_Symbol, PERIOD_CURRENT, MODE_HIGH, 20, 1)));
}

//+------------------------------------------------------------------+
//| Function to check consolidation patterns                         |
//+------------------------------------------------------------------+
bool IsConsolidationPattern()
{
    // Consolidation pattern logic (e.g., triangle, rectangle)
    return true; // Add your own logic here
}

//+------------------------------------------------------------------+
//| Function to check counter-trendline break                        |
//+------------------------------------------------------------------+
bool IsCounterTrendlineBreak()
{
    // Counter-trendline break logic
    return true; // Add your own logic here
}

//+------------------------------------------------------------------+
//| Function to open a trade                                         |
//+------------------------------------------------------------------+
void OpenTrade(int orderType, double fibLevel50, double fibLevel618)
{
    if (positionsOpened >= MaxPositionsPerTrade)
        return;

    double lotSize = CalculateLotSize();
    if (lotSize <= 0)
        return;

    MqlTradeRequest request;
    MqlTradeResult result;
    ZeroMemory(request);
    ZeroMemory(result);

    request.action = TRADE_ACTION_DEAL;
    request.symbol = _Symbol;
    request.volume = lotSize;
    request.type = orderType;
    request.price = (orderType == ORDER_TYPE_BUY) ? Ask : Bid;
    request.sl = (orderType == ORDER_TYPE_BUY) ? fibLevel50 : fibLevel618;
    request.tp = CalculateTakeProfitLevels(orderType, fibLevel50, fibLevel618);
    request.deviation = 10;
    request.type_filling = ORDER_FILLING_FOK;

    if (!OrderSend(request, result))
    {
        Print("OrderSend failed: ", result.retcode);
    }
    else
    {
        positionsOpened++;
    }
}

//+------------------------------------------------------------------+
//| Function to calculate take-profit levels                         |
//+------------------------------------------------------------------+
double CalculateTakeProfitLevels(int orderType, double fibLevel50, double fibLevel618)
{
    if (orderType == ORDER_TYPE_BUY)
    {
        takeProfitLevel1 = Close[0] + (fibLevel618 - fibLevel50) * 0.3; // 30% Fib retracement
        takeProfitLevel2 = Close[0] + (fibLevel618 - fibLevel50) * 0.4; // 40% Full retracement
        takeProfitLevel3 = Close[0] + (fibLevel618 - fibLevel50) * 1.27; // 1.27 Fib extension
    }
    else
    {
        takeProfitLevel1 = Close[0] - (fibLevel50 - fibLevel618) * 0.3; // 30% Fib retracement
        takeProfitLevel2 = Close[0] - (fibLevel50 - fibLevel618) * 0.4; // 40% Full retracement
        takeProfitLevel3 = Close[0] - (fibLevel50 - fibLevel618) * 1.27; // 1.27 Fib extension
    }

    return takeProfitLevel3; // Default to the highest take-profit level
}

//+------------------------------------------------------------------+
//| Function to check take-profit levels                             |
//+------------------------------------------------------------------+
void CheckTakeProfitLevels()
{
    for (int i = 0; i < PositionsTotal(); i++)
    {
        ulong ticket = PositionGetTicket(i);
        if (PositionSelectByTicket(ticket))
        {
            double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
            if ((PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY && currentPrice >= takeProfitLevel1) ||
                (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_SELL && currentPrice <= takeProfitLevel1))
            {
                ClosePosition(ticket);
            }
        }
    }
}

//+------------------------------------------------------------------+
//| Function to close a position                                     |
//+------------------------------------------------------------------+
void ClosePosition(ulong ticket)
{
    MqlTradeRequest request;
    MqlTradeResult result;
    ZeroMemory(request);
    ZeroMemory(result);

    request.action = TRADE_ACTION_DEAL;
    request.position = ticket;
    request.symbol = _Symbol;
    request.volume = PositionGetDouble(POSITION_VOLUME);
    request.type = (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY) ? ORDER_TYPE_SELL : ORDER_TYPE_BUY;
    request.price = (request.type == ORDER_TYPE_SELL) ? Bid : Ask;

    if (!OrderSend(request, result))
    {
        Print("ClosePosition failed: ", result.retcode);
    }
}

//+------------------------------------------------------------------+
//| Function to calculate lot size                                   |
//+------------------------------------------------------------------+
double CalculateLotSize()
{
    double riskAmount = AccountInfoDouble(ACCOUNT_BALANCE) * RiskPerTrade / 100;
    double tickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
    double pointValue = SymbolInfoDouble(_Symbol, SYMBOL_POINT);
    double riskDistance = MathAbs(Close[0] - stopLossLevel) / pointValue;
    double lotSize = riskAmount / (riskDistance * tickValue);
    return NormalizeDouble(lotSize, 2);
}

//+------------------------------------------------------------------+
//| Function to manage trailing stop                                 |
//+------------------------------------------------------------------+
void TrailingStop()
{
    for (int i = 0; i < PositionsTotal(); i++)
    {
        ulong ticket = PositionGetTicket(i);
        if (PositionSelectByTicket(ticket))
        {
            double currentStopLoss = PositionGetDouble(POSITION_SL);
            double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
            double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
            double newStopLoss = 0;

            if (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY)
            {
                newStopLoss = currentPrice * (1 - TrailingStopPct / 100);
                if (newStopLoss > currentStopLoss && newStopLoss > openPrice)
                {
                    ModifyStopLoss(ticket, newStopLoss);
                }
            }
            else if (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_SELL)
            {
                newStopLoss = currentPrice * (1 + TrailingStopPct / 100);
                if (newStopLoss < currentStopLoss && newStopLoss < openPrice)
                {
                    ModifyStopLoss(ticket, newStopLoss);
                }
            }
        }
    }
}

//+------------------------------------------------------------------+
//| Function to modify stop loss                                     |
//+------------------------------------------------------------------+
void ModifyStopLoss(ulong ticket, double sl)
{
    MqlTradeRequest request;
    MqlTradeResult result;
    ZeroMemory(request);
    ZeroMemory(result);

    request.action = TRADE_ACTION_SLTP;
    request.position = ticket;
    request.sl = sl;
    request.symbol = _Symbol;

    if (!OrderSend(request, result))
    {
        Print("ModifyStopLoss failed: ", result.retcode);
    }
}
