# Contact Center Analytics - Data Model Architecture

## 🏗️ Data Model Overview

The Contact Center Analytics platform utilizes a **star schema design** optimized for performance with Power BI, integrating data from Five9 call center, Salesforce CRM, and SharePoint employee hierarchy systems.

## 📊 Complete Data Model Architecture

```
                    ┌─────────────┐
                    │   DimDate   │
                    │             │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│    Task     │    │Call_Data_SF │    │  RTL_Deal   │
│             │    │             │    │             │
└─────┬───────┘    └─────┬───────┘    └─────┬───────┘
      │                  │                  │
      │                  │                  │
      ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Seller    │    │    User     │    │ RTL_Quote   │
│             │    │             │    │             │
└─────────────┘    └─────────────┘    └─────────────┘
                           │
                           │
                           ▼
                   ┌─────────────┐    ┌─────────────┐
                   │ dimEmpHier  │    │  RTL_BUD    │
                   │(Employee    │    │             │
                   │ Hierarchy)  │    └─────────────┘
                   └─────────────┘
```

## 📋 Complete Table Overview

### Multiple Fact Tables (Parallel Architecture)
- **Task** - Task management and call activities
- **Call_Data_SF** - Salesforce call data integration  
- **RTL_Deal** - Deal pipeline tracking and revenue analytics

### Dimension Tables
- **DimDate** - Comprehensive date dimension for time-based analysis
- **Seller** - Seller/account master data with channel information
- **User** - User/employee basic information
- **dimEmpHier** - Employee hierarchy with Manager-to-AE relationships

### Supporting Tables
- **RTL_Quote** - Quote and pricing information
- **RTL_BUD** - Budget and financial planning data

## 🔗 Complete Relationships & Hierarchy

### Multi-Fact Architecture with Employee Hierarchy

```
DimDate (1) ──┬─── (*) Task
              ├─── (*) Call_Data_SF  
              └─── (*) RTL_Deal
                          │
Seller (1) ─────── (*) Task
                          │
User (1) ───────── (*) Call_Data_SF
                          │
User (1) ───────── (*) dimEmpHier (Manager-to-AE relationships)
                          │
RTL_Deal (1) ────── (*) RTL_Quote
                          │
dimEmpHier ─────── (*) RTL_BUD (Budget planning by hierarchy)
```

### Employee Hierarchy Structure

**dimEmpHier enables Manager-to-Account Executive matrix reporting:**
- **AE_ID** - Account Executive identifier
- **BL_SE_ID** - Sales Executive/Manager identifier (hierarchy relationship)
- **Hire Date** - Employee tenure calculation
- **Termination Date** - Active employee filtering
- **BL_Expectation_BUD_LKP** - Budget expectation lookup integration

### Architecture Characteristics

**Manager-to-AE Matrix:**
- dimEmpHier provides hierarchical relationships for matrix reporting
- Enables drill-down from Manager level to individual AE performance
- Supports tenure calculations for performance context

**Multi-Domain Integration:**
- Parallel fact table architecture across business domains
- Employee hierarchy cuts across all fact tables
- Budget integration through hierarchy relationships
- Time intelligence consistent across all domains

## 🔗 Relationships & Cardinality

### Primary Relationships

```
DIM_Employee (1) ────── (*) FACT_Tasks
    │                         │
    │                         │
DIM_Account  (1) ────── (*) │
    │                         │
    │                         │
DIM_Calendar (1) ────── (*) │
                              │
DIM_Employee (1) ────── (*) FACT_Deals
    │
    │
DIM_Account  (1) ────── (*) FACT_Deals
```

### Relationship Details

**Employee Relationships:**
- `DIM_Employee[EmployeeID] → FACT_Tasks[EmployeeID]` (One-to-Many)
- `DIM_Employee[EmployeeID] → DIM_Account[OwnerID]` (One-to-Many)
- `DIM_Employee[EmployeeID] → FACT_Deals[EmployeeID]` (One-to-Many)

**Account Relationships:**
- `DIM_Account[AccountID] → FACT_Tasks[AccountID]` (One-to-Many)
- `DIM_Account[AccountID] → FACT_Deals[AccountID]` (One-to-Many)

**Time Intelligence:**
- `DIM_Calendar[DateKey] → FACT_Tasks[ActivityDate]` (One-to-Many)
- `DIM_Calendar[DateKey] → FACT_Deals[SubmissionDate]` (One-to-Many)

## 📊 Data Sources Integration

### Five9 Call Center Integration
**Connection Type:** Direct API integration with scheduled refresh
**Data Volume:** ~50,000 daily call records
**Refresh Schedule:** Every 15 minutes during business hours

**Key Data Points:**
- Call disposition and outcome data
- Agent performance metrics
- Queue and routing information
- Call duration and timing
- ANI/DNIS phone number data

### Salesforce CRM Integration
**Connection Type:** Salesforce connector with incremental refresh
**Data Volume:** 400,000+ accounts, 1M+ activities
**Refresh Schedule:** Hourly incremental updates

**Key Data Points:**
- Account and contact information
- Text messaging activity
- Deal pipeline and revenue data
- Account ownership assignments
- Custom object relationships

### SharePoint Employee Data
**Connection Type:** SharePoint list connector
**Data Volume:** ~500 employee records
**Refresh Schedule:** Daily refresh

**Key Data Points:**
- Employee hierarchy structure
- Manager-to-AE reporting relationships
- Tenure and role information
- Department assignments

## 🎯 Performance Optimizations

### Indexing Strategy

**Fact Table Optimizations:**
```sql
-- Primary clustered indexes on fact tables
FACT_Tasks: Clustered Index on (ActivityDate, EmployeeID)
FACT_Deals: Clustered Index on (SubmissionDate, EmployeeID)

-- Non-clustered indexes for common filters
FACT_Tasks: Index on (CallDisposition, IsContact)
FACT_Tasks: Index on (Subject, IsOwnedAccount)
```

**Dimension Optimizations:**
```sql
-- Ensure proper primary key indexing
DIM_Employee: Primary Key on EmployeeID
DIM_Account: Primary Key on AccountID  
DIM_Calendar: Primary Key on DateKey
```

### Data Model Best Practices

**Column-Level Optimizations:**
- Mark non-essential columns as "Don't Summarize"
- Set proper data types (Integer for IDs, Boolean for flags)
- Hide technical columns from report view
- Create calculated columns only when necessary

**Relationship Optimizations:**
- Single-direction filtering where possible
- Bi-directional filtering only for many-to-many scenarios
- Proper cardinality settings for performance

**Memory Optimization:**
- Remove unused columns from fact tables
- Use surrogate keys for dimension relationships
- Implement proper data compression

## 🔧 Data Quality & Governance

### Data Validation Rules

**Five9 Data Quality:**
- Validate phone number formats (10-digit US numbers)
- Ensure call duration > 0 for completed calls
- Validate disposition values against approved list

**Salesforce Data Quality:**
- Account ID format validation
- Employee ID existence verification
- Required field completeness checks

**Cross-System Validation:**
- Employee ID consistency across systems
- Account ID matching between Salesforce and Five9
- Date range validation for activity records

### Error Handling & Monitoring

**Data Load Monitoring:**
- Automated alerts for failed refreshes
- Row count validation after each load
- Data freshness monitoring dashboards
- Performance metric tracking

**Data Quality Dashboards:**
- Missing data identification
- Duplicate record detection
- Cross-system consistency reporting
- Data lineage documentation

This data model architecture provides the foundation for the sophisticated analytics and 100+ DAX measures that drive the Contact Center Analytics platform's business value and operational insights.
