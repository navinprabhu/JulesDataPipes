# API Documentation

REST API endpoints for accessing curated datasets.

## Authentication
All endpoints require Azure AD OAuth2 authentication.

## Endpoints

### Customers
- `GET /api/customers/{id}` - Customer 360 profile
- `GET /api/customers/{id}/devices` - Customer devices

### Sales  
- `GET /api/sales/daily` - Daily sales aggregates
- `GET /api/sales/trends` - Sales trend analysis

### Devices
- `GET /api/devices/{id}/status` - Device status and health
- `GET /api/devices/alerts` - Devices requiring attention

For detailed implementation, see [src/api/context.md](../src/api/context.md).