# Quote and Shipment Comparison

- [Back to English README](README.md)

## Overview
- Explains the practical difference between quote and shipment flows in the two published versions.
- Use this guide to choose the right version before integrating.

## Rule of Use
- The standard integration flow uses the first version endpoints for quote and shipment.
- The second version is recommended when you do not want to send more than country data in the quote.

## Quote
- v1 supports the complete flow with origin, destination, boxes, and items.
- v2 is leaner for integrations that start with less data.

## Shipment
- v1 covers creation, status, documents, cancellation, updates, and attachments.
- v2 exposes only shipment creation in the analyzed source.

## When to Use Each One
1. Use v1 when you need the full operational flow.
2. Use v2 when the initial quote payload must be simpler.
3. If your flow depends on status, documents, or cancellation, keep v1.
4. If you create through v2, follow-up actions happen through v1 APIs.
