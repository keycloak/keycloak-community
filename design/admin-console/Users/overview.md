# Users Redesign

The following improvements have been made to the Users section:

1. Display different default pages according to the total amount of users in the database

  In the current console, the user list is hidden by default. You must hit “View all users” to make it visible. In the new design, we want to simplify this action. But when the user amount is large, directly show all the users can cause performance issues. With this consideration, we decided to have two versions of default pages:

    * When there are no more than 100 local users, and no external users, directly show the user list
    * In the other cases, just show a search box

  More details are provided in the [Default page](https://www.keycloak.org/keycloak-community/design/admin-console/#/Users/default).

2. Add status to users

	In the first phase, we are going to support the “temporarily locked” and “disabled” status. Users with the status will have labels attached to them in the list. More status will be added in the future.

3. Use a table to show the user groups and make it more flexible

	The current group pattern is not fit bulk actions or other complicated operations. In the new design, we use a table instead to reorganize the whole function to make it easier to use.

This website only records the main changes in each function. The whole prototype can be accessed through the following links.

* [User list](https://marvelapp.com/prototype/7i7ja65/screen/75693218)
* [Groups](https://marvelapp.com/prototype/f13gid4/screen/72158283)
* [Consents](https://marvelapp.com/prototype/f734jga/screen/73531548)
* [Identity provider links](https://marvelapp.com/prototype/fgg6e0h/screen/75696510)
