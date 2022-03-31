# Add new row



## Add a single new row at a time

### Initial state
![AddSingleRow-1](./images/AddSingleRow-1.png)
  * The color of Add and Remove are both disabled grey (#6A6E73).
  * The **Add** button should say “**Add noun** ”(singular).

### Filled with something
![AddSingleRow-2](./images/AddSingleRow-2.png)
  * When all the existing rows are filled with something, the **Add** button changes to be active blue (#0066CC).


### Add new row
![AddSingleRow-3](./images/AddSingleRow-3.png)
  * When there are multiple rows, all the **Remove** buttons change to be active blue, users can remove any of the rows.
  * When there is an existing row which is empty, the **Add** button is disabled again.


### New row is filled
![AddSingleRow-4](./images/AddSingleRow-4.png)

### Hovering state
![AddSingleRow-5](./images/AddSingleRow-5.png)
  * Hovering on the remove icons will always show the tooltips.




## Attributes
In this scenario, only the key is required at most time. Sometime the key and the value are both required.

### Initial state
![Attributes-1](./images/Attributes-1.png)

  * The asterisk is used to indicate whether the **Key** or **Value** is required.
  * The color of **Add** and **Remove** are both disabled grey.
  * **Save** button is enabled all the time.

### Only value is provided
![Attributes-2](./images/Attributes-2.png)

  * The **Add** button is still disabled when the **Key** isn’t filled. If the **Key** and **Value** are both required, then the **Add** button will only be enabled after both fields are filled.

### Only key is provided
![Attributes-3](./images/Attributes-3.png)

  * The **Add** button is enabled after the key is filled, the value is allowed to be empty.


### Multiple rows
![Attributes-4](./images/Attributes-4.png)

  * When there are multiple rows, all the **Remove** buttons change to be active blue.
  * When there is an existing row which is empty, the **Add** button is disabled again.

### Validation
![Attributes-5](./images/Attributes-5.png)
  * When saving with required fields empty, a validation error displays below the corresponding field.




## Single label for key and value
In this scenario, whether the key and value are required depends on whether there is an asterisk attached to the field label.

### Initial state
![SingleLabel-1@2x](./images/SingleLabel-1@2x.png)
  * The color of **Add** and **Remove** are both disabled grey.

### Only key or value is provided
![SingleLabel-2](./images/SingleLabel-2.png)
  * The **Add** button is still disabled when only the Key or value is filled.

### Both key and value are provided
![SingleLabel-3@2x](./images/SingleLabel-3@2x.png)
  * The **Add** button is enabled after both key and value are filled.

### Multiple rows
![SingleLabel-4@2x](./images/SingleLabel-4@2x.png)
  * When there are multiple rows, all the **Remove** buttons change to be active blue.
  * When there is an existing row which is empty, the **Add** button is disabled again.

### Validation
#### Validation of key
![SingleLabel-5@2x](./images/SingleLabel-5@2x.png)
#### Validation of value
![SingleLabel-6@2x](./images/SingleLabel-6@2x.png)
  * The validation will appear after the field losing the focus.



### Select from the existing list
![SingleLabel-7@2x](./images/SingleLabel-7@2x.png)
  * The key and value fields could be dropdown instead of text input fields if there are existing options to select from.
