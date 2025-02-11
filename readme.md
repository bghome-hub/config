Test Case ID,Marital Status,Allowances,Additional Withholding,Income Bracket,Exception Year,Expected Outcome
TC-001,Single,0,0,$0 - $1,500,0,Flat tax rate correctly applied
TC-002,Married,0,0,$1,501 - $3,000,>0,Exception year applies correctly
TC-003,Null,0,0,$3,000+,Null,System handles missing marital status
TC-004,NA,Non-Zero,0,$0 - $1,500,0,System handles NA marital status
TC-005,Single,Null,Non-Zero,$1,501 - $3,000,>0,Additional withholding is correctly added
TC-006,Married,0,Null,$3,000+,0,System handles null additional withholding
TC-007,Single,Non-Zero,0,$0 - $1,500,Null,Allowances correctly reduce taxable income
TC-008,Married,0,Non-Zero,$1,501 - $3,000,0,Flat tax + additional withholding applied
TC-009,Null,Null,Null,$3,000+,>0,System correctly handles all null inputs
TC-010,NA,0,0,$0 - $1,500,0,System handles NA and applies tax rate
