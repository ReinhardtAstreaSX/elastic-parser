Sub RefangIndicators()
    Dim cell As Range
    Dim selectedRange As Range
    Dim text As String
    
    ' Work only on the selected cells to save memory
    Set selectedRange = Selection
    
    ' Optimize Excel performance during execution
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual
    
    For Each cell In selectedRange
        If Not cell.HasFormula And cell.Value <> "" Then
            text = cell.Value
            
            ' Remove bracketed and parenthesized dots
            text = Replace(text, "[.]", ".")
            text = Replace(text, "(.)", ".")
            text = Replace(text, "{.}", ".")
            
            ' Remove bracketed and parenthesized colons
            text = Replace(text, "[:]", ":")
            text = Replace(text, "(:)", ":")
            
            ' Fix protocol defanging
            text = Replace(text, "hxxps://", "https://")
            text = Replace(text, "hxxp://", "http://")
            text = Replace(text, "fxps://", "https://")
            text = Replace(text, "fxp://", "http://")
            
            ' Update cell value
            cell.Value = text
        End If
    Next cell
    
    ' Restore Excel settings
    Application.ScreenUpdating = True
    Application.Calculation = xlCalculationAutomatic
    
    MsgBox "Indicators successfully refanged!", vbInformation, "Done"
End Sub