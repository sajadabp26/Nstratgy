//+------------------------------------------------------------------+
//| BreakoutRetestLimitEA.mq4                                        |
//| Copyright 2025, Your Name                                        |
//| https://www.mql5.com                                             |
//+------------------------------------------------------------------+
#property copyright "Copyright 2025, Your Name"
#property link      "https://www.mql5.com"
#property version   "1.04" // Version updated with corrected structure
#property strict

//--- Standard MQL4 Includes ---
#include <stdlib.mqh>
//#include <Trade\Trade.mqh> // For CTrade class
#include <Indicators\Indicators.mqh> // For iMA
//#include <Objects\Objects.mqh> // For drawing objects

//--- Global Variables ---
datetime lastBarTime = 0;
int      pendingOrderTicket = -1;
int      pendingOrderType = -1; // Stores the type of the pending order (OP_BUYLIMIT/OP_SELLLIMIT)
datetime pendingOrderPlacementTime = 0; // Time when the pending order was placed

// Variables to store identified level and SL point for the pending trade
double identifiedLevelPrice = 0.0;
int    identifiedLevelBar = -1;
int    identifiedLevelType = 0; // 1 for Resistance (Buy), -1 for Support (Sell)

double identifiedSLPrice = 0.0;
int    identifiedSLBar = -1;

//--- Global Trade Object ---
CTrade trade; // Declare the CTrade object here

//--- Expert Inputs ---
// Declare inputs before they are used in global variable initializations or functions
extern int    MagicNumber = 123456;
extern double Slippage    = 3; // Slippage in points

// Pattern Identification Inputs (from user's previous snippet and new requirements)
extern int    MinSwingStrength = 4; // Kept from user's initial snippet, though maybe not directly used in the new logic
extern int    MaxSwingStrength = 10; // Kept from user's initial snippet
extern int    PatternLookbackBars_P1 = 200; // Kept from user's initial snippet
extern int    PatternLookbackBars_P2 = 50; // Kept from user's initial snippet
extern int    ResistanceSupportLookbackBars = 100; // N bars back to find the level (New input)
extern int    SLSwingStrength = 2; // Swing strength for finding the SL point (New input)


// Trade Management Inputs
extern double EntryBufferPips = 2.0;  // Buffer added/subtracted from level for entry price
extern double SLBufferPips    = 2.0;  // Buffer added/subtracted from SL swing point for SL price
extern double RewardRiskRatio = 2.0;
extern double RiskPercentage  = 2.0;
extern int    MinStopLossPips = 10;
extern int    MaxStopLossPips = 50;
extern int    PendingOrderExpiryBars = 30; // Pending order expiry in bars (New input)
// extern int    PendingOrderExpiryMinutes = 30; // Deprecated in favor of bars
extern bool   AllowManageOpenTrades = false; // Kept from user's initial snippet
extern double BreakevenBufferPips = 5.0; // Kept from user's initial snippet
extern int    StopLevelBufferPoints = 10; // Kept from user's initial snippet


// Trend Filter Inputs (from user's initial snippet)
extern bool   UseMATrendFilter = false; // Default to off as requested
extern int    MovingAveragePeriodShort = 20;
extern int    MovingAveragePeriodLong  = 50;
extern ENUM_MA_METHOD MovingAverageMethod = MODE_SMA;
extern ENUM_APPLIED_PRICE MovingAverageAppliedPrice = PRICE_CLOSE;

// Drawing Inputs (New inputs for line drawing)
extern bool   DrawPatternLines = true; // Default to true to show the levels
extern color  ResistanceLineColor = Red;
extern color  SupportLineColor    = Blue;
extern color  PendingOrderLineColor = Gray;
extern ENUM_LINE_STYLE DrawnLineStyle = STYLE_DASH;
extern int    DrawnLineWidth = 1;
extern int    DrawnLineDurationBars = 20; // Duration in bars for the drawn line

// Debugging Input (from user's initial snippet)
extern bool   Debug_Verbose = true;


// String names for chart objects - Initialize after MagicNumber is declared
string resistanceLineName;
string supportLineName;
string pendingOrderLineName;


//+------------------------------------------------------------------+
//| Custom Helper Functions                                          |
//+------------------------------------------------------------------+

// Converts pips to price units based on symbol digits
double PipsToPrice(double pips)
{
    if (_Digits == 3 || _Digits == 5)
        return pips * 10 * _Point;
    else
        return pips * _Point;
}

// Gets MQL4 error description string
string GetMQL4ErrorDescription(int error_code)
{
    return ErrorDescription(error_code);
}

// Helper function to convert OrderType to string for printing
string OrderTypeToString(int type)
{
    switch(type)
    {
        case OP_BUY: return "BUY";
        case OP_SELL: return "SELL";
        case OP_BUYLIMIT: return "BUY LIMIT";
        case OP_SELLLIMIT: return "SELL LIMIT";
        case OP_BUYSTOP: return "BUY STOP";
        case OP_SELLSTOP: return "SELL STOP";
        default: return "UNKNOWN (" + IntegerToString(type) + ")";
    }
}

// Calculates lot size based on risk percentage and stop loss in pips
double CalculateLotSize(double riskPercentage, double stopLossPips)
{
    if (riskPercentage <= 0 || stopLossPips <= 0)
    {
        if (Debug_Verbose) Print("DEBUG: CalculateLotSize - Invalid input: riskPercentage (", riskPercentage, ") or stopLossPips (", stopLossPips, ") is zero or negative.");
        return 0.0;
    }

    double accountBalance = AccountBalance();
    if (accountBalance <= 0)
    {
        if (Debug_Verbose) Print("DEBUG: CalculateLotSize - Invalid Account Balance (", accountBalance, ").");
        return 0.0;
    }

    double tickValue = MarketInfo(Symbol(), MODE_TICKVALUE);
    double tickSize  = MarketInfo(Symbol(), MODE_TICKSIZE);

    if (tickSize == 0)
    {
        if (Debug_Verbose) Print("DEBUG: CalculateLotSize - Invalid TickSize (0.0). Check symbol properties.");
        return 0.0;
    }

    // Value of 1 pip for 1 standard lot based on symbol properties
    double valuePerPipPerStandardLot = tickValue / tickSize * _Point;

    // Calculate risk per lot
    double riskPerLot = stopLossPips * valuePerPipPerStandardLot;

    if (riskPerLot <= 0)
    {
         if (Debug_Verbose) Print("DEBUG: CalculateLotSize - Calculated risk per lot is zero or negative (", riskPerLot, "). Stop loss too small?");
        return 0.0;
    }

    double riskAmount = accountBalance * (riskPercentage / 100.0);

    double lotSize = riskAmount / riskPerLot;

    double minLot = MarketInfo(Symbol(), MODE_MINLOT);
    double maxLot = MarketInfo(Symbol(), MODE_MAXLOT);
    double lotStep = MarketInfo(Symbol(), MODE_LOTSTEP);

    if (minLot <= 0 || maxLot <= 0 || lotStep <= 0)
    {
         if (Debug_Verbose) Print("DEBUG: CalculateLotSize - Invalid min/max lot or lot step (", minLot, "/", maxLot, "/", lotStep, "). Check symbol properties.");
        return 0.0;
    }

    // Normalize lot size to lot step and clamp between min/max lot
    lotSize = MathFloor(lotSize / lotStep) * lotStep;
    lotSize = MathMax(minLot, MathMin(lotSize, maxLot));

    if (Debug_Verbose)
    {
        Print("DEBUG: CalculateLotSize - Balance=", accountBalance, ", Risk %=", riskPercentage, ", Risk Amount=", riskAmount,
              ", SL Pips=", stopLossPips, ", ValuePerPipPerStandardLot=", valuePerPipPerStandardLot, ", RiskPerLot=", riskPerLot, ", Final Lot=", lotSize);
    }

    if (lotSize < minLot) // Check again after clamping
    {
        if (Debug_Verbose) Print("DEBUG: CalculateLotSize - Calculated lot size (", NormalizeDouble(lotSize, 2), ") is less than min lot (", minLot, "). Returning 0.0.");
        return 0.0;
    }

    // Normalize the final lot size to appropriate digits for lots (usually 2 or 3)
    return NormalizeDouble(lotSize, 2); // Assuming lot step precision is at most 0.01

}


// Gets trend direction based on two moving averages
int GetMATrendDirection()
{
    // Need enough bars for both MAs, plus one more for comparison
    if (Bars < MathMax(MovingAveragePeriodShort, MovingAveragePeriodLong) + 2) // Need at least 2 bars for MA comparison (current and previous closed)
    {
        if (Debug_Verbose) Print("DEBUG: GetMATrendDirection - Not enough bars (", Bars, ") for MA calculation. Need at least ", MathMax(MovingAveragePeriodShort, MovingAveragePeriodLong) + 2);
        return 0; // Undefined trend
    }

    // Get MA values for the most recent closed bar (index 1)
    double maShort1 = iMA(Symbol(), Period(), MovingAveragePeriodShort, 0, MovingAverageMethod, MovingAverageAppliedPrice, 1);
    double maLong1  = iMA(Symbol(), Period(), MovingAveragePeriodLong,  0, MovingAverageMethod, MovingAverageAppliedPrice, 1);
    // Get MA values for the bar before the most recent closed bar (index 2) for confirmation
    double maShort2 = iMA(Symbol(), Period(), MovingAveragePeriodShort, 0, MovingAverageMethod, MovingAverageAppliedPrice, 2);
    double maLong2  = iMA(Symbol(), Period(), MovingAveragePeriodLong,  0, MovingAverageMethod, MovingAverageAppliedPrice, 2);


    if (maShort1 == EMPTY_VALUE || maLong1 == EMPTY_VALUE || maShort2 == EMPTY_VALUE || maLong2 == EMPTY_VALUE)
    {
        if (Debug_Verbose) Print("DEBUG: GetMATrendDirection - iMA returned EMPTY_VALUE. Data not ready?");
        return 0; // Undefined trend
    }

    // Check for a confirmed crossover or sustained position
    if (maShort1 > maLong1 && maShort2 <= maLong2) return 1; // Bullish crossover
    if (maShort1 > maLong1 && maShort2 > maLong2) return 1;  // Sustained bullish
    if (maShort1 < maLong1 && maShort2 >= maLong2) return -1; // Bearish crossover
    if (maShort1 < maLong1 && maShort2 < maLong2) return -1; // Sustained bearish


    return 0; // Sideways or undefined
}

// Finds the bar index of a swing point (High or Low) within a range
// This function is used by FindLastSwingLowBeforeBar/FindLastSwingHighBeforeBar
int FindSwingPoint(int searchStartBar, int searchEndBar, int swingStrength, int swingType)
{
    if (swingStrength <= 0)
    {
        if (Debug_Verbose) Print("DEBUG: FindSwingPoint - Invalid swingStrength (", swingStrength, ").");
        return -1;
    }

    // Ensure enough bars exist for the swing strength calculation around any potential point in the range
    if (Bars < swingStrength * 2 + 1)
    {
        // This check should ideally be done before calling, but included for safety
        if (Debug_Verbose) Print("DEBUG: FindSwingPoint - Not enough total bars (", Bars, ") for swingStrength ", swingStrength);
        return -1;
    }

    // Adjust search range to avoid looking beyond available bars and boundaries required for swing check
    // Search is from older bars (larger index) to more recent bars (smaller index)
    // The range [searchEndBar, searchStartBar] is the area to look for the swing point.
    // Any bar 'i' in this range must have swingStrength bars before (i-swingStrength) and after (i+swingStrength).
    int actualStart = MathMin(searchStartBar, Bars - 1 - swingStrength);
    int actualEnd   = MathMax(searchEndBar, swingStrength);

    if (actualStart < actualEnd)
    {
        if (Debug_Verbose) Print("DEBUG: FindSwingPoint - Invalid search range after adjustment: [", actualStart, " down to ", actualEnd, "] for strength ", swingStrength);
        return -1; // Invalid range
    }

    for (int i = actualStart; i >= actualEnd; i--)
    {
        // Ensure there are enough bars before and after the current bar 'i' for swing check
        if (i < swingStrength || i >= Bars - swingStrength) continue;

        bool isSwing = true;

        if (swingType == 1) // Swing High
        {
            double currentHigh = High[i];
            for (int j = 1; j <= swingStrength; j++)
            {
                if (High[i - j] >= currentHigh || High[i + j] >= currentHigh) // Check bars before and after
                {
                    isSwing = false;
                    break;
                }
            }
            if (isSwing)
            {
                if (Debug_Verbose) Print("DEBUG: FindSwingPoint - Found Swing HIGH (Type 1) with strength ", swingStrength, " at Bar ", i, " Price=", NormalizeDouble(currentHigh, _Digits));
                return i; // Return the bar index of the swing point
            }
        }
        else if (swingType == -1) // Swing Low
        {
            double currentLow = Low[i];
            for (int j = 1; j <= swingStrength; j++)
            {
                if (Low[i - j] <= currentLow || Low[i + j] <= currentLow) // Check bars before and after
                {
                    isSwing = false;
                    break;
                }
            }
            if (isSwing)
            {
                if (Debug_Verbose) Print("DEBUG: FindSwingPoint - Found Swing LOW (Type -1) with strength ", swingStrength, " at Bar ", i, " Price=", NormalizeDouble(currentLow, _Digits));
                return i; // Return the bar index of the swing point
            }
        }
    }

    return -1; // Swing point not found in the specified range
}

// FindPattern function from user snippet - likely not used in the main logic based on the new strategy description.
// Keeping it here but commented out of the main logic flow for now.

bool FindPattern(int& out_point1Bar, double& out_point1Price,
                 int& out_point2Bar, double& out_point2Price,
                 int& out_intendedTradeDirection)
{
    out_point1Bar = -1;
    out_point1Price = 0.0;
    out_point2Bar = -1; out_point2Price = 0.0;
    out_intendedTradeDirection = 0;

    // Initial check for enough bars (can be complex depending on exact needs)
    // The existing check is kept for now, might need refinement based on PatternLookbackBars usage
    if (Bars < PatternLookbackBars_P1 + PatternLookbackBars_P2 + MaxSwingStrength + 2)
    {
        if (Debug_Verbose) Print("DEBUG: FindPattern - Not enough total bars (", Bars, ") for pattern search with lookbacks P1:", PatternLookbackBars_P1, " P2:", PatternLookbackBars_P2);
        return false;
    }

    for (int currentSwingStrength = MaxSwingStrength; currentSwingStrength >= MinSwingStrength; currentSwingStrength--)
    {
        if (Debug_Verbose) Print("DEBUG: FindPattern - Searching with SwingStrength = ", currentSwingStrength);

        // Check for enough bars for the current swing strength search range
        // This check was potentially confusing; leaving it as is for now but might need review
        if (Bars < currentSwingStrength * 2 + currentSwingStrength * 2 + 3)
        {
            if (Debug_Verbose) Print("DEBUG: FindPattern - Not enough bars for currentSwingStrength ", currentSwingStrength);
            continue;
        }

        // Calculate the search range for P1
        int p1_searchStartBar = MathMin(PatternLookbackBars_P1, Bars - 1 - (currentSwingStrength + 1 + 1));
        int p1_searchEndBar   = currentSwingStrength + PatternLookbackBars_P2 + 1;
        p1_searchEndBar = MathMax(p1_searchEndBar, currentSwingStrength);

        if (p1_searchStartBar < p1_searchEndBar)
        {
            if (Debug_Verbose) Print("DEBUG: FindPattern - Invalid P1 search range [", p1_searchStartBar, " down to ", p1_searchEndBar, "] for strength ", currentSwingStrength);
            continue;
        }

        // --- Search for BUY pattern ---
        // Search for P1: Swing High (Resistance)
        int p1_bar = FindSwingPoint(p1_searchStartBar, p1_searchEndBar, currentSwingStrength, 1);

        if (p1_bar != -1)
        {
            double p1_price = High[p1_bar];
            if (Debug_Verbose) Print("DEBUG: FindPattern (BUY) - Potential P1 (HH1 - Resistance) at Bar ", p1_bar, " Price=", NormalizeDouble(p1_price, _Digits));

            // Calculate search range for P2 (must be after P1 and within P2 lookback)
            int p2_searchStartBar_temp = p1_bar - 1;
            int p2_searchEndBar_temp   = MathMax(currentSwingStrength + 1, p1_bar - PatternLookbackBars_P2); // P2 must be at least 'currentSwingStrength + 1' bars from bar 0

            if (p2_searchStartBar_temp < p2_searchEndBar_temp)
            {
                 if (Debug_Verbose) Print("DEBUG: FindPattern (BUY) - Invalid P2 search range [", p2_searchStartBar_temp, " down to ", p2_searchEndBar_temp, "] after P1 at bar ", p1_bar);
            }
            else // If P2 search range is valid
            {
                // Search for P2: Swing Low (Retest point after potential breakout)
                int p2_bar = FindSwingPoint(p2_searchStartBar_temp, p2_searchEndBar_temp, currentSwingStrength, -1);

                if (p2_bar != -1)
                {
                    double p2_price = Low[p2_bar];

                    // --- IMPORTANT: Review and correct this price condition for BUY retest ---
                    // Current condition: P2 Low < P1 High (Likely incorrect for retest)
                    // For a retest BUY, P2 Low should be near or above P1 High after breakout
                    if (p2_price < p1_price) // <-- THIS CONDITION NEEDS TO BE VERIFIED/CORRECTED BASED ON STRATEGY
                    {
                        // Pattern found
                        out_point1Bar = p1_bar;
                        out_point1Price = p1_price;
                        out_point2Bar = p2_bar;
                        out_point2Price = p2_price;
                        out_intendedTradeDirection = 1; // BUY

                        if (Debug_Verbose) Print("DEBUG: FindPattern - Found BUY Pattern: P1(HH1) Bar=", p1_bar, " Price=", NormalizeDouble(p1_price,_Digits),
                                                  "; P2(HL1) Bar=", p2_bar, " Price=", NormalizeDouble(p2_price,_Digits));

                        return true; // Pattern found, exit function
                    }
                    else
                    {
                        if (Debug_Verbose) Print("DEBUG: FindPattern (BUY) - P2 Low (", NormalizeDouble(p2_price,_Digits), ") did not meet price condition relative to P1 High (", NormalizeDouble(p1_price,_Digits), ")");
                    }
                }
                else
                {
                     if (Debug_Verbose) Print("DEBUG: FindPattern (BUY) - Could not find P2 (HL1) after P1 at Bar ", p1_bar, " in range [", p2_searchStartBar_temp, " down to ", p2_searchEndBar_temp, "]");
                }
            }
        }

        // --- Search for SELL pattern ---
        // Search for P1: Swing Low (Support)
        p1_bar = FindSwingPoint(p1_searchStartBar, p1_searchEndBar, currentSwingStrength, -1);

        if (p1_bar != -1)
        {
            double p1_price = Low[p1_bar];
            if (Debug_Verbose) Print("DEBUG: FindPattern (SELL) - Potential P1 (LL1 - Support) at Bar ", p1_bar, " Price=", NormalizeDouble(p1_price, _Digits));

            // Calculate search range for P2 (must be after P1 and within P2 lookback)
            int p2_searchStartBar_temp = p1_bar - 1;
            int p2_searchEndBar_temp   = MathMax(currentSwingStrength + 1, p1_bar - PatternLookbackBars_P2); // P2 must be at least 'currentSwingStrength + 1' bars from bar 0

            if (p2_searchStartBar_temp < p2_searchEndBar_temp)
            {
                 if (Debug_Verbose) Print("DEBUG: FindPattern (SELL) - Invalid P2 search range [", p2_searchStartBar_temp, " down to ", p2_searchEndBar_temp, "] after P1 at bar ", p1_bar);
            }
            else // If P2 search range is valid
            {
                // Search for P2: Swing High (Retest point after potential breakout)
                int p2_bar = FindSwingPoint(p2_searchStartBar_temp, p2_searchEndBar_temp, currentSwingStrength, 1);

                if (p2_bar != -1)
                {
                    double p2_price = High[p2_bar];

                    // --- IMPORTANT: Review and correct this price condition for SELL retest ---
                    // Current condition: P2 High > P1 Low (Likely incorrect for retest)
                    // For a retest SELL, P2 High should be near or below P1 Low after breakout
                    if (p2_price > p1_price) // <-- THIS CONDITION NEEDS TO BE VERIFIED/CORRECTED BASED ON STRATEGY
                    {
                        // Pattern found
                        out_point1Bar = p1_bar;
                        out_point1Price = p1_price;
                        out_point2Bar = p2_bar;
                        out_point2Price = p2_price;
                        out_intendedTradeDirection = -1; // SELL

                        if (Debug_Verbose) Print("DEBUG: FindPattern - Found SELL Pattern: P1(LL1) Bar=", p1_bar, " Price=", NormalizeDouble(p1_price,_Digits),
                                                  "; P2(LH1) Bar=", p2_bar, " Price=", NormalizeDouble(p2_price,_Digits));

                        return true; // Pattern found, exit function
                    }
                    else
                    {
                        if (Debug_Verbose) Print("DEBUG: FindPattern (SELL) - P2 High (", NormalizeDouble(p2_price,_Digits), ") did not meet price condition relative to P1 Low (", NormalizeDouble(p1_price,_Digits), ")");
                    }
                }
                else
                {
                     if (Debug_Verbose) Print("DEBUG: FindPattern (SELL) - Could not find P2 (LH1) after P1 at Bar ", p1_bar, " in range [", p2_searchStartBar_temp, " down to ", p2_searchEndBar_temp, "]");
                }
            }
        }
    }

    // If the loop finishes without finding a pattern
    return false;
}


//+------------------------------------------------------------------+
//| New Helper Functions for Strategy Logic                          |
//+------------------------------------------------------------------+

// Finds the bar index of the highest high in a specified range (from startBar back by count)
// search is from startBar (more recent) to startBar + count - 1 (older)
int FindHighestHighBarInRange(int count, int startBar)
{
    if (count <= 0 || startBar < 0)
    {
        if (Debug_Verbose) Print("DEBUG: FindHighestHighBarInRange - Invalid input. Count=", count, ", StartBar=", startBar);
        return -1;
    }

    // Ensure enough bars are available for the requested range
    if (startBar + count > Bars)
    {
        if (Debug_Verbose) Print("DEBUG: FindHighestHighBarInRange - Not enough bars. Bars=", Bars, ", Need at least ", startBar + count);
        return -1;
    }

    int highestBar = -1;
    double highestPrice = -1.0; // Initialize with a very low price

    // Iterate from the start bar backwards for 'count' bars
    for (int i = startBar; i < startBar + count; i++)
    {
        if (High[i] > highestPrice)
        {
            highestPrice = High[i];
            highestBar = i;
        }
    }

    if (Debug_Verbose && highestBar != -1)
    {
        Print("DEBUG: FindHighestHighBarInRange - Found highest high at bar ", highestBar, " Price=", NormalizeDouble(highestPrice, _Digits), " in range [", startBar, " to ", startBar + count - 1, "]");
    }

    return highestBar;
}

// Finds the bar index of the lowest low in a specified range (from startBar back by count)
// search is from startBar (more recent) to startBar + count - 1 (older)
int FindLowestLowBarInRange(int count, int startBar)
{
    if (count <= 0 || startBar < 0)
    {
        if (Debug_Verbose) Print("DEBUG: FindLowestLowBarInRange - Invalid input. Count=", count, ", StartBar=", startBar);
        return -1;
    }

    // Ensure enough bars are available for the requested range
    if (startBar + count > Bars)
    {
        if (Debug_Verbose) Print("DEBUG: FindLowestLowBarInRange - Not enough bars. Bars=", Bars, ", Need at least ", startBar + count);
        return -1;
    }


    int lowestBar = -1;
    double lowestPrice = MarketInfo(Symbol(), MODE_LOW); // Initialize with a reasonably high price


    // Iterate from the start bar backwards for 'count' bars
    for (int i = startBar; i < startBar + count; i++)
    {
        if (Low[i] < lowestPrice)
        {
            lowestPrice = Low[i];
            lowestBar = i;
        }
    }

    if (Debug_Verbose && lowestBar != -1)
    {
        Print("DEBUG: FindLowestLowBarInRange - Found lowest low at bar ", lowestBar, " Price=", NormalizeDouble(lowestPrice, _Digits), " in range [", startBar, " to ", startBar + count - 1, "]");
    }

    return lowestBar;
}


// Finds the most recent swing low before a specific bar within a lookback
// Searches backward from startSearchBar up to startSearchBar + lookbackBars
int FindLastSwingLowBeforeBar(int startSearchBar, int lookbackBars, int swingStrength)
{
    if (lookbackBars <= 0 || startSearchBar < 0)
    {
        if (Debug_Verbose) Print("DEBUG: FindLastSwingLowBeforeBar - Invalid input. StartSearchBar=", startSearchBar, ", Lookback=", lookbackBars);
        return -1;
    }

    // Define the search range for FindSwingPoint (older bar index, more recent bar index)
    int searchEndBar = startSearchBar; // Most recent bar index to search from (exclusive, as we search *before* startSearchBar)
    int searchStartBar = MathMin(startSearchBar + lookbackBars -1, Bars - 1); // Older bar index to search up to

    // Ensure enough bars for swing point calculation around any point in the search range
    if (Bars < swingStrength * 2 + 1 || searchStartBar < swingStrength || searchEndBar + swingStrength >= Bars)
    {
        if (Debug_Verbose) Print("DEBUG: FindLastSwingLowBeforeBar - Not enough bars for swing strength or search range. Bars=", Bars, ", SwingStrength=", swingStrength, ", SearchRange=[", searchEndBar, " to ", searchStartBar, "]");
        return -1;
    }


    // The FindSwingPoint function searches backward from start to end.
    // We want to search backward from 'startSearchBar' up to 'searchStartBar + lookbackBars'.
    // Let's define the search range for FindSwingPoint correctly.
    // We are looking for a swing point with index 'i' such that i < startSearchBar.
    // The search should go from startSearchBar - 1 backwards.
    // The furthest bar back to search is startSearchBar + lookbackBars - 1.
    // The FindSwingPoint searches from start (older index) to end (more recent index) backwards.
    // So, the start for FindSwingPoint is startSearchBar + lookbackBars - 1.
    // The end for FindSwingPoint is startSearchBar.

    int searchRangeEnd = startSearchBar;
    int searchRangeStart = MathMin(startSearchBar + lookbackBars, Bars); // Adjusted to not go beyond Bars

    // FindSwingPoint(older index, more recent index, ...)
    return FindSwingPoint(searchRangeStart -1, searchRangeEnd, swingStrength, -1); // Search for Swing Low
}

// Finds the most recent swing high before a specific bar within a lookback
// Searches backward from startSearchBar up to startSearchBar + lookbackBars
int FindLastSwingHighBeforeBar(int startSearchBar, int lookbackBars, int swingStrength)
{
    if (lookbackBars <= 0 || startSearchBar < 0)
    {
        if (Debug_Verbose) Print("DEBUG: FindLastSwingHighBeforeBar - Invalid input. StartSearchBar=", startSearchBar, ", Lookback=", lookbackBars);
        return -1;
    }

    // Define the search range for FindSwingPoint
    int searchEndBar = startSearchBar; // Most recent bar index to search from (exclusive)
    int searchStartBar = MathMin(startSearchBar + lookbackBars - 1, Bars - 1); // Older bar index to search up to

    // Ensure enough bars for swing point calculation around any point in the search range
    if (Bars < swingStrength * 2 + 1 || searchStartBar < swingStrength || searchEndBar + swingStrength >= Bars)
    {
        if (Debug_Verbose) Print("DEBUG: FindLastSwingHighBeforeBar - Not enough bars for swing strength or search range. Bars=", Bars, ", SwingStrength=", swingStrength, ", SearchRange=[", searchEndBar, " to ", searchStartBar, "]");
        return -1;
    }

    // Call FindSwingPoint to search backward for a Swing High
    // FindSwingPoint(older index, more recent index, ...)
    int searchRangeEnd = startSearchBar;
    int searchRangeStart = MathMin(startSearchBar + lookbackBars, Bars); // Adjusted to not go beyond Bars

    return FindSwingPoint(searchRangeStart - 1, searchRangeEnd, swingStrength, 1); // Search for Swing High
}


// Helper function to draw a horizontal line
void DrawHorizontalLine(string name, double price, int startBar, int durationBars, color clr, int style, int width)
{
    if (!DrawPatternLines)
    {
        // If drawing is disabled, ensure the object is deleted if it exists
        ObjectDelete(name);
        return;
    }

    // Ensure enough bars are available for the duration
    if (startBar < 0 || startBar >= Bars)
    {
        if (Debug_Verbose) Print("DEBUG: DrawHorizontalLine - Invalid startBar (", startBar, "). Skipping drawing.");
        ObjectDelete(name);
        return;
    }

    // The line will be drawn from Time[startBar] to Time[startBar + durationBars]
    int endBar = startBar + durationBars;
    if (endBar < 0) endBar = 0; // Ensure end bar is not negative
    if (endBar >= Bars) endBar = Bars - 1; // Ensure end bar is within available bars

    if (startBar > endBar) // If duration is negative, swap start and end bars
    {
        int temp = startBar;
        startBar = endBar;
        endBar = temp;
    }


    // Delete the object if it already exists to redraw it
    ObjectDelete(name);

    // Create the horizontal line object
    if (ObjectCreate(0, name, OBJ_HLINE, 0, Time[startBar], price))
    {
        // Set object properties
        ObjectSetInteger(0, name, OBJPROP_COLOR, clr);
        ObjectSetInteger(0, name, OBJPROP_STYLE, style);
        ObjectSetInteger(0, name, OBJPROP_WIDTH, width);
        ObjectSetInteger(0, name, OBJPROP_RAY, false); // Do not extend to infinity
        ObjectSetInteger(0, name, OBJPROP_SELECTABLE, false); // Make it non-selectable
        ObjectSetInteger(0, name, OBJPROP_HIDDEN, false); // Make it visible
        ObjectSetInteger(0, name, OBJPROP_BACK, false); // Draw on top of chart

        // Set the end time for the line object (end bar's time)
        if (endBar < Bars) // Ensure endBar is a valid index
          ObjectSetInteger(0, name, OBJPROP_TIME1, Time[endBar]);
        else
          ObjectSetInteger(0, name, OBJPROP_TIME1, Time[Bars - 1]); // Draw up to the last bar


        // Set the description (tooltip)
        ObjectSetString(0, name, OBJPROP_TEXT, name + " at " + DoubleToString(price, _Digits));

        if (Debug_Verbose) Print("DEBUG: DrawHorizontalLine - Drawn line '", name, "' at price ", NormalizeDouble(price, _Digits), " from bar ", startBar, " to bar ", endBar);

    }
    else
    {
        if (Debug_Verbose) Print("DEBUG: DrawHorizontalLine - Failed to create line object '", name, "'. Error: ", GetMQL4ErrorDescription(GetLastError()));
    }
}

// Helper function to delete a chart object
void DeleteObject(string name)
{
    if (ObjectFind(0, name) >= 0)
    {
        if (ObjectDelete(name))
        {
            if (Debug_Verbose) Print("DEBUG: DeleteObject - Object '", name, "' deleted.");
        }
        else
        {
            if (Debug_Verbose) Print("DEBUG: DeleteObject - Failed to delete object '", name, "'. Error: ", GetMQL4ErrorDescription(GetLastError()));
        }
    }
}

// Helper function to delete all objects created by this EA's magic number
void DeleteAllObjectsByMagic()
{
    ObjectsDeleteAll(0, -1, MagicNumber);
    if (Debug_Verbose) Print("DEBUG: All objects with MagicNumber ", MagicNumber, " deleted.");
}


//+------------------------------------------------------------------+
//| Expert initialization function                                     |
//+------------------------------------------------------------------+
int OnInit()
{
    //--- Expert advisor initialization code ---

    // Initialize object names here now that MagicNumber is declared
    resistanceLineName = "ResistanceLine_" + IntegerToString(MagicNumber);
    supportLineName = "SupportLine_" + IntegerToString(MagicNumber);
    pendingOrderLineName = "PendingOrderLevel_" + IntegerToString(MagicNumber); // Line at the entry price


    // Check minimum and maximum stop loss inputs
    if (MinStopLossPips <= 0)
    {
        Print("ERROR: Minimum Stop Loss (MinStopLossPips) must be greater than 0.");
        return(INIT_FAILED);
    }
    if (MaxStopLossPips < MinStopLossPips)
    {
        Print("ERROR: Maximum Stop Loss (MaxStopLossPips) cannot be less than Minimum Stop Loss.");
        return(INIT_FAILED);
    }

    // Check risk percentage
    if (RiskPercentage <= 0 || RiskPercentage > 100)
    {
        Print("ERROR: Risk Percentage (RiskPercentage) must be between 0 and 100.");
        return(INIT_FAILED);
    }

    // Check lookback bars
    if (ResistanceSupportLookbackBars <= 0)
    {
        Print("ERROR: Resistance/Support Lookback Bars (ResistanceSupportLookbackBars) must be greater than 0.");
        return(INIT_FAILED);
    }

    // Check SL swing strength
    if (SLSwingStrength <= 0)
    {
        Print("ERROR: SL Swing Strength (SLSwingStrength) must be greater than 0.");
        return(INIT_FAILED);
    }
    // Ensure enough bars for SL swing strength calculation
    if (Bars < SLSwingStrength * 2 + 1)
    {
        Print("ERROR: Not enough historical data for the specified SL Swing Strength. Need at least ", SLSwingStrength * 2 + 1, " bars.");
        return(INIT_FAILED);
    }

    // Check drawing duration
    if (DrawnLineDurationBars <= 0)
    {
        if (Debug_Verbose) Print("WARNING: Drawn Line Duration Bars (DrawnLineDurationBars) is <= 0. Lines will not be drawn forward.");
    }


    // Initialize the CTrade object
    trade.SetExpertMagic(MagicNumber);
    trade.SetDeviationInPoints(Slippage); // Use Slippage input for deviation

    // Initial check for sufficient bars for pattern detection and SL search
    // Need enough bars for the lookback (ResistanceSupportLookbackBars)
    // and enough bars before the breakout bar for the SL swing point search (SLSwingStrength * 2 + 1)
    // The SL search range goes back from bar 1 by a lookback amount.
    // Let's ensure we have enough bars for the initial level search (bars 1 to ResistanceSupportLookbackBars)
    // and for the SL search *before* bar 1. The SL search goes back from bar 1.
    // A conservative check is needed. We need at least `ResistanceSupportLookbackBars` bars for the initial level search
    // and potentially up to `SLSwingStrength * 2 + 1` bars before the SL point itself.
    // Let's ensure enough bars for the lookback + the maximum potential index accessed during SL search.
    // The SL search starts from bar 1 and goes back. The furthest bar accessed is 1 + lookbackBars + swingStrength.
    if (Bars < ResistanceSupportLookbackBars + SLSwingStrength * 2 + 5) // Adding buffer
    {
        Print("ERROR: Not enough historical data for the specified lookback bars and SL swing strength.");
        return(INIT_FAILED);
    }


    if (Debug_Verbose) Print("DEBUG: OnInit completed successfully.");
    return(INIT_SUCCEEDED);
}
//+------------------------------------------------------------------+
//| Expert deinitialization function                               |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
    //--- Expert deinitialization code ---
    if (Debug_Verbose) Print("DEBUG: OnDeinit called, Reason: ", reason);

    // Delete all chart objects created by this EA's magic number
    DeleteAllObjectsByMagic();

    // Clean up the CTrade object (though not strictly necessary, good practice)
    // trade.Deinit(); // CTrade does not have a Deinit method

}
//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
{
    // Check for new bar
    if (lastBarTime != Time[0])
    {
        lastBarTime = Time[0];
        // New bar has started, execute logic

        if (Debug_Verbose) Print("DEBUG: New Bar at ", Time[0]);

        // Need enough bars for calculations (MA, Swing Points, etc.)
        // A comprehensive check was done in OnInit, but ensure enough bars for current operations if needed
        if (Bars < MathMax(MathMax(MovingAveragePeriodShort, MovingAveragePeriodLong), SLSwingStrength * 2 + 1) + 2)
        {
            if (Debug_Verbose) Print("DEBUG: Not enough bars on new bar for calculations. Bars=", Bars);
            return; // Wait for more bars
        }


        //--- Check for expired pending order ---
        if (pendingOrderTicket != -1)
        {
            bool foundOrder = false;
            for (int i = 0; i < OrdersTotal(); i++)
            {
                if (OrderSelect(i, SELECT_BY_POS, MODE_TRADES))
                {
                    if (OrderMagicNumber() == MagicNumber && OrderTicket() == pendingOrderTicket)
                    {
                        foundOrder = true;

                        // Check if the pending order has expired based on bars
                        // iBarShift returns the bar index for a given time.
                        int placementBar = iBarShift(Symbol(), Period(), pendingOrderPlacementTime);

                        if (placementBar != -1) // Check if placement bar was found
                        {
                            int barsSincePlacement = placementBar - 0; // Bars since placement = placement bar index - current bar index (0)
                            if (barsSincePlacement >= PendingOrderExpiryBars)
                            {
                                if (Debug_Verbose) Print("DEBUG: Pending order #", pendingOrderTicket, " expired (", barsSincePlacement, " bars since placement). Cancelling.");
                                if (trade.OrderDelete(pendingOrderTicket))
                                {
                                    pendingOrderTicket = -1; // Reset ticket
                                    pendingOrderType = -1;
                                    pendingOrderPlacementTime = 0;
                                    // Delete the pending order entry line
                                    DeleteObject(pendingOrderLineName);
                                }
                                else
                                {
                                    if (Debug_Verbose) Print("DEBUG: Failed to cancel pending order #", pendingOrderTicket, ". Error: ", GetMQL4ErrorDescription(GetLastError()));
                                }
                            }
                            // If the pending order was filled, reset the ticket
                            else if (OrderType() == OP_BUY || OrderType() == OP_SELL)
                            {
                                if (Debug_Verbose) Print("DEBUG: Pending order #", pendingOrderTicket, " was filled.");
                                pendingOrderTicket = -1;
                                pendingOrderType = -1;
                                pendingOrderPlacementTime = 0;
                                // Delete the pending order entry line
                                DeleteObject(pendingOrderLineName);
                            }
                        }
                        else
                        {
                            if (Debug_Verbose) Print("DEBUG: Could not find placement bar for pending order #", pendingOrderTicket, ". Time=", pendingOrderPlacementTime);
                            // If placement bar not found, maybe data history changed or error occurred.
                            // Consider cancelling or just printing a warning. Let's print warning for now.
                        }
                        break; // Found the order, exit loop
                    }
                }
            }
            if (!foundOrder)
            {
                // If the order ticket is set but the order is no longer found (e.g., manually closed, expired by broker)
                if (Debug_Verbose) Print("DEBUG: Pending order ticket #", pendingOrderTicket, " not found. Resetting state.");
                pendingOrderTicket = -1;
                pendingOrderType = -1;
                pendingOrderPlacementTime = 0;
                DeleteObject(pendingOrderLineName);
            }
        }

        //--- Strategy Logic: Find Pattern and Place Order if no pending order ---
        if (pendingOrderTicket == -1)
        {
            if (Debug_Verbose) Print("DEBUG: Searching for a new pattern.");

            int tradeDirection = 0; // 1 for BUY, -1 for SELL

            // --- Trend Filter Check ---
            if (UseMATrendFilter)
            {
                int trend = GetMATrendDirection();
                if (Debug_Verbose) Print("DEBUG: MA Trend Direction: ", trend);
                if (trend == 1) tradeDirection = 1; // Only look for BUY in bullish trend
                else if (trend == -1) tradeDirection = -1; // Only look for SELL in bearish trend
                else
                {
                    if (Debug_Verbose) Print("DEBUG: Trend is neutral based on MA filter. Skipping trade search.");
                    // If trend is neutral and filter is on, do not look for trades
                    identifiedLevelBar = -1; // Reset identified level
                    identifiedLevelPrice = 0.0;
                    identifiedLevelType = 0;
                    // Delete old drawn lines if they exist
                    DeleteObject(resistanceLineName);
                    DeleteObject(supportLineName);
                    return; // Exit OnTick as no trade allowed
                }
            }
            else
            {
                // If MA filter is off, check for both BUY and SELL patterns
                tradeDirection = 0; // Will be set to 1 or -1 if a pattern is found
            }

            // Ensure enough bars for the lookback for level finding
            if (Bars < ResistanceSupportLookbackBars + 1)
            {
                if (Debug_Verbose) Print("DEBUG: Not enough bars (", Bars, ") for Resistance/Support lookback (", ResistanceSupportLookbackBars, ").");
                return; // Cannot proceed without enough data
            }


            // --- Find Resistance (for BUY) or Support (for SELL) ---
            int currentLevelBar = -1;
            double currentLevelPrice = 0.0;
            int currentLevelType = 0;

            bool patternFound = false;

            // Find potential Resistance (Highest High) - Search from bar 1 backwards
            int resistanceBar = FindHighestHighBarInRange(ResistanceSupportLookbackBars, 1);

            if (resistanceBar != -1)
            {
                double resistancePrice = High[resistanceBar];
                if (Debug_Verbose) Print("DEBUG: Identified potential Resistance at Bar ", resistanceBar, " Price=", NormalizeDouble(resistancePrice, _Digits));

                // Check for BUY Breakout: Current candle closed above resistance + buffer
                double breakoutLevelBuy = resistancePrice + PipsToPrice(EntryBufferPips);
                if (Close[0] > breakoutLevelBuy) // Check if the current candle closed above the breakout level
                {
                    if (!UseMATrendFilter || (UseMATrendFilter && GetMATrendDirection() == 1)) // Check MA filter if enabled
                    {
                        if (Debug_Verbose) Print("DEBUG: BUY Breakout confirmed at price ", NormalizeDouble(Close[0], _Digits), " above Resistance at ", NormalizeDouble(resistancePrice, _Digits), ". Checking for trade conditions.");

                        // Find the last Higher Low before the breakout bar (bar 0)
                        // Search from bar 1 backwards up to a reasonable lookback (e.g., the resistance bar + some buffer)
                        int slSearchLookbackBars = resistanceBar + SLSwingStrength * 2 + 5; // Define a reasonable lookback for SL search

                        int slBar = FindLastSwingLowBeforeBar(1, slSearchLookbackBars, SLSwingStrength); // Search from bar 1 backwards


                        if (slBar != -1)
                        {
                            double slPrice = Low[slBar] - PipsToPrice(SLBufferPips);

                            // Validate Stop Loss distance (must be positive and within min/max)
                            double entryPrice = resistancePrice + PipsToPrice(EntryBufferPips);
                            double stopLossDistancePips = NormalizeDouble((entryPrice - slPrice) / _Point, 1); // Distance in pips

                            if (Debug_Verbose) Print("DEBUG: Calculated SL Distance Pips: ", stopLossDistancePips, " Min SL Pips: ", MinStopLossPips, " Max SL Pips: ", MaxStopLossPips);


                            if (stopLossDistancePips > 0 && stopLossDistancePips >= MinStopLossPips && stopLossDistancePips <= MaxStopLossPips)
                            {
                                // Calculate Take Profit price
                                // User's rule: TP distance = (SL distance in pips + SL Buffer Pips) * RewardRiskRatio
                                // Note: SLBufferPips is already included in stopLossDistancePips calculation based on SLPrice
                                // Let's re-read the rule: "ریوارد دوبرابر فاصله استاپ +بافر" implies TP distance is (SL distance * RewardRiskRatio) + some buffer.
                                // The user's text might mean the TP is 2*SL distance + SLBufferPips added to the TP price.
                                // Let's assume TP distance is (Raw SL distance from entry to SL point) * RRR + SLBufferPips (or another buffer)
                                // Let's go with the simplest interpretation based on "ریوارد دوبرابر فاصله استاپ +بافر":
                                // TP price = Entry Price + (SL Distance in Price) * RewardRiskRatio
                                double slDistancePrice = entryPrice - slPrice;
                                double tpDistancePrice = slDistancePrice * RewardRiskRatio;
                                double tpPrice = entryPrice + tpDistancePrice;

                                // Calculate Lot Size
                                double lotSize = CalculateLotSize(RiskPercentage, stopLossDistancePips); // Calculate based on SL distance in pips

                                if (lotSize > 0)
                                {
                                    // Place BUY LIMIT order
                                    double normalizedEntryPrice = NormalizeDouble(entryPrice, _Digits);
                                    double normalizedSlPrice    = NormalizeDouble(slPrice, _Digits);
                                    double normalizedTpPrice    = NormalizeDouble(tpPrice, _Digits);

                                    if (Debug_Verbose) Print("DEBUG: Placing BUY LIMIT order. Entry=", normalizedEntryPrice, ", SL=", normalizedSlPrice, ", TP=", normalizedTpPrice, ", Lot=", NormalizeDouble(lotSize, 2));

                                    if (trade.BuyLimit(lotSize, normalizedEntryPrice, normalizedSlPrice, normalizedTpPrice))
                                    {
                                        pendingOrderTicket = trade.ResultOrder();
                                        pendingOrderType = OP_BUYLIMIT;
                                        pendingOrderPlacementTime = Time[0]; // Store placement time (current bar time)

                                        identifiedLevelBar = resistanceBar; // Store the bar/price of the resistance level that triggered the trade
                                        identifiedLevelPrice = resistancePrice;
                                        identifiedLevelType = 1; // BUY

                                        identifiedSLBar = slBar; // Store the bar/price of the SL swing point
                                        identifiedSLPrice = Low[slBar];

                                        patternFound = true; // Indicate pattern found and order placed

                                        if (Debug_Verbose) Print("DEBUG: BUY LIMIT order placed successfully. Ticket #", pendingOrderTicket);

                                        // Draw the relevant lines
                                        if (DrawPatternLines)
                                        {
                                            // Draw the original resistance line up to the breakout bar (bar 0)
                                            DrawHorizontalLine(resistanceLineName, resistancePrice, resistanceBar, 0 - resistanceBar, ResistanceLineColor, DrawnLineStyle, DrawnLineWidth);

                                            // Draw the pending order entry line from the breakout bar (bar 0) for the specified duration
                                            DrawHorizontalLine(pendingOrderLineName, normalizedEntryPrice, 0, DrawnLineDurationBars, PendingOrderLineColor, DrawnLineStyle, DrawnLineWidth);
                                        }
                                    }
                                    else
                                    {
                                        if (Debug_Verbose) Print("DEBUG: Failed to place BUY LIMIT order. Error: ", GetMQL4ErrorDescription(GetLastError()));
                                    }
                                }
                                else
                                {
                                    if (Debug_Verbose) Print("DEBUG: Calculated Lot Size (", NormalizeDouble(lotSize, 2), ") is not valid (<= 0).");
                                }
                            }
                            else
                            {
                                if (Debug_Verbose) Print("DEBUG: Stop Loss distance (", stopLossDistancePips, " pips) is outside the allowed range [", MinStopLossPips, ", ", MaxStopLossPips, "]. Skipping trade.");
                            }
                        }
                        else
                        {
                            if (Debug_Verbose) Print("DEBUG: Could not find a valid SL Swing Low before the breakout bar. Skipping trade.");
                        }
                    }
                    else
                    {
                        if (Debug_Verbose) Print("DEBUG: BUY Breakout occurred, but MA Trend Filter is ON and trend is not bullish. Skipping trade.");
                    }
                }
                else
                {
                    // No breakout yet, just draw/update the resistance line
                    if (DrawPatternLines)
                    {
                        // Draw the resistance line from where it was found up to the current bar
                        DrawHorizontalLine(resistanceLineName, resistancePrice, resistanceBar, 0 - resistanceBar, ResistanceLineColor, DrawnLineStyle, DrawnLineWidth);
                    }
                }
            }
            else
            {
                if (Debug_Verbose) Print("DEBUG: No Resistance found in the last ", ResistanceSupportLookbackBars, " bars (excluding current).");
                // If no resistance found, ensure the line is deleted
                DeleteObject(resistanceLineName);
            }

            // If a BUY pattern was found and order placed, we stop searching for a SELL pattern in this tick
            if (patternFound) return;


            // --- Find Support (for SELL) ---
            // Find potential Support (Lowest Low) - Search from bar 1 backwards
            int supportBar = FindLowestLowBarInRange(ResistanceSupportLookbackBars, 1);

            if (supportBar != -1)
            {
                double supportPrice = Low[supportBar];
                if (Debug_Verbose) Print("DEBUG: Identified potential Support at Bar ", supportBar, " Price=", NormalizeDouble(supportPrice, _Digits));

                // Check for SELL Breakout: Current candle closed below support - buffer
                double breakoutLevelSell = supportPrice - PipsToPrice(EntryBufferPips);
                if (Close[0] < breakoutLevelSell) // Check if the current candle closed below the breakout level
                {
                    if (!UseMATrendFilter || (UseMATrendFilter && GetMATrendDirection() == -1)) // Check MA filter if enabled
                    {
                        if (Debug_Verbose) Print("DEBUG: SELL Breakout confirmed at price ", NormalizeDouble(Close[0], _Digits), " below Support at ", NormalizeDouble(supportPrice, _Digits), ". Checking for trade conditions.");

                        // Find the last Lower High before the breakout bar (bar 0)
                        // Search from bar 1 backwards up to a reasonable lookback
                        int slSearchLookbackBars = supportBar + SLSwingStrength * 2 + 5; // Define a reasonable lookback for SL search
                        int slBar = FindLastSwingHighBeforeBar(1, slSearchLookbackBars, SLSwingStrength); // Search from bar 1 backwards

                        if (slBar != -1)
                        {
                            double slPrice = High[slBar] + PipsToPrice(SLBufferPips);

                            // Validate Stop Loss distance (must be positive and within min/max)
                            double entryPrice = supportPrice - PipsToPrice(EntryBufferPips);
                            double stopLossDistancePips = NormalizeDouble((slPrice - entryPrice) / _Point, 1); // Distance in pips

                            if (Debug_Verbose) Print("DEBUG: Calculated SL Distance Pips: ", stopLossDistancePips, " Min SL Pips: ", MinStopLossPips, " Max SL Pips: ", MaxStopLossPips);

                            if (stopLossDistancePips > 0 && stopLossDistancePips >= MinStopLossPips && stopLossDistancePips <= MaxStopLossPips)
                            {
                                // Calculate Take Profit price
                                // TP price = Entry Price - (SL Distance in Price) * RewardRiskRatio
                                double slDistancePrice = slPrice - entryPrice;
                                double tpDistancePrice = slDistancePrice * RewardRiskRatio;
                                double tpPrice = entryPrice - tpDistancePrice;


                                // Calculate Lot Size
                                double lotSize = CalculateLotSize(RiskPercentage, stopLossDistancePips); // Calculate based on SL distance in pips

                                if (lotSize > 0)
                                {
                                    // Place SELL LIMIT order
                                    double normalizedEntryPrice = NormalizeDouble(entryPrice, _Digits);
                                    double normalizedSlPrice    = NormalizeDouble(slPrice, _Digits);
                                    double normalizedTpPrice    = NormalizeDouble(tpPrice, _Digits);


                                    if (Debug_Verbose) Print("DEBUG: Placing SELL LIMIT order. Entry=", normalizedEntryPrice, ", SL=", normalizedSlPrice, ", TP=", normalizedTpPrice, ", Lot=", NormalizeDouble(lotSize, 2));

                                    if (trade.SellLimit(lotSize, normalizedEntryPrice, normalizedSlPrice, normalizedTpPrice))
                                    {
                                        pendingOrderTicket = trade.ResultOrder();
                                        pendingOrderType = OP_SELLLIMIT;
                                        pendingOrderPlacementTime = Time[0]; // Store placement time (current bar time)

                                        identifiedLevelBar = supportBar; // Store the bar/price of the support level that triggered the trade
                                        identifiedLevelPrice = supportPrice;
                                        identifiedLevelType = -1; // SELL

                                        identifiedSLBar = slBar; // Store the bar/price of the SL swing point
                                        identifiedSLPrice = High[slBar];


                                        patternFound = true; // Indicate pattern found and order placed

                                        if (Debug_Verbose) Print("DEBUG: SELL LIMIT order placed successfully. Ticket #", pendingOrderTicket);

                                        // Draw the relevant lines
                                        if (DrawPatternLines)
                                        {
                                            // Draw the original support line up to the breakout bar (bar 0)
                                            DrawHorizontalLine(supportLineName, supportPrice, supportBar, 0 - supportBar, SupportLineColor, DrawnLineStyle, DrawnLineWidth);

                                            // Draw the pending order entry line from the breakout bar (bar 0) for the specified duration
                                            DrawHorizontalLine(pendingOrderLineName, normalizedEntryPrice, 0, DrawnLineDurationBars, PendingOrderLineColor, DrawnLineStyle, DrawnLineWidth);
                                        }
                                    }
                                    else
                                    {
                                        if (Debug_Verbose) Print("DEBUG: Failed to place SELL LIMIT order. Error: ", GetMQL4ErrorDescription(GetLastError()));
                                    }
                                }
                                else
                                {
                                    if (Debug_Verbose) Print("DEBUG: Calculated Lot Size (", NormalizeDouble(lotSize, 2), ") is not valid (<= 0).");
                                }
                            }
                            else
                            {
                                if (Debug_Verbose) Print("DEBUG: Stop Loss distance (", stopLossDistancePips, " pips) is outside the allowed range [", MinStopLossPips, ", ", MaxStopLossPips, "]. Skipping trade.");
                            }
                        }
                        else
                        {
                            if (Debug_Verbose) Print("DEBUG: Could not find a valid SL Swing High before the breakout bar. Skipping trade.");
                        }
                    }
                    else
                    {
                        if (Debug_Verbose) Print("DEBUG: SELL Breakout occurred, but MA Trend Filter is ON and trend is not bearish. Skipping trade.");
                    }
                }
                else
                {
                    // No breakout yet, just draw/update the support line
                    if (DrawPatternLines)
                    {
                        // Draw the support line from where it was found up to the current bar
                        DrawHorizontalLine(supportLineName, supportPrice, supportBar, 0 - supportBar, SupportLineColor, DrawnLineStyle, DrawnLineWidth);
                    }
                }
            }
            else
            {
                if (Debug_Verbose) Print("DEBUG: No Support found in the last ", ResistanceSupportLookbackBars, " bars (excluding current).");
                // If no support found, ensure the line is deleted
                DeleteObject(supportLineName);
            }

            // Clean up the pending order line if no pending order exists after checks
            if (pendingOrderTicket == -1)
            {
                DeleteObject(pendingOrderLineName);
            }
        }
        else
        {
            // If a pending order exists, update the drawn lines to extend to the current bar
            if (DrawPatternLines)
            {
                // Update the pending order entry line
                if (ObjectFind(0, pendingOrderLineName) >= 0)
                {
                    ObjectSetInteger(0, pendingOrderLineName, OBJPROP_TIME1, Time[0]);
                }

                // Update the resistance/support line that triggered the trade
                if (identifiedLevelBar != -1)
                {
                    string levelLineName = (identifiedLevelType == 1) ? resistanceLineName : supportLineName;
                    if (ObjectFind(0, levelLineName) >= 0)
                    {
                        // Extend the line from its start bar up to the current bar
                        ObjectSetInteger(0, levelLineName, OBJPROP_TIME1, Time[0]);
                    }
                }
            }
        }
    }

    // Additional logic for tick-by-tick processing can go here if needed,
    // but the main strategy logic runs only on a new bar.
}
//+------------------------------------------------------------------+
