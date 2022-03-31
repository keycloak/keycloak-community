# Save button bar

In the old console, the Keycloak is broadly using the “Cancel” button to revert the changes that haven’t been saved in a form. In the new console, all the “Cancel” buttons are only used to terminate a process and return to the previous page. Meanwhile, the “Reload” button is used to cover the functionality that revert the changes that haven’t been saved in forms.

This documentation detailed records which buttons should be used in different cases.

### Create/add (full-page/modal)
![SaveButton-1](./images/SaveButton-1.png)
* There should only be two buttons -
![SaveButton-1.1](./images/SaveButton-1.1.png)
  "Cancel" is used to terminate the create/add progress and go back to the previous page.



### Create (wizard)
![SaveButton-2](./images/SaveButton-2.png)
* This component follows PF4 [guidelines](https://www.patternfly.org/v4/components/wizard#basic).


### Details/edit/settings
The previous level isn’t embedded in a tab/doesn’t have previous level.

![SaveButton-3](./images/SaveButton-3.png)
* There should be only two buttons
![SaveButton-3.1](./images/SaveButton-3.1.png)
  Users are supposed to use breadcrumb to go back to the previous level.



### Details/edit
The previous level is embedded in a tab.
![SaveButton-4](./images/SaveButton-4@2x.png)
* There should be only two buttons
![SaveButton-4.1](./images/SaveButton-4.1.png)
  When users click the previous level in the breadcrumb, it’s supposed to go to the tab that contains this object instead of the first tab.




### Read-only details/view
![SaveButton-5](./images/SaveButton-5@2x.png)
* There should only be one button:
  ![SaveButton-5.1](./images/SaveButton-5.1.png)

  Users can return to the previous level by using it.


### Code editor
![SaveButton-6](./images/SaveButton-6@2x.png)
* There should be only two buttons
![SaveButton-6.1](./images/SaveButton-6.1.png)

### Settings in cards
![SaveButton-7](./images/SaveButton-7.png)
* There should be only two buttons
![SaveButton-7.1](./images/SaveButton-7.1.png)
  Save is usually using the secondary button style.
