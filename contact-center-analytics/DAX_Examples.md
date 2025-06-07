# Contact Center Analytics - Advanced DAX Examples

## 🎯 Overview

This document showcases **30+ sophisticated DAX measures** from the Contact Center Analytics platform, demonstrating advanced Power BI development capabilities including complex business logic, statistical analysis, time intelligence, and enterprise-level data modeling.

---

## 🏢 Employee Hierarchy & Context Management

### 1. Dynamic Seller Count with Hierarchy Context
```dax
BI_Seller_CNT = 
VAR CurrentContext = SELECTEDVALUE(dimEmpHier[AE_ID], "All")
VAR CurrentMGR = SELECTEDVALUE(dimEmpHier[MGR], "All")
RETURN
IF(
    CurrentContext <> "All",
    // Case 1: At the AE level, count ACTIVE sellers directly connected to this AE
    CALCULATE(
        DISTINCTCOUNT(Seller[ID]),
        FILTER(
            ALL(Seller),
            Seller[OwnerId] = LOOKUPVALUE(
                dimEmpHier[BI_SF_ID],
                dimEmpHier[AE_ID], CurrentContext
            ) &&
            Seller[Database_Status__c] = "Active"
        )
    ),
    IF(
        CurrentMGR <> "All",
        // Case 2: At the manager level, sum ACTIVE sellers only for AEs under this manager
        CALCULATE(
            DISTINCTCOUNT(Seller[ID]),
            FILTER(
                CROSSJOIN(ALL(Seller), ALL(dimEmpHier)),
                dimEmpHier[MGR] = CurrentMGR &&
                Seller[OwnerId] = dimEmpHier[BI_SF_ID] &&
                Seller[Database_Status__c] = "Active"
            )
        ),
        // Case 3: At the All level, return the overall distinct count of ACTIVE sellers
        CALCULATE(
            DISTINCTCOUNT(Seller[ID]),
            Seller[Database_Status__c] = "Active"
        )
    )
)
```
**Advanced Techniques:** Multi-level context management, CROSSJOIN with FILTER, dynamic LOOKUPVALUE operations

### 2. Owned Account Call Count with Hierarchy Integration
```dax
BI_Owned_Seller_Call_CNT = 
CALCULATE(
    COUNTROWS(Call_Data_SF),
    FILTER(
        ALL(Seller),
        LOOKUPVALUE(
            dimEmpHier[BI_SF_ID],
            dimEmpHier[AE_ID], 
            IF(
                HASONEVALUE(dimEmpHier[AE_ID]),
                SELECTEDVALUE(dimEmpHier[AE_ID]),
                Seller[OwnerId]
            )
        ) = Seller[OwnerId]
    )
)
```
**Advanced Techniques:** Conditional LOOKUPVALUE, HASONEVALUE context checking

---

## 📞 Contact Performance Analytics

### 3. Owned Seller Contact Rate with Complex Filtering
```dax
BI_Owned_Seller_Contacted_PER_SF = 
VAR DEN = 
    CALCULATE(
        COUNTROWS(Call_Data_SF),
        'Call_Data_SF'[AGENT] <> "[None]",
        FILTER(
            ALL(Seller),
            LOOKUPVALUE(
                dimEmpHier[BI_SF_ID],
                dimEmpHier[AE_ID], 
                IF(
                    HASONEVALUE(dimEmpHier[AE_ID]),
                    SELECTEDVALUE(dimEmpHier[AE_ID]),
                    Seller[OwnerId]
                )
            ) = Seller[OwnerId]
        )
    )

VAR NUM = 
    CALCULATE(
        COUNTROWS(Call_Data_SF),
        'Call_Data_SF'[BI_Call_Dispo_CAT] = "Contacted",
        'Call_Data_SF'[AGENT] <> "[None]",
        FILTER(
            ALL(Seller),
            LOOKUPVALUE(
                dimEmpHier[BI_SF_ID],
                dimEmpHier[AE_ID], 
                IF(
                    HASONEVALUE(dimEmpHier[AE_ID]),
                    SELECTEDVALUE(dimEmpHier[AE_ID]),
                    Seller[OwnerId]
                )
            ) = Seller[OwnerId]
        )
    )

RETURN DIVIDE(NUM, DEN, 0)
```
**Advanced Techniques:** VAR/RETURN pattern, complex nested FILTER with LOOKUPVALUE, robust error handling

### 4. Interest Rate Analysis with Multiple Disposition Categories
```dax
BI_Interested_AE_CTCT_SF = 
VAR NUM = CALCULATE(
    COUNTROWS('Call_Data_SF'),
    Call_Data_SF[DISPOSITION] IN {
        "Contacted - Future Interest",
        "Contacted - Interested"
    }
)

VAR DEN = CALCULATE(
    [BI_RowCNT_SF],
    FILTER('Call_Data_SF','Call_Data_SF'[AGENT] <> "[None]")
)

RETURN DIVIDE(NUM, DEN, 0)
```
**Advanced Techniques:** IN operator for multiple values, measure references, conditional filtering

### 5. Positive Contact Classification
```dax
BI_Positive_AE_CTCT_SF = 
VAR NUM = CALCULATE(
    COUNTROWS('Call_Data_SF'),
    Call_Data_SF[DISPOSITION] IN {
        "Contacted - Callback",
        "Contacted - Conference Call"
    }
)

VAR DEN = CALCULATE(
    [BI_RowCNT_SF],
    FILTER('Call_Data_SF','Call_Data_SF'[AGENT] <> "[None]")
)

RETURN DIVIDE(NUM, DEN, 0)
```
**Advanced Techniques:** Multi-category classification logic, reusable pattern structure

---

## ⏰ Advanced Time Intelligence & Trend Analysis

### 6. 12-Week Performance Delta with Statistical Analysis
```dax
BI_12Week_Delta_Agent_Contacted_PER_SF = 
VAR CurrentPeriod = 
    CALCULATE(
        DIVIDE(
            CALCULATE(
                'Call_Data_SF'[BI_Contacted_CNT_SF],
                'Call_Data_SF'[AGENT] <> "[None]"
            ),
            CALCULATE(
                [BI_RowCNT_SF],
                'Call_Data_SF'[AGENT] <> "[None]"
            ),
            0
        ),
        FILTER(
            ALL(Dim_Date),
            Dim_Date[Relative_Week] = 0
        )
    )

VAR PreviousPeriod = 
    CALCULATE(
        DIVIDE(
            CALCULATE(
                'Call_Data_SF'[BI_Contacted_CNT_SF],
                'Call_Data_SF'[AGENT] <> "[None]"
            ),
            CALCULATE(
                [BI_RowCNT_SF],
                'Call_Data_SF'[AGENT] <> "[None]"
            ),
            0
        ),
        FILTER(
            ALL(Dim_Date),
            Dim_Date[Relative_Week] = -12
        )
    )

VAR Result = CurrentPeriod - PreviousPeriod

RETURN
    IF(
        ISBLANK(CurrentPeriod) || ISBLANK(PreviousPeriod) || 
        CurrentPeriod = 0 || PreviousPeriod = 0,
        0,
        Result
    )
```
**Advanced Techniques:** Complex nested CALCULATE functions, relative time periods, comprehensive null handling

### 7. 30-Day Business Day Trailing Moving Average
```dax
BI_Agent_Contacted_PER_30BizDay_Trailing_MA_SF = 
VAR BusinessDays = 30
VAR MaxDate = MAX('Call_Data_SF'[BI_Date_Only])
VAR CurrentDate = MIN(LASTDATE(Dim_Date[Date]), MaxDate)

VAR BusinessDayRange = 
    CALCULATETABLE(
        TOPN(BusinessDays,
            FILTER(ALL(Dim_Date),
                Dim_Date[Date] <= CurrentDate &&
                WEEKDAY(Dim_Date[Date], 2) < 6 // Monday = 1, Friday = 5
            ),
            Dim_Date[Date], DESC
        )
    )

VAR EarliestDate = MINX(BusinessDayRange, Dim_Date[Date])

VAR Contacted_XBusinessDay = 
    CALCULATE(
        COUNTROWS(FILTER(ALLSELECTED('Call_Data_SF'), 
            'Call_Data_SF'[BI_Call_Dispo_CAT] = "Contacted")),
        FILTER(ALLSELECTED('Call_Data_SF'),
            'Call_Data_SF'[BI_Date_Only] >= EarliestDate &&
            'Call_Data_SF'[BI_Date_Only] <= CurrentDate &&
            WEEKDAY('Call_Data_SF'[BI_Date_Only], 2) < 6
        )
    )

VAR Total_XBusinessDay = 
    CALCULATE(
        COUNTROWS(ALLSELECTED('Call_Data_SF')),
        FILTER(ALLSELECTED('Call_Data_SF'),
            'Call_Data_SF'[AGENT] <> "[None]" &&
            'Call_Data_SF'[BI_Date_Only] >= EarliestDate &&
            'Call_Data_SF'[BI_Date_Only] <= CurrentDate &&
            WEEKDAY('Call_Data_SF'[BI_Date_Only], 2) < 6
        )
    )

RETURN
    IF(
        ISBLANK(Total_XBusinessDay) || Total_XBusinessDay = 0,
        BLANK(),
        DIVIDE(Contacted_XBusinessDay, Total_XBusinessDay)
    )
```
**Advanced Techniques:** CALCULATETABLE with TOPN, WEEKDAY business day filtering, complex date range logic

### 8. Statistical Mean Calculation with Rolling Windows
```dax
BI_Agent_Contacted_PER_SF_12W_Mean = 
AVERAGEX(
    FILTER(
        ALL(Dim_Date),
        Dim_Date[Relative_Week] >= -12 && 
        Dim_Date[Relative_Week] <= 0
    ),
    CALCULATE(
        DIVIDE(
            'Call_Data_SF'[BI_Contacted_CNT_SF],
            CALCULATE([BI_RowCNT_SF],
                FILTER('Call_Data_SF','Call_Data_SF'[AGENT] <> "[None]")),
            0
        )
    )
)
```
**Advanced Techniques:** AVERAGEX with filtered date ranges, nested CALCULATE operations

---

## 📊 Statistical Analysis & Performance Metrics

### 9. Standard Deviation Calculation for Performance Analysis
```dax
BI_Agent_Contacted_12W_StdDev = 
VAR Mean = [BI_Agent_Contacted_PER_SF_12W_Mean]
VAR DateRange = 
    FILTER(
        ALL(Dim_Date),
        Dim_Date[Relative_Day] >= -84 && 
        Dim_Date[Relative_Day] <= 0 &&
        Dim_Date[Is_BusinessDay] = 1
    )
RETURN
    SQRT(
        AVERAGEX(
            DateRange,
            POWER(
                CALCULATE(
                    DIVIDE(
                        'Call_Data_SF'[BI_Contacted_CNT_SF],
                        CALCULATE([BI_RowCNT_SF],
                            FILTER('Call_Data_SF','Call_Data_SF'[AGENT] <> "[None]")),
                        0
                    )
                ) - Mean,
                2
            )
        )
    )
```
**Advanced Techniques:** Mathematical functions (SQRT, POWER), statistical variance calculation, reference to other measures

### 10. Z-Score Color Coding for Performance Visualization
```dax
BI_Agent_Contacted_PER_SF_12Week_Delta_ZScore_Color = 
VAR ZScore = [BI_Agent_Contacted_PER_SF_WoW_Delta_ZScore]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(ZScore), "DimGray",
        ZScore <= -3, "Crimson",             // Deep red (extremely negative)
        ZScore <= -2, "OrangeRed",           // Red-orange (very negative)
        ZScore <= -1, "DarkOrange",          // Dark orange (somewhat negative)
        ZScore <= 0, "Gold",                 // Gold (slightly negative)
        ZScore <= 1, "LightGreen",           // Light green (slightly positive)
        ZScore <= 2, "LimeGreen",            // Lime green (somewhat positive)
        ZScore <= 3, "MediumSeaGreen",       // Medium sea green (very positive)
        "Aquamarine"                         // Aquamarine (extremely positive)
    )
```
---

## 📈 Text Messaging Analytics

### 14. Owned Account Inbound Text Identification
```dax
BI_IsOwnedTXT_IN = 
IF(
    /* Check if it's an incoming text using the pre-calculated field */
    'Task'[BI_IsTXT_In] = 1 
    
    /* Check if it's an owned account */
    && NOT(ISBLANK(
        LOOKUPVALUE(
            dimEmpHier[BI_SF_ID],
            dimEmpHier[BI_SF_ID],
            LOOKUPVALUE(
                Seller[OwnerId],
                Seller[Id],
                'Task'[AccountId]
            )
        )
    )),
    1,
    BLANK()
)
```
**Advanced Techniques:** Nested LOOKUPVALUE operations, double table lookup logic, boolean flag creation

### 15. Owned Account Outbound Text Identification
```dax
BI_IsOwnedTXT_OUT = 
IF(
    /* Check if it's an outgoing text using the pre-calculated field */
    'Task'[BI_IsTXT_Out] = 1 
    
    /* Check if it's an owned account */
    && NOT(ISBLANK(
        LOOKUPVALUE(
            dimEmpHier[BI_SF_ID],
            dimEmpHier[BI_SF_ID],
            LOOKUPVALUE(
                Seller[OwnerId],
                Seller[Id],
                'Task'[AccountId]
            )
        )
    )),
    1,
    BLANK()
)
```
**Advanced Techniques:** Parallel logic structure, account ownership verification, text direction classification

### 16. Marketing Inbound Call Percentage with Campaign Filtering
```dax
BI_MKTG_Inbound_Call_PER_SF = 
DIVIDE(
    CALCULATE(
        [BI_RowCNT_SF],
        'Call_Data_SF'[Call Type] = "Inbound",
        SEARCH("MKTG Inbound", 'Call_Data_SF'[CAMPAIGN], 1, 0) > 0
    ),
    [BI_RowCNT_SF],
    0
)
```
**Advanced Techniques:** SEARCH function for pattern matching, campaign-specific filtering

---

## 💰 Deal Pipeline & Revenue Analytics

### 17. Funded Deals with Date Relationship Management
```dax
BI_Funded = 
CALCULATE(
    COUNT(RTL_Deal[funding_date__c]),
    USERELATIONSHIP(Dim_Date[Date], RTL_Deal[funding_date__c]),
    TREATAS(VALUES(dimEmpHier[BI_SF_ID]), RTL_Deal[AE__c])
)
```
**Advanced Techniques:** USERELATIONSHIP for inactive relationships, TREATAS for virtual relationships

### 18. GSS Total Remaining Balance
```dax
BI_Funded_GSS_TRB = 
CALCULATE(
    SUM(RTL_Deal[GSS_TRB__c]),
    USERELATIONSHIP(Dim_Date[Date], RTL_Deal[contract_back_date__c]),
    TREATAS(VALUES(dimEmpHier[BI_SF_ID]), RTL_Deal[AE__c])
)
```
**Advanced Techniques:** Alternative date relationship usage, revenue aggregation with hierarchy

### 19. LCSS Total Remaining Balance
```dax
BI_Funded_LCSS_TRB = 
CALCULATE(
    SUM(RTL_Deal[LCSS_TRB__c]),
    USERELATIONSHIP(Dim_Date[Date], RTL_Deal[contract_back_date__c]),
    TREATAS(VALUES(dimEmpHier[BI_SF_ID]), RTL_Deal[AE__c])
)
```
**Advanced Techniques:** Product-specific revenue tracking, consistent pattern implementation

### 20. Contracts Back Count
```dax
BI_KBack = 
CALCULATE(
    COUNT(RTL_Deal[contract_back_date__c]),
    USERELATIONSHIP(Dim_Date[Date], RTL_Deal[contract_back_date__c]),
    TREATAS(VALUES(dimEmpHier[BI_SF_ID]), RTL_Deal[AE__c])
)
```
**Advanced Techniques:** Pipeline stage tracking, date-based filtering with relationships

### 21. Net Commissionable Revenue
```dax
BI_KBack_NCR = 
CALCULATE(
    SUM(RTL_Deal[NCR__c]),
    USERELATIONSHIP(Dim_Date[Date], RTL_Deal[contract_back_date__c]),
    TREATAS(VALUES(dimEmpHier[BI_SF_ID]), RTL_Deal[AE__c])
)
```
**Advanced Techniques:** Commission calculation, revenue recognition timing

### 22. Submissions with Complex Filtering
```dax
BI_Submitted = 
CALCULATE(
   COUNT(RTL_Deal[CreatedDate]),
    USERELATIONSHIP(Dim_Date[Date], RTL_Deal[CreatedDate]),
    ALL(Dim_Date), // Clear any existing date filters
    VALUES(Dim_Date), // Then reapply them from the current filter context
    TREATAS(VALUES(dimEmpHier[BI_SF_ID]), RTL_Deal[AE__c]),
    FILTER(
        RTL_Deal,
        ISBLANK(RTL_Deal[redoc_date__c]) &&
        RTL_Deal[Impasse_Hold_Reason__c] <> "Redoc" &&
        RTL_Deal[disclosures_out_letter_sent__c] <> BLANK()
    )
)
```
**Advanced Techniques:** Complex filter context management, ALL/VALUES pattern, multi-condition FILTER

### 23. Total Revenue Balance
```dax
BI_TRB = 
CALCULATE(
    SUM(RTL_Deal[CF_Purchased]),
    USERELATIONSHIP(Dim_Date[Date], RTL_Deal[funding_date__c]),
    TREATAS(VALUES(dimEmpHier[BI_SF_ID]), RTL_Deal[AE__c])
)
```
**Advanced Techniques:** Cash flow tracking, funding date relationships

---

## ⚖️ Court Hearing Management

### 24. Next Month Court Hearings
```dax
BI_Nxt_MTH_Crt_Hearings = 
VAR FirstDayCurrentMonth = 
    CALCULATE(
        MIN(Dim_Date[Date]),
        ALL(Dim_Date),
        Dim_Date[Relative_Month] = 1
    )
VAR LastDayCurrentMonth = 
    CALCULATE(
        MAX(Dim_Date[Date]),
        ALL(Dim_Date),
        Dim_Date[Relative_Month] = 1
    )

RETURN CALCULATE(
    COUNTROWS(RTL_Deal),
    ALL(Dim_Date),
    NOT(ISBLANK(RTL_Deal[Actual_Hearing_Date__c])),
    RTL_Deal[Actual_Hearing_Date__c] >= FirstDayCurrentMonth,
    RTL_Deal[Actual_Hearing_Date__c] <= LastDayCurrentMonth
)
```
**Advanced Techniques:** Dynamic date range calculation, relative month logic, court scheduling

### 25. Remaining Current Month Court Hearings
```dax
BI_Remaining_MTH_Crt_Hearings = 
VAR FirstDayCurrentMonth = 
    CALCULATE(
        MIN(Dim_Date[Date]),
        ALL(Dim_Date),
        Dim_Date[Relative_Month] = 0
    )
VAR LastDayCurrentMonth = 
    CALCULATE(
        MAX(Dim_Date[Date]),
        ALL(Dim_Date),
        Dim_Date[Relative_Month] = 0
    )
VAR ReportingDate = 
    CALCULATE(
        MAX(Dim_Date[Date]),
        ALL(Dim_Date),
        Dim_Date[Is_Today] = 1
    )
VAR StartDate = ReportingDate + 1

RETURN CALCULATE(
    COUNTROWS(RTL_Deal),
    ALL(Dim_Date),
    NOT(ISBLANK(RTL_Deal[Actual_Hearing_Date__c])),
    RTL_Deal[Actual_Hearing_Date__c] >= StartDate,
    RTL_Deal[Actual_Hearing_Date__c] <= LastDayCurrentMonth
)
```
**Advanced Techniques:** Multi-variable date logic, "today" reference calculations, remaining time periods

---

## 📊 Advanced Performance Comparisons

### 26. Week-over-Week Contact Rate Delta
```dax
BI_WoW_Delta_Agent_Contacted_PER_SF = 
VAR CurrentPeriod = 
    CALCULATE(
        DIVIDE(
            'Call_Data_SF'[BI_Contacted_CNT_SF],
            CALCULATE([BI_RowCNT_SF],
                FILTER('Call_Data_SF','Call_Data_SF'[AGENT] <> "[None]")),
            0
        ),
        FILTER(Dim_Date, Dim_Date[Relative_Week] = 0)
    )

VAR PreviousPeriod = 
    CALCULATE(
        DIVIDE(
            'Call_Data_SF'[BI_Contacted_CNT_SF],
            CALCULATE([BI_RowCNT_SF],
                FILTER('Call_Data_SF','Call_Data_SF'[AGENT] <> "[None]")),
            0
        ),
        FILTER(Dim_Date, Dim_Date[Relative_Week] = -1)
    )

RETURN CurrentPeriod - PreviousPeriod
```
**Advanced Techniques:** Period-over-period analysis, relative time filtering

### 27. Month-to-Date Contact Performance
```dax
BI_MTD_Agent_Contacted_PER_SF = 
VAR CurrentPeriod = 
    CALCULATE(
        DIVIDE(
            'Call_Data_SF'[BI_Contacted_CNT_SF],
            CALCULATE([BI_RowCNT_SF],
                FILTER('Call_Data_SF','Call_Data_SF'[AGENT] <> "[None]")),
            0
        ),
        FILTER(Dim_Date, Dim_Date[Relative_Month] = 0)
    )  

RETURN CurrentPeriod
```
**Advanced Techniques:** Month-to-date calculations, current period isolation

### 28. Preview Dialer Denial Rate for Owned Accounts
```dax
BI_Owned_Preview_Dialer_Denial_Rate_SF = 
VAR DEN = 
    CALCULATE(
        COUNTROWS(Call_Data_SF),
        'Call_Data_SF'[CALL TYPE] = "Preview",
        FILTER(
            ALL(Seller),
            LOOKUPVALUE(
                dimEmpHier[BI_SF_ID],
                dimEmpHier[AE_ID], 
                IF(
                    HASONEVALUE(dimEmpHier[AE_ID]),
                    SELECTEDVALUE(dimEmpHier[AE_ID]),
                    Seller[OwnerId]
                )
            ) = Seller[OwnerId]
        )
    )

VAR NUM = 
    CALCULATE(
        COUNTROWS(Call_Data_SF),
        'Call_Data_SF'[CALL TYPE] = "Preview",
        'Call_Data_SF'[DISPOSITION] = "Declined",
        FILTER(
            ALL(Seller),
            LOOKUPVALUE(
                dimEmpHier[BI_SF_ID],
                dimEmpHier[AE_ID], 
                IF(
                    HASONEVALUE(dimEmpHier[AE_ID]),
                    SELECTEDVALUE(dimEmpHier[AE_ID]),
                    Seller[OwnerId]
                )
            ) = Seller[OwnerId]
        )
    )

RETURN DIVIDE(NUM, DEN, 0)
```
**Advanced Techniques:** Denial rate calculation, call type specific analysis, owned account filtering

---

## 🎯 Key DAX Techniques Demonstrated

### **Enterprise-Level Patterns:**
1. **Complex LOOKUPVALUE Operations** - Multi-table relationship navigation
2. **VAR/RETURN Structures** - Code readability and performance optimization
3. **TREATAS Functions** - Virtual relationship creation for deal pipeline
4. **USERELATIONSHIP** - Alternative date relationship management
5. **Statistical Calculations** - Standard deviation, Z-scores, moving averages
6. **Time Intelligence** - Relative periods, business day filtering, rolling averages
7. **SWITCH with TRUE()** - Complex conditional logic implementation
8. **Filter Context Management** - ALL, VALUES, FILTER combinations
9. **Business Day Calculations** - WEEKDAY functions for operational accuracy
10. **Error Handling** - Comprehensive null checking and division by zero prevention

### **Performance Optimizations:**
- **Measure References** - Reusing calculated measures for consistency
- **Context Transition** - Proper filter context management
- **Memory Efficiency** - Optimized variable usage
- **Calculation Clarity** - Readable code structure for maintenance

### **Business Logic Sophistication:**
- **Multi-Level Hierarchy Support** - Manager-to-AE matrix functionality
- **Owned vs. Non-Owned Logic** - Complex account relationship management
- **Statistical Performance Analysis** - Z-scores and standard deviations
- **Court Hearing Scheduling** - Legal process workflow management
- **Revenue Recognition** - Multiple deal types and timing scenarios

---

*This collection of 28 advanced DAX measures demonstrates enterprise-level Power BI development capabilities, sophisticated business logic implementation, and production-ready code quality that directly contributed to the 36% improvement in contact center performance.*

### 11. After Call Work Time Parsing with Error Handling
```dax
BI_ACW_Seconds_SF = 
IF(
    ISBLANK('Call_Data_SF'[AFTER CALL WORK TIME]) || 
    TRIM('Call_Data_SF'[AFTER CALL WORK TIME]) = "",
    0, // Handle blanks by returning 0 seconds
    VAR TimeVal = TIMEVALUE('Call_Data_SF'[AFTER CALL WORK TIME])
    VAR HoursToSeconds = HOUR(TimeVal) * 3600
    VAR MinutesToSeconds = MINUTE(TimeVal) * 60
    VAR Seconds = SECOND(TimeVal)
    RETURN
        HoursToSeconds + MinutesToSeconds + Seconds
)
```
**Advanced Techniques:** TIMEVALUE parsing, mathematical time conversion, comprehensive null handling

### 12. Complex Call Disposition Categorization
```dax
BI_Call_Dispo_CAT = 
SWITCH(
    TRUE(),
    CONTAINSSTRING(LOWER('Call_Data_SF'[DISPOSITION]), "contacted"), "Contacted",
    'Call_Data_SF'[DISPOSITION] = "No Answer", "No Answer",
    'Call_Data_SF'[DISPOSITION] = "No Answer.", "No Answer",
    'Call_Data_SF'[DISPOSITION] = "Busy", "Busy",
    'Call_Data_SF'[DISPOSITION] = "Busy.", "Busy",
    'Call_Data_SF'[DISPOSITION] = "Disconnected", "Disco",
    'Call_Data_SF'[DISPOSITION] = "Abandon", "System",
    'Call_Data_SF'[DISPOSITION] = "Agent Error", "System",
    'Call_Data_SF'[DISPOSITION] = "Caller Disconnected", "System",
    'Call_Data_SF'[DISPOSITION] = "Declined", "System",
    'Call_Data_SF'[DISPOSITION] = "Dial Error", "System",
    'Call_Data_SF'[DISPOSITION] = "Helpdesk Email", "System",
    'Call_Data_SF'[DISPOSITION] = "Internal Call", "System",
    'Call_Data_SF'[DISPOSITION] = "No Disposition", "System",
    'Call_Data_SF'[DISPOSITION] = "NULL", "System",
    'Call_Data_SF'[DISPOSITION] = "Operator Intercept", "System",
    'Call_Data_SF'[DISPOSITION] = "Queue Callback Timeout", "System",
    'Call_Data_SF'[DISPOSITION] = "Resource Unavailable", "System",
    'Call_Data_SF'[DISPOSITION] = "Sent To Voicemail", "System",
    'Call_Data_SF'[DISPOSITION] = "Station Session", "System",
    'Call_Data_SF'[DISPOSITION] = "System Error", "System",
    'Call_Data_SF'[DISPOSITION] = "System Shutdown", "System",
    'Call_Data_SF'[DISPOSITION] = "Voicemail Dump", "System",
    'Call_Data_SF'[DISPOSITION] = "Voicemail Processed", "System",
    'Call_Data_SF'[DISPOSITION] = "Voicemail Returned", "System",
    'Call_Data_SF'[DISPOSITION]
)
```
**Advanced Techniques:** Extensive SWITCH logic, CONTAINSSTRING for pattern matching, comprehensive categorization

### 13. Advanced Dialer Program Classification
```dax
BI_Dial_Program = 
VAR IsAgentCampaign = SEARCH("Agent", [CAMPAIGN], 1, 0) > 0
RETURN SWITCH(
    TRUE(),
    -- Research category
    ([Call Type] = "Outbound" && [CAMPAIGN] = "Research_Dialer_Unassigned_Power") 
    || ([Call Type] = "Preview" && [CAMPAIGN] = "Research_Dialer_Preview"),
    "Research",
    
    -- Non-AE category
    [Call Type] = "Outbound" && [CAMPAIGN] = "Non AE Assigned",
    "Non-AE Owned",
    
    -- Preview category
    [Call Type] = "Preview" 
    && NOT(IsAgentCampaign)
    && [CAMPAIGN] <> "TAM Test",
    "Preview",
    
    -- AE Owned Preview category
    [Call Type] = "Preview" && IsAgentCampaign,
    "AE Owned Preview",
    
    -- List Mining category
    [Call Type] = "Manual" && IsAgentCampaign,
    "List Mining",
    
    -- Default case
    "Other"
)
```
**Advanced Techniques:** VAR with SEARCH function, complex boolean logic, multi-condition classification
