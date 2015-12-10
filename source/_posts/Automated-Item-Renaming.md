title: Automated Item Renaming
date: 2015-01-16 13:08:29
tags:
 - X++
 - Development
categories:
 - AX 2009
 - Data
---
We have recently started a long-term project to standardize our purchased and manufactured part numbers - not a small task. We're okay with doing a straight rename on the item card (which flows to all documentation historically), but renaming items has two side effects: the user who triggered the rename has their client session effectively locked until the rename is complete (which can take up to half an hour), and other users who use items on a regular basis, such as those doing production scheduling and entries, notice the system running considerably slower during the rename. In a few cases, we have even run into deadlock issues.

To help combat this, we've constructed a periodic task for renaming items. Simply, a table holds a queue of what items should be renamed to what, and a batch task is scheduled to process the renaming after-hours. Recently, this was expanded so any BOMs related to the item would be renamed to coincide with the new item number as well (our BOMs are named after the item numbers, with versions of the BOM being named [Item Number].[Version]).

Here is what the process code looks like:

```CSharp 
public void run()
{
    System.Text.RegularExpressions.Regex            regex;
    System.Text.RegularExpressions.Match            regexMatch;
    System.Text.RegularExpressions.GroupCollection  groups;
    System.Text.RegularExpressions.Group            grop;
    System.Text.RegularExpressions.RegexOptions     options;

    MaintBatchRenaming                              renaming,
                                                    renaming2;
    InventTable                                     inventTable;
    fieldId                                         inventFieldId   = fieldnum(InventTable, ItemId),
                                                    bomFieldId      = fieldnum(BOMTable, bomId);

    BOMVersion                                      bomVersion;
    BOMTable                                        bomTable;
    str 45                                          newBomId;
    InteropPermission                               permission;
    str                                             temp;
    ;

    permission = new InteropPermission(InteropKind::ClrInterop);
    permission.assert();

    ttsbegin;

    update_recordset renaming
        setting InProgress = NoYes::Yes
        where renaming.Completed == NoYes::No;

    ttscommit;

    setprefix("Mass rename items");

    while select forupdate renaming
        where renaming.InProgress == NoYes::Yes
    {
        setprefix(strfmt("%1 => %2", renaming.ItemId, renaming.NewItemId));

        if (InventTable::find(renaming.NewItemId).RecId)
        {
            error(strfmt("%1 already exists. Cannot rename.", renaming.NewItemId));

        }
        else
        {

            try
            {
                ttsbegin;

                inventTable = InventTable::find(renaming.ItemId);
                info(strfmt("Renaming item %1 to %2", renaming.ItemId, renaming.NewItemId));

                inventTable.ItemId = renaming.NewItemId;
                inventTable.renamePrimaryKey();

                info(strfmt("Item rename complete"));

                if (renaming.RenameBom == NoYes::Yes)
                {
                    options = System.Text.RegularExpressions.RegexOptions::IgnoreCase;

                    //BP Deviation Documented
                    regex = new System.Text.RegularExpressions.Regex("([a-z0-9\-/]+)(\.[0-9]{0,3})?", options);

                    while select bomVersion
                        join bomTable
                        where bomVersion.ItemId == renaming.NewItemId
                           && bomTable.bomId == bomVersion.bomId
                    {
                        setprefix(strfmt("BOM %1", bomTable.bomId));
                        regexMatch = regex.Match(bomTable.bomId);

                        if (regexMatch.get_Success())
                        {
                            groups = regexMatch.get_Groups();
                            grop = groups.get_Item(1);
                            temp = grop.get_Value();

                            if (temp == renaming.OrigItemId)
                            {
                                grop = groups.get_Item(2);
                                temp = grop.get_Value();

                                newBomId = strfmt("%1%2", renaming.NewItemId, temp);

                                info(strfmt("Renaming BOM %1 to %2", bomTable.bomId, newBomId));

                                bomTable.bomId = newBomId;
                                bomTable.renamePrimaryKey();

                                info(strfmt("BOM Rename complete"));
                            }
                        }
                    }
                }

                select firstonly forupdate renaming2
                    where renaming2.RecId == renaming.RecId;

                renaming2.Completed         = NoYes::Yes;
                renaming2.Error             = NoYes::No;
                renaming2.InProgress        = NoYes::No;
                renaming2.CompletedDateTime = DateTimeUtil::getSystemDateTime();
                renaming2.update();

                ttscommit;
            }
            catch
            {
                Global::exceptionTextFallThrough();
            }
        }
    }
}
```

The process is fairly straight forward. We have a definition or queue table, `MaintBatchRenaming`. We begin by setting all non-complete records on that table to be marked as 'In Progress', and then loop through all records that are marked as such (this helps catch records that were previously marked but never finished). For each item, we check to see if the new item number already exists, and if it does we skip the process. Otherwise, we pull the `InventTable` record, set the `ItemId` to the new item number and run the `renamePrimaryKey` method. Contrary to 99% of the examples you find when researching the `renamePrimaryKey` method, you do not need to call `CCPrimaryKey::renamePrimaryKey` unless you have the Human Resources I or Questionnaire II configuration keys enabled (plus a few other requirements), which we do not.

During the process, we also look at all the BOMs associated with the item, and run it against a regex to determine what the BOM should be renamed to - for example, if the item ABC is renamed to XYZ, the BOM ABC.1 should be renamed to XYZ.1, ABC.2 should be renamed to XYZ.2, etc. If the BOM does not match the [Item Number].[Version] pattern, it is not renamed.

Once all the renaming is complete, the record in the queue is marked as complete and no longer in progress with current the date and time, and the process moves to the next item. If there is a problem renaming the item and/or BOMs, the record is skipped until the next processing.

 

One of the challenges associated with this is validation and historical record keeping. We want make sure the item number to be changed is an actual item number - this can be accomplished by associating the field with the EDT `ItemId`, which enforces the association. However, once the rename is complete, we have no record of what the original item number was because it was renamed everywhere that association is enforced.

To combat this, we created another field on our queue table, `OrigItemId`, which is a string of the same size as `ItemId`, and added an edit method on the table:

```CSharp
public edit ItemId itemId(boolean _set, ItemId _itemId)
{
    ;

    if (this.Completed)
        return this.OrigItemId;

    if (_set)
    {
        this.ItemId = _itemId;
    }

    return this.ItemId;
}
```

We also had to update two other methods on the table to make everything work correctly:

```CSharp
public void update()
{
    if (this.orig().ItemId != this.NewItemId &&
        this.orig().ItemId != this.ItemId)
    {
        this.OrigItemId = this.ItemId;
    }

    super();
}
```
```CSharp
public void insert()
{
    this.OrigItemId = this.ItemId;

    super();
}
```
We then use the edit method on the form instead of directly adding the item number field. Because it has a return type of ItemId, it is also associated with the Item table and gets the validation on input, but the return value is not validated, allowing us to show a string that is no longer an item number in the form after the rename is complete. 

The end result is a system that allows users to add renaming tasks to, which complete overnight in batch. Not only do the renames process faster on the server, but they also don't have as many issues because fewer users are in the system. 