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
    )

RETURN Result

