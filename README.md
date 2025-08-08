Sub ExportWordFormattingToJSON()
    Dim doc As Document
    Set doc = ActiveDocument
    
    Dim js As String
    Dim para As Paragraph
    Dim foundTitle As Boolean, foundSection As Boolean, foundSubsection As Boolean, foundPoint As Boolean
    
    ' 1. Margins
    js = "{""margins"":{""top"":" & doc.PageSetup.TopMargin / 72 _
       & ",""bottom"":" & doc.PageSetup.BottomMargin / 72 _
       & ",""left"":" & doc.PageSetup.LeftMargin / 72 _
       & ",""right"":" & doc.PageSetup.RightMargin / 72 & "},"
    
    ' 2. Loop through paragraphs and extract formatting by style
    For Each para In doc.Paragraphs
        Select Case LCase(para.Style)
            Case "title"
                If Not foundTitle Then
                    js = js & """title"":" & FormatParaAsJSON(para) & ","
                    foundTitle = True
                End If
            Case "heading 1"
                If Not foundSection Then
                    js = js & """section"":" & FormatParaAsJSON(para) & ","
                    foundSection = True
                End If
            Case "heading 2"
                If Not foundSubsection Then
                    js = js & """subsection"":" & FormatParaAsJSON(para) & ","
                    foundSubsection = True
                End If
            Case "normal"
                If Not foundPoint Then
                    js = js & """point"":" & FormatParaAsJSON(para) & ","
                    foundPoint = True
                End If
        End Select
    Next para
    
    ' Remove trailing comma and close JSON
    If Right(js, 1) = "," Then js = Left(js, Len(js) - 1)
    js = js & "}"
    
    ' Output
    Debug.Print js
    MsgBox "JSON formatting exported! See Immediate window (Ctrl+G)."
End Sub

Function FormatParaAsJSON(para As Paragraph) As String
    Dim f As Font: Set f = para.Range.Font
    Dim col As Long: col = f.Color
    Dim r As Long, g As Long, b As Long
    RGBToComponents col, r, g, b
    FormatParaAsJSON = "{""font"":""" & f.Name & """,""size"":" & f.Size & ",""bold"":" & LCase(f.Bold = True) _
        & ",""alignment"":" & para.Alignment _
        & ",""color"":{""r"":" & r & ",""g"":" & g & ",""b"":" & b & "},""indent"":" & para.LeftIndent / 28.35 & _
        ",""space_before"":" & para.SpaceBefore & ",""space_after"":" & para.SpaceAfter & _
        ",""underline"":" & LCase(f.Underline <> wdUnderlineNone) & "}"
End Function

Sub RGBToComponents(col As Long, ByRef r As Long, ByRef g As Long, ByRef b As Long)
    r = col Mod 256
    g = (col \ 256) Mod 256
    b = (col \ 65536) Mod 256
End Sub
