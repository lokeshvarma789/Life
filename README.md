
Get-ChildItem -Recurse -File -Filter "*.fex" |
    Select-String -SimpleMatch -Pattern "PLAN_ID", "CVG_PLAN_ID" |
    Select-Object Path, LineNumber, Line |
    Format-List
