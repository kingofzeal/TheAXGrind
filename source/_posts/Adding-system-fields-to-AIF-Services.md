title: Adding system fields to AIF Services
tags:
 - X++
 - Development
date: 2015-12-23 06:00:00
categories:
 - AX 2009
 - AIF
---
We have recently been able to really dive in to the AX AIF Webservices, and like most things the road was bumpy and filled with potholes. This isn't to say that webservices had a lot of problems. In fact, using webservices was certainly beneficial since much of the work has already been done. However, there were some things we had to do to make sure evrything was working the way we want it to.

One of the biggest things that we ran into was making system fields, such as `modifiedDateTime`, `modifiedBy`, `createdDateTime` and `createdBy`, available for consumption by subscribers of the webservice. These fields, while available for use within AX and thus also available via the Business Connector, are not natively generated and added to the webservice classes.

In our case, we wanted to make sure we had the createdDateTime field available in the InvoiceService. The examples I will use will focus around this use case, but they should be transferable to any other similar case.

First, we need to make sure the table actually has the field enabled. On the table properties, simply make sure the `createdDateTime` property is set to "Yes":

![](TableProperties.png)

If this property is not set, change it and syncronize the table. You may also want to run the AIF "Update Document Service" process, but that is not strictly mandatory.

From there, we need to modify the appropriate AIF class to be able to read from the field. The class name we are looking for is named `Ax[TableName]`; since we are focusing on the `CustInvoiceJour` table, the class will be `AxCustInvoiceJour`. 

Within this class, we will need to create a method that matches the format `parm[FieldName]`. Again, as we are trying to bring the `createdDateTime` field forward, the method will be `parmCreatedDateTime`. The method should be similar to other parm-type methods: 

```axapta
public [DataType] parm[FieldName]([DataType] _[FieldName] = [DataTypeDefault])
{
    [Value] = _[FieldName];

    return [Value];
}
```

In out case, this is our new method:

```axapta
public UtcDateTime parmCreatedDateTime(UtcDateTime _createdDateTime = DateTimeUtil::minValue())
{
    return custInvoiceJour.createdDateTime;
}
```

You will notice that we left the assignment line of the normal method off our implementation. This is because the system fields are always readonly. This change allows someone to specify a new createdDateTime, but since the parameter is ignored, nothing will actually happen.

Finally, the last step is to "publish" the changes. This is done by opening the AIF Services form (found at `Basic / Setup / Application Integration Framework / Services`) and click the 'Refresh' button and then click the 'Generate' button. This will regenerate the WDSL files for the service, which can then be updated wherever the service is used.