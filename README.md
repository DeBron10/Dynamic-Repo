I'll build this as a complete ASP.NET Web Forms + C# WebMethods + SQL Server + jQuery AJAX example, from selecting columns in the frontend to executing a filtered SQL query and displaying the returned report.

We'll use your existing report-builder structure: `selectedTables`, `selectedColumns`, and filter objects containing `columnKey`, `condition`, `value`, `value2`, and `selectedValues`.

## 1. How the complete flow works

Frontend: Report Builder

User selects a table, columns and filters

JavaScript builds the request

`JSON.stringify()` converts selections and filters to JSON

jQuery AJAX → C# WebMethod

Receives the JSON and validates the requested fields and conditions

SQL Server

Builds a safe `SELECT` query and parameterizes filter values

JSON response → Report table

C# returns the rows, and JavaScript renders them

I'll use sample `dbo.ProjectDetails` data with these columns: `ProjectId`, `ProjectName`, `StartDate`, `Cost`, `Category`, `Place`, and `Status`.

## 2. Sample SQL Server database and data

Run this script in SQL Server Management Studio (SSMS). If your database already exists, create the table and sample rows in that database instead.

```
CREATE DATABASE ReportBuilderDB;
GO

USE ReportBuilderDB;
GO

CREATE TABLE dbo.ProjectDetails
(
    ProjectId   INT           NOT NULL PRIMARY KEY,
    ProjectName NVARCHAR(100) NOT NULL,
    StartDate   DATE          NOT NULL,
    Cost        DECIMAL(12,2) NOT NULL,
    Category    NVARCHAR(50)  NOT NULL,
    Place       NVARCHAR(50)  NOT NULL,
    Status      NVARCHAR(30)  NOT NULL
);
GO

INSERT INTO dbo.ProjectDetails
    (ProjectId, ProjectName, StartDate, Cost, Category, Place, Status)
VALUES
(1, 'Road Development',  '2026-01-10', 250000, 'Infrastructure', 'Pune',   'Completed'),
(2, 'School Renovation', '2026-02-15', 150000, 'Education',      'Mumbai', 'In Progress'),
(3, 'Water Supply',      '2026-03-01', 400000, 'Infrastructure', 'Nashik', 'Pending'),
(4, 'Hospital Upgrade',  '2026-03-20', 350000, 'Healthcare',     'Pune',   'In Progress'),
(5, 'Library Project',   '2026-04-05',  90000, 'Education',      'Nashik', 'Completed'),
(6, 'Bridge Repair',     '2026-04-12', 550000, 'Infrastructure', 'Mumbai', 'Pending'),
(7, 'Clinic Setup',      '2026-05-01', 180000, 'Healthcare',     'Pune',   'Completed'),
(8, 'School Extension',  '2026-05-18', 220000, 'Education',      'Mumbai', 'In Progress');
GO
```

For example, the user might select `ProjectName`, `Cost`, `Place`, and `Status`, then add two filters:

- `Cost > 200000`
- `Place IN ('Pune', 'Mumbai')`

The backend should return only rows matching both filters.

## 3. Configure the ASP.NET project

Create an ASP.NET Web Forms project in Visual Studio with these files:

```
ReportBuilder/
├── ReportBuilder.aspx
├── ReportBuilder.aspx.cs
└── Web.config
```

Use .NET Framework with `System.Web` and `System.Data.SqlClient`.

### `Web.config`

Add the connection string inside the existing `<configuration>` element. Replace the server name with your SQL Server instance.

```
<configuration>
  <connectionStrings>
    <add name="ReportBuilderDB"
         connectionString="Data Source=YOUR_SERVER_NAME;
                           Initial Catalog=ReportBuilderDB;
                           Integrated Security=True;
                           TrustServerCertificate=True"
         providerName="System.Data.SqlClient" />
  </connectionStrings>
</configuration>
```

For example, your local server might be `.\SQLEXPRESS` or `(localdb)\MSSQLLocalDB`. Use the instance that actually hosts your database. For production, use your organization's approved connection and authentication settings.

## 4. Frontend: `ReportBuilder.aspx`

This is a working example of the page. It retrieves the available columns and their filter options from the backend, creates column checkboxes, creates type-specific filter controls, sends the request, and displays the returned report.

```

<%@ Page Language="C#" AutoEventWireup="true"
    CodeBehind="ReportBuilder.aspx.cs"
    Inherits="ReportBuilder.ReportBuilder" %>

<!DOCTYPE html>
<html>
<head runat="server">
    <title>Dynamic Report Builder</title>

    <script src="https://code.jquery.com/jquery-3.7.1.min.js"></script>

    <style>
        body {
            font-family: Arial, sans-serif;
            background: #f4f6f9;
            margin: 0;
            padding: 24px;
            color: #222;
        }

        .container {
            max-width: 1100px;
            margin: auto;
            background: white;
            padding: 24px;
            border-radius: 12px;
            box-shadow: 0 4px 16px #00000010;
        }

        h1, h2 { margin-top: 0; }

        .section {
            border: 1px solid #e1e5eb;
            border-radius: 8px;
            padding: 16px;
            margin: 18px 0;
        }

        .field-list, .filter-row {
            display: flex;
            flex-wrap: wrap;
            gap: 12px;
            align-items: center;
        }

        .field-item {
            background: #f6f8fb;
            padding: 10px;
            border-radius: 6px;
        }

        select, input, button {
            padding: 9px;
            border: 1px solid #ccd2dc;
            border-radius: 6px;
            font: inherit;
        }

        button {
            cursor: pointer;
            border: none;
            color: white;
            font-weight: bold;
        }

        .primary { background: #2563eb; }
        .success { background: #16803c; }
        .warning { background: #b45309; }
        .danger  { background: #dc2626; }

        .filter-card {
            padding: 14px;
            margin: 12px 0;
            background: #f8fafc;
            border: 1px solid #e2e8f0;
            border-radius: 8px;
        }

        .message { margin: 12px 0; white-space: pre-wrap; }
        .error { color: #b91c1c; }
        .ok { color: #166534; }

        .table-wrap { overflow-x: auto; }

        table {
            border-collapse: collapse;
            width: 100%;
            margin-top: 12px;
        }

        th, td {
            padding: 10px;
            border: 1px solid #e5e7eb;
            text-align: left;
        }

        th { background: #eef2ff; }

        .hidden { display: none; }

        .filter-values {
            display: flex;
            flex-wrap: wrap;
            gap: 10px;
            align-items: center;
        }
    </style>
</head>

<body>
<form id="form1" runat="server">
<div class="container">

    <h1>Dynamic Report Builder</h1>
    <p>Select a data table, choose the columns you need, apply filters,
       and generate a report.</p>

    <div id="message" class="message"></div>

    <div class="section">
        <h2>1. Choose a table</h2>
        <select id="tableSelect">
            <option value="">Loading tables...</option>
        </select>
    </div>

    <div class="section">
        <h2>2. Choose columns</h2>
        <div id="fieldList" class="field-list"></div>
        <p>Selected columns: <strong id="columnCount">0</strong></p>
        <button type="button" class="warning" id="clearColumns">
            Clear columns
        </button>
    </div>

    <div class="section">
        <h2>3. Add filters</h2>
        <div id="filterList"></div>
        <button type="button" class="success" id="addFilter">
            + Add filter
        </button>
    </div>

    <div class="section">
        <h2>4. Generate your report</h2>
        <button type="button" class="primary" id="generateReport">
            Generate report
        </button>
        <button type="button" class="warning" id="exportCsv">
            Export CSV
        </button>
        <p id="reportSummary"></p>
    </div>

    <div class="section">
        <h2>Report results</h2>
        <div id="reportOutput" class="table-wrap">
            Your report will appear here.
        </div>
    </div>

</div>
</form>

<script>
"use strict";

let tables = {};
let selectedTables = [];
let selectedColumns = [];
let filters = [];
let nextFilterId = 1;
let reportRows = [];
let reportColumns = [];

function showMessage(message, isError) {
    $("#message")
        .text(message || "")
        .toggleClass("error", !!isError)
        .toggleClass("ok", !isError && !!message);
}

// ---------------- LOAD DATABASE SCHEMA ----------------

function loadSchema() {
    $.ajax({
        type: "POST",
        url: "ReportBuilder.aspx/GetReportSchema",
        data: JSON.stringify({}),
        contentType: "application/json; charset=utf-8",
        dataType: "json",

        success: function (response) {
            const schema = response.d || [];
            tables = {};

            schema.forEach(function (table) {
                tables[table.key] = table;
            });

            const $select = $("#tableSelect").empty();

            schema.forEach(function (table) {
                $("<option>")
                    .val(table.key)
                    .text(table.name)
                    .appendTo($select);
            });

            if (schema.length === 0) {
                showMessage("No report tables are available.", true);
                return;
            }

            // This sample supports one selected table per report.
            selectedTables = [schema[0].key];
            $select.val(schema[0].key);

            displayFields();
            displayFilters();
            showMessage("Database schema loaded successfully.", false);
        },

        error: function (xhr) {
            showMessage(
                "Could not load schema: " + getAjaxError(xhr), true
            );
        }
    });
}

function getAjaxError(xhr) {
    if (xhr.responseJSON && xhr.responseJSON.Message) {
        return xhr.responseJSON.Message;
    }

    try {
        const response = JSON.parse(xhr.responseText);
        if (response && response.Message) return response.Message;
    } catch (_) {}

    return xhr.statusText || "Unknown server error";
}

// ---------------- COLUMN CHECKBOXES ----------------

function displayFields() {
    const tableKey = $("#tableSelect").val();
    const table = tables[tableKey];
    const $list = $("#fieldList").empty();

    selectedColumns = [];

    if (!table) return;

    table.columns.forEach(function (column) {
        const fullKey = tableKey + "." + column.key;
        const id = "column_" + column.key;

        const $label = $("<label>")
            .addClass("field-item")
            .attr("for", id);

        const $checkbox = $("<input>", {
            type: "checkbox",
            id: id,
            value: fullKey
        });

        $checkbox.on("change", function () {
            readSelectedFields();
        });

        $label.append($checkbox);
        $label.append(" " + column.name);
        $list.append($label);
    });

    updateColumnCount();
}

function readSelectedFields() {
    selectedColumns = [];

    $("#fieldList input[type=checkbox]:checked").each(function () {
        selectedColumns.push($(this).val());
    });

    updateColumnCount();
}

function updateColumnCount() {
    $("#columnCount").text(selectedColumns.length);
}

// ---------------- FILTER BUILDER ----------------

function addFilter() {
    filters.push({
        id: nextFilterId++,
        columnKey: "",
        condition: "",
        value: "",
        value2: "",
        selectedValues: []
    });

    displayFilters();
}

function getColumnDetails(fullKey) {
    if (!fullKey) return null;

    const parts = fullKey.split(".");
    const table = tables[parts[0]];

    if (!table) return null;

    return table.columns.find(function (column) {
        return column.key === parts[1];
    }) || null;
}

function getFilterConditions(type) {
    if (type === "number") {
        return [
            { value: ">", text: "Greater than" },
            { value: "<", text: "Less than" },
            { value: "=", text: "Equal to" },
            { value: "between", text: "Between" }
        ];
    }

    if (type === "date") {
        return [
            { value: "between", text: "Between dates" }
        ];
    }

    if (type === "multi") {
        return [{ value: "in", text: "Is any of" }];
    }

    return [
        { value: "contains", text: "Contains" },
        { value: "=", text: "Equal to" }
    ];
}

function appendOptions($select, options, selectedValue) {
    options.forEach(function (option) {
        const $option = $("<option>")
            .val(option.value)
            .text(option.text);

        if (option.value === selectedValue) {
            $option.prop("selected", true);
        }

        $select.append($option);
    });
}

function displayFilters() {
    const $list = $("#filterList").empty();
    const tableKey = $("#tableSelect").val();
    const table = tables[tableKey];

    filters.forEach(function (filter) {
        const $card = $("<div>").addClass("filter-card");
        const $row = $("<div>").addClass("filter-row");

        const $field = $("<select>").append(
            $("<option>").val("").text("Choose column")
        );

        if (table) {
            table.columns.forEach(function (column) {
                $field.append(
                    $("<option>")
                        .val(tableKey + "." + column.key)
                        .text(column.name)
                );
            });
        }

        $field.val(filter.columnKey);

        const $condition = $("<select>").append(
            $("<option>").val("").text("Choose condition")
        );

        const column = getColumnDetails(filter.columnKey);
        const conditions = column
            ? getFilterConditions(column.type)
            : [];

        appendOptions($condition, conditions, filter.condition);

        const $values = $("<div>").addClass("filter-values");

        if (column && filter.condition) {
            if (column.type === "multi") {
                // Values are loaded from the database schema response.
                (column.options || []).forEach(function (value) {
                    const checked =
                        filter.selectedValues.indexOf(value) !== -1;

                    const $label = $("<label>");
                    const $checkbox = $("<input>", {
                        type: "checkbox",
                        value: value
                    }).prop("checked", checked);

                    $checkbox.on("change", function () {
                        readFilterValues();
                    });

                    $label.append($checkbox);
                    $label.append(" " + value);
                    $values.append($label);
                });

                if (!column.options || column.options.length === 0) {
                    $values.text("No available values.");
                }
            } else {
                const inputType = column.type === "date" ? "date" :
                    column.type === "number" ? "number" : "text";

                const $first = $("<input>", {
                    type: inputType,
                    placeholder: filter.condition === "between"
                        ? "From"
                        : "Enter value",
                    step: column.type === "number" ? "any" : undefined
                }).val(filter.value);

                $first.on("input change", function () {
                    readFilterValues();
                });

                $values.append($first);

                if (filter.condition === "between") {
                    const $second = $("<input>", {
                        type: inputType,
                        placeholder: "To",
                        step: column.type === "number" ? "any" : undefined
                    }).val(filter.value2);

                    $second.on("input change", function () {
                        readFilterValues();
                    });

                    $values.append($second);
                }
            }
        }

        $field.on("change", function () {
            filter.columnKey = $(this).val();
            filter.condition = "";
            filter.value = "";
            filter.value2 = "";
            filter.selectedValues = [];
            displayFilters();
        });

        $condition.on("change", function () {
            filter.condition = $(this).val();
            filter.value = "";
            filter.value2 = "";
            filter.selectedValues = [];
            displayFilters();
        });

        const $remove = $("<button>", {
            type: "button",
            text: "Remove"
        }).addClass("danger");

        $remove.on("click", function () {
            filters = filters.filter(function (item) {
                return item.id !== filter.id;
            });
            displayFilters();
        });

        $row.append($field, $condition, $values, $remove);
        $card.append($row);
        $list.append($card);
    });
}

function readFilterValues() {
    // Read the current UI controls back into the filter objects.
    $("#filterList .filter-card").each(function (index) {
        const filter = filters[index];
        if (!filter) return;

        const $card = $(this);
        filter.columnKey = $card.find("select").eq(0).val() || "";
        filter.condition = $card.find("select").eq(1).val() || "";

        const column = getColumnDetails(filter.columnKey);
        filter.value = "";
        filter.value2 = "";
        filter.selectedValues = [];

        if (!column || !filter.condition) return;

        if (column.type === "multi") {
            $card.find(".filter-values input:checked").each(function () {
                filter.selectedValues.push($(this).val());
            });
        } else {
            const $inputs = $card.find(".filter-values input");
            filter.value = $inputs.eq(0).val() || "";

            if (filter.condition === "between") {
                filter.value2 = $inputs.eq(1).val() || "";
            }
        }
    });
}

// ---------------- BUILD AND SEND REPORT REQUEST ----------------

function generateReport() {
    readSelectedFields();
    readFilterValues();

    if (selectedColumns.length === 0) {
        showMessage("Please select at least one column.", true);
        return;
    }

    const requestData = {
        tableName: $("#tableSelect").val(),
        selectedColumns: selectedColumns,
        filters: filters
    };

    // This is the exact object sent to the WebMethod.
    console.log("Request object:", requestData);
    console.log("JSON sent to backend:", JSON.stringify({
        request: requestData
    }));

    $("#generateReport").prop("disabled", true).text("Generating...");
    showMessage("", false);

    $.ajax({
        type: "POST",
        url: "ReportBuilder.aspx/GenerateReport",
        data: JSON.stringify({ request: requestData }),
        contentType: "application/json; charset=utf-8",
        dataType: "json",

        success: function (response) {
            const result = response.d;

            reportColumns = result.columns || [];
            reportRows = result.rows || [];

            displayReport();

            $("#reportSummary").text(
                "Rows returned: " + reportRows.length
            );

            showMessage("Report generated successfully.", false);
        },

        error: function (xhr) {
            showMessage(
                "Report generation failed: " + getAjaxError(xhr), true
            );
        },

        complete: function () {
            $("#generateReport")
                .prop("disabled", false)
                .text("Generate report");
        }
    });
}

// ---------------- DISPLAY RESULT ----------------

function displayReport() {
    const $output = $("#reportOutput").empty();

    if (!reportColumns.length) {
        $output.text("No columns were returned.");
        return;
    }

    const $table = $("<table>");
    const $thead = $("<thead>");
    const $headerRow = $("<tr>");

    reportColumns.forEach(function (column) {
        $headerRow.append($("<th>").text(column.name));
    });

    $thead.append($headerRow);
    $table.append($thead);

    const $tbody = $("<tbody>");

    if (reportRows.length === 0) {
        const $tr = $("<tr>");
        $tr.append(
            $("<td>")
                .attr("colspan", reportColumns.length)
                .text("No records match your filters.")
        );
        $tbody.append($tr);
    } else {
        reportRows.forEach(function (record) {
            const $tr = $("<tr>");

            reportColumns.forEach(function (column) {
                $tr.append(
                    $("<td>").text(
                        record[column.key] == null
                            ? ""
                            : String(record[column.key])
                    )
                );
            });

            $tbody.append($tr);
        });
    }

    $table.append($tbody);
    $output.append($table);
}

// ---------------- CSV EXPORT ----------------

function exportCSV() {
    if (!reportColumns.length) {
        showMessage("Generate a report before exporting.", true);
        return;
    }

    function csvEscape(value) {
        const text = value == null ? "" : String(value);
        return '"' + text.replace(/"/g, '""') + '"';
    }

    const lines = [];

    lines.push(
        reportColumns.map(function (column) {
            return csvEscape(column.name);
        }).join(",")
    );

    reportRows.forEach(function (record) {
        lines.push(
            reportColumns.map(function (column) {
                return csvEscape(record[column.key]);
            }).join(",")
        );
    });

    const blob = new Blob(
        ["\uFEFF" + lines.join("\r\n")],
        { type: "text/csv;charset=utf-8;" }
    );

    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");

    link.href = url;
    link.download = "report.csv";
    link.click();

    URL.revokeObjectURL(url);
}

// ---------------- EVENTS ----------------

$(function () {
    $("#tableSelect").on("change", function () {
        selectedTables = [$(this).val()];
        selectedColumns = [];
        filters = [];
        nextFilterId = 1;

        displayFields();
        displayFilters();
        $("#reportOutput").text("Generate a new report to see results.");
        $("#reportSummary").text("");
    });

    $("#clearColumns").on("click", function () {
        $("#fieldList input[type=checkbox]").prop("checked", false);
        readSelectedFields();
    });

    $("#addFilter").on("click", addFilter);
    $("#generateReport").on("click", generateReport);
    $("#exportCsv").on("click", exportCSV);

    loadSchema();
});
</script>
</form>
</body>
</html>

```

Important: This page intentionally supports one table per report for now. It uses the table's actual column metadata and database-derived options; it doesn't hardcode the column list or filter values into the frontend.

## 5. Backend: `ReportBuilder.aspx.cs`

This is the main backend. It has two WebMethods:

- `GetReportSchema()` — returns the approved table, its columns, types, and available filter values.
- `GenerateReport(request)` — receives the user's selected columns and filters, validates them, builds SQL, executes it, and returns the rows.

The table and column names are checked against a server-side allow-list. Filter values are SQL parameters, not concatenated into the query.

```

using System;
using System.Collections.Generic;
using System.Configuration;
using System.Data;
using System.Data.SqlClient;
using System.Globalization;
using System.Linq;
using System.Text.RegularExpressions;
using System.Web.Script.Services;
using System.Web.Services;

namespace ReportBuilder
{
    public partial class ReportBuilder : System.Web.UI.Page
    {
        // ---------------- DATA MODELS ----------------

        public class ReportColumn
        {
            public string name { get; set; }       // Display label
            public string key { get; set; }        // Frontend key
            public string dbName { get; set; }     // Actual SQL column
            public string type { get; set; }       // number/date/text/multi
            public List<string> options { get; set; }
        }

        public class ReportTable
        {
            public string name { get; set; }
            public string key { get; set; }
            public List<ReportColumn> columns { get; set; }
        }

        public class ReportFilter
        {
            public int id { get; set; }
            public string columnKey { get; set; }
            public string condition { get; set; }
            public string value { get; set; }
            public string value2 { get; set; }
            public List<string> selectedValues { get; set; }
        }

        public class ReportRequest
        {
            public string tableName { get; set; }
            public List<string> selectedColumns { get; set; }
            public List<ReportFilter> filters { get; set; }
        }

        public class ReportResult
        {
            public List<ReportColumn> columns { get; set; }
            public List<Dictionary<string, object>> rows { get; set; }
        }

        private static string ConnectionString
        {
            get
            {
                return ConfigurationManager
                    .ConnectionStrings["ReportBuilderDB"]
                    .ConnectionString;
            }
        }

        // ---------------- TRUSTED REPORT CONFIGURATION ----------------
        //
        // Only tables and columns explicitly listed here are reportable.
        // Never trust table/column identifiers sent by the browser.

        private static ReportTable GetProjectDetailsDefinition()
        {
            return new ReportTable
            {
                key = "projectDetails",
                name = "Project Details",
                columns = new List<ReportColumn>
                {
                    new ReportColumn {
                        name = "Project ID",
                        key = "projectId",
                        dbName = "ProjectId",
                        type = "number"
                    },
                    new ReportColumn {
                        name = "Project Name",
                        key = "projectName",
                        dbName = "ProjectName",
                        type = "text"
                    },
                    new ReportColumn {
                        name = "Start Date",
                        key = "startDate",
                        dbName = "StartDate",
                        type = "date"
                    },
                    new ReportColumn {
                        name = "Cost",
                        key = "cost",
                        dbName = "Cost",
                        type = "number"
                    },
                    new ReportColumn {
                        name = "Category",
                        key = "category",
                        dbName = "Category",
                        type = "multi"
                    },
                    new ReportColumn {
                        name = "Place",
                        key = "place",
                        dbName = "Place",
                        type = "multi"
                    },
                    new ReportColumn {
                        name = "Status",
                        key = "status",
                        dbName = "Status",
                        type = "multi"
                    }
                }
            };
        }

        private static ReportTable GetAllowedTable(string tableKey)
        {
            if (tableKey == "projectDetails")
                return GetProjectDetailsDefinition();

            throw new ArgumentException("This report table is not allowed.");
        }

        private static ReportColumn GetAllowedColumn(
            ReportTable table, string columnKey)
        {
            if (table == null || String.IsNullOrWhiteSpace(columnKey))
                return null;

            // Only a column defined by the server can be used.
            return table.columns.FirstOrDefault(
                c => c.key == columnKey
            );
        }

        // ---------------- WEBMETHOD 1: LOAD SCHEMA ----------------

        [WebMethod]
        [ScriptMethod(ResponseFormat = ResponseFormat.Json)]
        public static List<ReportTable> GetReportSchema()
        {
            ReportTable table = GetAllowedTable("projectDetails");

            // Load actual available values for the multi-select filters.
            foreach (ReportColumn column in table.columns)
            {
                column.options = new List<string>();

                if (column.type != "multi")
                    continue;

                string sql =
                    "SELECT DISTINCT [" + column.dbName + "] " +
                    "FROM dbo.ProjectDetails " +
                    "WHERE [" + column.dbName + "] IS NOT NULL " +
                    "ORDER BY [" + column.dbName + "]";

                using (SqlConnection con =
                    new SqlConnection(ConnectionString))
                using (SqlCommand cmd = new SqlCommand(sql, con))
                {
                    con.Open();

                    using (SqlDataReader reader = cmd.ExecuteReader())
                    {
                        while (reader.Read())
                        {
                            column.options.Add(
                                Convert.ToString(
                                    reader[0],
                                    CultureInfo.InvariantCulture
                                )
                            );
                        }
                    }
                }
            }

            return new List<ReportTable> { table };
        }

        // ---------------- WEBMETHOD 2: GENERATE REPORT ----------------

        [WebMethod]
        [ScriptMethod(ResponseFormat = ResponseFormat.Json)]
        public static ReportResult GenerateReport(ReportRequest request)
        {
            if (request == null)
                throw new ArgumentException("Request data is missing.");

            // Validate the table against the server's allow-list.
            ReportTable table = GetAllowedTable(request.tableName);

            if (request.selectedColumns == null ||
                request.selectedColumns.Count == 0)
            {
                throw new ArgumentException(
                    "Select at least one report column."
                );
            }

            // The frontend sends keys like:
            // projectDetails.projectName
            List<ReportColumn> chosenColumns =
                new List<ReportColumn>();

            foreach (string fullKey in request.selectedColumns)
            {
                string[] parts = (fullKey ?? "").Split('.');

                if (parts.Length != 2 ||
                    parts[0] != table.key)
                {
                    throw new ArgumentException(
                        "An invalid report column was supplied."
                    );
                }

                ReportColumn column =
                    GetAllowedColumn(table, parts[1]);

                if (column == null)
                {
                    throw new ArgumentException(
                        "A selected column is not allowed."
                    );
                }

                if (!chosenColumns.Any(c => c.key == column.key))
                    chosenColumns.Add(column);
            }

            // SELECT identifiers come only from trusted server config.
            List<string> selectParts = chosenColumns.Select(c =>
                "[" + c.dbName + "] AS [" + c.key + "]"
            ).ToList();

            string sql =
                "SELECT " + String.Join(", ", selectParts) +
                " FROM dbo.ProjectDetails WHERE 1 = 1";

            List<SqlParameter> parameters =
                new List<SqlParameter>();

            int parameterIndex = 0;

            // Each filter is joined with AND.
            if (request.filters != null)
            {
                foreach (ReportFilter filter in request.filters)
                {
                    if (filter == null ||
                        String.IsNullOrWhiteSpace(filter.columnKey))
                    {
                        continue; // Ignore an unused filter row.
                    }

                    string[] parts = filter.columnKey.Split('.');

                    if (parts.Length != 2 ||
                        parts[0] != table.key)
                    {
                        throw new ArgumentException(
                            "A filter column is invalid."
                        );
                    }

                    ReportColumn column =
                        GetAllowedColumn(table, parts[1]);

                    if (column == null)
                    {
                        throw new ArgumentException(
                            "A filter column is not allowed."
                        );
                    }

                    string dbColumn = "[" + column.dbName + "]";
                    string condition = filter.condition ?? "";

                    // Validate the operator for this column type.
                    bool validCondition =
                        (column.type == "number" &&
                            new[] { ">", "<", "=", "between" }
                                .Contains(condition))
                        ||
                        (column.type == "date" &&
                            condition == "between")
                        ||
                        (column.type == "multi" &&
                            condition == "in")
                        ||
                        (column.type == "text" &&
                            new[] { "=", "contains" }
                                .Contains(condition));

                    if (!validCondition)
                    {
                        throw new ArgumentException(
                            "The filter condition is invalid for " +
                            column.name + "."
                        );
                    }

                    // NUMBER and TEXT: =, >, <, contains
                    if (condition == ">" ||
                        condition == "<" ||
                        condition == "=" ||
                        condition == "contains")
                    {
                        if (String.IsNullOrWhiteSpace(filter.value))
                        {
                            throw new ArgumentException(
                                "Enter a value for " + column.name + "."
                            );
                        }

                        string parameterName =
                            "@p" + parameterIndex++;

                        if (condition == "contains")
                        {
                            sql += " AND " + dbColumn +
                                   " LIKE " + parameterName;

                            parameters.Add(new SqlParameter(
                                parameterName,
                                SqlDbType.NVarChar, 4000)
                            {
                                Value = "%" + filter.value + "%"
                            });
                        }
                        else
                        {
                            sql += " AND " + dbColumn + " " +
                                   condition + " " + parameterName;

                            parameters.Add(
                                CreateTypedParameter(
                                    parameterName, column, filter.value
                                )
                            );
                        }
                    }

                    // NUMBER or DATE: BETWEEN value AND value2
                    else if (condition == "between")
                    {
                        if (String.IsNullOrWhiteSpace(filter.value) ||
                            String.IsNullOrWhiteSpace(filter.value2))
                        {
                            throw new ArgumentException(
                                "Enter both From and To values for " +
                                column.name + "."
                            );
                        }

                        object from = ParseValue(column, filter.value);
                        object to = ParseValue(column, filter.value2);

                        if (ComparerFor(column, from, to) > 0)
                        {
                            throw new ArgumentException(
                                "The From value cannot exceed the To value " +
                                "for " + column.name + "."
                            );
                        }

                        string p1 = "@p" + parameterIndex++;
                        string p2 = "@p" + parameterIndex++;

                        sql += " AND " + dbColumn +
                               " BETWEEN " + p1 + " AND " + p2;

                        parameters.Add(
                            CreateTypedParameter(p1, column, filter.value)
                        );
                        parameters.Add(
                            CreateTypedParameter(p2, column, filter.value2)
                        );
                    }

                    // MULTI-SELECT: IN (@p0, @p1, @p2)
                    else if (condition == "in")
                    {
                        List<string> values =
                            filter.selectedValues ?? new List<string>();

                        // No selections means no restriction from this filter.
                        if (values.Count == 0)
                            continue;

                        // Values must be options belonging to this column.
                        // This prevents a browser from inventing categories.
                        List<string> allowedValues =
                            GetDistinctValues(table, column);

                        foreach (string value in values)
                        {
                            if (!allowedValues.Contains(value))
                            {
                                throw new ArgumentException(
                                    "An invalid value was supplied for " +
                                    column.name + "."
                                );
                            }
                        }

                        List<string> placeholders =
                            new List<string>();

                        foreach (string value in values.Distinct())
                        {
                            string p = "@p" + parameterIndex++;
                            placeholders.Add(p);

                            parameters.Add(new SqlParameter(
                                p, SqlDbType.NVarChar, 4000)
                            {
                                Value = value
                            });
                        }

                        if (placeholders.Count > 0)
                        {
                            sql += " AND " + dbColumn +
                                   " IN (" +
                                   String.Join(", ", placeholders) + ")";
                        }
                    }
                }
            }

            // Execute the validated query.
            ReportResult result = new ReportResult
            {
                columns = chosenColumns.Select(c => new ReportColumn
                {
                    name = c.name,
                    key = c.key,
                    type = c.type
                }).ToList(),

                rows = new List<Dictionary<string, object>>()
            };

            using (SqlConnection con =
                new SqlConnection(ConnectionString))
            using (SqlCommand cmd = new SqlCommand(sql, con))
            {
                cmd.CommandType = CommandType.Text;
                cmd.CommandTimeout = 30;

                foreach (SqlParameter parameter in parameters)
                    cmd.Parameters.Add(parameter);

                con.Open();

                using (SqlDataReader reader = cmd.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        Dictionary<string, object> row =
                            new Dictionary<string, object>();

                        foreach (ReportColumn column in chosenColumns)
                        {
                            object value = reader[column.key];

                            if (value == DBNull.Value)
                            {
                                row[column.key] = null;
                            }
                            else if (value is DateTime)
                            {
                                row[column.key] =
                                    ((DateTime)value).ToString(
                                        "yyyy-MM-dd",
                                        CultureInfo.InvariantCulture
                                    );
                            }
                            else
                            {
                                row[column.key] = value;
                            }
                        }

                        result.rows.Add(row);
                    }
                }
            }

            return result;
        }

        // ---------------- SAFE VALUE PARSING ----------------

        private static object ParseValue(
            ReportColumn column, string value)
        {
            if (column.type == "number")
            {
                decimal number;

                if (!Decimal.TryParse(
                    value,
                    NumberStyles.Number,
                    CultureInfo.InvariantCulture,
                    out number))
                {
                    throw new ArgumentException(
                        "Enter a valid number for " + column.name + "."
                    );
                }

                return number;
            }

            if (column.type == "date")
            {
                DateTime date;

                if (!DateTime.TryParseExact(
                    value,
                    "yyyy-MM-dd",
                    CultureInfo.InvariantCulture,
                    DateTimeStyles.None,
                    out date))
                {
                    throw new ArgumentException(
                        "Enter a valid date for " + column.name + "."
                    );
                }

                return date;
            }

            return value;
        }

        private static SqlParameter CreateTypedParameter(
            string name, ReportColumn column, string value)
        {
            if (column.type == "number")
            {
                return new SqlParameter(name, SqlDbType.Decimal)
                {
                    Precision = 12,
                    Scale = 2,
                    Value = ParseValue(column, value)
                };
            }

            if (column.type == "date")
            {
                return new SqlParameter(name, SqlDbType.Date)
                {
                    Value = ParseValue(column, value)
                };
            }

            return new SqlParameter(
                name, SqlDbType.NVarChar, 4000)
            {
                Value = value
            };
        }

        private static int ComparerFor(
            ReportColumn column, object from, object to)
        {
            if (column.type == "number")
                return ((decimal)from).CompareTo((decimal)to);

            if (column.type == "date")
                return ((DateTime)from).CompareTo((DateTime)to);

            return 0;
        }

        private static List<string> GetDistinctValues(
            ReportTable table, ReportColumn column)
        {
            // Both identifiers come from trusted server configuration.
            string sql =
                "SELECT DISTINCT [" + column.dbName + "] " +
                "FROM dbo.ProjectDetails " +
                "WHERE [" + column.dbName + "] IS NOT NULL";

            List<string> values = new List<string>();

            using (SqlConnection con =
                new SqlConnection(ConnectionString))
            using (SqlCommand cmd = new SqlCommand(sql, con))
            {
                con.Open();

                using (SqlDataReader reader = cmd.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        values.Add(Convert.ToString(
                            reader[0],
                            CultureInfo.InvariantCulture
                        ));
                    }
                }
            }

            return values;
        }
    }
}

```

## 6. What exactly gets sent from frontend to backend?

Suppose the user selects these four columns:

- Project Name
- Cost
- Place
- Status

And adds two filters:

1. Cost is greater than `200000`.
2. Place is either `Pune` or `Mumbai`.

The frontend builds this JavaScript object:

```
const requestData = {
    tableName: "projectDetails",

    selectedColumns: [
        "projectDetails.projectName",
        "projectDetails.cost",
        "projectDetails.place",
        "projectDetails.status"
    ],

    filters: [
        {
            id: 1,
            columnKey: "projectDetails.cost",
            condition: ">",
            value: "200000",
            value2: "",
            selectedValues: []
        },
        {
            id: 2,
            columnKey: "projectDetails.place",
            condition: "in",
            value: "",
            value2: "",
            selectedValues: ["Pune", "Mumbai"]
        }
    ]
};
```

Then this line converts it to JSON and sends it through AJAX:

```
data: JSON.stringify({ request: requestData })
```

The `{ request: requestData }` wrapper is important because the C# WebMethod accepts a parameter named `request`.

### How C# receives it

ASP.NET automatically deserializes the JSON into your `ReportRequest` object.

| JSON property     | C# property               | What it contains                 |
| ----------------- | ------------------------- | -------------------------------- |
| `tableName`       | `request.tableName`       | `projectDetails`                 |
| `selectedColumns` | `request.selectedColumns` | List of selected column keys     |
| `filters`         | `request.filters`         | List of filter objects           |
| `columnKey`       | `filter.columnKey`        | Column the filter applies to     |
| `condition`       | `filter.condition`        | `>`, `between`, `in`, etc.       |
| `value`           | `filter.value`            | First filter value               |
| `value2`          | `filter.value2`           | Second value for a range         |
| `selectedValues`  | `filter.selectedValues`   | Selected values for multi-select |

You don't need to manually read the JSON string or call `JSON.parse()` in C#. The WebMethod parameter binding does this.

## 7. What SQL query does the backend build?

For the example request above, the backend constructs this query:

```
SELECT
    [ProjectName] AS [projectName],
    [Cost]        AS [cost],
    [Place]       AS [place],
    [Status]      AS [status]
FROM dbo.ProjectDetails
WHERE 1 = 1
  AND [Cost] > @p0
  AND [Place] IN (@p1, @p2);
```

The actual parameter values are:

| Parameter | SQL type | Value    |
| --------- | -------- | -------- |
| `@p0`     | Decimal  | `200000` |
| `@p1`     | NVARCHAR | `Pune`   |
| `@p2`     | NVARCHAR | `Mumbai` |

The database receives the filter values separately from the SQL command. This is the correct approach; never concatenate a user's filter text directly into SQL.

Notice how the selected frontend key `projectDetails.projectName` maps to the trusted SQL column `ProjectName`, with `AS [projectName]` ensuring the returned field matches the frontend key.

## 8. What result comes back from SQL Server?

The C# method converts the database results into a JSON-serializable object. ASP.NET wraps the WebMethod result inside `response.d`.

The result is conceptually equivalent to:

```
{
  "d": {
    "columns": [
      {"name": "Project Name", "key": "projectName", "type": "text"},
      {"name": "Cost", "key": "cost", "type": "number"},
      {"name": "Place", "key": "place", "type": "multi"},
      {"name": "Status", "key": "status", "type": "multi"}
    ],
    "rows": [
      {
        "projectName": "Road Development",
        "cost": 250000.00,
        "place": "Pune",
        "status": "Completed"
      },
      {
        "projectName": "Hospital Upgrade",
        "cost": 350000.00,
        "place": "Pune",
        "status": "In Progress"
      },
      {
        "projectName": "Bridge Repair",
        "cost": 550000.00,
        "place": "Mumbai",
        "status": "Pending"
      },
      {
        "projectName": "School Extension",
        "cost": 220000.00,
        "place": "Mumbai",
        "status": "In Progress"
      }
    ]
  }
}
```

The frontend reads the result using:

```
success: function (response) {
    const result = response.d;

    reportColumns = result.columns || [];
    reportRows = result.rows || [];

    displayReport();
}
```

`displayReport()` then creates the table headings and cells from the returned column metadata and rows. The same returned data can also be exported to CSV.

## 9. How the different filter conditions work

| UI selection                   | Filter object           | SQL behavior                    |
| ------------------------------ | ----------------------- | ------------------------------- |
| Cost greater than 200000       | `condition: ">"`        | `Cost > @p0`                    |
| Cost less than 300000          | `condition: "<"`        | `Cost < @p0`                    |
| Cost equal to 250000           | `condition: "="`        | `Cost = @p0`                    |
| Cost between 100000 and 300000 | `condition: "between"`  | `Cost BETWEEN @p0 AND @p1`      |
| Start Date between two dates   | `condition: "between"`  | `StartDate BETWEEN @p0 AND @p1` |
| Place Pune or Mumbai           | `condition: "in"`       | `Place IN (@p0, @p1)`           |
| Project Name contains `School` | `condition: "contains"` | `ProjectName LIKE @p0`          |

All active filters are combined with `AND` in this example. For instance, a row must meet both the cost and place conditions.

## 10. Important setup checks

Before running it, check these details:

1. Namespace: The `.aspx` page's `Inherits="ReportBuilder.ReportBuilder"` must match the namespace and class in your `.aspx.cs` file. If your project uses a different namespace, update both.
2. Connection string: `YOUR_SERVER_NAME` must be replaced with the SQL Server instance where `ReportBuilderDB` exists.
3. Form tag: In the frontend code above, keep only one closing `</form>` — remove the closing form tag immediately before the `<script>` block and leave the final closing tag after the script.
4. Filter completeness: The sample backend rejects missing values for active comparison and range filters. An empty multi-select filter currently imposes no restriction.
5. Database permissions: Give the application account only the database permissions it needs. Keep the reportable table and column allow-list on the server.
6. Errors: The sample propagates validation errors through WebMethod exceptions. For production, add server-side logging and return controlled error messages instead of exposing database details.
7. Multiple tables: This working example supports `ProjectDetails` only. Supporting several tables and selecting columns across them requires explicit table allow-lists and approved join relationships.

One important distinction: The frontend's `GetReportSchema()` call gets its filter options from distinct database values, but the backend's trusted configuration defines which columns and filter types are allowed. That's intentional: actual database values can be dynamic without allowing a browser to request arbitrary database tables or columns.
