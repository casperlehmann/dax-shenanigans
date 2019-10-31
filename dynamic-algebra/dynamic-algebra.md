> How to dynamically parse and evaluate algebraic formulars in DAX

From an answer I gave on [Reddit](https://www.reddit.com/r/PowerBI/comments/avfu4e/help_with_custom_totals/)

Say you have a Data table like this:

Table:

```Data = DATATABLE("Group"; STRING; "Value"; INTEGER; {{"A"; 1}; {"B"; 2}; {"C"; 3}; {"D"; 4}})```

And a series of calculations you want, represented by a Calculations table like this:

Table:

```Calcs = DATATABLE("Calculation"; STRING; {{"A"}; {"B"}; {"B/A"}; {"C*D^2"}})```

Meaure:

```Dynamic Algebra = 
VAR ABC = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
VAR input = FIRSTNONBLANK(Calcs[Calculation]; TRUE())
VAR CALC = IF(HASONEVALUE(Calcs[Calculation]); SUBSTITUTE(input; "^"; "*"); "")
VAR SUB_SPACE = SUBSTITUTE(CALC; " "; "")
VAR SUB_PLUS = SUBSTITUTE(SUB_SPACE; "+"; "|")
VAR SPLIT_PLUS = 
ADDCOLUMNS(
    SELECTCOLUMNS(GENERATESERIES(1; PATHLENGTH(SUB_PLUS)); "Iter plus"; [Value]);
    "Path Split Plus"; PATHITEM(SUB_PLUS; [Iter plus]; 0);
    "Result";
    VAR SUB_MINUS = SUBSTITUTE(PATHITEM(SUB_PLUS; [Iter plus]; 0); "-"; "|")
    VAR MINUS_TABLE = ADDCOLUMNS(
        SELECTCOLUMNS(GENERATESERIES(1; PATHLENGTH(SUB_MINUS)); "Iter minus"; [Value]);
        "Path Split Minus"; PATHITEM(SUB_MINUS; [Iter minus]; 0);
        "Result";
        VAR SUB_MULT = SUBSTITUTE(PATHITEM(SUB_MINUS; [Iter minus]; 0); "*"; "|")
        VAR MULT_TABLE = ADDCOLUMNS(
            SELECTCOLUMNS(GENERATESERIES(1; PATHLENGTH(SUB_MULT)); "Iter mult"; [Value]);
            "Result";
            VAR SUB_DIV = SUBSTITUTE(PATHITEM(SUB_MULT; [Iter mult]; 0); "/"; "|")
            VAR DIV_TABLE = ADDCOLUMNS(
                SELECTCOLUMNS(GENERATESERIES(1; PATHLENGTH(SUB_DIV)); "Iter div"; [Value]);
                "Result";
                VAR RAW_VALUE = PATHITEM(SUB_DIV; [Iter div]; 0)
                VAR LOOKUP_VALUE = IFERROR(SEARCH(RAW_VALUE; ABC); BLANK())
                RETURN IF(LOOKUP_VALUE; CALCULATE(SUM(Data[Value]); Data[Group] = RAW_VALUE); VALUE(RAW_VALUE))
            )
            // We don't have access to any function called DIVIDEX,
            // so we go the poor man's route and divide by the inverse.
            // As we cannot slice lists in DAX, instead we square the
            // first value.
            VAR RAW_VALUE = PATHITEM(SUB_DIV; 1; 0)
            VAR LOOKUP_VALUE = IFERROR(SEARCH(RAW_VALUE; ABC); BLANK())
            VAR FIRST_VALUE = IF(LOOKUP_VALUE; CALCULATE(SUM(Data[Value]); Data[Group] = RAW_VALUE); VALUE(RAW_VALUE))
            VAR FIRST_VALUE_SQUARED = FIRST_VALUE^2
            RETURN FIRST_VALUE_SQUARED*PRODUCTX(SELECTCOLUMNS(DIV_TABLE; "Result"; [Result]); 1/[Result])
        )
        RETURN PRODUCTX(MULT_TABLE; [Result])
    )
    RETURN SUMX(MINUS_TABLE; IF([Iter minus]=1; 1; -1) * [Result])
)
VAR RESULT = IF(
    FIND("^"; input; 1; -1)=-1;
    SUMX(SPLIT_PLUS; [Result]);
    "Not supported"
)
RETURN RESULT
```
