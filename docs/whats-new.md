# What's new

[← Documentation index](../README.md#documentation)

Release notes for every version are on the [releases page](https://github.com/gabgiani/dafo-sap-connect-downloads/releases).

## 0.1.0-20261015.5

### Explore the knowledge map from any box

Click any box of the map — a customer, a sales order, a production order, an item — choose the field to search by and enter a value. The gateway reads that box and follows every relationship it can execute, **forwards and backwards**. Each box opens in its own window, placed and connected exactly like the map, with the records SAP returned and expandable line details. Relationships not yet validated on real data are drawn dashed. If a box has no data, its window says why: no read tool, nothing to follow, a failed read or no executable relationship.

![Choose a box, a field and a value](images/knowledge-map-explore.png)

### A clearer SAP query Generator

The Generator now works in three steps — **Describe the query**, **Review the SQL and save it in SAP**, **Test and publish**. Progress, checks and SAP messages appear next to the buttons you just used, the action bar stays visible while you scroll, and the SQL is not shown twice after saving.

![Review the generated SQL before saving it in SAP](images/generator-review.png)

### Field meanings for the assistant

When choosing the fields of a read tool, each field shows its meaning from the SAP database reference (REFDB) or the SAP user-field definition. Administrators can confirm or write their own meanings; the assistant receives them through `describe_sap_tool_fields`.

![Field meanings while choosing the fields of a read tool](images/read-tool-field-meanings.png)

### Also in this release

- Every data tool is an administrator-managed read tool; investigations are published only when all their read tools exist.
- SAP authorization guidance for integration users, including authorization groups and data ownership, and a clear *Read permitted · 0 rows* result when SAP hides records from the user.

*Screenshots use sample data.*
