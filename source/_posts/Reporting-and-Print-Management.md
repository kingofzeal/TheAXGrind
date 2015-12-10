title: Reporting and Print Management
date: 2015-04-27 07:00:00
tags:
 - X++
categories:
 - AX 2009
 - Reporting
---
No code, but a helpful story this time.

Some time ago, we created two posting-style reports for our implementation of AX. These reports follow the same process as other posting processes (like invoices, packing slips, etc), so while not the easiest report to create due to all the new classes required by the `FormLetter` framework, we were able to get them in place and workable. To help expedite the development process, instead of creating the new classes from scratch we duplicated the class that was similar to it, changed a few of the parameters and wired it up to the new report layout. For example, we duplicated the `SalesFormLetter_Invoice` class to `SalesFormLetter_OtherInvoice`, `SalesFormLetterReport_Invoice` to `SalesFormLetterReport_OtherInvoice`, etc.

One thing we have been fighting since the development of these reports is the print settings. From a users perspective, the settings can be changed, but are never used. In fact, no matter what you set on these new reports, it will always use the print settings on the report that was copied; in the above example, when you would go to run the Sales Order Other Invoice, it would use the Sales Order Invoice's print settings. This caused some especially odd errors when users moved desk locations and had a different printer to print to, more so if the user did not have access to the original report. In this situation, the only way we could figure to address the issue was, after such a move, security would be opened for that user to be able to run the original report, make the necessary changes to the print settings and revoke access. This was cumbersome, but thankfully did not occur terribly often.

More recently, one of our users that has to deal with this issue was bouncing between two different desk locations on a regular basis, and each of the locations had a different printer to print to. Since the printers were local (not networked), the printer the report tried to print to was considered 'not installed', and they were unable to print at all. The default was to send the report directly to the printer, and as we were unable to change the settings on the parent report, it caused a fair amount of legwork to get everything working until the user could settle down to a single desk location.

Finally being able to take the time to comb through these reports, I found that this issue was cause because after duplicating the classes, not all the settings were taken into account - particularly the settings revolving around Print Management. Previously, this had been overlooked because we don't actively use the Print Management framework.

Following the [Microsoft White Paper AX 2009 Print Management Integration Guide](http://www.microsoft.com/en-us/download/details.aspx?id=4393), I created new document types for these reports, associated them in the same node locations as the original reports, and updated the trouble reports to use the new document types. Once the default module-level Print Management settings were created for the new document types, the reports began to print to the location requested in the posting dialog without impacting the original report settings.

If anyone seems to have the same issue, be sure to look closely at the `printMgmtDocumentType` method of the `FormLetter` class for the report (eg `SalesFormLetter_Invoice`, `PurchFormLetter_PackingSlip`), and make sure it references a unique document in the Print Management framework. It's easy to overlook, especially if you do not use the framework for printing.