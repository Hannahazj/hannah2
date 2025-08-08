Option Explicit

'== Prereqs: Tools > References > Microsoft Scripting Runtime
'== And import JsonConverter.bas (VBA-JSON)

Private Const PPI As Double = 72#

'--- Default mapping of your JSON buckets to Word styles
Private Const STYLE_TITLE As String = "Title"
Private Const STYLE_SECTION As String = "Heading 1"
Private Const STYLE_SUBSECTION As String = "Heading 2"
Private Const STYLE_POINT As String = "List Paragraph"  ' fallback: "Normal" if you want

'==============================
' PUBLIC ENTRYPOINTS
'==============================

'Export current doc's formatting to JSON (opens a new doc with the JSON and also prints to Immediate)
Public Sub ExportFormatToJson()
    Dim json As String
    json = BuildJsonForDoc()

    Debug.Print json
    Documents.Add.Content.Text = json
End Sub

'Apply JSON to the current doc.
'Usage: select JSON text in Word and run this macro; or leave no selection to paste JSON into an InputBox.
Public Sub ApplyFormatFromSelectedJson()
    Dim src As String
    If Selection.Range.Characters.Count > 0 Then
        src = Selection.Text
    Else
        src = InputBox("Paste JSON to apply:", "Document Format JSON")
        If Len(src) = 0 Then Exit Sub
    End If
    ApplyJsonToDoc src
    MsgBox "Formatting applied.", vbInformation
End Sub

'==============================
' CORE: EXPORT
'==============================

Private Function BuildJsonForDoc() As String
    Dim ps As PageSetup: Set ps = ActiveDocument.PageSetup

    Dim topIn As Double, botIn As Double, leftIn As Double, rightIn As Double
    topIn = PointsToInches(ps.TopMargin)
    botIn = PointsToInches(ps.BottomMargin)
    leftIn = PointsToInches(ps.LeftMargin)
    rightIn = PointsToInches(ps.RightMargin)

    Dim titleObj As String, sectionObj As String, subObj As String, pointObj As String
    titleObj = StyleToJsonObject(STYLE_TITLE, True)              ' include alignment for title
    sectionObj = StyleToJsonObject(STYLE_SECTION, False)
    subObj = StyleToJsonObject(STYLE_SUBSECTION, False, True)    ' include indent for sub/point
    pointObj = StyleToJsonObject(STYLE_POINT, False, True)

    BuildJsonForDoc = _
        "{""margins"":{" & _
            """top"":" & Num(topIn) & "," & _
            """bottom"":" & Num(botIn) & "," & _
            """left"":" & Num(leftIn) & "," & _
            """right"":" & Num(rightIn) & _
        "}," & _
        """title"":" & titleObj & "," & _
        """section"":" & sectionObj & "," & _
        """subsection"":" & subObj & "," & _
        """point"":" & pointObj & _
        "}"
End Function

Private Function StyleToJsonObject(styleName As String, _
                                   Optional includeAlignment As Boolean = False, _
                                   Optional includeIndent As Boolean = False) As String
    Dim sty As Style
    On Error Resume Next
    Set sty = ActiveDocument.Styles(styleName)
    On Error GoTo 0

    If sty Is Nothing Then
        StyleToJsonObject = "{}"
        Exit Function
    End If

    Dim fnt As Font:        Set fnt = sty.Font
    Dim para As ParagraphFormat: Set para = sty.ParagraphFormat

    Dim r As Long, g As Long, b As Long
    RGBSplit SafeFontRGB(fnt), r, g, b

    Dim parts As Collection: Set parts = New Collection
    parts.Add """font"":""" & SafeStr(fnt.Name) & """"
    parts.Add """size"":" & Num(fnt.Size)
    parts.Add """bold"":" & LCase$(CStr(fnt.Bold <> 0))
    parts.Add """color"":{""r"":" & r & ",""g"":" & g & ",""b"":" & b & "}"

    If includeAlignment Then
        parts.Add """alignment"":" & CStr(NormalizeAlignment(para.Alignment))
    End If

    ' spacing in points (matches your sample integers)
    parts.Add """space_before"":" & Num(para.SpaceBefore)
    parts.Add """space_after"":" & Num(para.SpaceAfter)

    ' underline as bool
    parts.Add """underline"":" & LCase$(CStr(fnt.Underline <> wdUnderlineNone))

    If includeIndent Then
        parts.Add """indent"":" & Num(PointsToInches(para.LeftIndent))
    End If

    StyleToJsonObject = "{" & JoinCollection(parts, ",") & "}"
End Function

'==============================
' CORE: IMPORT/APPLY
'==============================

Private Sub ApplyJsonToDoc(ByVal jsonText As String)
    Dim parsed As Scripting.Dictionary
    Set parsed = JsonConverter.ParseJson(jsonText)

    '--- Margins
    Dim m As Scripting.Dictionary: Set m = parsed("margins")
    With ActiveDocument.PageSetup
        .TopMargin = InchesToPoints(CDbl(m("top")))
        .BottomMargin = InchesToPoints(CDbl(m("bottom")))
        .LeftMargin = InchesToPoints(CDbl(m("left")))
        .RightMargin = InchesToPoints(CDbl(m("right")))
    End With

    '--- Styles (update existing Word styles)
    UpdateStyleFromJson STYLE_TITLE, parsed("title")
    UpdateStyleFromJson STYLE_SECTION, parsed("section")
    UpdateStyleFromJson STYLE_SUBSECTION, parsed("subsection")
    UpdateStyleFromJson STYLE_POINT, parsed("point")
End Sub

Private Sub UpdateStyleFromJson(ByVal styleName As String, ByVal obj As Variant)
    Dim sty As Style: Set sty = EnsureParagraphStyle(styleName)

    Dim fnt As Font:        Set fnt = sty.Font
    Dim para As ParagraphFormat: Set para = sty.ParagraphFormat

    ' Font
    If ExistsKey(obj, "font") Then fnt.Name = CStr(obj("font"))
    If ExistsKey(obj, "size") Then fnt.Size = CDbl(obj("size"))

    ' Prefer "bold" if present; fallback to "heading_bold"
    If ExistsKey(obj, "bold") Then
        fnt.Bold = IIf(CBool(obj("bold")), True, False)
    ElseIf ExistsKey(obj, "heading_bold") Then
        fnt.Bold = IIf(CBool(obj("heading_bold")), True, False)
    End If

    If ExistsKey(obj, "underline") Then
        fnt.Underline = IIf(CBool(obj("underline")), wdUnderlineSingle, wdUnderlineNone)
    End If

    ' Color
    If ExistsKey(obj, "color") Then
        Dim c As Scripting.Dictionary: Set c = obj("color")
        fnt.TextColor.RGB = RGB(CInt(c("r")), CInt(c("g")), CInt(c("b")))
    End If

    ' Paragraph
    If ExistsKey(obj, "alignment") Then
        para.Alignment = AlignFromJson(CInt(obj("alignment")))
    End If
    If ExistsKey(obj, "space_before") Then para.SpaceBefore = CSng(obj("space_before"))
    If ExistsKey(obj, "space_after") Then para.SpaceAfter = CSng(obj("space_after"))
    If ExistsKey(obj, "indent") Then para.LeftIndent = InchesToPoints(CDbl(obj("indent")))
End Sub

'==============================
' HELPERS
'==============================

Private Function EnsureParagraphStyle(ByVal name As String) As Style
    Dim sty As Style
    On Error Resume Next
    Set sty = ActiveDocument.Styles(name)
    On Error GoTo 0

    If sty Is Nothing Then
        Set sty = ActiveDocument.Styles.Add(Name:=name, Type:=wdStyleTypeParagraph)
        sty.BaseStyle = "Normal"
    End If
    Set EnsureParagraphStyle = sty
End Function

Private Function SafeStr(ByVal s As String) As String
    SafeStr = Replace(s, """", """""")
End Function

Private Function JoinCollection(col As Collection, ByVal sep As String) As String
    Dim i As Long, s As String
    For i = 1 To col.Count
        If i > 1 Then s = s & sep
        s = s & CStr(col(i))
    Next
    JoinCollection = s
End Function

Private Function PointsToInches(ByVal pts As Double) As Double
    PointsToInches = pts / PPI
End Function

Private Function Num(ByVal d As Double) As String
    ' JSON needs dot decimal; avoid locale commas
    Num = Replace(Format$(d, "0.###"), ",", ".")
End Function

Private Sub RGBSplit(ByVal rgbVal As Long, ByRef r As Long, ByRef g As Long, ByRef b As Long)
    r = (rgbVal And &HFF&)
    g = (rgbVal \ 256) And &HFF&
    b = (rgbVal \ 65536) And &HFF&
End Sub

Private Function SafeFontRGB(f As Font) As Long
    On Error Resume Next
    SafeFontRGB = f.TextColor.RGB
    If Err.Number <> 0 Then
        Err.Clear
        SafeFontRGB = RGB(0, 0, 0)
    End If
    On Error GoTo 0
End Function

Private Function NormalizeAlignment(ByVal wdAlign As WdParagraphAlignment) As Long
    ' Map Word enum to simple [0:left,1:center,2:right,3:justify]
    Select Case wdAlign
        Case wdAlignParagraphCenter: NormalizeAlignment = 1
        Case wdAlignParagraphRight:  NormalizeAlignment = 2
        Case wdAlignParagraphJustify:NormalizeAlignment = 3
        Case Else:                   NormalizeAlignment = 0
    End Select
End Function

Private Function AlignFromJson(ByVal n As Long) As WdParagraphAlignment
    Select Case n
        Case 1: AlignFromJson = wdAlignParagraphCenter
        Case 2: AlignFromJson = wdAlignParagraphRight
        Case 3: AlignFromJson = wdAlignParagraphJustify
        Case Else: AlignFromJson = wdAlignParagraphLeft
    End Select
End Function

Private Function ExistsKey(ByVal dict As Variant, ByVal k As String) As Boolean
    On Error Resume Next
    ExistsKey = dict.Exists(k)
    On Error GoTo 0
End Function
