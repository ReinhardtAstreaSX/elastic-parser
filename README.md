Option Explicit

Sub IOCDefang()

    Dim selectedRange As Range
    Dim cell As Range
    Dim text As String

    Dim ipv4Dict As Object
    Dim urlDict As Object

    Dim reIP As Object
    Dim reURL As Object

    Dim fso As Object
    Dim ts As Object

    Dim arr As Variant
    Dim i As Long

    Dim outputFolder As String
    Dim fileDate As String

    ' Store unique IPv4 addresses
    Set ipv4Dict = CreateObject("Scripting.Dictionary")

    ' Store unique URLs and domains
    Set urlDict = CreateObject("Scripting.Dictionary")

    ' Auto-select all non-empty cells in Column A
    With ActiveSheet
        If Application.WorksheetFunction.CountA(.Columns("A")) = 0 Then
            MsgBox "Column A is empty.", vbExclamation, "No Data"
            Exit Sub
        End If
    
        Set selectedRange = .Range("A1", .Cells(.Rows.Count, "A").End(xlUp))
    End With

    ' Match IPv4 with optional /32
    Set reIP = CreateObject("VBScript.RegExp")
    With reIP
        .Global = False
        .IgnoreCase = True
        .Pattern = "^(\d{1,3}\.){3}\d{1,3}(/32)?$"
    End With

    ' Match domains and HTTP/HTTPS URLs
    Set reURL = CreateObject("VBScript.RegExp")
    With reURL
        .Global = False
        .IgnoreCase = True
        .Pattern = "^(https?://)?(([A-Za-z0-9]([A-Za-z0-9-]{0,61}[A-Za-z0-9])?)\.)+[A-Za-z]{2,63}(:\d+)?([/?#].*)?$"
    End With

    ' Improve processing speed
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual

    For Each cell In selectedRange

        If Not cell.HasFormula Then

            text = CStr(cell.Value)

            ' Remove whitespace
            text = Replace(text, " ", "")
            text = Replace(text, vbTab, "")
            text = Replace(text, Chr(160), "")
            text = Trim(text)

            If text <> "" Then

                ' Restore defanged IOC formats
                text = Replace(text, "[.]", ".")
                text = Replace(text, "(.)", ".")
                text = Replace(text, "{.}", ".")

                text = Replace(text, "[:]", ":")
                text = Replace(text, "(:)", ":")

                text = Replace(text, "hxxps://", "https://")
                text = Replace(text, "hxxp://", "http://")
                text = Replace(text, "fxps://", "https://")
                text = Replace(text, "fxp://", "http://")

                ' Normalize IPv4 and add /32
                If reIP.Test(text) Then

                    If InStr(text, "/") = 0 Then
                        text = text & "/32"
                    End If

                    If Not ipv4Dict.Exists(text) Then
                        ipv4Dict.Add text, ""
                    End If

                ' Store unique domains/URLs
                ElseIf reURL.Test(text) Then

                    If Not urlDict.Exists(text) Then
                        urlDict.Add text, ""
                    End If

                End If

            End If

        End If

    Next cell

    Application.ScreenUpdating = True
    Application.Calculation = xlCalculationAutomatic

    ' Stop if no IOCs were found
    If ipv4Dict.Count = 0 And urlDict.Count = 0 Then
        Application.ScreenUpdating = True
        Application.Calculation = xlCalculationAutomatic
    
        MsgBox "No valid IPv4 addresses or URLs/Domains were found in Column A.", _
               vbExclamation, "No Results"
        Exit Sub
    End If

    ' Save output files to Desktop
    outputFolder = CreateObject("WScript.Shell").SpecialFolders("Desktop")
    fileDate = Format(Date, "yyyymmdd")

    Set fso = CreateObject("Scripting.FileSystemObject")

    ' Export sorted IPv4 list
    If ipv4Dict.Count > 0 Then

        arr = ipv4Dict.Keys
        QuickSort arr, LBound(arr), UBound(arr)

        Set ts = fso.CreateTextFile(outputFolder & "\ipv4_" & fileDate & ".txt", True)

        For i = LBound(arr) To UBound(arr)
            ts.WriteLine arr(i)
        Next i

        ts.Close

    End If

    ' Export sorted URL/domain list
    If urlDict.Count > 0 Then
    
        arr = urlDict.Keys
        QuickSort arr, LBound(arr), UBound(arr)
    
        Set ts = fso.CreateTextFile(outputFolder & "\urls_" & fileDate & ".txt", True)
    
        For i = LBound(arr) To UBound(arr)
            ts.WriteLine arr(i)
        Next i
    
        ts.Close
    
    End If
    
    ' Clear processed data only after successful export
    selectedRange.Clear
    
    MsgBox _
        "IOC export completed successfully!" & vbCrLf & vbCrLf & _
        "IPv4: " & ipv4Dict.Count & vbCrLf & _
        "URLs/Domains: " & urlDict.Count & vbCrLf & vbCrLf & _
        "Files saved to Desktop:" & vbCrLf & _
        "ipv4_" & fileDate & ".txt" & vbCrLf & _
        "urls_" & fileDate & ".txt", _
        vbInformation, "IOC Export"

End Sub

Private Sub QuickSort(arr As Variant, first As Long, last As Long)

    Dim low As Long
    Dim high As Long
    Dim mid As Variant
    Dim temp As Variant

    low = first
    high = last
    mid = arr((first + last) \ 2)

    Do While low <= high

        Do While arr(low) < mid
            low = low + 1
        Loop

        Do While arr(high) > mid
            high = high - 1
        Loop

        If low <= high Then

            temp = arr(low)
            arr(low) = arr(high)
            arr(high) = temp

            low = low + 1
            high = high - 1

        End If

    Loop

    If first < high Then QuickSort arr, first, high
    If low < last Then QuickSort arr, low, last

End Sub


