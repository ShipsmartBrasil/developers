# Quotation and Shipping Comparison

- [Back to language README](README.md)

## Overview
- Explains the practical difference between quotation and shipping flows in the two published versions.
- Use this guide to choose the correct version before integrating.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Staging: `https://hmlapi.shipsmart.com.br`

## Authentication
- Bearer token in the `Authorization` header.
- Language header: `lang`.

## Usage Rule
- The standard integration flow is: quotation and shipping using the first version endpoints.
- The second version is recommended when you do not want to provide data beyond the country in the quotation.

## Quotation
### Practical difference
- In the first version, the quotation accepts the full flow with origin, destination, boxes, and items.
- In the second version, the quotation is more streamlined for scenarios where the integrator wants to provide less data at the beginning of the process.

### Comparison
| Topic | First version | Second version | Impact |
|---|---|---|---|
| Objective | Full quotation | Simplified quotation | Choose the level of detail |
| Address | More fields in the contract | Leaner contract | Less input data |
| Items | Full and advanced flow | Explicit flow for items | Better for lean payloads |
| Retrieval | Retrieval and status available in the first version endpoint | Information not identified in the analyzed source | Both operational retrievals remain in the first version |

## Shipping
### Practical difference
- In the first version, shipping covers creation, status retrieval, document retrieval, cancellation, update, and attachments.
- In the second version, the analyzed source shows only shipment creation.

### Comparison
| Topic | First version | Second version | Impact |
|---|---|---|---|
| Creation | Available | Available | Common base flow |
| Status | Available | Information not identified in the analyzed source | Operational tracking depends on the first version |
| Documents | Available | Information not identified in the analyzed source | Document download remains in the first version |
| Cancellation | Available | Information not identified in the analyzed source | Full cycle remains in the first version |
| Update | Available only in exceptional cases | Information not identified in the analyzed source | As a rule, the shipment is irreversible after creation |

## When to Use Each One
1. Use the first version when you want the full operational flow.
2. Use the second version when the initial quotation payload needs to be simpler.
3. If your flow depends on status, documents, or cancellation, keep the first version.
4. If you create through the second version, action tracking is done through the first version APIs.

## Final Notes
- The analyzed source does not provide complete equivalent second version documentation for status, docs, and cancellation.
- For full operation, the first version remains the functional reference.
