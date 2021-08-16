# Override Pages
In some cases, the default pages do not offer all the functionality or modification possibilities needed. It is possible to override the pages, though this should be the last resort as it increases resources needed for maintainability.

! IMPORTANT: This is valid for `invenio-app-rdm==v6.0.2` !


### Install Module Again
In order to pick up all the new files and to rebuild webpack, it is necessary to install `invenio-theme-tugraz` again. If no such changes happened, this step can be skipped.

`invenio-cli packages install /path/to/module`


### JavaScript and React Components
In order to provide functionality, all the needed components either have to be created or imported from other modules. To achieve this, create or update the following file `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/{componentToOverride}/components.js`. This will make it easier to keep a good overview of the dependencies.

Add all needed imports:
```js
import { Button, Card} from "semantic-ui-react";

// import component from invenio module
import { SearchBar } from "@js/invenio_search_ui/components";
```

Create new components:
```js
export const RDMRecordSearchBarContainer = () => {
  return (
    <Overridable id={"SearchApp.searchbar"}>
      <SearchBar />
    </Overridable>
  );
};
```


## Search Pages

### Main Search
The component to be overridden is `search` so the path for this looks like `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/search/`.
Place the components in `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/search/components.js`.

#### Creating Search App
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

#### Adding to webpack
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

#### Adding Search Template
To make use of the new search, a search template will be added. It will import our search app as previously defined in webpack.
To achieve this, we will simply copy the current search page from invenio-rdm and replace the standard search app import with ours.
Create the following file `invenio_theme_tugraz/templates/invenio_theme_tugraz/search.html` with the content of the [base search template from invenio-app-rdm ](https://github.com/inveniosoftware/invenio-app-rdm/blob/v6.0.2/invenio_app_rdm/records_ui/templates/semantic-ui/invenio_app_rdm/records/search.html).

In that file, all that has to be done is to replace 
```python
{%- block javascript %}
    {{ super() }}
    {{ webpack['invenio-app-rdm-search.js'] }}
{%- endblock %}
```
with
```python
{%- block javascript %}
    {{ super() }}
    {{ webpack['invenio-theme-tugraz-rdm-search.js'] }}
{%- endblock %}
```

#### Override Config Variable
Finally, invenio-rdm must know that it should use our new template for the search. This can be achieved by overriding the config variable `SEARCH_UI_SEARCH_TEMPLATE`.
In `invenio_theme_tugraz/config.py` add or override the following line:
```python
SEARCH_UI_SEARCH_TEMPLATE = "invenio_theme_tugraz/search.html"
```
Now invenio-rdm will use the new search template, which will use the new search app.


### User Record Search
The component to be overridden is `user_records_search` so the path for this looks like `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/user_records_search/`.
Place the components in `invenio_theme_tugraz/assets/semantic-ui/js/invenio_theme_tugraz/user_records_search/components.js`.

#### Creating Search App
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

#### Adding to webpack
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

#### Adding Search Template
To make use of the new search, a search template will be added. It will import our search app as previously defined in webpack.
To achieve this, we will simply copy the current search page from invenio-rdm and replace the standard search app import with ours.
Create the following file `invenio_theme_tugraz/templates/invenio_theme_tugraz/records/search_deposit.html` with the content of the [base user records search template from invenio-app-rdm ](https://github.com/inveniosoftware/invenio-app-rdm/blob/v6.0.2/invenio_app_rdm/records_ui/templates/semantic-ui/invenio_app_rdm/records/search_deposit.html).

In that file, all that has to be done is to replace 
```python
{%- block javascript %}
  {{ super() }}
  {{ webpack['invenio-app-rdm-user-records-search.js'] }}
{%- endblock javascript %}
```
with
```python
{%- block javascript %}
  {{ super() }}
  {{ webpack['invenio-theme-tugraz-rdm-user-records-search.js'] }}
{%- endblock javascript %}
```

#### Adding Render Function
Next, in `invenio_theme_tugraz/deposits.py` we define a function which will render the previously created template and pass the `searchbar_config` as argument.
```python
@login_required
def deposit_search():
    """List of user deposits page."""
    return render_template(
        "invenio_theme_tugraz/records/search_deposit.html",
        searchbar_config=dict(searchUrl=get_search_url()),
    )
```

#### Adding URL Route
All that is left is to add a URL rule, so that the new function is called instead of the one from the base implementation. In `invenio_theme_tugraz/ext.py` import the previously defined function and inside the `init_app` function add:

`app.add_url_rule("/uploads", "deposit_search", deposit_search)`t

Now invenio-rdm will use the new user record search template, which will use the new search app.



## Record Landing Page
Since we do not need to add additional functionality to the record landing page, there is no need for new components or updating the webpack file. We will simply add and modify the html file and add the URL route. If there is need for additional functionality, follow the steps from the search page guide.

#### Adding Search Template
Create the following file `invenio_theme_tugraz/templates/invenio_theme_tugraz/records/detail.html` with the content of the [base record detail template from invenio-app-rdm ](https://github.com/inveniosoftware/invenio-app-rdm/blob/v6.0.2/invenio_app_rdm/records_ui/templates/semantic-ui/invenio_app_rdm/records/detail.html).

Modify the template file as you see fit.

#### Adding Render Function
Next, in `invenio_theme_tugraz/deposits.py` we define a function which will render the previously created template and pass arguments needed by the template.
```python
@pass_is_preview
@pass_record_or_draft
@pass_record_files
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

#### Adding URL Route
All that is left is to add a URL rule, so that the new function is called instead of the one from the base implementation. In `invenio_theme_tugraz/ext.py` import the previously defined function and inside the `init_app` function add:

`app.add_url_rule("/records/<pid_value>", "record_detail", record_detail)`

Now invenio-rdm will use the new record detail template when displaying a record.

