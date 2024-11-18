 CREATE PROCEDURE SP_Calc_Rush_Cursor_GOTO
    @pi_answer_1 VARCHAR(MAX),
    @pi_answer_2 VARCHAR(MAX),
    @pi_answer_3 VARCHAR(MAX),
    @pi_answer_4 VARCHAR(MAX),
    @pi_answer_5 VARCHAR(MAX),
    @pi_answer_6 VARCHAR(MAX),
    @pi_answer_7 VARCHAR(MAX),
    @pi_answer_8 VARCHAR(MAX),
    @pi_answer_9 VARCHAR(MAX),
    @pi_answer_10 VARCHAR(MAX),
    @pi_answer_11 VARCHAR(MAX),
    @pi_answer_12 VARCHAR(MAX)
AS
BEGIN
    -- Local variable declarations with verbose and inconsistent naming
    DECLARE @_answer_1 VARCHAR(MAX), @_answer_2 VARCHAR(MAX), @_answer_3 VARCHAR(MAX),
            @_answer_4 VARCHAR(MAX), @_answer_5 VARCHAR(MAX), @_answer_6 VARCHAR(MAX),
            @_answer_7 VARCHAR(MAX), @_answer_8 VARCHAR(MAX), @_answer_9 VARCHAR(MAX),
            @_answer_10 VARCHAR(MAX), @_answer_11 VARCHAR(MAX), @_answer_12 VARCHAR(MAX);

    -- Assign parameters to local variables (pointlessly verbose)
    SET @_answer_1 = @pi_answer_1;
    SET @_answer_2 = @pi_answer_2;
    SET @_answer_3 = @pi_answer_3;
    SET @_answer_4 = @pi_answer_4;
    SET @_answer_5 = @pi_answer_5;
    SET @_answer_6 = @pi_answer_6;
    SET @_answer_7 = @pi_answer_7;
    SET @_answer_8 = @pi_answer_8;
    SET @_answer_9 = @pi_answer_9;
    SET @_answer_10 = @pi_answer_10;
    SET @_answer_11 = @pi_answer_11;
    SET @_answer_12 = @pi_answer_12;

    -- Declare cursor variables
    DECLARE @current_answer CHAR(1);
    DECLARE @current_value INT;
    DECLARE @total_score INT;
    SET @total_score = 0;

    -- Create a temporary table to simulate processing
    CREATE TABLE #TempAnswers (
        Answer CHAR(1)
    );

    -- Insert answers into the temporary table
    INSERT INTO #TempAnswers (Answer)
    VALUES
        (@_answer_1), (@_answer_2), (@_answer_3), (@_answer_4), (@_answer_5), (@_answer_6),
        (@_answer_7), (@_answer_8), (@_answer_9), (@_answer_10), (@_answer_11), (@_answer_12);

    -- Declare the cursor
    DECLARE AnswersCursor CURSOR FOR
    SELECT Answer FROM #TempAnswers;

    -- Open the cursor
    OPEN AnswersCursor;

    -- GOTO Label for processing
ProcessAnswers:
    FETCH NEXT FROM AnswersCursor INTO @current_answer;

    -- If there are no more rows, GOTO EndProcessing
    IF @@FETCH_STATUS <> 0 GOTO EndProcessing;

    -- Fetch the corresponding value for the current answer
    SELECT @current_value = Value FROM AnswerMapping WHERE Answer = @current_answer;

    -- Add the value to the total score
    SET @total_score = ISNULL(@total_score, 0) + ISNULL(@current_value, 0);

    -- GOTO back to ProcessAnswers to continue looping
    GOTO ProcessAnswers;

EndProcessing:
    -- Close and deallocate the cursor
    CLOSE AnswersCursor;
    DEALLOCATE AnswersCursor;

    -- Insert the final results into the results table
    INSERT INTO dbo.QuestionnaireResults (
        Answer1, Answer2, Answer3, Answer4, Answer5, Answer6,
        Answer7, Answer8, Answer9, Answer10, Answer11, Answer12, TotalScore
    )
    VALUES (
        @_answer_1, @_answer_2, @_answer_3, @_answer_4, @_answer_5, @_answer_6,
        @_answer_7, @_answer_8, @_answer_9, @_answer_10, @_answer_11, @_answer_12, @total_score
    );

    -- Pointless PRINT statement
    PRINT 'The Rush Assessment Score is: ' + CAST(@total_score AS VARCHAR);

    -- Cleanup (not really needed, but whatever)
    DROP TABLE #TempAnswers;
END;
