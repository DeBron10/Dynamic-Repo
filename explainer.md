# Understanding your Custom Report Builder — from start to finish

I’ve reviewed your uploaded HTML file. Let's understand it in the same order that the code is written, using your actual variable names and functions.

The most important thing to understand first is that your current code generates reports entirely in the frontend using sample data. It does not yet retrieve the table schema or report rows from SQL Server.

Your page has three main parts:

1\. HTML + CSS

Creates the page, buttons, field checkboxes, filter controls, and report preview.

2\. JavaScript

Stores the sample records, remembers selections, applies filters, and displays or exports the report.

3\. Backend and SQL Server — not connected yet

This is where you will eventually retrieve real table metadata and rows, and execute the report query.

## 1. HTML and CSS: what the user sees

Your HTML is the visual interface. The CSS determines its layout and appearance.

| HTML element / ID         | Purpose                                  |
| ------------------------- | ---------------------------------------- |
| `id="builder"`            | Contains the report-building controls    |
| `id="fieldList"`          | Displays available columns as checkboxes |
| `id="fieldCount"`         | Shows how many fields are selected       |
| `id="filterList"`         | Holds the dynamically generated filters  |
| `id="recordCount"`        | Displays the number of report rows       |
| `id="selectedFieldCount"` | Displays the number of selected columns  |
| `id="activeFilterCount"`  | Displays the number of active filters    |
| `id="reportTable"`        | Displays the report preview              |

For example:

```
<div class="field-grid" id="fieldList"></div>
<div class="filter-list" id="filterList"></div>
```

These two containers are initially empty. JavaScript creates the checkboxes and filter controls and inserts them into these containers.

When you click the buttons, HTML calls JavaScript functions through attributes such as:

```
<button onclick="addFilter()">+ Add Filter</button>

<button onclick="generateReport()">Generate Report</button>
```

`onclick` means that when the button is clicked, the named JavaScript function runs.

## 2. Step 1: the `tables` object — your sample database

Your file defines this structure around lines 713–850:

```
var tables = {
    projectDetails: {
        name: "Project Details",

        columns: [
            {
                name: "Project ID",
                key: "projectId",
                type: "number"
            },
            {
                name: "Cost",
                key: "cost",
                type: "number"
            },
            {
                name: "Place",
                key: "place",
                type: "multi"
            }
        ],

        rows: [
            {
                projectId: 101,
                cost: 50000,
                place: "Pune"
            }
        ]
    }
};
```

This is a simplified illustration of the structure in your file. Your actual file contains seven columns and eight sample records.

Think of it as a miniature database created inside JavaScript.

`tables`

`projectDetails`

The internal table key

`name`

Display name: Project Details

`columns`

Metadata describing each column, its label, key and type.

`rows`

The actual sample records used by your current report engine.

Notice the difference between these three properties:

- `name`: the label shown to the user.
- `key`: the actual property name used to access a value in JavaScript.
- `type`: tells the filtering logic which kind of control to display.

For example, `type: "number"` makes the UI offer numeric comparisons, while `type: "multi"` makes it offer a multi-select list.

This object is hardcoded. When you connect the application to SQL Server, the backend should provide the allowed table/column metadata and the report data should come from the database instead of `rows`.

## 3. Step 2: understand the global variables

Your code declares these variables around lines 858–868:

```
var selectedTables = ["projectDetails"];

var selectedColumns = [];

var filters = [];

var nextFilterId = 1;

var reportRows = [];

var reportColumns = [];
```

These variables act as the memory of the report builder while the user interacts with the page.

| Variable          | What it stores                   | Example                                                 |
| ----------------- | -------------------------------- | ------------------------------------------------------- |
| `selectedTables`  | Tables being used                | `["projectDetails"]`                                    |
| `selectedColumns` | Columns the user checked         | `["projectDetails.projectName", "projectDetails.cost"]` |
| `filters`         | All configured filter conditions | Cost greater than 50,000                                |
| `nextFilterId`    | The ID to give the next filter   | `3`                                                     |
| `reportRows`      | Records that matched the filters | The records included in the report                      |
| `reportColumns`   | Columns to display in the result | Project Name, Cost                                      |

Let's see how these variables change as you use the application.

INTERACTIVE EXAMPLE

A. User selects columns

Project Name

Cost

Place

Status

Equivalent `selectedColumns` array:

[   "projectDetails.projectName",   "projectDetails.cost" ]

B. User configures a cost filter

Greater than

Equivalent filter object:

{   "id": 1,   "columnKey": "projectDetails.cost",   "condition": ">",   "value": "50000",   "value2": "",   "selectedValues": [] }

C. The backend request you could eventually send

{   "selectedTables": [     "projectDetails"   ],   "selectedColumns": [     "projectDetails.projectName",     "projectDetails.cost"   ],   "filters": [     {       "columnKey": "projectDetails.cost",       "condition": ">",       "value": "50000",       "value2": "",       "selectedValues": []     }   ] }

This example illustrates the kind of information your current frontend can collect. Your current `generateReport()` function does not send this JSON to a backend yet.

## 4. Step 3: `displayFields()` — create the field checkboxes

This function starts around line 875.

Its job is to read `tables` and build the list of fields the user can select.

The important logic is:

```
var tableKey = selectedTables[tableIndex];

var currentTable = tables[tableKey];
```

Here's what happens:

1. `selectedTables[tableIndex]` retrieves the table key, `projectDetails`.
2. `tables[tableKey]` retrieves that table's configuration.
3. The function loops through `currentTable.columns`.
4. It creates a checkbox for each column.
5. It inserts each checkbox into `fieldList`.

### Why does it build `fullColumnKey`?

```
var fullColumnKey = tableKey + "." + column.key;
```

If `tableKey` is `projectDetails` and `column.key` is `cost`, the result is:

```
projectDetails.cost
```

This combined key tells the application both the table and the column.

The checkbox receives that key as its value:

```
checkbox.value = fullColumnKey;
```

When the user checks a box, the `onchange` event calls:

```
readSelectedFields();
```

That function updates the `selectedColumns` array.

The line below also restores previously selected checkboxes whenever the fields are redrawn:

```
checkbox.checked = isFieldSelected(fullColumnKey);
```

That is why the selection can persist when the UI is rebuilt.

## 5. Step 4: `isFieldSelected()` and `readSelectedFields()`

### `isFieldSelected(fieldKey)`

This function loops through `selectedColumns` and checks whether the supplied key is already in the array.

It returns:

- `true` if the field was selected.
- `false` if it was not selected.

### `readSelectedFields()`

This function reads the current checkboxes from the page:

```
selectedColumns = [];

var checkboxes = fieldList.getElementsByTagName("input");

for (var i = 0; i < checkboxes.length; i++) {
    if (checkboxes[i].checked === true) {
        selectedColumns.push(checkboxes[i].value);
    }
}
```

The first line clears the old selection. The loop then rebuilds it from the checkboxes that are currently checked.

For example, if the user checks Project Name and Cost, the array becomes:

```
[
    "projectDetails.projectName",
    "projectDetails.cost"
]
```

Finally, `updateFieldCount()` updates the field counter displayed on the page.

The `clearFieldSelection()` function simply empties the array and calls `displayFields()` again to redraw the checkboxes as unchecked.

## 6. Step 5: `getColumnDetails()` — find a column's metadata

This function is important because different fields need different filter controls.

Suppose it receives:

```
getColumnDetails("projectDetails.cost");
```

It splits the key:

```
var parts = fullColumnKey.split(".");

var tableKey = parts[0];
var columnKey = parts[1];
```

The resulting values are:

```
tableKey  = "projectDetails"
columnKey = "cost"
```

Then it looks up the table:

```
var currentTable = tables[tableKey];
```

Finally, it loops through that table's `columns` array until it finds a column whose `key` matches `cost`.

It returns the entire column object, such as:

```
{
    name: "Cost",
    key: "cost",
    type: "number"
}
```

This means other functions can determine the type of a field without hardcoding a separate lookup for every individual column.

If the table or column cannot be found, the function returns `null`.

## 7. Step 6: `getAvailableValues()` — populate Category, Place and Status

Your multi-select filter needs to show the distinct values available in the data.

For example, the sample data has places such as Pune, Mumbai, Nashik and Nagpur.

The function:

```
getAvailableValues("projectDetails.place");
```

does the following:

1. Splits the combined key into table and column names.
2. Retrieves the table's `rows`.
3. Loops through each record.
4. Reads that record's value for the requested column.
5. Checks whether the value has already been added.
6. Adds only new values.
7. Sorts the resulting list and returns it.

For the sample Place data, the result is:

```
["Mumbai", "Nagpur", "Nashik", "Pune"]
```

The `displayFilters()` function then turns these values into options in a multi-select control.

Important: this function reads from `tables[tableKey].rows`, which is currently your hardcoded sample data. Once connected to SQL Server, the backend should supply the available filter values, or the report query can use database-backed lookup values.

## 8. Step 7: `addFilter()` — add a new filter row

When the user clicks + Add Filter, this HTML runs:

```
<button onclick="addFilter()">+ Add Filter</button>
```

The `addFilter()` function creates a new filter object:

```
var newFilter = {
    id: nextFilterId,
    columnKey: "",
    condition: "=",
    value: "",
    value2: "",
    selectedValues: []
};
```

Let's understand each property.

| Property         | Purpose                                           |
| ---------------- | ------------------------------------------------- |
| `id`             | Unique identifier for this filter row             |
| `columnKey`      | The selected field, such as `projectDetails.cost` |
| `condition`      | Comparison, such as `>`, `=`, or `between`        |
| `value`          | The first input value                             |
| `value2`         | The second value for a range                      |
| `selectedValues` | Selected items for Category, Place or Status      |

For a Cost filter between 40,000 and 80,000, the object might be:

```
{
    id: 1,
    columnKey: "projectDetails.cost",
    condition: "between",
    value: "40000",
    value2: "80000",
    selectedValues: []
}
```

Then:

```
nextFilterId = nextFilterId + 1;
filters.push(newFilter);
displayFilters();
```

- The first line increments the next ID, so the next filter can have a different ID.
- `filters.push(newFilter)` stores the new filter in the array.
- `displayFilters()` redraws the filter interface.

Notice that the object is stored separately from the HTML controls. The object is the application's data; the HTML is how the user edits that data.

## 9. Step 8: `displayFilters()` — dynamically create the filter controls

This is one of the most important functions in your project. It begins around line 1152.

Its job is to create a filter row for every object in the `filters` array.

Each row contains:

1. A field dropdown.
2. A condition dropdown.
3. One or more value inputs.
4. A remove button.

### A. The field dropdown

The function loops through every selected table and every available column to build options such as:

```
Choose a field
Project ID
Project Name
Start Date
Cost
Category
Place
Status
```

Each option's value is a combined table/column key:

```
option.value = tableKey + "." + column.key;
```

The user sees a friendly label, but the program stores a key it can use to find the field later.

When the user changes the field, the code executes:

```
readFilterValues();
displayFilters();
```

This saves the existing values before redrawing the controls for the newly selected field.

### B. The condition dropdown

The code calls `getColumnDetails()` to find the selected column's type, then displays the appropriate conditions.

| Field type | Available conditions                       |
| ---------- | ------------------------------------------ |
| `number`   | Greater than, Less than, Equal to, Between |
| `date`     | Between                                    |
| `multi`    | Is any of                                  |
| `text`     | Contains, Equal to                         |

For example, when Cost is selected, the dropdown offers `>`, `<`, `=`, and `between`.

When Place is selected, the dropdown offers `in`, displayed as “Is any of.”

The `addCondition()` helper creates each option and inserts it into the condition dropdown.

### C. The value input

After identifying the field type and condition, the function creates the appropriate input.

Number — Cost

For `>`, `<`, or `=`, it creates one numeric input. For `between`, it creates two inputs: minimum and maximum.

Date — Start Date

It creates two date inputs to represent the start and end of a date range.

Multi-select — Category, Place, Status

It creates a multiple-selection dropdown populated using `getAvailableValues()`.

Text — Project Name

It creates a text input for Contains or Equal to.

Each input gets an ID tied to the filter's ID. For example:

```
numberInput.id = "filterValue_" + filter.id;
```

If the filter ID is `2`, the resulting HTML ID is `filterValue_2`.

This lets the program read the correct input for the correct filter, even when multiple filters exist.

### D. The remove button

Each filter row has an `X` button. It stores the filter ID in a `data-filter-id` attribute.

When clicked, it calls:

```
removeFilter(filterId);
```

This removes the matching object from the `filters` array and redraws the filter list.

## 10. Step 9: `readFilterValues()` — save what the user entered

When the user edits a filter, the visible inputs must be copied into the corresponding object in the `filters` array.

That is what `readFilterValues()` does, starting around line 1578.

For every filter object, it looks up the corresponding controls using their IDs:

```
var fieldSelect = document.getElementById(
    "filterField_" + filter.id
);

var conditionSelect = document.getElementById(
    "filterCondition_" + filter.id
);
```

It then updates the object's properties:

```
filter.columnKey = fieldSelect.value;
filter.condition = conditionSelect.value;
```

The function also reads:

- `filterValue_ID` into `filter.value`.
- `filterValue2_ID` into `filter.value2`.
- Selected options in `filterValues_ID` into `filter.selectedValues`.

For a Place filter where the user selects Pune and Nashik, the resulting object could be:

```
{
    id: 1,
    columnKey: "projectDetails.place",
    condition: "in",
    value: "",
    value2: "",
    selectedValues: ["Pune", "Nashik"]
}
```

The function calls `updateSummary()` at the end so the active-filter count can be refreshed.

Why is this important? When you eventually send a report request to your backend, you want to send the filter objects containing the user's latest selections—not just the original empty objects.

## 11. Step 10: `removeFilter()` and `updateSummary()`

### `removeFilter(filterId)`

The function creates a new array, copies every filter except the one whose ID matches `filterId`, assigns that new array to `filters`, and calls `displayFilters()`.

For example, if `filters` contains IDs 1, 2 and 3, removing ID 2 leaves IDs 1 and 3.

### `updateSummary()`

This function updates the summary counters shown above the report preview.

It counts a filter as active only when it has a selected field and a sufficiently complete value:

- For `in`, at least one selected value is required.
- For `between`, both range values are required.
- For other conditions, the first value must be nonempty.

It then writes the count into `activeFilterCount`.

This counter is for display purposes. It does not execute the filters or query the database.

## 12. Step 11: `recordMatchesFilter()` — decide whether a record passes a filter

This function is the core of your current filtering engine. It takes two arguments:

```
recordMatchesFilter(record, filter);
```

- `record` is one sample data object.
- `filter` is one configured filter object.

It retrieves the relevant column from the record, then applies logic based on the condition and field type.

### Example: Cost greater than 50,000

Suppose the record is:

```
{
    projectId: 102,
    projectName: "Office Renovation",
    cost: 85000,
    place: "Mumbai"
}
```

And the filter is:

```
{
    columnKey: "projectDetails.cost",
    condition: ">",
    value: "50000",
    value2: "",
    selectedValues: []
}
```

The function converts the record's cost and the input value to numbers:

```
var numberValue = Number(recordValue);
var firstNumber = Number(filter.value);
```

Then evaluates:

```
return numberValue > firstNumber;
```

Since 85,000 is greater than 50,000, it returns `true`.

### What each filter does

| Condition  | Logic                                                       |
| ---------- | ----------------------------------------------------------- |
| `>`        | Record value is greater than the input                      |
| `<`        | Record value is less than the input                         |
| `=`        | Record value equals the input                               |
| `between`  | Record value is within both bounds, including the endpoints |
| `in`       | Record value matches one of the selected options            |
| `contains` | Text contains the search string, ignoring case              |

For a date range, your code compares the date strings. This works for the ISO-style dates used in your sample rows, such as `2026-03-05`, provided the values use a consistent format.

An important behaviour in your current code: an empty filter input generally returns `true`, meaning it does not restrict the record. The function does not treat an unfinished filter as a reason to reject all records.

## 13. Step 12: `recordMatchesAllFilters()` — apply every filter

A record might pass one filter but fail another. Your code combines applicable filters using AND logic.

For example, the user chooses:

- Cost greater than 50,000.
- Place is any of Pune or Nashik.

A record must satisfy both conditions to be included.

| Project             | Cost    | Place  | Included?                             |
| ------------------- | ------- | ------ | ------------------------------------- |
| Website Development | ₹50,000 | Pune   | No — cost is not greater than ₹50,000 |
| Office Renovation   | ₹85,000 | Mumbai | No — place doesn't match              |
| Mobile Application  | ₹70,000 | Pune   | Yes                                   |
| Road Repair         | ₹40,000 | Nashik | No — cost is too low                  |

The function loops through `filters` and calls:

```
if (recordMatchesFilter(record, filter) === false) {
    return false;
}
```

As soon as an applicable filter fails, the record is rejected. If all applicable filters pass, the function returns `true`.

Your code also skips filters that belong to a different table key. That's useful as a basic separation mechanism, but it does not implement joins between tables. Your current sample uses one table, `projectDetails`.

## 14. Step 13: `generateReport()` — the main report-generation function

This function runs when the user clicks Generate Report.

It coordinates the earlier functions in a specific order.

1. Read selections

   Calls `readSelectedFields()` and `readFilterValues()` to collect the latest UI values.
2. Validate input

   Ensures at least one column is selected. It also checks that numeric ranges and date ranges aren't reversed.
3. Prepare output arrays

   Clears `reportRows` and `reportColumns`, then copies the selected column keys into `reportColumns`.
4. Loop through records

   It iterates over each selected table's `rows` array.
5. Apply the filters

   Calls `recordMatchesAllFilters()` for each record. Only records that pass are added to `reportRows`.
6. Render the report

   Calls `displayReport()`, updates the report description, and hides the builder.

The crucial part is this loop in your current code:

```
for (rowIndex = 0;
     rowIndex < currentTable.rows.length;
     rowIndex++) {

    var record = currentTable.rows[rowIndex];

    if (recordMatchesAllFilters(tableKey, record) === true) {
        reportRows.push({
            tableKey: tableKey,
            data: record
        });
    }
}
```

The data is coming from:

```
currentTable.rows
```

That is the reason your report currently works without a database connection: the sample records are already stored in JavaScript.

### What needs to change for the real backend?

The frontend will still collect the selected fields and filters. But instead of looping through `currentTable.rows`, `generateReport()` will send a request to your ASP.NET WebMethod.

The backend will validate the request, execute SQL Server queries, and return the result rows. The frontend will then call `displayReport()` to render those returned rows.

So you can retain much of your UI and report-rendering code while replacing the sample-data processing.

## 15. Step 14: `displayReport()` — render the result table

This function takes the values stored in `reportColumns` and `reportRows` and constructs the HTML for the result table.

### A. Create column headings

It loops through `reportColumns` and calls `getColumnDetails()` to find each column's friendly name.

For example, the internal key:

```
projectDetails.projectName
```

becomes the heading:

```
Project Name
```

### B. Create data cells

For every report record, it loops through the selected columns and retrieves the corresponding values.

Your code checks the table key before reading a value:

```
if (columnTableKey === reportRecord.tableKey) {
    cellValue = reportRecord.data[columnKey];
}
```

This is sufficient for your current single-table example. With joined tables, the response format and cell lookup logic would need to support values from multiple tables or, preferably, return flat report rows with one property per selected column.

### C. Style the Status column

If the column key is `status`, your code applies a CSS class:

- `Completed` gets the green style.
- `In Progress` gets the amber style.
- Other statuses get the default style.

This is presentation logic; it doesn't affect filtering.

### D. Handle no matching records

If `reportRows.length === 0`, the function displays “No records found.”

Finally, it inserts the completed table into the page:

```
reportTable.innerHTML = headingHTML + bodyHTML;
```

Your code uses `escapeHTML()` on headings and values before inserting them. That helps prevent text from being interpreted as HTML.

## 16. Step 15: `escapeHTML()` — display text safely

The function replaces characters that have special meaning in HTML:

```
result = result.replace(/&/g, "&amp;");
result = result.replace(/</g, "&lt;");
result = result.replace(/>/g, "&gt;");
result = result.replace(/"/g, "&quot;");
result = result.replace(/'/g, "&#39;");
```

For example, a project name containing:

```
Research <Development>
```

is converted to HTML-safe text so the browser displays the angle brackets instead of interpreting the value as markup.

This is useful whenever you display data from a database or user input. It is not a substitute for validating input or protecting SQL queries on the backend.

## 17. Step 16: `toggleBuilder()` and `resetBuilder()`

### `toggleBuilder()`

This function shows or hides the report builder:

```
builder.classList.toggle("hidden");
```

Your CSS defines `.builder.hidden` as `display: none`, so the class controls whether the builder is visible.

### `resetBuilder()`

This function restores the report builder to its initial state. It:

- Selects the default table again.
- Clears the selected columns.
- Removes all filters.
- Resets the next filter ID.
- Clears the report rows and columns.
- Shows the builder.
- Restores the initial report-preview message.
- Updates the counters.

It does not delete or modify anything in a database. At present, all these operations affect only the browser's JavaScript state and HTML.

## 18. Step 17: `exportCSV()` — download the report

This function exports the report currently held in `reportColumns` and `reportRows`.

The process is:

1. Check that a report has been generated.
2. Build the CSV header from the selected column names.
3. Loop through the report records and build each CSV row.
4. Escape double quotes by doubling them, and wrap each value in double quotes.
5. Create a `Blob` with the CSV contents.
6. Create a temporary download link and trigger it.

For example, a report containing Project Name and Cost might export as:

```
"Project Name","Cost"
"Website Development","50000"
"Mobile Application","70000"
```

The CSV is generated in the browser. It does not require a backend export method, provided the report data is already present in `reportRows`.

If you later generate the report from SQL Server, you can still use this function, as long as you populate the frontend report arrays with the returned results.

## 19. Step 18: initialize the page

At the bottom of your script, you have:

```
displayFields();

displayFilters();

updateSummary();
```

These functions run when the script executes and initialize the interface:

- `displayFields()` draws the available field checkboxes.
- `displayFilters()` draws any existing filter rows; initially, there are none.
- `updateSummary()` initializes the summary counters.

Because your current `tables` object is already hardcoded, these functions can run immediately.

Once your table schema comes from an AJAX request, you must initialize the field UI after the schema request succeeds. Otherwise, `displayFields()` might run before `tables` has been populated.

## 20. How everything connects together

Here is the complete flow of your current application:

Page loads

`tables` already contains sample metadata and records

`displayFields()`

Creates field checkboxes

User selects fields and filters

`readSelectedFields()` and `readFilterValues()` update the arrays

`generateReport()`

Validates the inputs and loops through sample records

`recordMatchesAllFilters()`

Decides which records match every applicable condition

`displayReport()`

Renders the selected columns and matching records

`exportCSV()`

Downloads the generated report if requested

## 21. What changes when you connect SQL Server?

You don't have to rebuild the whole interface. The main change is where the data comes from and where the filtering happens.

| Current implementation                           | Database-connected implementation                             |
| ------------------------------------------------ | ------------------------------------------------------------- |
| `tables` is hardcoded                            | Load approved table/column metadata using `GetReportSchema()` |
| `rows` contains sample records                   | `GenerateReport()` returns actual SQL query results           |
| `getAvailableValues()` reads sample rows         | Retrieve filter options from SQL Server                       |
| `generateReport()` filters records in JavaScript | `generateReport()` sends a JSON request using AJAX            |
| `displayReport()` renders local results          | `displayReport()` renders results returned by the backend     |
| `exportCSV()` exports local report arrays        | Keep it, populating the arrays with backend results           |

The new report-generation flow will look like this:

Frontend collects selections

`selectedTables`, `selectedColumns`, `filters`

AJAX + `JSON.stringify()`

Sends the report request to `GenerateReport()`

C# WebMethod

Validates tables, columns, and filter conditions

SQL Server

Executes a validated, parameterized query

JSON response → `displayReport()`

Displays actual database results

A final important point: your current code supports selecting columns and filtering records from one sample table. It doesn't yet implement real database joins, database-driven filter options, or AJAX report generation.

My recommended next step: keep the HTML, CSS, and filter-building functions. First replace the hardcoded column metadata with `GetReportSchema()`. Then replace the sample-row loop inside `generateReport()` with an AJAX call to `GenerateReport()`. This lets you connect the backend incrementally without rewriting your entire project.
