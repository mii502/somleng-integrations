# somleng-integrations - Claude Code Context

## What This Is

Collection of integration plugins that extend Somleng functionality. Currently contains the Inventory Manager for automated phone number ordering.

**Upstream:** https://github.com/somleng/somleng-integrations
**Fork:** https://github.com/mii502/somleng-integrations

## Inventory Manager

Automated DID (phone number) ordering system that maintains stock levels by ordering from suppliers.

### How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                  INVENTORY MANAGER                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Scheduled Runner (cron/Lambda)                            │
│       │                                                     │
│       │ 1. Check current stock                              │
│       ▼                                                     │
│   ┌─────────────────────────────────────────┐              │
│   │         Somleng Carrier API             │              │
│   │   GET /carrier/phone_numbers?available  │              │
│   └─────────────────────────────────────────┘              │
│       │                                                     │
│       │ 2. If stock < MIN_STOCK                            │
│       ▼                                                     │
│   ┌─────────────────────────────────────────┐              │
│   │         Supplier API (Skyetel)          │              │
│   │   Order DIDs until stock = MAX_STOCK    │              │
│   └─────────────────────────────────────────┘              │
│       │                                                     │
│       │ 3. Provision numbers                                │
│       ▼                                                     │
│   ┌─────────────────────────────────────────┐              │
│   │         Somleng Carrier API             │              │
│   │   POST /carrier/phone_numbers           │              │
│   └─────────────────────────────────────────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Key Files

```
inventory_manager/
├── app.rb                    # Sinatra application (Lambda handler)
├── runner.rb                 # Main ordering logic
├── lib/
│   ├── inventory_manager.rb  # Stock checking and ordering
│   └── suppliers/
│       └── skyetel.rb       # Skyetel API client
├── config/
│   └── cities.csv           # City configurations for ordering
├── Dockerfile                # AWS Lambda deployment
└── Gemfile                   # Ruby dependencies
```

### Configuration

```bash
# Somleng Carrier API
SOMLENG_API_KEY=<carrier-api-key>
SOMLENG_API_URL=https://api.cpaas.voip.ing

# Supplier credentials (Skyetel)
SUPPLIER=skyetel
SKYETEL_USERNAME=<username>
SKYETEL_PASSWORD=<password>

# Inventory settings
MIN_STOCK=10          # Order when below this
MAX_STOCK=50          # Order up to this
DRY_RUN=false         # Set true for testing
```

### City Configuration (cities.csv)

```csv
city,state,country,rate_center
Los Angeles,CA,US,LSAN DA 01
New York,NY,US,NWYRCYZN01
Chicago,IL,US,CHCGIL01
```

### Running

**As Lambda:**
```bash
# Build Docker image
docker build -t inventory-manager .

# Deploy to AWS Lambda
aws lambda update-function-code ...
```

**Locally:**
```bash
cd inventory_manager
bundle install
DRY_RUN=true ruby runner.rb
```

**As Cron:**
```bash
# Run every hour
0 * * * * cd /path/to/inventory_manager && ruby runner.rb >> /var/log/inventory.log 2>&1
```

## Supplier: Skyetel

Currently the only supported supplier.

### API Endpoints Used

| Endpoint | Purpose |
|----------|---------|
| `GET /inventory/search` | Search available DIDs |
| `POST /inventory/order` | Order DIDs |
| `GET /inventory/orders/{id}` | Check order status |

### Adding New Suppliers

1. Create `lib/suppliers/new_supplier.rb`
2. Implement the supplier interface:
   ```ruby
   class NewSupplier
     def search_numbers(criteria)
       # Return available numbers
     end

     def order_number(number)
       # Order the number
     end
   end
   ```
3. Register in `inventory_manager.rb`

## Integration with Our Platform

### When to Use

- **White-label carriers** need phone number inventory
- **Automated provisioning** without manual ordering
- **Multi-city coverage** with stock maintenance

### Not Currently Used

We currently provision phone numbers manually via:
1. Carrier dashboard (https://carrier.app.cpaas.voip.ing)
2. Direct Carrier API calls

The inventory manager would be useful if we:
- Have high-volume number requirements
- Need automatic restocking
- Want to offer customers self-service number selection

## Carrier API Reference

The Inventory Manager uses Somleng's **Carrier API** (not the Twilio-compatible API):

```bash
# List available phone numbers
curl -H "Authorization: Bearer $CARRIER_API_KEY" \
  "https://api.cpaas.voip.ing/carrier/phone_numbers?available=true"

# Create phone number
curl -X POST -H "Authorization: Bearer $CARRIER_API_KEY" \
  "https://api.cpaas.voip.ing/carrier/phone_numbers" \
  -d '{"number": "+12025551234", "type": "local"}'
```

See `/somleng-carrier` skill for full Carrier API documentation.
