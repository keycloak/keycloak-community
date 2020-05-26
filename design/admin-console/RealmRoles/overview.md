# Roles Redesign

The following improvements have been made to the Roles section:

1. The Default Roles tab is removed from this section.

   This tab defines the default roles that will be automatically assigned to any user when the user is newly created or imported through an identity provider. The default roles not only include realm roles, but also client roles defined in different clients. What’s more, the current design cannot clearly show the relationship between default roles and users. Based on these concerns, we decide to move the Default Roles tab together with the Default Groups tab (in Groups) to Realm Settings. There will be a new User Registration tab where you can configure all the default settings for new users.

2. Roles will be renamed to Realm roles in the side navigation.

   As the Default Roles tab is removed, this section will only include a table of all the realm roles.

3. Separate the Composite Roles function to a new tab.

   In the current console, the Composite Roles function is embedded in the Details page of roles. To make this function more focused, we move it to a new tab. Users can enable this tab by assigning associated roles to this role, and that would turn this role into a composite role.

4. The table of users in the “Users in Role” tab can be added/removed.

   In the current console, you can only view which users have this role assigned under the tab. If you want to assign this role to a user, you can only go to the Role Mappings tab of the user and then assign it. In the new design, the table under this tab is updated to allow adding new users or removing users who have this role assigned. This makes the role assigned function more convenient.

This website only records the main changes in each function. The whole prototype can be accessed through the following links.

* [Role list](https://marvelapp.com/70aabf8/screen/69465732)
* [Create role](https://marvelapp.com/70aabf8/screen/69465883)
* [Delete role](https://marvelapp.com/70aabf8/screen/69466244)
* [Role details](https://marvelapp.com/70aabf8/screen/69466280)
* [Composite role](https://marvelapp.com/70aabf8/screen/69466425)
* [Manage users in Role](https://marvelapp.com/70aabf8/screen/69466546)
