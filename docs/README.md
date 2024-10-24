# Matching tables correctly

While aagerblad's solution is an improvement that allows for Jinja parameters to not cause major syntax highlighting
failures, it is far from perfect, and other issues still remain. In this document I add some notes regarding these
issues.

## Current status

There are two remaining issues, which I explain below.

### FROM UNNEST issue

The second issue is that we don't always find a table name after the `FROM` keyword. In the structure `FROM UNNEST(...)`
the highlighter thinks that `UNNEST(...)` is a table.

![unnest is table](img/02_unnest_is_table.png)

This is especially problematic with expressions that contain characters not considered in the matching strategy. When
this happens, the matching stops, and quotations or backticks can be interpreted as opening ones, and thus cause the
entire rest of the code to be miss-highlighted, until the closing one is found.

![wrong quotations](img/03_wrong_quotations.png)

Breaking the line in which the expression is found would solve this issue. But this is not a desired solution since
there's no other reason why the line should be broken.

![broken line](img/04_broken_line.png)

Even though I've chosen to call this the `FROM UNNEST(...)` issue, this also happens with `JOIN UNNEST(...)`
expressions.

---
This issue is resolved in [4][1] and [5][2].

### Expressions with no backticks

Firstly, the current solution works when the targeted table is surrounded by backticks (`table_name`). The problem is
that when the targeted table is the result of a previous CTE, backticks are not allowed. The following image shows this
problem:

![backticks versus no backticks](img/01_issue_one.png)

## Current regex

Because the second issue is much harder to resolve, let's start by understanding how all of this works.

In order to build a syntax highlighter for VS Code one needs to "simply" provide the `syntaxes` configuration. In it,
one provides patterns to be matched. These patterns need to have capture groups. To each capture group a classification
(`name`) gets assigned. The rest (i.e. applying color to different words and tokens in the code) is not handled by this
extension, but by the VS Code engine.

The pattern in question here is the following one (it includes the solution to problem one):

```json
{
  "$schema": "https://raw.githubusercontent.com/martinring/tmlanguage/master/tmlanguage.json",
  "name": "SQL (BigQuery)",
  "patterns": [
    {
      "captures": {
        "1": {
          "name": "keyword.other.select.sql"
        },
        "4": {
          "name": "entity.name.function.table.standard.sql"
        },
        "5": {
          "name": "invalid.illegal.table.delimiter.sql"
        },
        "6": {
          "name": "variable.parameter.table.partition_decorator.sql"
        },
        "7": {
          "name": "entity.name.function.table.legacy.sql"
        },
        "8": {
          "name": "invalid.illegal.table.delimiter.sql"
        },
        "9": {
          "name": "variable.parameter.table.partition_decorator.legacy.sql"
        }
      },
      "match": "(?i)(from(?!\\s+unnest)|join(?!\\s+unnest)|delete\\s+from|delete(?!\\s+from)|update|using)\\s+((`?)((?:[\\w\\-\\{\\}]+(?:\\.|(\\:)))?(?:[\\w\\{\\}]+\\.)?(?:INFORMATION_SCHEMA\\.\\w+|[\\w\\{\\}\\.\\(\\=\\\"\\'\\-\\,\\s\\)]+))(\\*|\\$\\w+)?\\3?|\\[((?:[\\w\\-\\{\\}]+(?:\\:|(\\.)))?(?:[\\w\\{\\}]+\\.)?[\\w\\{\\}]+)(\\*|\\$\\w+)?\\])",
      "name": "meta.create.from.sql"
    }
  ],
  "repository": {},
  "scopeName": "source.sql-bigquery"
}
```

We can see that this pattern will generate (at least) 9 capturing-groups, 7 of which will be assigned to categories.
The 9 capturing groups are:

| Group | Expression (shortened when needed)                                                            |
|:-----:|-----------------------------------------------------------------------------------------------|
| 1     | `(from(?!\\s+unnest)\|join(?!\\s+unnest)\|delete\\s+from\|delete(?!\\s+from)\|update\|using)` |
| 2     | group that contains 3, 4, 5, 6, 7, 8, and 9 as sub-groups                                     |
| 3     | ``(`?)``                                                                                      |
| 4     | group that contains group 5 as a sub-group, plus more regex expressions                       |
| 5     | `(\\:)`                                                                                       |
| 6     | `(\\*\|\\$\\w+)?`                                                                             |
| 7     | group that contains group 8 as a sub-group, plus more regex expressions                       |
| 8     | `(\\.)`                                                                                       |
| 9     | `(\\*\|\\$\\w+)?`                                                                             |

```txt
(?i)
( <!-- start group 1 -->
    from(?!\\s+unnest)|join(?!\\s+unnest)|delete\\s+from|delete(?!\\s+from)|update|using
) <!-- end group 1 -->
\\s+
( <!-- start group 2 -->
    (`?) <!-- group 3 -->
    ( <!-- start group 4 -->
        (?:
            [\\w\\-\\{\\}]+
            (?:
                \\.
                |
                (\\:) <!-- group 5 -->
            )
        )?
        (?:
            [\\w\\{\\}]+\\.
        )?
        (?:
            INFORMATION_SCHEMA\\.\\w+
            |
            [\\w\\{\\}\\.\\(\\=\\\"\\'\\-\\,\\s\\)]+
        )
    ) <!-- end group 4 -->
    (\\*|\\$\\w+)? <!-- group 6 -->
    \\3?
    |
    \\[
        ( <!-- start group 7 -->
            (?:
                [\\w\\-\\{\\}]+
                (?:
                    \\:
                    |
                    (\\.) <!-- group 8 -->
                )
            )?
            (?:[\\w\\{\\}]+\\.)?
            [\\w\\{\\}]+
        ) <!-- end group 7 -->
        (\\*|\\$\\w+)? <!-- group 9 -->
    \\]
) <!-- end group 2 -->
```

[1]: https://github.com/daczarne/vscode-language-sql-bigquery/pull/4
[2]: https://github.com/daczarne/vscode-language-sql-bigquery/pull/5
