# Override Pages
In some cases, the default pages do not offer all the functionality or modification possibilities needed. It is possible to override the pages, though this should be the last resort as it increases resources needed for maintainability.

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
Create the following file `invenio_theme_tugraz/templates/invenio_theme_tugraz/search.html` with content
```html
{#
  Copyright (C) 2020 CERN.
  Copyright (C) 2020 Northwestern University.
  Copyright (C) 2021 Graz University of Technology.

  Invenio App RDM is free software; you can redistribute it and/or modify it
  under the terms of the MIT License; see LICENSE file for more details.
#}
{%- set title = _("Search results") %}
{%- extends config.BASE_TEMPLATE %}

{%- block javascript %}
    {{ super() }}
    <!-- That's the important part! -->
    {{ webpack['invenio-theme-tugraz-rdm-search.js'] }}
{%- endblock %}

{%- block page_body %}

<div data-invenio-search-config='{
  "aggs": [
    {
      "aggName": "access_status",
      "field": "access.status",
      "title": "{{_("Access status")}}"
    },
    {
      "aggName": "resource_type",
      "field": "resource_type.type",
      "title": "{{_("Resource type")}}",
      "childAgg": {
        "aggName": "subtype",
        "field": "resource_type.subtype",
        "title": "{{_("Resource type")}}"
      }
    },
    {
      "aggName": "languages",
      "field": "languages",
      "title": "{{_("Languages")}}"
    }
  ],
  "appId": "rdm-search",
  "initialQueryState": {
    "hiddenParams": null,
    "size": 10,
    "page": 1,
    "sortBy": "bestmatch"
  },
  "layoutOptions": {
    "gridView": false,
    "listView": true
  },
  "paginationOptions": {
    "defaultValue": 10,
    "resultsPerPage": [
      {
        "text": "10",
        "value": 10
      },
      {
        "text": "20",
        "value": 20
      },
      {
        "text": "50",
        "value": 50
      }
    ]
  },
  "searchApi": {
    "axios": {
      "headers": {
        "Accept": "application/vnd.inveniordm.v1+json"
      },
      "url": "/api/records",
      "withCredentials": true
    },
    "invenio": {
      "requestSerializer": "InvenioRecordsResourcesRequestSerializer"
    }
  },
  "sortOrderDisabled": true,
  "defaultSortingOnEmptyQueryString": {
    "sortBy": "newest"
  },
  "sortOptions": [
    {
      "sortBy": "bestmatch",
      "text": "{{_("Best match")}}"
    },
    {
      "sortBy": "newest",
      "text": "{{_("Newest")}}"
    },
    {
      "sortBy": "oldest",
      "text": "{{_("Oldest")}}"
    },
    {
      "sortBy": "version",
      "text": "{{_("Version")}}"
    }
  ]
}'></div>

{%- endblock page_body %}

```

#### Override Config Variable
Finally, invenio-rdm must know that it should use our new template for the search. This can be achieved by overriding the config variable `SEARCH_UI_SEARCH_TEMPLATE`.
In `invenio_theme_tugraz/config.py` add or override the following line:
```python
SEARCH_UI_SEARCH_TEMPLATE = "invenio_theme_tugraz/search.html"
```

Now invenio-rdm will use the new search template, which will use the new search app.
