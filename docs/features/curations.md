# Curations feature

[invenio-curations](https://github.com/tu-graz-library/invenio-curations) is an Invenio package that adds curation reviews to TU Graz
Repository.

Out of the box, the Repository already provides reviews for records as part of the submission or inclusion into communities.
However, there is no requirement per default for records to be part of any community at all.
Thus, it is generally easy for users to self-publish records in the Repository without any further review.

## Setup

After the successful installation of `Invenio-Curations`, it still needs to be configured properly to work.
The following sections should guide you through the required adaptations.

### invenio.cfg

Because this package is now part of the Repository, we included its overriden
configurations in [invenio-config-tugraz](https://github.com/tu-graz-library/invenio-config-tugraz). But, even so, we still need to update
some core InvenioRDM configs in order to get it working. For default configurations please refer to the [curations](https://github.com/tu-graz-library/invenio-curations) package.

- Include the curation component in the RDM components list

```
from invenio_config_tugraz.components import TUGRAZ_RDM_RECORDS_SERVICE_COMPONENTS

RDM_RECORDS_SERVICE_COMPONENTS = TUGRAZ_RDM_RECORDS_SERVICE_COMPONENTS + [RDMToGlobalSearchComponent]
``` 

- Update Requests permission policy
- Update Notification Builders
- Update Requests Facets
- Update Registered Event types

```
from invenio_config_tugraz.permissions import TUGrazRDMRequestsPermissionPolicy
from invenio_config_tugraz.facets import TUGRAZ_REQUESTS_FACETS
from invenio_config_tugraz.notifications import TUGRAZ_NOTIFICATIONS_BUILDERS
from invenio_config_tugraz.requests import TUGRAZ_REQUESTS_REGISTERED_EVENT_TYPES

REQUESTS_PERMISSION_POLICY = TUGrazRDMRequestsPermissionPolicy
REQUESTS_FACETS = TUGRAZ_REQUESTS_FACETS
NOTIFICATIONS_BUILDERS = TUGRAZ_NOTIFICATIONS_BUILDERS
REQUESTS_REGISTERED_EVENT_TYPES = TUGRAZ_REQUESTS_REGISTERED_EVENT_TYPES
```

### Curations UI
The changes so far have dealt with setting up the mechanism for the curation workflow in the backend. To also make the workflow accessible for users through the UI, some frontend components have to be updated as well.

Invenio-Curations provides a few component overrides. These overrides need to be registered in the overridable registry (i.e. in the repository's **assets/js/invenio_app_rdm/overridableRegistry/mapping.js**):

```
import { curationComponentOverrides } from "@js/invenio_curations/requests";
import { DepositBox } from "@js/invenio_curations/deposit/DepositBox";

export const overriddenComponents = {
    // ... after your other overrides ...
    ...curationComponentOverrides,
    "InvenioAppRdm.Deposit.CardDepositStatusBox.container": DepositBox,
};
``` 

### Curator role
The permission to manage curation requests is controlled by a specific role in the system. The default configured role is **administration-rdm-records-curation**. Add this role to the curators for TU Graz repository in order for them to have access to the flow.

Commands to achieve this:
```
invenio roles create administration-rdm-records-curation
invenio roles add <curator_mail> administration-rdm-records-curation
``` 
