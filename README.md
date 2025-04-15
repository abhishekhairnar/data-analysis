Conditional_KPI_Sum_DR :=
VAR SelectedDetail = SELECTEDVALUE('jv condition'[Details])

-- Get all rows for the selected KPI
VAR FilterRows =
    FILTER(
        'jv condition',
        'jv condition'[Details] = SelectedDetail
    )

VAR Result =
    SUMX(
        FilterRows,
        VAR StatusCond = 'jv condition'[Status]
        VAR TypeCond = 'jv condition'[Type]
        VAR CycleCond = 'jv condition'[Transaction Cycle]
        VAR InOwCond = 'jv condition'[In/Ow]
        VAR AddColumnName = 'jv condition'[add dr]

        -- Extract operator and value from Transaction Cycle
        VAR CycleOperator =
            IF(
                ISBLANK(CycleCond), BLANK(),
                IF(
                    FIND("!=", CycleCond, 1, 0) > 0, "!=",
                    IF(FIND("==", CycleCond, 1, 0) > 0, "==", BLANK())
                )
            )

        VAR CycleValue =
            IF(
                CycleOperator = "!=",
                TRIM(SUBSTITUTE(CycleCond, "!=", "")),
                IF(
                    CycleOperator = "==",
                    TRIM(SUBSTITUTE(CycleCond, "==", "")),
                    BLANK()
                )
            )

        -- Create the filter expression
        VAR DataFilter =
            FILTER(
                raw_data,
                (ISBLANK(StatusCond) || raw_data[Status] = StatusCond) &&
                (ISBLANK(TypeCond) || raw_data[Type] = TypeCond) &&
                (ISBLANK(InOwCond) || raw_data[In/Ow] = InOwCond) &&
                (
                    ISBLANK(CycleOperator) ||
                    (
                        CycleOperator = "==" && raw_data[Transaction Cycle] = CycleValue ||
                        CycleOperator = "!=" && raw_data[Transaction Cycle] <> CycleValue
                    )
                )
            )

        -- Dynamic measure: switch based on column name in 'add dr'
        VAR ValueToSum =
            SWITCH(
                TRUE(),
                AddColumnName = "setamdr", CALCULATE(SUM(raw_data[setamdr]), DataFilter),
                AddColumnName = "Int fee amt cr", CALCULATE(SUM(raw_data[Int fee amt cr]), DataFilter),
                AddColumnName = "oth fee gst dr", CALCULATE(SUM(raw_data[oth fee gst dr]), DataFilter),
                0
            )

        RETURN ValueToSum
    )

RETURN Result





KPI Conditional Sum DR :=
VAR SelectedKPI = SELECTEDVALUE('JV Condition'[Details])

VAR RowsForKPI =
    FILTER(
        'JV Condition',
        'JV Condition'[Details] = SelectedKPI
    )

VAR Result =
    SUMX(
        RowsForKPI,
        VAR StatusCond = 'JV Condition'[Status]
        VAR TypeCond = 'JV Condition'[Transaction type]
        VAR CycleCond = 'JV Condition'[Transaction Cycle]
        VAR InOwCond = 'JV Condition'[In/Ow]
        VAR ColToSum = 'JV Condition'[add dr]

        VAR HasStatus = NOT ISBLANK(StatusCond)
        VAR HasType = NOT ISBLANK(TypeCond)
        VAR HasCycle = NOT ISBLANK(CycleCond)
        VAR HasInOw = NOT ISBLANK(InOwCond)

        -- Extract operator and value from cycle
        VAR CycleOperator =
            IF(
                FIND("!=", CycleCond, 1, 0) > 0, "!=",
                IF(FIND("==", CycleCond, 1, 0) > 0, "==", BLANK())
            )

        VAR CycleValue =
            TRIM(
                SUBSTITUTE(
                    SUBSTITUTE(CycleCond, "!=", ""),
                    "==", ""
                )
            )

        -- Filtered raw data
        VAR FilteredData =
            FILTER(
                'DSR Report DOM',
                (NOT HasStatus || 'DSR Report DOM'[Status(Approved/Declined)] = StatusCond) &&
                (NOT HasType   || 'DSR Report DOM'[Transaction type] = TypeCond) &&
                (NOT HasInOw   || 'DSR Report DOM'[In/Ow] = InOwCond) &&
                (
                    NOT HasCycle ||
                    (
                        CycleOperator = "!=" && 'DSR Report DOM'[Transaction Cycle] <> CycleValue ||
                        CycleOperator = "==" && 'DSR Report DOM'[Transaction Cycle] = CycleValue
                    )
                )
            )

        -- Sum the correct column
        VAR SumValue =
            SWITCH(
                ColToSum,
                "setamdr", CALCULATE(SUM('DSR Report DOM'[setamdr]), FilteredData),
                "Int fee amt cr", CALCULATE(SUM('DSR Report DOM'[Int fee amt cr]), FilteredData),
                "oth fee gst dr", CALCULATE(SUM('DSR Report DOM'[oth fee gst dr]), FilteredData),
                0
            )

        RETURN SumValue







let
    Source = SharePoint.Contents("https://fiservcorp.sharepoint.com/sites/APACOPSPMO/", [ApiVersion = 15]),
    #"Shared Documents" = Source{[Name="Shared Documents"]}[Content],
    General = #"Shared Documents"{[Name="General"]}[Content],
    #"Monthly APAC Service council" = General{[Name="Monthly APAC Service council"]}[Content],
    #"Filtered Rows" = Table.SelectRows(#"Monthly APAC Service council", each ([Extension] = ".xlsx") and ([Name] = "Monthly APAC Deck_M-O-M trend- 2025.xlsx")),
    #"Filtered Hidden Files1" = Table.SelectRows(#"Filtered Rows", each [Attributes]?[Hidden]? <> true),
    #"Invoke Custom Function1" = Table.AddColumn(#"Filtered Hidden Files1", "Transform File (2)", each #"Transform File (2)"([Content])),
    #"Renamed Columns1" = Table.RenameColumns(#"Invoke Custom Function1", {"Name", "Source.Name"}),
    #"Removed Other Columns1" = Table.SelectColumns(#"Renamed Columns1", {"Source.Name", "Transform File (2)"}),
    #"Expanded Table Column1" = Table.ExpandTableColumn(#"Removed Other Columns1", "Transform File (2)", Table.ColumnNames(#"Transform File (2)"(#"Sample File (2)"))),
    #"Changed Type" = Table.TransformColumnTypes(#"Expanded Table Column1",{{"Source.Name", type text}, {"Name", type text}, {"Data", type any}, {"Item", type text}, {"Kind", type text}, {"Hidden", type logical}}),
    Data = #"Changed Type"{0}[Data],
    #"Filtered Rows1" = Table.SelectRows(Data, each not (([Column1] = null) and ([Column2] = null))),
    #"Transposed Table" = Table.Transpose(#"Filtered Rows1"),
    #"Filtered Rows2" = Table.SelectRows(#"Transposed Table", each ([Column2] <> null)),
    #"Promoted Headers" = Table.PromoteHeaders(#"Filtered Rows2", [PromoteAllScalars=true]),
    Data1 = #"Changed Type"{1}[Data],
    #"Filtered Rows3" = Table.SelectRows(Data1, each not (([Column1] = null) and ([Column2] = null))),
    #"Transposed Table1" = Table.Transpose(#"Filtered Rows3"),
    #"Filtered Rows4" = Table.SelectRows(#"Transposed Table1", each ([Column2] <> null)),
    #"Promoted Headers2" = Table.PromoteHeaders(#"Filtered Rows4", [PromoteAllScalars=true]),
    #"Appended Query" = Table.Combine({#"Promoted Headers2", #"Promoted Headers"})
in
    #"Appended Query"
    )

RETURN Result








let
    Source = SharePoint.Contents("https://fiservcorp.sharepoint.com/sites/APACOPSPMO/", [ApiVersion = 15]),
    #"Shared Documents" = Source{[Name="Shared Documents"]}[Content],
    General = #"Shared Documents"{[Name="General"]}[Content],
    #"Monthly APAC Service council" = General{[Name="Monthly APAC Service council"]}[Content],
    #"Filtered Rows" = Table.SelectRows(#"Monthly APAC Service council", each ([Extension] = ".xlsx") and ([Name] = "Monthly APAC Deck_M-O-M trend- 2025.xlsx")),
    #"Filtered Hidden Files1" = Table.SelectRows(#"Filtered Rows", each [Attributes]?[Hidden]? <> true),

    // Load the workbook
    Workbook = Excel.Workbook(#"Filtered Hidden Files1"{0}[Content], null, true),

    // Keep only sheet contents
    SheetsOnly = Table.SelectRows(Workbook, each [Kind] = "Sheet"),

    // Apply the custom transformation function to each sheet
    TransformSheet = (sheet as table) as table =>
        let
            FilteredRows = Table.SelectRows(sheet, each not (([Column1] = null) and ([Column2] = null))),
            TransposedTable = Table.Transpose(FilteredRows),
            FilteredRows2 = Table.SelectRows(TransposedTable, each ([Column2] <> null)),
            PromotedHeaders = Table.PromoteHeaders(FilteredRows2, [PromoteAllScalars=true])
        in
            PromotedHeaders,

    TransformedSheets = Table.AddColumn(SheetsOnly, "Transformed Data", each TransformSheet([Data])),

    // Combine all sheets into one table
    CombinedData = Table.Combine(TransformedSheets[Transformed Data])

in
    CombinedData




------------------------------------------------------------------------------------------------

3MonthAvgTotalChargePerEventID =
VAR CurrentMonth = MAX('Invoice'[BILLING CYCLE DATE])
VAR PreviousMonths =
    CALCULATETABLE(
        Invoice,
        DATESINPERIOD(
            'Invoice'[BILLING CYCLE DATE],
            MAX('Invoice'[BILLING CYCLE DATE]),
            -3,
            MONTH
        )
    )
VAR TotalChargeLast3Months =
    CALCULATE(
        SUM(Invoice[TOTAL CHARGE]),
        PreviousMonths
    )
VAR EventIDCountLast3Months =
    CALCULATE(
        COUNT(Invoice[EVENT ID]),
        PreviousMonths
    )
RETURN
    DIVIDE(TotalChargeLast3Months, EventIDCountLast3Months)
 
The following syntax error occurred during parsing: Invalid token, Line 4, Offset 2,  .
