# Global Search feature

[invenio-global-search](https://github.com/tu-graz-library/invenio-global-search) is an Invenio package that provides a unified search interface across different record types (RDM records, MARC21 publications, LOM educational resources) in the Repository.

**Note:** Global search is most useful when you have multiple data models (e.g., RDM + MARC21, or RDM + LOM). If you only have RDM records, the built-in RDM search is sufficient and global search may not be necessary.

This guide explains how to set up global search for research results, with examples for adding additional record types.

## Setup

After the successful installation of `invenio-global-search`, it needs to be configured properly to work. The following sections should guide you through the required adaptations.

### Installation

Add `invenio-global-search` to your `pyproject.toml`:

```toml
[project]
dependencies = [
    "invenio-global-search>=0.3.0",
    # ... other dependencies
]
```

Then update your lock file:

```bash
uv sync
```

### invenio.cfg

The global search component needs to be added to the RDM records service components, and you need to configure the global search interface to show research results.

Add the following configuration to your theme's `invenio.cfg` (e.g., `themes/YOUR_THEME/invenio.cfg`):

```python
GLOBAL_SEARCH_SCHEMAS = {
    "rdm": {
        "schema": "rdm",
        "name_l10n": "Research Result",
    },
}
"""Mapping of schemas for global search - only RDM (Research Results) are shown."""
```

**Note:** If you want to add additional record types (e.g., MARC21 publications or LOM educational resources), you must have the corresponding packages installed (`invenio-records-marc21` for publications, `invenio-records-lom` for educational resources). See the [Adding Additional Record Types](#adding-additional-record-types) section below.

### Component Configuration

For each record type you want to include in global search, you need to add the corresponding component to the service components. This ensures that records are automatically indexed in the global search database when they are created or updated.

**For RDM records:**

```python
from invenio_global_search.components import RDMToGlobalSearchComponent
from invenio_rdm_records.services.components import (
    DefaultRecordsComponents as RDMDefaultRecordsComponents,
)

RDM_RECORDS_SERVICE_COMPONENTS = RDMDefaultRecordsComponents + [
    RDMToGlobalSearchComponent
]
```

**For MARC21 publications (if installed):**

```python
from invenio_global_search.components import Marc21ToGlobalSearchComponent
from invenio_records_marc21.services.components import (
    DefaultRecordsComponents as Marc21DefaultRecordsComponents,
)

MARC21_RECORDS_SERVICE_COMPONENTS = Marc21DefaultRecordsComponents + [
    Marc21ToGlobalSearchComponent
]
```

**For LOM educational resources (if installed):**

```python
from invenio_global_search.components import LOMToGlobalSearchComponent
from invenio_records_lom.services.components import (
    DefaultRecordsComponents as LOMDefaultRecordsComponents,
)

LOM_RECORDS_SERVICE_COMPONENTS = LOMDefaultRecordsComponents + [
    LOMToGlobalSearchComponent
]
```

### Initialize Global Search Database

After configuring global search, you need to initialize the global search database:

```bash
invenio global-search rebuild-database
```

This command creates the necessary database tables for storing global search records.


## UI Customization

The global search feature uses the global search template for the main search page. Configure this in your `invenio.cfg`:

```python
SEARCH_UI_SEARCH_TEMPLATE = "invenio_records_global_search/search/search.html"
```

This ensures that users hitting "Enter" in the search bar are directed to the global search interface, which shows research results.

**Important:** After setting `SEARCH_UI_SEARCH_TEMPLATE` to use the global search template, the default RDM records search route will no longer be accessible. You need to create a new route for RDM records search if you want to provide direct access to RDM-only search. This is typically done by adding a custom view/route in your theme package.

## Adding Additional Record Types

To make global search more useful, you can add additional record types such as MARC21 publications or LOM educational resources.

**Prerequisites:** You must have the corresponding packages installed:
- For MARC21 publications: `invenio-records-marc21`
- For LOM educational resources: `invenio-records-lom`

After installing the respective packages, you need to:
1. Add the component to the service components (see [Component Configuration](#component-configuration) above)
2. Update your `GLOBAL_SEARCH_SCHEMAS` configuration

**Example: Adding MARC21 Publications**

```python
GLOBAL_SEARCH_SCHEMAS = {
    "rdm": {
        "schema": "rdm",
        "name_l10n": "Research Result",
    },
    "marc21": {
        "schema": "marc21",
        "name_l10n": "Publication",
    },
}
```

**Example: Adding LOM Educational Resources**

```python
GLOBAL_SEARCH_SCHEMAS = {
    "rdm": {
        "schema": "rdm",
        "name_l10n": "Research Result",
    },
    "lom": {
        "schema": "lom",
        "name_l10n": "OER",
    },
}
```
