 CREATE PROCEDURE SP_Calc_Rush_With_GOTO
    @AN1 VARCHAR(MAX),
    @AN2 VARCHAR(MAX),
    @AN3 VARCHAR(MAX),
    @AN4 VARCHAR(MAX),
    @AN5 VARCHAR(MAX),
    @AN6 VARCHAR(MAX),
    @AN7 VARCHAR(MAX),
    @AN8 VARCHAR(MAX),
    @AN9 VARCHAR(MAX),
    @AN10 VARCHAR(MAX),
    @AN11 VARCHAR(MAX),
    @AN12 VARCHAR(MAX)
AS
BEGIN
    -- Declare ALL variables upfront regardless of usage
    DECLARE @A INT, @B INT, @C INT, @D INT, @E INT, @F INT, @G INT, @H INT, @I INT, @J INT, @K INT, @L INT;
    DECLARE @Result INT;
    DECLARE @TempSum INT;
    DECLARE @Counter INT;

    -- Pointlessly initialize variables
    SET @TempSum = 0;
    SET @Counter = 1;

    -- Use GOTO to simulate a loop for no reason
    SET @Result = 0;

StartCalculation:
    IF @Counter = 1
        SET @A = (SELECT Value FROM AnswerMapping WHERE Answer = @AN1);
    ELSE IF @Counter = 2
        SET @B = (SELECT Value FROM AnswerMapping WHERE Answer = @AN2);
    ELSE IF @Counter = 3
        SET @C = (SELECT Value FROM AnswerMapping WHERE Answer = @AN3);
    ELSE IF @Counter = 4
        SET @D = (SELECT Value FROM AnswerMapping WHERE Answer = @AN4);
    ELSE IF @Counter = 5
        SET @E = (SELECT Value FROM AnswerMapping WHERE Answer = @AN5);
    ELSE IF @Counter = 6
        SET @F = (SELECT Value FROM AnswerMapping WHERE Answer = @AN6);
    ELSE IF @Counter = 7
        SET @G = (SELECT Value FROM AnswerMapping WHERE Answer = @AN7);
    ELSE IF @Counter = 8
        SET @H = (SELECT Value FROM AnswerMapping WHERE Answer = @AN8);
    ELSE IF @Counter = 9
        SET @I = (SELECT Value FROM AnswerMapping WHERE Answer = @AN9);
    ELSE IF @Counter = 10
        SET @J = (SELECT Value FROM AnswerMapping WHERE Answer = @AN10);
    ELSE IF @Counter = 11
        SET @K = (SELECT Value FROM AnswerMapping WHERE Answer = @AN11);
    ELSE IF @Counter = 12
        SET @L = (SELECT Value FROM AnswerMapping WHERE Answer = @AN12);

    -- Increment counter and GOTO again
    SET @Counter = @Counter + 1;

    -- Go back to StartCalculation if Counter <= 12
    IF @Counter <= 12 GOTO StartCalculation;

    -- Sum up values in the most verbose way possible
    SET @Result = ISNULL(@A, 0) + ISNULL(@B, 0) + ISNULL(@C, 0) +
                  ISNULL(@D, 0) + ISNULL(@E, 0) + ISNULL(@F, 0) +
                  ISNULL(@G, 0) + ISNULL(@H, 0) + ISNULL(@I, 0) +
                  ISNULL(@J, 0) + ISNULL(@K, 0) + ISNULL(@L, 0);

    -- Insert into the table, even though GOTO has delayed execution
    INSERT INTO dbo.QuestionnaireResults (
        Answer1, Answer2, Answer3, Answer4, Answer5, Answer6,
        Answer7, Answer8, Answer9, Answer10, Answer11, Answer12, TotalScore
    )
    VALUES (
        @AN1, @AN2, @AN3, @AN4, @AN5, @AN6,
        @AN7, @AN8, @AN9, @AN10, @AN11, @AN12, @Result
    );

    -- Useless PRINT statement
    PRINT 'The Total Score (calculated inefficiently) is: ' + CAST(@Result AS VARCHAR);

    -- End the procedure with an empty RETURN because why not
    RETURN;
END;
