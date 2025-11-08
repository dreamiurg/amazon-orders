# User Guide

This guide covers common workflows and practical usage patterns for the `amazon-orders` library.

## Table of Contents

1. [Getting Started](#getting-started)
2. [Authentication](#authentication)
3. [Retrieving Order History](#retrieving-order-history)
4. [Working with Orders](#working-with-orders)
5. [Transaction History](#transaction-history)
6. [Error Handling](#error-handling)
7. [Performance Optimization](#performance-optimization)
8. [Advanced Usage](#advanced-usage)

## Getting Started

### Installation

```bash
pip install amazon-orders --upgrade
```

For best compatibility, pin to the minor version:

```bash
pip install amazon-orders==4.0.*
```

### Basic Example

```python
from amazonorders.session import AmazonSession
from amazonorders.orders import AmazonOrders

# Create and authenticate session
amazon_session = AmazonSession("your@email.com", "your_password")
amazon_session.login()

# Get order history
amazon_orders = AmazonOrders(amazon_session)
orders = amazon_orders.get_order_history(year=2024)

# Display orders
for order in orders:
    print(f"Order {order.order_number}: ${order.grand_total}")
```

### CLI Usage

The library includes a command-line interface:

```bash
# Login (stores session for reuse)
amazon-orders login

# Get order history
amazon-orders history --year 2024

# Get specific order details
amazon-orders order --order-id 123-4567890-1234567

# Get transaction history
amazon-orders transactions --days 90
```

## Authentication

### Method 1: Direct Credentials

```python
from amazonorders.session import AmazonSession

session = AmazonSession(
    username="your@email.com",
    password="your_password"
)
session.login()
```

### Method 2: Environment Variables

```bash
export AMAZON_USERNAME="your@email.com"
export AMAZON_PASSWORD="your_password"
```

```python
from amazonorders.session import AmazonSession

# Credentials loaded automatically from environment
session = AmazonSession()
session.login()
```

### Method 3: Configuration File

Create `~/.config/amazon-orders/config.yml`:

```yaml
username: your@email.com
password: your_password
output_dir: ~/amazon-orders-output
max_auth_attempts: 5
```

```python
from amazonorders.session import AmazonSession
from amazonorders.conf import AmazonOrdersConfig

config = AmazonOrdersConfig()
session = AmazonSession(config=config)
session.login()
```

### Handling Multi-Factor Authentication (MFA)

#### Interactive MFA

When MFA is required, the library will prompt for the code:

```python
session = AmazonSession("your@email.com", "your_password")
session.login()
# --> Enter OTP code: 123456
```

#### Automated MFA with OTP Secret

For automated workflows, configure OTP secret key:

```bash
export AMAZON_OTP_SECRET_KEY="YOUR_SECRET_KEY_FROM_AMAZON"
```

Or in code:

```python
session = AmazonSession(
    username="your@email.com",
    password="your_password",
    otp_secret_key="YOUR_SECRET_KEY_FROM_AMAZON"
)
session.login()  # OTP codes generated automatically
```

**Getting your OTP secret:**
1. Go to Amazon's 2FA settings
2. Choose "Authenticator App"
3. When shown QR code, click "Can't scan the barcode?"
4. Copy the secret key shown
5. Use this key in your configuration

### Handling CAPTCHAs

#### Static Image CAPTCHAs

The library can auto-solve simple CAPTCHAs or prompt for manual input:

```python
session = AmazonSession("your@email.com", "your_password")
session.login()
# If CAPTCHA appears, you'll be prompted to enter the text
```

#### Interactive/Puzzle CAPTCHAs

JavaScript-based puzzle CAPTCHAs cannot be solved. Workarounds:

1. **Reduce CAPTCHA frequency:**
   - Use session persistence (cookies saved automatically)
   - Login less frequently
   - Use residential IP addresses

2. **Login via browser first:**
   ```python
   # Login manually in browser, then export cookies
   # Use browser extension to export cookies as JSON
   # Place in ~/.config/amazon-orders/cookies.json

   session = AmazonSession()
   # Will load existing cookies
   session.login()
   ```

3. **Use environment with lower security triggers:**
   - Avoid cloud IPs
   - Use consistent user agent
   - Don't query too rapidly

### Session Persistence

Sessions are automatically saved and reused:

```python
# First run - performs authentication
session = AmazonSession("your@email.com", "your_password")
session.login()

# Subsequent runs - reuses saved session
session = AmazonSession("your@email.com", "your_password")
session.login()  # Skips auth if session valid
```

Session cookies stored at: `~/.config/amazon-orders/cookies.json`

To force re-authentication:

```python
import os
from amazonorders.conf import AmazonOrdersConfig

config = AmazonOrdersConfig()
cookies_file = os.path.join(config.config_dir, "cookies.json")
if os.path.exists(cookies_file):
    os.remove(cookies_file)

session = AmazonSession("your@email.com", "your_password")
session.login()  # Will perform fresh authentication
```

## Retrieving Order History

### Basic Order History

```python
from amazonorders.orders import AmazonOrders

amazon_orders = AmazonOrders(amazon_session)

# Get all orders from 2024
orders = amazon_orders.get_order_history(year=2024)

for order in orders:
    print(f"Order: {order.order_number}")
    print(f"Date: {order.order_placed_date}")
    print(f"Total: ${order.grand_total}")
    print(f"Items: {len(order.items)}")
    print("---")
```

### Full Order Details

By default, order history returns summary data. For complete details:

```python
# Get full details (slower - makes additional request per order)
orders = amazon_orders.get_order_history(
    year=2024,
    full_details=True
)

for order in orders:
    print(f"Payment Method: {order.payment_method}")
    print(f"Last 4 digits: {order.payment_method_last_4}")

    for item in order.items:
        print(f"  - {item.title}: ${item.price}")
        print(f"    Seller: {item.seller.name if item.seller else 'Amazon'}")
        print(f"    Quantity: {item.quantity}")
```

**Fields only available with `full_details=True`:**
- `order.payment_method`
- `order.payment_method_last_4`
- `order.refund_completed_date`
- `order.refund_total`
- Complete item details

### Pagination Control

```python
# Get only first page
orders = amazon_orders.get_order_history(
    year=2024,
    keep_paging=False
)

# Resume from specific index (if previous call errored)
orders = amazon_orders.get_order_history(
    year=2024,
    start_index=50
)
```

### Multiple Years

```python
import datetime

all_orders = []
current_year = datetime.date.today().year

for year in range(current_year - 5, current_year + 1):
    orders = amazon_orders.get_order_history(year=year)
    all_orders.extend(orders)
    print(f"Retrieved {len(orders)} orders from {year}")

print(f"Total orders: {len(all_orders)}")
```

## Working with Orders

### Accessing Order Details

```python
order = orders[0]

# Basic fields
print(f"Order Number: {order.order_number}")
print(f"Placed: {order.order_placed_date}")
print(f"Total: ${order.grand_total}")

# Recipient information
if order.recipient:
    print(f"Recipient: {order.recipient.name}")
    print(f"Address: {order.recipient.address}")

# Shipment tracking
for shipment in order.shipments:
    print(f"Status: {shipment.delivery_status}")
    print(f"Shipped: {shipment.shipped_date}")
    print(f"Delivered: {shipment.delivered_date}")
```

### Working with Items

```python
for item in order.items:
    print(f"Title: {item.title}")
    print(f"Price: ${item.price}")
    print(f"Quantity: {item.quantity}")
    print(f"Condition: {item.condition}")
    print(f"Link: {item.link}")

    if item.seller:
        print(f"Seller: {item.seller.name}")
        print(f"Seller Link: {item.seller.link}")

    if item.return_eligible_date:
        print(f"Return by: {item.return_eligible_date}")

    if item.image_link:
        print(f"Image: {item.image_link}")
```

### Getting Specific Order by ID

```python
# Get specific order
order = amazon_orders.get_order(order_id="123-4567890-1234567")

print(f"Order {order.order_number}")
print(f"Total: ${order.grand_total}")
```

### Filtering Orders

```python
# Orders over $100
expensive_orders = [o for o in orders if o.grand_total > 100]

# Orders from specific month
import datetime
jan_orders = [
    o for o in orders
    if o.order_placed_date.month == 1
]

# Orders with specific item
def has_item_with_text(order, text):
    return any(text.lower() in item.title.lower() for item in order.items)

electronics = [o for o in orders if has_item_with_text(o, "laptop")]

# Orders from specific seller
def has_seller(order, seller_name):
    return any(
        item.seller and seller_name.lower() in item.seller.name.lower()
        for item in order.items
    )

third_party = [o for o in orders if has_seller(o, "SomeStore")]
```

### Exporting Orders to CSV

```python
import csv
from datetime import datetime

orders = amazon_orders.get_order_history(year=2024, full_details=True)

with open('orders_2024.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f)
    writer.writerow([
        'Order Number', 'Date', 'Total', 'Payment Method',
        'Item Count', 'Items'
    ])

    for order in orders:
        items_str = '; '.join([item.title for item in order.items])
        writer.writerow([
            order.order_number,
            order.order_placed_date.isoformat(),
            order.grand_total,
            order.payment_method or 'N/A',
            len(order.items),
            items_str
        ])

print("Orders exported to orders_2024.csv")
```

### Exporting to JSON

```python
import json
from datetime import date

class DateEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, date):
            return obj.isoformat()
        return super().default(obj)

# Convert order to dict
def order_to_dict(order):
    return {
        'order_number': order.order_number,
        'order_placed_date': order.order_placed_date,
        'grand_total': order.grand_total,
        'payment_method': order.payment_method,
        'items': [
            {
                'title': item.title,
                'price': item.price,
                'quantity': item.quantity,
                'link': item.link
            }
            for item in order.items
        ]
    }

orders_data = [order_to_dict(o) for o in orders]

with open('orders_2024.json', 'w', encoding='utf-8') as f:
    json.dump(orders_data, f, indent=2, cls=DateEncoder)
```

## Transaction History

### Basic Transaction Retrieval

```python
from amazonorders.transactions import AmazonTransactions

amazon_trans = AmazonTransactions(amazon_session)

# Get last 90 days of transactions
transactions = amazon_trans.get_transactions(days=90)

for trans in transactions:
    print(f"Date: {trans.completed_date}")
    print(f"Order: {trans.order_number}")
    print(f"Amount: ${trans.grand_total}")
    print(f"Seller: {trans.seller}")
    print(f"Refund: {trans.is_refund}")
    print("---")
```

### Filtering Transactions

```python
# Get 1 year of transactions
transactions = amazon_trans.get_transactions(days=365)

# Only refunds
refunds = [t for t in transactions if t.is_refund]
total_refunded = sum(t.grand_total for t in refunds)
print(f"Total refunded: ${total_refunded}")

# Only charges (non-refunds)
charges = [t for t in transactions if not t.is_refund]
total_spent = sum(abs(t.grand_total) for t in charges)
print(f"Total spent: ${total_spent}")

# Transactions by seller
from collections import defaultdict
by_seller = defaultdict(list)
for trans in transactions:
    by_seller[trans.seller].append(trans)

for seller, trans_list in by_seller.items():
    total = sum(abs(t.grand_total) for t in trans_list)
    print(f"{seller}: ${total} ({len(trans_list)} transactions)")
```

### Monthly Spending Report

```python
from datetime import date
from collections import defaultdict

transactions = amazon_trans.get_transactions(days=365)

# Group by month
monthly = defaultdict(float)
for trans in transactions:
    if not trans.is_refund:  # Exclude refunds
        month_key = trans.completed_date.strftime("%Y-%m")
        monthly[month_key] += abs(trans.grand_total)

# Display report
print("Monthly Amazon Spending:")
for month in sorted(monthly.keys()):
    print(f"{month}: ${monthly[month]:.2f}")

# Average monthly spending
avg_monthly = sum(monthly.values()) / len(monthly) if monthly else 0
print(f"\nAverage: ${avg_monthly:.2f}/month")
```

### Pagination Control

```python
# Get only first page
transactions = amazon_trans.get_transactions(
    days=365,
    keep_paging=False
)

# Resume from where it left off (if error occurred)
# The error will include next_page_data in meta
try:
    transactions = amazon_trans.get_transactions(days=365)
except Exception as e:
    next_page_data = getattr(e, 'meta', None)
    if next_page_data:
        # Resume
        more_transactions = amazon_trans.get_transactions(
            days=365,
            next_page_data=next_page_data
        )
```

## Error Handling

### Basic Error Handling

```python
from amazonorders.exception import (
    AmazonOrdersError,
    AmazonOrdersAuthError,
    AmazonOrdersNotFoundError
)

try:
    session = AmazonSession("email@example.com", "password")
    session.login()

    amazon_orders = AmazonOrders(session)
    orders = amazon_orders.get_order_history(year=2024)

except AmazonOrdersAuthError as e:
    print(f"Authentication failed: {e}")
    print("Check credentials or handle CAPTCHA/MFA")

except AmazonOrdersNotFoundError as e:
    print(f"Order not found: {e}")

except AmazonOrdersError as e:
    print(f"General error: {e}")
    if hasattr(e, 'meta'):
        print(f"Context: {e.meta}")
```

### Handling Pagination Errors

```python
from amazonorders.exception import AmazonOrdersError

try:
    orders = amazon_orders.get_order_history(year=2024, full_details=True)
except AmazonOrdersError as e:
    print(f"Error occurred: {e}")

    # Check if we can resume
    if hasattr(e, 'meta') and 'index' in e.meta:
        print(f"Error at index: {e.meta['index']}")
        print("Retrying from that index...")

        orders = amazon_orders.get_order_history(
            year=2024,
            start_index=e.meta['index'],
            full_details=True
        )
```

### Retry Logic

```python
import time

def get_orders_with_retry(amazon_orders, year, max_retries=3):
    """Get order history with automatic retry on failure."""
    for attempt in range(max_retries):
        try:
            orders = amazon_orders.get_order_history(year=year)
            return orders
        except AmazonOrdersError as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Attempt {attempt + 1} failed. Retrying in {wait_time}s...")
                time.sleep(wait_time)
            else:
                print(f"Failed after {max_retries} attempts")
                raise

# Usage
orders = get_orders_with_retry(amazon_orders, 2024)
```

## Performance Optimization

### Minimize Full Details Requests

```python
# Bad: Always fetches full details
orders = amazon_orders.get_order_history(year=2024, full_details=True)

# Good: Fetch full details only for specific orders
orders = amazon_orders.get_order_history(year=2024)  # Fast summary
expensive_orders = [o for o in orders if o.grand_total > 500]

# Get full details only for expensive orders
for order in expensive_orders:
    detailed_order = amazon_orders.get_order(order.order_number)
    print(f"Payment method: {detailed_order.payment_method}")
```

### Configure Thread Pool Size

```python
from amazonorders.conf import AmazonOrdersConfig

config = AmazonOrdersConfig(data={'thread_pool_size': 10})
amazon_orders = AmazonOrders(amazon_session, config=config)

# Async processing uses configured pool size
orders = amazon_orders.get_order_history(year=2024, full_details=True)
```

### Limit Pagination

```python
# Get only recent orders
orders = amazon_orders.get_order_history(
    year=2024,
    keep_paging=False  # Only first page
)

# Process incrementally
processed_count = 0
start_index = 0

while True:
    page_orders = amazon_orders.get_order_history(
        year=2024,
        start_index=start_index,
        keep_paging=False
    )

    if not page_orders:
        break

    # Process orders
    for order in page_orders:
        # Do something with order
        processed_count += 1

    start_index += len(page_orders)
    print(f"Processed {processed_count} orders...")
```

### Cache Results

```python
import pickle
from pathlib import Path

def get_cached_orders(amazon_orders, year, cache_dir="./cache"):
    """Get orders with file caching."""
    cache_path = Path(cache_dir) / f"orders_{year}.pkl"

    if cache_path.exists():
        # Load from cache
        with open(cache_path, 'rb') as f:
            return pickle.load(f)

    # Fetch from Amazon
    orders = amazon_orders.get_order_history(year=year)

    # Save to cache
    cache_path.parent.mkdir(exist_ok=True)
    with open(cache_path, 'wb') as f:
        pickle.dump(orders, f)

    return orders

# Usage
orders = get_cached_orders(amazon_orders, 2024)
```

## Advanced Usage

### Custom I/O Handler

```python
from amazonorders.session import IODefault

class CustomIO(IODefault):
    def echo(self, msg, **kwargs):
        # Custom logging
        import logging
        logging.info(msg)

    def prompt(self, msg, **kwargs):
        # Get input from custom source
        # e.g., web form, API, etc.
        return self.get_from_custom_source(msg)

    def get_from_custom_source(self, prompt):
        # Your custom implementation
        pass

session = AmazonSession(
    "email@example.com",
    "password",
    io=CustomIO()
)
```

### Custom Configuration

```python
from amazonorders.conf import AmazonOrdersConfig

config = AmazonOrdersConfig(
    config_path="/custom/path/config.yml",
    data={
        'output_dir': '/custom/output',
        'max_auth_attempts': 10,
        'thread_pool_size': 20
    }
)

session = AmazonSession(config=config)
amazon_orders = AmazonOrders(session)
```

### Debug Mode

```python
# Enable debug logging
session = AmazonSession(
    "email@example.com",
    "password",
    debug=True
)

# Or for specific components
amazon_orders = AmazonOrders(session, debug=True)
amazon_trans = AmazonTransactions(session, debug=True)

# Debug output goes to stderr and debug files in output directory
```

### Working with Multiple Accounts

```python
def get_orders_for_account(username, password):
    session = AmazonSession(username, password)
    session.login()

    amazon_orders = AmazonOrders(session)
    orders = amazon_orders.get_order_history(year=2024)

    return orders

accounts = [
    ("account1@example.com", "password1"),
    ("account2@example.com", "password2"),
]

all_orders = []
for username, password in accounts:
    orders = get_orders_for_account(username, password)
    all_orders.extend(orders)
    print(f"Retrieved {len(orders)} orders for {username}")

print(f"Total orders across all accounts: {len(all_orders)}")
```

### Integration with Data Analysis

```python
import pandas as pd

# Convert orders to DataFrame
def orders_to_dataframe(orders):
    data = []
    for order in orders:
        for item in order.items:
            data.append({
                'order_number': order.order_number,
                'order_date': order.order_placed_date,
                'order_total': order.grand_total,
                'item_title': item.title,
                'item_price': item.price,
                'item_quantity': item.quantity,
                'seller': item.seller.name if item.seller else 'Amazon'
            })
    return pd.DataFrame(data)

# Get orders and convert
orders = amazon_orders.get_order_history(year=2024)
df = orders_to_dataframe(orders)

# Analysis
print(df.groupby('seller')['item_price'].sum())
print(df['item_title'].value_counts().head(10))
```

## CLI Examples

### Basic Commands

```bash
# Login
amazon-orders login --username user@example.com --password mypass

# Order history
amazon-orders history --year 2024

# Order history with full details
amazon-orders history --year 2024 --full-details

# Specific order
amazon-orders order --order-id 123-4567890-1234567

# Transactions
amazon-orders transactions --days 90
```

### Advanced CLI Usage

```bash
# Custom config path
amazon-orders --config-path /path/to/config.yml history --year 2024

# Debug mode
amazon-orders --debug history --year 2024

# Custom output directory
amazon-orders --output-dir /tmp/amazon history --year 2024

# Max auth attempts
amazon-orders --max-auth-attempts 10 login
```

### Environment Variables for CLI

```bash
export AMAZON_USERNAME="user@example.com"
export AMAZON_PASSWORD="mypass"
export AMAZON_OTP_SECRET_KEY="secret123"

# No need to pass credentials
amazon-orders login
amazon-orders history --year 2024
```

## Troubleshooting

See the [troubleshooting guide](troubleshooting.rst) for detailed help with:

- CAPTCHA challenges
- MFA/OTP issues
- Session persistence problems
- HTML parsing errors
- Rate limiting
- Regional differences

## Best Practices

1. **Use environment variables** for credentials in production
2. **Enable session persistence** to minimize authentication
3. **Configure OTP secret** for automated workflows
4. **Use `full_details=False`** by default, enable only when needed
5. **Implement retry logic** for production environments
6. **Cache results** when appropriate
7. **Handle errors gracefully** with proper exception handling
8. **Monitor for library updates** as Amazon may change HTML structure
9. **Respect rate limits** to avoid triggering security measures
10. **Use debug mode** only during development
