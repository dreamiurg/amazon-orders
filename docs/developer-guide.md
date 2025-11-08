# Developer Contributing Guide

This guide is for developers who want to contribute to the `amazon-orders` library, whether fixing bugs, adding features, or improving documentation.

## Table of Contents

1. [Getting Started](#getting-started)
2. [Development Environment Setup](#development-environment-setup)
3. [Project Structure](#project-structure)
4. [Development Workflow](#development-workflow)
5. [Testing](#testing)
6. [Code Quality](#code-quality)
7. [Documentation](#documentation)
8. [Release Process](#release-process)
9. [Common Development Tasks](#common-development-tasks)

## Getting Started

### Prerequisites

- Python 3.9 or higher
- Git
- Make (for using Makefile commands)
- A valid Amazon account for integration testing

### Initial Setup

1. **Fork the repository**

   Visit [github.com/alexdlaird/amazon-orders](https://github.com/alexdlaird/amazon-orders) and click "Fork"

2. **Clone your fork**

   ```bash
   git clone https://github.com/YOUR_USERNAME/amazon-orders.git
   cd amazon-orders
   ```

3. **Add upstream remote**

   ```bash
   git remote add upstream https://github.com/alexdlaird/amazon-orders.git
   ```

## Development Environment Setup

### Using Make (Recommended)

The project includes a Makefile with common development tasks:

```bash
# Install in virtual environment
make install

# Build and install locally
make local

# Run unit tests
make test

# Run integration tests
make test-integration

# Run code quality checks
make check

# Build documentation
make docs

# Run everything (install, check, test)
make all

# Clean build artifacts
make clean
```

### Manual Setup

If you prefer manual setup:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install package in development mode
pip install -e .

# Install development dependencies
pip install -e ".[dev]"

# Install documentation dependencies
pip install -e ".[docs]"

# Install integration test dependencies
pip install -e ".[integration]"
```

### Environment Variables

For testing and development, configure these environment variables:

```bash
# Required for integration tests
export AMAZON_USERNAME="your@email.com"
export AMAZON_PASSWORD="your_password"
export AMAZON_OTP_SECRET_KEY="your_otp_secret"  # Optional but recommended

# Optional configuration
export AMAZON_BASE_URL="https://www.amazon.com"  # Override base URL
```

Create a `.env` file (git-ignored) for convenience:

```bash
# .env
AMAZON_USERNAME=your@email.com
AMAZON_PASSWORD=your_password
AMAZON_OTP_SECRET_KEY=your_otp_secret
```

Load with:

```bash
export $(cat .env | xargs)
```

## Project Structure

```
amazon-orders/
├── amazonorders/           # Main package
│   ├── __init__.py        # Package version and metadata
│   ├── cli.py             # Command-line interface
│   ├── conf.py            # Configuration management
│   ├── constants.py       # URL constants and patterns
│   ├── exception.py       # Custom exceptions
│   ├── forms.py           # Authentication form handlers
│   ├── orders.py          # Order retrieval logic
│   ├── selectors.py       # CSS selectors for parsing
│   ├── session.py         # Session management
│   ├── transactions.py    # Transaction retrieval logic
│   ├── util.py            # Utility functions
│   └── entity/            # Entity models
│       ├── parsable.py    # Base entity class
│       ├── item.py        # Item entity
│       ├── order.py       # Order entity
│       ├── recipient.py   # Recipient entity
│       ├── seller.py      # Seller entity
│       ├── shipment.py    # Shipment entity
│       └── transaction.py # Transaction entity
├── tests/                 # Test suite
│   ├── unit/             # Unit tests
│   ├── integration/      # Integration tests
│   ├── resources/        # Test fixtures
│   ├── testcase.py       # Base test case
│   ├── unittestcase.py   # Unit test base
│   └── integrationtestcase.py  # Integration test base
├── docs/                  # Documentation
│   ├── conf.py           # Sphinx configuration
│   ├── index.rst         # Documentation home
│   ├── api.rst           # API reference
│   ├── troubleshooting.rst  # Troubleshooting guide
│   ├── architecture.md   # Architecture documentation
│   ├── user-guide.md     # User guide
│   └── developer-guide.md # This file
├── scripts/              # Build and utility scripts
├── .github/              # GitHub Actions workflows
│   └── workflows/
│       ├── build.yml     # Build and test
│       ├── integration.yml  # Nightly integration tests
│       ├── release.yml   # Release automation
│       └── ...
├── Makefile              # Development commands
├── pyproject.toml        # Package configuration
├── CONTRIBUTING.rst      # Contributing guide
└── README.md             # Project README
```

## Development Workflow

### 1. Create a Branch

```bash
# Update main branch
git checkout main
git pull upstream main

# Create feature branch
git checkout -b feature/my-feature

# Or for bugfix
git checkout -b fix/issue-123
```

### 2. Make Changes

Follow these guidelines:

- **Code Style**: Follow PEP 8 conventions
- **Type Hints**: Use type hints for function signatures
- **Docstrings**: Add docstrings for public APIs
- **Line Length**: Max 119 characters (configured in flake8)
- **Imports**: Organize imports (standard library, third-party, local)

Example:

```python
from typing import List, Optional
from bs4 import Tag
from amazonorders.conf import AmazonOrdersConfig

def parse_items(tags: List[Tag], config: AmazonOrdersConfig) -> List[Item]:
    """
    Parse item tags into Item entities.

    :param tags: List of BeautifulSoup tags representing items.
    :param config: Configuration object with selectors.
    :return: List of parsed Item entities.
    """
    items = []
    for tag in tags:
        item = Item(tag, config)
        items.append(item)
    return items
```

### 3. Write Tests

Every change should include tests:

**Unit Test Example** (`tests/unit/test_orders.py`):

```python
from tests.unittestcase import UnitTestCase
from amazonorders.orders import AmazonOrders

class TestAmazonOrders(UnitTestCase):
    def test_parse_order_number(self):
        """Test order number parsing."""
        # Setup
        html = self.get_resource('order_details.html')
        parsed = BeautifulSoup(html, 'html.parser')

        # Execute
        order = Order(parsed, self.config)

        # Assert
        self.assertEqual(order.order_number, "123-4567890-1234567")

    def test_order_total_parsing(self):
        """Test order total is correctly parsed as float."""
        html = self.get_resource('order_details.html')
        parsed = BeautifulSoup(html, 'html.parser')

        order = Order(parsed, self.config)

        self.assertIsInstance(order.grand_total, float)
        self.assertGreater(order.grand_total, 0)
```

**Integration Test Example** (`tests/integration/test_orders_integration.py`):

```python
from tests.integrationtestcase import IntegrationTestCase

class TestOrdersIntegration(IntegrationTestCase):
    def test_get_order_history(self):
        """Test retrieving real order history."""
        orders = self.amazon_orders.get_order_history(
            year=2024,
            keep_paging=False  # Only first page for faster testing
        )

        self.assertIsInstance(orders, list)
        if orders:  # Only assert if user has orders
            order = orders[0]
            self.assertIsNotNone(order.order_number)
            self.assertIsNotNone(order.grand_total)
            self.assertIsNotNone(order.order_placed_date)
```

### 4. Run Tests Locally

```bash
# Run unit tests
make test

# Run integration tests (requires valid Amazon credentials)
make test-integration

# Run specific test file
source venv/bin/activate
pytest tests/unit/test_orders.py -v

# Run specific test
pytest tests/unit/test_orders.py::TestAmazonOrders::test_parse_order_number -v
```

### 5. Check Code Quality

```bash
# Run all checks (mypy + flake8)
make check

# Run mypy only
source venv/bin/activate
mypy amazonorders

# Run flake8 only
flake8
```

### 6. Build Documentation

```bash
# Build docs
make docs

# View docs
open build/docs/html/index.html  # macOS
xdg-open build/docs/html/index.html  # Linux
```

### 7. Commit Changes

Use clear, descriptive commit messages:

```bash
git add .
git commit -m "Add support for parsing digital orders

- Add digital_item flag to Item entity
- Update selectors for digital content
- Add unit tests for digital order parsing
- Update documentation with digital order examples

Fixes #123"
```

### 8. Push and Create Pull Request

```bash
# Push to your fork
git push origin feature/my-feature

# Create PR on GitHub
# Navigate to https://github.com/alexdlaird/amazon-orders
# Click "New Pull Request"
```

## Testing

### Test Structure

The project has two types of tests:

1. **Unit Tests** (`tests/unit/`)
   - Fast, isolated tests
   - Mock external dependencies
   - Test individual components
   - Run on every commit

2. **Integration Tests** (`tests/integration/`)
   - Slower, real Amazon requests
   - Require valid credentials
   - Test end-to-end workflows
   - Run nightly via GitHub Actions

### Running Tests

```bash
# All unit tests
make test

# All integration tests
make test-integration

# Specific test file
pytest tests/unit/test_session.py -v

# Tests matching pattern
pytest -k "test_order" -v

# With coverage report
pytest --cov=amazonorders --cov-report=html

# Stop on first failure
pytest -x

# Show print statements
pytest -s
```

### Test Configuration

Integration tests support retry logic for flaky tests:

```bash
# Retry failing tests up to 2 times with 5 minute delay
make test-integration

# Custom retry configuration
INTEGRATION_TEST_RERUN=5 INTEGRATION_TEST_RERUN_DELAY=60 make test-integration
```

### Writing Good Tests

**Do:**
- Test one thing per test method
- Use descriptive test names
- Include docstrings explaining what's tested
- Use assertions that provide clear failure messages
- Mock external dependencies in unit tests
- Clean up resources in tearDown

**Don't:**
- Write tests that depend on each other
- Use time.sleep() (use retry logic instead)
- Hard-code credentials in test files
- Commit test resources with sensitive data
- Skip tests without good reason

### Test Fixtures

Use resource files for test data:

```python
class MyTestCase(UnitTestCase):
    def test_parsing(self):
        # Load HTML from tests/resources/
        html = self.get_resource('order_page.html')
        parsed = BeautifulSoup(html, 'html.parser')

        # Test parsing
        order = Order(parsed, self.config)
        self.assertEqual(order.order_number, "123-4567890-1234567")
```

To create test resources:

```bash
# Build test resources from real Amazon data
make build-test-resources
```

## Code Quality

### Type Checking with mypy

The project uses type hints and mypy for type checking:

```python
from typing import List, Optional

def get_orders(year: int) -> List[Order]:
    """Get orders for a year."""
    pass

def find_order(order_id: str) -> Optional[Order]:
    """Find order by ID, returns None if not found."""
    pass
```

Run type checker:

```bash
mypy amazonorders
```

### Linting with flake8

The project follows PEP 8 with some adjustments:

- Max line length: 119 characters
- Use double quotes for strings
- 4 spaces for indentation

Configuration in `pyproject.toml`:

```toml
[tool.flake8]
max-line-length = 119
statistics = true
exclude = "scripts/*,docs/*,venv/*,build/*"
```

Run linter:

```bash
flake8
```

### Coverage Requirements

Maintain high test coverage:

```bash
# Generate coverage report
pytest --cov=amazonorders --cov-report=html

# View report
open build/coverage/index.html
```

Coverage configuration in `pyproject.toml`:

```toml
[tool.coverage.report]
precision = 2
exclude_lines = [
    "if TYPE_CHECKING:",
    "pragma: no cover",
    "def __repr__",
    "raise NotImplementedError",
]
```

## Documentation

### Building Documentation

The project uses Sphinx for documentation:

```bash
# Build HTML docs
make docs

# View docs
open build/docs/html/index.html
```

### Documentation Structure

- **README.md**: Quick start and overview
- **docs/index.rst**: Main documentation page
- **docs/api.rst**: API reference (auto-generated from docstrings)
- **docs/troubleshooting.rst**: Common issues and solutions
- **docs/architecture.md**: Architecture and design
- **docs/user-guide.md**: User guide with examples
- **docs/developer-guide.md**: This developer guide

### Writing Docstrings

Use reStructuredText format for docstrings:

```python
def get_order_history(
    self,
    year: int = datetime.date.today().year,
    start_index: Optional[int] = None,
    full_details: bool = False,
    keep_paging: bool = True
) -> List[Order]:
    """
    Get the Amazon Order history for a given year.

    :param year: The year for which to get history.
    :param start_index: The index to start fetching from. See
        :attr:`~amazonorders.entity.order.Order.index` to correlate.
    :param full_details: Get the full details for each Order (slower).
    :param keep_paging: ``False`` if only one page should be fetched.
    :return: A list of the requested Orders.
    :raises AmazonOrdersError: If not authenticated or parsing fails.

    Example::

        orders = amazon_orders.get_order_history(year=2024, full_details=True)
        for order in orders:
            print(f"{order.order_number}: ${order.grand_total}")
    """
    pass
```

### Updating Documentation

When adding features:

1. Update API docstrings
2. Add examples to user guide
3. Update architecture docs if structure changes
4. Add troubleshooting entries for known issues
5. Update README if it affects quick start

## Release Process

Releases are managed by maintainers following semantic versioning.

### Version Numbering

- **Major** (x.0.0): Breaking changes
- **Minor** (4.x.0): New features, backwards compatible
- **Patch** (4.0.x): Bug fixes

### Release Checklist

1. **Update version** in these files:
   - `amazonorders/__init__.py`
   - `docs/index.rst`
   - `README.md`

2. **Update CHANGELOG.md**
   - Add version and date
   - List all changes

3. **Validate release**:
   ```bash
   VERSION=4.0.17 make validate-release
   ```

4. **Create release commit**:
   ```bash
   git commit -m "Release v4.0.17"
   git tag v4.0.17
   git push origin main --tags
   ```

5. **GitHub Actions** automatically:
   - Runs tests
   - Builds package
   - Publishes to PyPI
   - Creates GitHub release

### Manual Release (if needed)

```bash
# Build package
make local

# Upload to PyPI (requires credentials)
make upload
```

## Common Development Tasks

### Adding a New Entity Field

1. **Add to entity class** (`amazonorders/entity/order.py`):

```python
class Order(Parsable):
    def __init__(self, parsed, config, **kwargs):
        # ... existing code ...

        #: The order's gift message.
        self.gift_message: Optional[str] = self.safe_simple_parse(
            selector=self.config.selectors.FIELD_ORDER_GIFT_MESSAGE_SELECTOR
        )
```

2. **Add selector** (`amazonorders/selectors.py`):

```python
class Selectors:
    # ... existing selectors ...

    #: Order gift message selector
    FIELD_ORDER_GIFT_MESSAGE_SELECTOR = ".gift-message"
```

3. **Add test** (`tests/unit/test_order.py`):

```python
def test_gift_message_parsing(self):
    """Test gift message is parsed correctly."""
    html = self.get_resource('order_with_gift.html')
    order = Order(BeautifulSoup(html, 'html.parser'), self.config)

    self.assertEqual(order.gift_message, "Happy Birthday!")
```

4. **Update documentation**:
   - Add to docstring in entity class
   - Mention in user guide if user-facing
   - Add example usage

### Adding a New Authentication Form

1. **Create form class** (`amazonorders/forms.py`):

```python
class NewAuthForm(AuthForm):
    """Handler for new authentication challenge."""

    def __init__(self, config, form_selector=None, error_selector=None):
        super().__init__(
            config,
            form_selector or config.selectors.NEW_AUTH_FORM_SELECTOR,
            error_selector or config.selectors.NEW_AUTH_ERROR_SELECTOR
        )

    def submit(self, session, parsed, io):
        """Submit the new auth form."""
        # Parse form fields
        field = self.simple_parse(parsed, ".auth-field")

        # Get user input
        value = io.prompt("Enter auth value")

        # Submit form
        return session.post(self.config.constants.AUTH_URL, data={
            "field": field,
            "value": value
        })
```

2. **Add selectors** (`amazonorders/selectors.py`):

```python
#: New auth form selector
NEW_AUTH_FORM_SELECTOR = "form#new-auth-form"

#: New auth error selector
NEW_AUTH_ERROR_SELECTOR = ".new-auth-error"
```

3. **Register in session** (`amazonorders/session.py`):

```python
def __init__(self, username=None, password=None, **kwargs):
    if not auth_forms:
        auth_forms = [
            # ... existing forms ...
            NewAuthForm(config),  # Add new form
        ]
```

4. **Test**:

```python
def test_new_auth_form_submission(self):
    """Test new auth form is handled correctly."""
    # Mock the form page
    # Test submission logic
    # Verify correct data posted
```

### Updating Selectors for Amazon HTML Changes

When Amazon changes their HTML:

1. **Identify broken selector**:
   - Check failing tests
   - Run with debug mode: `amazon-orders --debug history`
   - Examine HTML in `~/.config/amazon-orders/output/`

2. **Update selector** (`amazonorders/selectors.py`):

```python
# Old selector (broken)
FIELD_ORDER_NUMBER_SELECTOR = ".order-number"

# New selector (updated)
FIELD_ORDER_NUMBER_SELECTOR = ".order-info .order-id"
```

3. **Test locally**:

```bash
make test-integration
```

4. **Update test fixtures** if needed:

```bash
make build-test-resources
```

5. **Document the change**:

```
Update order number selector for Amazon HTML change

Amazon updated their order page HTML structure, changing
the class used for order numbers from .order-number to
.order-info .order-id.

Updated selector and verified with integration tests.
```

### Debugging Integration Tests

```python
# Enable debug mode in tests
class TestOrdersIntegration(IntegrationTestCase):
    def setUp(self):
        super().setUp()
        self.amazon_session.debug = True
        self.amazon_orders.debug = True
```

Debug output includes:
- HTTP requests/responses
- HTML snapshots (saved to output directory)
- Parsing errors
- Authentication flow

Debug files location: `~/.config/amazon-orders/output/`

### Performance Profiling

Profile slow operations:

```python
import cProfile
import pstats

# Profile order history retrieval
profiler = cProfile.Profile()
profiler.enable()

orders = amazon_orders.get_order_history(year=2024, full_details=True)

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)  # Top 20 slowest functions
```

## Getting Help

- **Documentation**: [amazon-orders.readthedocs.io](https://amazon-orders.readthedocs.io)
- **Issues**: [GitHub Issues](https://github.com/alexdlaird/amazon-orders/issues)
- **Discussions**: [GitHub Discussions](https://github.com/alexdlaird/amazon-orders/discussions)
- **Email**: contact@alexlaird.com

## Code of Conduct

Please review the [Code of Conduct](https://github.com/alexdlaird/amazon-orders?tab=coc-ov-file) before contributing.

## License

By contributing to `amazon-orders`, you agree that your contributions will be licensed under the MIT License.
