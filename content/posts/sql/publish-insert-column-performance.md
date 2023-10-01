---
title: 'SQL Server column add deployment performance'
date: 2023-09-22
# weight: 1
# aliases: ["/first"]
tags: ['sql', 'dacpac', 'sql server']
author: 'Jamie Sharpe'
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: 'A catch with SQL server database deployments and column insertions'
disableShare: false
disableHLJS: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: false
UseHugoToc: true
cover:
    image: '<image path/url>' # image path/url
    alt: '<alt text>' # alt text
    caption: '<text>' # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---

## Introduction

We recently carried out a deployment where our SQL Server database took much longer than normal to deploy.

This was caused by a seemingly harmless change where a column was inserted within the middle of the columns rather than the end. When a column is inserted in the middle of a SQL table, the entire table needs to be dropped and rebuilt as part of deployment. When you insert a column at the end of the table, the deployment executes a simple `ALTER TABLE {table_name} ADD {column_name} {datatype}` statement. When publishing the database on the dev machine with a much smaller amount of data the increase in deployment/publish time was not noticed, however as we moved to a real environment with more data the increase in time became apparent.

This can easily be reproduced to see what really happens.

## Setup

In Visual Studio I created a simple database project with a single table:

```SQL
CREATE TABLE [dbo].[MyTable]
(
    [Id] INT NOT NULL IDENTITY(1,1),
    [AddressLine1] NVARCHAR(256) NOT NULL,
    [AddressLine2] NVARCHAR(256) NULL,
    [AddressLine3] NVARCHAR(256) NULL,
    [AddressLine4] NVARCHAR(256) NULL,
    [AddressLine5] NVARCHAR(256) NULL,
    [Postcode] VARCHAR(256) NOT NULL,
    [LongText1] NVARCHAR(MAX) NOT NULL,
    CONSTRAINT [PK_MyTable] PRIMARY KEY ([Id])
)
```

And then I populated it with 200,000 rows of data, the key here is the "LongText1" column which is filled with 20,000 character long text.

```SQL
DECLARE @counter INT = 1;
DECLARE @longText VARCHAR(MAX);
SET @longText = REPLICATE('X', 20000); -- 20,000-character-long text

WHILE @counter <= 200000
BEGIN
    INSERT INTO MyTable (
        AddressLine1,
        AddressLine2,
        AddressLine3,
        AddressLine4,
        AddressLine5,
        Postcode,
        LongText1
    )
    VALUES (
        'AddressLine1-' + CAST(@counter AS VARCHAR(10)),
        'AddressLine2-' + CAST(@counter AS VARCHAR(10)),
        'AddressLine3-' + CAST(@counter AS VARCHAR(10)),
        'AddressLine4-' + CAST(@counter AS VARCHAR(10)),
        'AddressLine5-' + CAST(@counter AS VARCHAR(10)),
        'Postcode-' + CAST(@counter AS VARCHAR(10)),
        @longText -- Insert the 20,000-character-long text
    );
    SET @counter = @counter + 1;
END;
```

Now we are ready to repeat the problem.

## Inserting a column in the middle

In Visual Studio I updated the table definition to be as follows, with the `InsertMiddleCol` added.

```SQL
CREATE TABLE [dbo].[MyTable]
(
    [Id] INT NOT NULL IDENTITY(1,1),
    [AddressLine1] NVARCHAR(256) NOT NULL,
    [AddressLine2] NVARCHAR(256) NULL,
    [AddressLine3] NVARCHAR(256) NULL,
    [AddressLine4] NVARCHAR(256) NULL,
    [AddressLine5] NVARCHAR(256) NULL,
    [InsertMiddleCol] NVARCHAR(256) NULL,
    [Postcode] VARCHAR(256) NOT NULL,
    [LongText1] NVARCHAR(MAX) NOT NULL,
    CONSTRAINT [PK_MyTable] PRIMARY KEY ([Id])
)
```

And then I published the database, as seen below this took 1 minute and 18 seconds to publish.

![Publish time of 1 minute 18 seconds inserting column in the middle](/Publish_Database_Insert_Col_Middle.png)

## Inserting a column at the end

In Visual Studio I updated the table definition to be as follows, with the `InsertEndCol` added.

```SQL
CREATE TABLE [dbo].[MyTable]
(
    [Id] INT NOT NULL IDENTITY(1,1),
    [AddressLine1] NVARCHAR(256) NOT NULL,
    [AddressLine2] NVARCHAR(256) NULL,
    [AddressLine3] NVARCHAR(256) NULL,
    [AddressLine4] NVARCHAR(256) NULL,
    [AddressLine5] NVARCHAR(256) NULL,
    [InsertMiddleCol] NVARCHAR(256) NULL,
    [Postcode] VARCHAR(256) NOT NULL,
    [LongText1] NVARCHAR(MAX) NOT NULL,
    [InsertEndCol] NVARCHAR (MAX) NULL
    CONSTRAINT [PK_MyTable] PRIMARY KEY ([Id])
)
```

And again I published the database. This time the publish completed in 1 second.

![Publish time of 1 second inserting column to the end](/Publish_Database_Insert_Col_End.png)

## A deeper dive

We can view the scripts that will be executed during the publish of each of the above changes within Visual Studio by click the "Generate Script" button rather than "Publish" from the "Publish Database" wizard. This produces the below scripts.

First the insertion to the middle of the table:

```SQL
GO
USE [$(DatabaseName)];

GO
PRINT N'Starting rebuilding table [dbo].[MyTable]...';

GO
BEGIN TRANSACTION;

SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

CREATE TABLE [dbo].[tmp_ms_xx_MyTable] (
    [Id]              INT            IDENTITY (1, 1) NOT NULL,
    [AddressLine1]    NVARCHAR (256) NOT NULL,
    [AddressLine2]    NVARCHAR (256) NULL,
    [AddressLine3]    NVARCHAR (256) NULL,
    [AddressLine4]    NVARCHAR (256) NULL,
    [AddressLine5]    NVARCHAR (256) NULL,
    [InsertMiddleCol] NVARCHAR (256) NULL,
    [Postcode]        VARCHAR (256)  NOT NULL,
    [LongText1]       NVARCHAR (MAX) NOT NULL,
    CONSTRAINT [tmp_ms_xx_constraint_PK_MyTable1] PRIMARY KEY CLUSTERED ([Id] ASC)
);

IF EXISTS (SELECT TOP 1 1
           FROM   [dbo].[MyTable])
    BEGIN
        SET IDENTITY_INSERT [dbo].[tmp_ms_xx_MyTable] ON;
        INSERT INTO [dbo].[tmp_ms_xx_MyTable] ([Id], [AddressLine1], [AddressLine2], [AddressLine3], [AddressLine4], [AddressLine5], [Postcode], [LongText1])
        SELECT   [Id],
                 [AddressLine1],
                 [AddressLine2],
                 [AddressLine3],
                 [AddressLine4],
                 [AddressLine5],
                 [Postcode],
                 [LongText1]
        FROM     [dbo].[MyTable]
        ORDER BY [Id] ASC;
        SET IDENTITY_INSERT [dbo].[tmp_ms_xx_MyTable] OFF;
    END

DROP TABLE [dbo].[MyTable];

EXECUTE sp_rename N'[dbo].[tmp_ms_xx_MyTable]', N'MyTable';

EXECUTE sp_rename N'[dbo].[tmp_ms_xx_constraint_PK_MyTable1]', N'PK_MyTable', N'OBJECT';

COMMIT TRANSACTION;

SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

GO
PRINT N'Update complete.';

GO
```

And now the insertion to the end of table:

```SQL
GO
USE [$(DatabaseName)];

GO
PRINT N'Altering Table [dbo].[MyTable]...';

GO
ALTER TABLE [dbo].[MyTable]
    ADD [InsertEndCol] NVARCHAR (MAX) NULL;

GO
PRINT N'Update complete.';

```

As you can clearly see the insertion to middle causes a full rebuild of table versus a simple `ALTER TABLE` script for the insertion to the end.

## Conclusion

Inserting a column in the middle of a SQL Server table causing the table to be rebuilt on publish/deployment. Depending on the data in the table this may be a very long operation.

When making changes to tables check your publish script carefully and consider the impact of what looks like a simple harmless change.
