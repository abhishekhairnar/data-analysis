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
