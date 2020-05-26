# Manage users in Role

The Users in Role tab lists all the users who have this role assigned. In the current console, these users can only be viewed. But in the new design, users can be directly added to/removed from this tab.

## Empty state

An empty state occurs when no users have this role directly assigned. That means if this role is assigned to a user as an associated role, that user won’t display under this tab. Also, if this role is assigned to a group that includes some users, those users won’t display here as well.

![UsersEmptyState](./images/users-empty-state.png)

## Add users

* Users can be added to this role through the Add user function. Upon clicking the Add button, a modal with the whole list of all available users will pop up.
* Users that already have this role as an effective role won’t appear in the list and cannot be added here.

![AddUsers](./images/add-users.png)

* A toast alert will appear when users are added successfully.
* The added users will appear under this tab.

![Alert](./images/add-users-alert.png)

## Remove users

The users can either be removed one by one or bulk removed.

### Remove a single user

* Clicking the kebab shows the Remove user action. This action can remove a single user.

![SingleRemove](./images/single-remove.png)

* Clicking the Remove user button will pop up a confirmation modal.

![Confirmation](./images/confirmation.png)

### Bulk remove users

* If multiple users are selected, the Remove user action in the toolbar will be enabled. This action can remove multiple users.

![BulkRemove](./images/bulk-remove.png)

* Clicking the Remove user button will pop up a confirmation modal.

![BulkConfirmation](./images/bulk-confirmation.png)
