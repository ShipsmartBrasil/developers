# Technical Integration Guide

## Access
- Documentation: `https://api.shipsmart.global/docs`
- Base URL: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`

## Overview
- Quote first, then create a shipment.
- V1 is the complete flow.
- V2 is the simplified flow for quote and shipment, with country-only input.
- Shipment is irreversible after creation, except for NF-e and attachments in exceptional cases.
- Tracking returns `timeline` and `status` together.
- Manifest is used when a synchronous shipment flow is needed.

## Topics
- Quote
- Shipment
- Shipment documents
- Tracking
- Pickup scheduling
- Manifest
- Drafts
- Boxes
- Items
- Contracts
