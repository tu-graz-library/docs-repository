# Override Pages
In some cases, the default pages do not offer all the functionality or modification possibilities needed. It is possible to override the pages, though this should be the last resort as it increases resources needed for maintainability.

We will look into how to override some essential pages of `invenio-app-rdm` with our existing module `invenio-theme-tugraz`.

! IMPORTANT: This is valid for `invenio-app-rdm==v6.0.2` !

## Main Search
The component to be overridden is `search` so the path for this looks like `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/search/`.
Place the components in `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/search/components.js`.

The original file can be found here: [components.js](https://github.com/inveniosoftware/invenio-app-rdm/blob/v6.0.2/invenio_app_rdm/theme/assets/semantic-ui/js/invenio_app_rdm/search/components.js)


### Creating Search App
Now that all components have been created or imported, create the following file `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/search/index.js`.
In here, all the previously defined components will be imported and the search app will be defined.

```js
import { createSearchAppInit } from "@js/invenio_search_ui";
import {
  RDMBucketAggregationElement,
  RDMRecordFacets,
  RDMRecordFacetsValues,
  RDMRecordResultsGridItem,
  RDMRecordResultsListItem,
  RDMRecordSearchBarContainer,
  RDMRecordSearchBarElement,
  RDMToggleComponent,
  RDMCountComponent,
} from "./components";

const initSearchApp = createSearchAppInit({
  "BucketAggregation.element": RDMBucketAggregationElement,
  "BucketAggregationValues.element": RDMRecordFacetsValues,
  "ResultsGrid.item": RDMRecordResultsGridItem,
  "ResultsList.item": RDMRecordResultsListItem,
  "SearchApp.facets": RDMRecordFacets,
  "SearchApp.searchbarContainer": RDMRecordSearchBarContainer,
  "SearchBar.element": RDMRecordSearchBarElement,
  "SearchFilters.ToggleComponent": RDMToggleComponent,
  "Count.element": RDMCountComponent,
});
```

### Update webpack
Now that the functionality is done, it is important to add it to webpack. This makes it accessible to templates.
Adding the following line in the `invenio_theme_tugraz/webpack.py` file, under the `semantic-ui` entries will do the trick. It will look something like this:
```python
"semantic-ui": dict(
            entry={
                "invenio-theme-tugraz-theme": "./less/invenio_theme_tugraz/theme.less",
                "invenio-theme-tugraz-js": "./js/invenio_theme_tugraz/theme.js",
                # add search component
                'invenio-theme-tugraz-rdm-search': './js/invenio_theme_tugraz/search/index.js',
            }, ...
```

### Adding Search Template
To make use of the new search, a search template will be added. It will import our search app as previously defined in webpack.
To achieve this, we will simply copy the current search page from invenio-rdm and replace the standard search app import with ours.
Create the following file `invenio_theme_tugraz/templates/invenio_theme_tugraz/search.html` with the content of the [base search template from invenio-app-rdm ](https://github.com/inveniosoftware/invenio-app-rdm/blob/v6.0.2/invenio_app_rdm/records_ui/templates/semantic-ui/invenio_app_rdm/records/search.html).

In that file, all that has to be done is to replace the import:
```diff
{%- block javascript %}
  {{ super() }}
- {{ webpack['invenio-app-rdm-search.js'] }}
+ {{ webpack['invenio-theme-tugraz-rdm-search.js'] }}
{%- endblock %}
```

### Override Config Variable
Finally, invenio-rdm must know that it should use our new template for the search. This can be achieved by overriding the config variable `SEARCH_UI_SEARCH_TEMPLATE`.
In `invenio_theme_tugraz/config.py` add or override the following line:
```python
SEARCH_UI_SEARCH_TEMPLATE = "invenio_theme_tugraz/search.html"
```
Now invenio-rdm will use the new search template, which will use the new search app.


## User Record Search
The component to be overridden is `user_records_search` so the path for this looks like `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/user_records_search/`.
Place the components in `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/user_records_search/components.js`.

The original file can be found here: [components.js](https://github.com/inveniosoftware/invenio-app-rdm/blob/v6.0.2/invenio_app_rdm/theme/assets/semantic-ui/js/invenio_app_rdm/user_records_search/components.js)

### Creating Search App
Now that all components have been created or imported, create the following file `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/user_records_search/index.js`.
In here, all the previously defined components will be imported and the search app will be defined.

```js
import { createSearchAppInit } from "@js/invenio_search_ui";
import {
  RDMRecordResultsListItem,
  RDMRecordResultsGridItem,
  RDMDepositResults,
  RDMEmptyResults,
  RDMUserRecordsSearchLayout,
} from "./components";
import {
  RDMBucketAggregationElement,
  RDMCountComponent,
  RDMRecordFacets,
  RDMRecordFacetsValues,
  RDMRecordSearchBarElement,
  RDMToggleComponent,
} from "../search/components";

const initSearchApp = createSearchAppInit({
  "BucketAggregation.element": RDMBucketAggregationElement,
  "BucketAggregationValues.element": RDMRecordFacetsValues,
  "Count.element": RDMCountComponent,
  "EmptyResults.element": RDMEmptyResults,
  "ResultsList.item": RDMRecordResultsListItem,
  "ResultsGrid.item": RDMRecordResultsGridItem,
  "SearchApp.facets": RDMRecordFacets,
  "SearchApp.layout": RDMUserRecordsSearchLayout,
  "SearchApp.results": RDMDepositResults,
  "SearchBar.element": RDMRecordSearchBarElement,
  "BucketAggregation.element": RDMBucketAggregationElement,
  "BucketAggregationValues.element": RDMRecordFacetsValues,
  "SearchFilters.ToggleComponent": RDMToggleComponent,
});
```

### Update webpack
Now that the functionality is done, it is important to add it to webpack. This makes it accessible to templates.
Adding the following line in the `invenio_theme_tugraz/webpack.py` file, under the `semantic-ui` entries will do the trick. It will look something like this:
```python
"semantic-ui": dict(
            entry={
                "invenio-theme-tugraz-theme": "./less/invenio_theme_tugraz/theme.less",
                "invenio-theme-tugraz-js": "./js/invenio_theme_tugraz/theme.js",
                # add user record search component
                'invenio-theme-tugraz-rdm-user-records-search': './js/invenio_theme_tugraz/user_records_search/index.js',
            }, ...
```

### Adding Search Template
To make use of the new search, a search template will be added. It will import our search app as previously defined in webpack.
To achieve this, we will simply copy the current search page from invenio-rdm and replace the standard search app import with ours.
Create the following file `invenio_theme_tugraz/templates/invenio_theme_tugraz/records/search_deposit.html` with the content of the [base user records search template from invenio-app-rdm ](https://github.com/inveniosoftware/invenio-app-rdm/blob/v6.0.2/invenio_app_rdm/records_ui/templates/semantic-ui/invenio_app_rdm/records/search_deposit.html).

In that file, all that has to be done is to replace the import:
```diff
{%- block javascript %}
  {{ super() }}
- {{ webpack['invenio-app-rdm-user-records-search.js'] }}
+ {{ webpack['invenio-theme-tugraz-rdm-user-records-search.js'] }}
{%- endblock javascript %}
```

### Adding Render Function
Next, in `invenio_theme_tugraz/deposits.py` we define a function which will render the previously created template and pass the `searchbar_config` as argument.
```python
from flask import render_template
from flask_login import login_required
from invenio_app_rdm.records_ui.views.deposits import get_search_url


@login_required
def deposit_search():
    """List of user deposits page."""
    return render_template(
        "invenio_theme_tugraz/records/search_deposit.html",
        searchbar_config=dict(searchUrl=get_search_url()),
    )
```

### Adding URL Route
All that is left is to add a URL rule, so that the new function is called instead of the one from the base implementation. In `invenio_theme_tugraz/ext.py` import the previously defined function

```python
from invenio_theme_tugraz.deposits import deposit_search
```

and inside the `init_app` function add:

```python
app.add_url_rule("/uploads", "deposit_search", deposit_search)
```

Now invenio-rdm will use the new user record search template, which will use the new search app.



## Record Landing Page
Since we do not need to add additional functionality to the record landing page, there is no need for new components or updating the webpack file. We will simply add and modify the html file and add the URL route. If there is need for additional functionality, follow the steps from the search page guide.

### Adding Detail Template
Create the following file `invenio_theme_tugraz/templates/invenio_theme_tugraz/records/detail.html` with the content of the [base record detail template from invenio-app-rdm ](https://github.com/inveniosoftware/invenio-app-rdm/blob/v6.0.2/invenio_app_rdm/records_ui/templates/semantic-ui/invenio_app_rdm/records/detail.html).

Modify the template file as you see fit.

### Adding Render Function
Next, in `invenio_theme_tugraz/deposits.py` we define a function which will render the previously created template and pass arguments needed by the template.
```python
from flask import render_template
from invenio_app_rdm.records_ui.views.decorators import (
    pass_is_preview,
    pass_record_files,
    pass_record_or_draft,
)
from invenio_rdm_records.resources.serializers import UIJSONSerializer


@pass_is_preview
@pass_record_files
@pass_record_or_draft
def record_detail(record=None, files=None, pid_value=None, is_preview=False):
    """Record detail page (aka landing page)."""
    files_dict = None if files is None else files.to_dict()

    return render_template(
        "invenio_theme_tugraz/records/detail.html",
        record=UIJSONSerializer().serialize_object_to_dict(record.to_dict()),
        pid=pid_value,
        files=files_dict,
        permissions=record.has_permissions_to(['edit', 'new_version', 'manage',
                                               'update_draft', 'read_files']),
        is_preview=is_preview,
    )
```

### Adding URL Route
All that is left is to add a URL rule, so that the new function is called instead of the one from the base implementation. In `invenio_theme_tugraz/ext.py` import the previously defined function

```python
from invenio_theme_tugraz.deposits import record_detail
```

and inside the `init_app` function add:

```python
app.add_url_rule("/records/<pid_value>", "record_detail", record_detail)
```

Now invenio-rdm will use the new record detail template when displaying a record.


## Deposit Page
The component to be overridden is `deposit` so the path for this looks like `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/deposit/`.
Place the components in `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/deposit/RDMDepositForm.js`.

The original file can be found here: [RDMDepositForm.js](https://github.com/inveniosoftware/invenio-app-rdm/blob/v6.0.2/invenio_app_rdm/theme/assets/semantic-ui/js/invenio_app_rdm/deposit/RDMDepositForm.js)

### Rendering Deposit Form
Now that all components have been created or imported, create the following file `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/deposit/index.js`.
In here, all the previously defined components will be imported and the deposit form will be rendered.

```js
import React from "react";
import ReactDOM from "react-dom";
import "semantic-ui-css/semantic.min.css";

import { getInputFromDOM } from "react-invenio-deposit";
import { RDMDepositForm } from "./RDMDepositForm";

ReactDOM.render(
  <RDMDepositForm
    record={getInputFromDOM("deposits-record")}
    files={getInputFromDOM("deposits-record-files")}
    config={getInputFromDOM("deposits-config")}
    permissions={getInputFromDOM("deposits-record-permissions")}
  />,
  document.getElementById("deposit-form")
);
```

### Update webpack
Now that the functionality is done, it is important to add it to webpack. This makes it accessible to templates.
Adding the following line in the `invenio_theme_tugraz/webpack.py` file, under the `semantic-ui` entries will do the trick. It will look something like this:
```python
"semantic-ui": dict(
            entry={
                "invenio-theme-tugraz-theme": "./less/invenio_theme_tugraz/theme.less",
                "invenio-theme-tugraz-js": "./js/invenio_theme_tugraz/theme.js",
                # add deposit component
                'invenio-theme-tugraz-rdm-deposit': './js/invenio_theme_tugraz/deposit/index.js',
            }, ...
```

### Adding Deposit Template
To make use of the new deposit form, a template will be added. It will import the deposit form as previously defined in webpack.
To achieve this, we will simply copy the current deposit page from invenio-rdm and replace the standard deposit form import with ours.
Create the following file `invenio_theme_tugraz/templates/invenio_theme_tugraz/records/deposit.html` with the content of the [base deposit form template from invenio-app-rdm ](https://github.com/inveniosoftware/invenio-app-rdm/blob/v6.0.2/invenio_app_rdm/records_ui/templates/semantic-ui/invenio_app_rdm/records/deposit.html).

In that file, all that has to be done is to replace the import:
```diff
{%- block javascript %}
  {{ super() }}
- {{ webpack['invenio-app-rdm-deposit.js'] }}
+ {{ webpack['invenio-theme-tugraz-rdm-deposit.js'] }}
{%- endblock %}
```

### Adding Render Function
Next, in `invenio_theme_tugraz/deposits.py` we define two functions which will render the previously created template and pass arguments needed by the template.

First the create function, which is called when the user wants to create a new record:
```python
from flask import render_template
from flask_login import login_required
from invenio_app_rdm.records_ui.views.decorators import (
    pass_draft,
    pass_draft_files,
)
from invenio_app_rdm.records_ui.views.deposits import (
    get_form_config,
    get_search_url,
    new_record,
)
from invenio_rdm_records.resources.serializers import UIJSONSerializer


@login_required
def deposit_create():
    """Create a new deposit."""
    return render_template(
        "invenio_theme_tugraz/records/deposit.html",
        forms_config=get_form_config(createUrl=("/api/records")),
        searchbar_config=dict(searchUrl=get_search_url()),
        record=new_record(),
        files=dict(
            default_preview=None, entries=[], links={}
        ),
    )
```

Second the edit function, which is called when the user wants to edit an existing record:
```python
@login_required
@pass_draft
@pass_draft_files
def deposit_edit(draft=None, draft_files=None, pid_value=None):
    """Edit an existing deposit."""
    record = UIJSONSerializer().serialize_object_to_dict(draft.to_dict())

    return render_template(
        "invenio_theme_tugraz/records/deposit.html",
        forms_config=get_form_config(apiUrl=f"/api/records/{pid_value}/draft"),
        record=record,
        files=draft_files.to_dict(),
        searchbar_config=dict(searchUrl=get_search_url()),
        permissions=draft.has_permissions_to(['new_version'])
    )
```

### Adding URL Route
All that is left is to add URL rules, so that the new functions are called instead of the ones from the base implementation. In `invenio_theme_tugraz/ext.py` import the previously defined functions

```python
from invenio_theme_tugraz.deposits import deposit_create, deposit_edit
```

and inside the `init_app` function add:

- For new records: 
```python
app.add_url_rule("/uploads/new", "deposit_create", deposit_create)
```
- For editing existing records:
```python
app.add_url_rule("/uploads/<pid_value>", "deposit_edit", deposit_edit)`
```
Now invenio-rdm will use the new deposit form, when creating a new record or updating an existing record.



## Troubleshoot

### ManifestKeyNotFoundError
- check if the spelling of the webpack names matches with the import in the template.
- reinstall invenio-theme-tugraz
    - In order to pick up all the new files and to rebuild webpack, it is necessary to install `invenio-theme-tugraz` again. If no such changes happened, this step can be skipped.
    - `invenio-cli packages install /path/to/invenio-theme-tugraz`

