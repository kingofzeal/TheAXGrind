title: 'AX 2009: Export to Excel problems'
date: 2013-10-01 09:00:00
tags:
 - X++
 - Excel
categories: 
 - AX 2009
---
We have recently seen an issue with the Export to Excel feature of AX 2009, where a stray double quotation mark in the grid will cause all subsequent fields to be offset. Instead of getting nicely formatted rows and columns, we had a few well-formatted rows, and some other rows that weren't so nicely formatted. This is also shown in one or two other places around there internet (such as [http://community.dynamics.com/ax/f/33/t/102643.aspx](http://community.dynamics.com/ax/f/33/t/102643.aspx)), but as much as I looked I could not find a solution. We had looked at this problem earlier, as many of our part number include double quotes to represent inches. Previously, we modified the itemName method on the InventTable to replace double quotes with single quotes, as that would not break Excel and was an easy fix. However, we recently discovered that many other user fields were starting to have double quotes in them, and we needed a way to address all of them.

Taking a lead from the MSDN post [How does the Export to Excel feature work under the hood](http://blogs.msdn.com/b/emeadaxsupport/archive/2009/09/07/how-does-the-export-to-excel-feature-work-under-the-hood.aspx), I looked at the SysGridExportToExcel class, specifically the performPushAndFormatting method. I also began to monitor the Windows clipboard, since as that posts explains, the Export to Excel feature relies heavily on the clipboard.

I figured there are three ways I can attack this issue:

- Create an edit method for every field that would hold a double quotation mark, and reference that method instead of directly referencing the field on the form. This would cause the form filter to not work properly, plus the thought of doing this for every field seemed daunting.
- Modify the system so that when the Export to Excel process begins (before the clipboard is populated) all the incoming fields have their double quotes replaced to two single quotes. This is the ideal solution, since there would not be any reprocessing costs like there would be later in the process (after the clipboard has been populated)
- Modify the system so that after the clipboard is populated but before the information is pasted into Excel, all double quotes are replaced to single quotes. Looking at the information that was being sent to the clipboard showed that the information was formatted as Tab separated values, with most test field surrounded by double quotes, which would need to be preserved.


Since the first option would be a last resort, I began to look into the second option: modify the system to change how it generates the information to be sent to the clipboard. However, even searching online I could not find where the system did this. The only clue I had was the stack trace after hitting a breakpoint early in the performPushAndFormatting method, which seems to indicate it was built into the FormRun base class. Because it is a system base class, I cannot modify it (though it would be the appropriate place to do so). My only other option would be to create my own class that inherits from FormRun, override the Task to build in my own functionality, and proceed to update every form in the system to inherit from this new class. However, since I have no idea what is actually happening in this method AND I would have to do it on every form, this also seemed like a dead end. 

The last option, to modify the clipboard data after it has already been generated seemed to be my only option. I discovered the **TextBuffer** class in AX has a handy fromClipboard and toClipboard method, so I would use that. 

Within the performPushAndFormatting method, before any of the Excel work begins, I added the following code:

```axapta SysGridExportToExcel.performPushAndFormatting
TextBuffer buffer = new TextBuffer();
System.Text.RegularExpressions.Regex regex;
str cleanedText;
;

if (buffer.fromClipboard())
{
    regex = new System.Text.RegularExpressions.Regex("(?<=[^\\t\\r\\n])\"(?=[^\\t\\r\\n])");

    cleanedText = regex.Replace(buffer.getText(), "\"\"");

    buffer.setText(cleanedText);
    buffer.toClipboard();
}
```

This replaces all double quotes that are inside the field with two double quotes, which Excel interprets as an escaped double quote. String type fields, when copied from the clipboard, are surrounded by double quotes; the regular expression above excludes those and replaces everything else.

We attempted several ways of accomplishing this goal, from using a series of strReplace commands, to manually parsing the clipboard string character by character, but both of these options were slow when dealing with a large export set.