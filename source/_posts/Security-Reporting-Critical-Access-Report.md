title: Security Reporting - Critical Access Report
date: 2015-03-31 07:00:00
tags:
 - Security
 - X++
 - Reporting
categories:
 - AX 2009
 - Reporting
---
One of the more well-known drawbacks of AX 2009 is the lack of user security reporting. For privately held companies, this can be an issue, but one some may be comfortable overlooking. For a publicly traded company, this problem is expounded, as auditors will often demand some kind of documentation that users cannot get access to cross-functional responsibilities that would violate Segregation of Duties policies. Without third-party tools, this reporting normally means that the auditor in question would run the User Permissions report for the users, and compare that with a document that organizes what access a user must have to do a particular task. Normally this is a long, tedious and expensive process, which is done at least once a year.

To help combat this process, we created a reasonably simple report which does this automatically. By making the system check for permissions, we have reduced errors in these types of checks significantly and can get this information very quickly and reliably.

At the core of the report, there are two populated tables: one of the tables holds a list of all the functions we want to test for, while the other table holds what elements should be checked for each function. Since most functions actually need multiple permissions together to be able to work correctly, there is a 1:n relationship between these tables. For example:
![](CriticalFunctionAccess.png)
 

The Category column is simply a way our auditors prefer the functions to be grouped. Additionally, each security element needs to be identified by the actual AOT path. For the purposes of this report, all table access is governed by the table itself, not by the permissions on individual fields.

Additionally a user must have the specified access (or higher) for each element for the user to be considered to have access to the particular function. If they are missing the correct level of access to just one of the nodes, they are not considered to have access to the function.

Once the information is populated, the primary class is executed.

```axapta SecurityAudit
class SecurityAudit
{
    //Tables
    SecurityAuditFunction              functions;
    SecurityAuditFunctionDefinition    funcDefs;
    UserInfo                           userInfo;

    //Local vars
    SecurityKeySet                     sks;

    //Excel-related objects
    SysExcelApplication                application;
    SysExcelWorkbook                   workbook;
    SysExcelWorksheet                  worksheet;
    SysExcelCells                      cells;
    SysExcelCell                       cell;
    int                                row;
    int                                column;
}

private void loadSecurityKeySet(userId _userId, selectableDataArea _company)
{
    ;

    sks = new SecurityKeySet();
    sks.loadUserRights(_userId, _company);
}

private void loadSecurityKeySet(userId _userId, selectableDataArea _company)
{
    ;

    sks = new SecurityKeySet();
    sks.loadUserRights(_userId, _company);
}

private AccessType permissionForElement(str _elementAotPath)
{
    TreeNode    tn;
    ;

    if (!sks)
        throw error(strfmt("SecurityKeySet must be loaded before checking permissions"));

    tn = TreeNode::findNode(_elementAotPath);

    if (!tn)
        throw error(strfmt("AOT Path does not exist: %1", _elementAotPath));

    switch (tn.applObjectType())
    {
        case UtilElementType::OutputTool:
        case UtilElementType::DisplayTool:
        case UtilElementType::ActionTool:
            return sks.secureNodeAccess(tn.AOTname(), tn.applObjectType());
        case UtilElementType::Table:
            return sks.tableAccess(tn.applObjectId());
        case UtilElementType::TableField:
            return sks.fieldAccess(tn.AOTparent().AOTparent().applObjectId(), tn.applObjectId());
        case UtilElementType::SecurityKey:
            return sks.access(tn.applObjectId());
        default:
            throw error(strfmt("Could not determine security for %1 node %2", tn.applObjectType(), tn.AOTname()));
    }
}

private boolean permissionForFunction(SecurityAuditFunctionId _functionId)
{
    AccessType  userElementAccess;
    int         funcDefLines;
    ;

    while select funcDefs
        where funcDefs.FunctionId == _functionId
    {
        userElementAccess = this.permissionForElement(funcDefs.AotPath);
        funcDefLines++;

        //Short-circuit lookups if user doesn't have necessary access
        if (userElementAccess < funcDefs.RequiredAccess)
            return false;
    }

    //only return true if there was at least one definition record for the function
    return (funcDefLines > 0);
}

public void run()
{
    #AviFiles
    NoYes                   userHasAccess;
    CompanyDomainList       companyDomainList;
    SysOperationProgress    progress            = new SysOperationProgress(3);
    ;


    progress.setCaption("Generating Security Report");
    progress.setAnimation(#AviUpdate);

    select count(RecId)
        from companyDomainList
        where companyDomainList.DomainId == "Prod";
        
    select count(RecId)
        from userInfo
        where userInfo.Enable == NoYes::Yes;
        
    select count(RecId)
        from functions;

    progress.setTotal(companyDomainList.RecId * userInfo.RecId * functions.RecId, 1);
    progress.setTotal(userInfo.RecId * functions.RecId, 2);
    progress.setTotal(functions.RecId, 3);

    while select companyDomainList
        where companyDomainList.DomainId == "Prod"
    {
        progress.setText(companyDomainList.CompanyId, 1);
        progress.setCount(0, 2);
        this.setupExcel(companyDomainList.CompanyId);

        while select userInfo
            where userInfo.Enable == NoYes::Yes
        {
            //processes to do on a per-user basis
            progress.setText(userInfo.Id, 2);
            progress.setCount(0, 3);
            this.loadSecurityKeySet(userInfo.Id, companyDomainList.CompanyId);

            //Advance the Excel cell one row and back to beginning
            row++;
            column = 1;
            this.writeInExcel(userInfo.Id);

            while select functions
                order by FunctionCategory   asc,
                         FunctionId         asc
            {
                //processes to do on a per-function basis
                progress.setText(functions.FunctionId, 3);
                progress.incCount(1, 1);
                progress.incCount(1, 2);
                progress.incCount(1, 3);
                userHasAccess = this.permissionForFunction(functions.FunctionId);
                this.writeInExcel(enum2str(userHasAccess));
            }
        }
    }

    application.visible(true);
}

///Excel needs some header information added to it before the main work can be done
private void setupExcel(selectableDataArea _company)
{
    str     rangeStr;
    int     sheetCount,
            counter;
    ;

    if (!application)
    {
        application = SysExcelApplication::construct();
        //application.visible(true);
        workbook    = application
                            .workbooks()
                            .add();

        //Delete all but the first worksheet
        for (counter = workbook.worksheets().count(); counter > 1; counter --)
        {
            workbook
                .worksheets()
                .itemFromNum(counter)
                .delete();
        }

        worksheet   = application
                            .workbooks()
                            .item(1)
                            .worksheets()
                            .itemFromNum(1);

        worksheet.name(_company);
        cells       = worksheet.cells();
    }
    else
    {
        worksheet   = workbook
                            .worksheets()
                            .add();

        worksheet.name(_company);
        cells       = worksheet.cells();
    }

    //==== Write Categories ====
    row     = 7;
    column  = 2;

    while select FunctionCategory, count(RecId)
        from functions
        group by FunctionCategory
        order by FunctionCategory   asc
    {
        this.writeInExcel(functions.FunctionCategory);
        column += functions.RecId - 1; //write function advances by 1 automatically
    }

    //===== Write Functions ====
    row++;
    column = 2;

    while select functions
        order by FunctionCategory   asc,
                 FunctionId         asc
    {
        this.writeInExcel(functions.FunctionId);
    }
}

///Writes the provided text to the current cell coordinates in Excel
///Will automatically advance the cell to the next column
private void writeInExcel(str text)
{
    cells.item(row, column).value(text);
    column++;
}

static SecurityAudit construct()
{
    return new SecurityAudit();
}

static void main(Args args)
{
    SecurityAudit  audit;
    ;

    audit = SecurityAudit::construct();
    audit.run();
}
```

The report itself is generated directly to Excel, because the system can be set up to have any number of functions, and test any number of users for those functions. In addition, it is run for every company, creating a new tab for each one. In our case, we have a domain that contains all of our active companies which drives what companies the report will run for ("Prod", which is referenced a couple times in the code). Various methods are included to help easily manage the Excel side of things so we can focus on the more important part: determining access.

The heavy lifting is of course done by the Run method, which loops through each function and hands off to to the permissionForFunction method. This method, in turn, loops through the definition of the function and passes each AOT path to the permissionForElement function, which tells what level of access (if any) the user has to that particular AOT node. If at any time the permission for the AOT node falls below the required level for that node, we short-circuit a "No", which keeps the system from having to check every node for every user.

Finally, once we know if the user has access or not, we print the answer to the appropriate Excel cell. When the spreadsheet has been completely populated, we make the application visible and end the task.

 

This relatively simple report makes it easy to add or remove what we term Critical Access areas, which are generally SoD compliance type functions, but it could be used for other things as well. Since everything is run on-demand, and against the user's actual security, we can be sure that so long as the definitions are accurate, the report will also be accurate, and the results can be examined in minutes instead of days.

 