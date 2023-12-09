<!-- markdownlint-disable -->

## Inputs

| Name | Description | Default | Required |
|------|-------------|---------|----------|
| group-by | Group by query | empty | false |
| matrix | Matrix inputs (json array or object with include property passed as string or file path) | N/A | true |
| nested-matrices-count | Number of nested matrices that should be returned as the output (from 1 to 3) | 1 | false |
| sort-by | Sort by query | empty | false |


## Outputs

| Name | Description |
|------|-------------|
| matrix | A matrix suitable for extending matrix size workaround |
<!-- markdownlint-restore -->
