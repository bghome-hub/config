3.1 Functional Test Matrix
We will test the combinations of the following variables:

Marital Status: Married, Single, Null, NA
Allowances: Zero, Non-Zero, Null
Additional Withholding: Zero, Non-Zero, Null
Income Brackets:
$0 - $1,500/month
$1,501 - $3,000/month
$3,000+/month
Exception Year: Greater than zero vs. null or zero
Test Case ID	Marital Status	Allowances	Additional Withholding	Income Bracket	Exception Year	Expected Outcome
TC-001	Single	0	0	$0 - $1,500	0	Flat tax rate correctly applied
TC-002	Married	0	0	$1,501 - $3,000	>0	Exception year applies correctly
TC-003	Null	0	0	$3,000+	Null	System handles missing marital status
TC-004	NA	Non-Zero	0	$0 - $1,500	0	System handles NA marital status
TC-005	Single	Null	Non-Zero	$1,501 - $3,000	>0	Additional withholding is correctly added
TC-006	Married	0	Null	$3,000+	0	System handles null additional withholding
TC-007	Single	Non-Zero	0	$0 - $1,500	Null	Allowances correctly reduce taxable income
TC-008	Married	0	Non-Zero	$1,501 - $3,000	0	Flat tax + additional withholding applied
TC-009	Null	Null	Null	$3,000+	>0	System correctly handles all null inputs
TC-010	NA	0	0	$0 - $1,500	0	System handles NA and applies tax rate
3.2 Regression Test Cases
Test Case ID	Regression Area	Expected Outcome
TC-101	Payroll calculations	No unintended changes in net pay calculations
TC-102	Year-end reporting (W-2, 1099)	Reports reflect correct tax amounts
TC-103	Multi-state employee taxation	Georgia employees are taxed separately from other states
TC-104	Historical tax records	No retroactive changes to prior payroll data



