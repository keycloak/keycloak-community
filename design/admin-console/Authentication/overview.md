# Authentication Redesign

The Authentication redesign will be carried out in two steps:

  * Step 1 - Improve the visualization. Apply PatternFly 4 to the current interface. Optimize the user experience of the Authentication to a certain extent.
  * Step 2 - Simplify the whole Authentication flows edit experience. Provide an easier configuration method to the novice.

In this round of design, we mainly focus on the first step. Following changes have been made:

1. A flow list has been added. Users can manage (create, duplicate, delete, etc.) all the flows in this list.

2. Flows and Bindings tabs are combined. The in-use flows can be viewed and changed directly from the flows list. The flows which are used as overrides are also marked in the list.

3. Types are added as an attribute to the flows. All the built-in and new created flows are separated into 6 types - the browser flow, the registration flow, the client authentication (also include docker auth flow), the direct grant flow (also include http challenge flow), the reset credential flow and the broker login flow.

4. The layout of the flow details has been improved. Requirements have been simplified to only show the selected one. Sub-flows are represented by the expandable rows.

5. The interaction behaviors are improved when creating a conditional flow.

This website only records the main changes in each function. The whole prototype can be accessed through the following links.

* [Flow list](https://marvelapp.com/prototype/bh91013/section/1227851)
* [Create flow](https://marvelapp.com/prototype/bh91013/section/1227905)
* [Layout of built-in flows](https://marvelapp.com/prototype/bh91013/section/1228928)
* [Layout of custom flows](https://marvelapp.com/prototype/bh91013/section/1228929)
* [Conditional flow](https://marvelapp.com/prototype/bh91013/section/1228935)
* [Required actions](https://marvelapp.com/prototype/7i7g4cb/screen/75688306)
* [Policies](https://marvelapp.com/prototype/5ifgiai/screen/75687759)
