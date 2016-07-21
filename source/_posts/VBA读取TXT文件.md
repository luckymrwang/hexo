title: VBA读取TXT文件
date: 2016-06-24 14:41:07
tags: [VBA]
toc: true
---

用VBA读取TXT文件到EXCEL

```vb
    Dim objStream, strData    Set objStream = CreateObject("ADODB.Stream")    objStream.Charset = "utf-8"
    
    ‘ LINUX 换行符 objStream.LineSeparator = 10  默认是 -1 回车换行符
        objStream.Open    objStream.LoadFromFile ("C:\Users\Administrator\Desktop\test.txt")    rowNo = 1    Do While Not (objStream.EOS)        Worksheets("Sheet1").Cells(rowNo, 1).Value = objStream.ReadText(-2) '每次读一行        rowNo = rowNo + 1    Loop            ' 关闭    objStream.Close    Set objStream = Nothing
```