# Feature Data 2

<table class="table">
  <tr>
    <td>ID</td><td>featuredata2</td>
  </tr>
  <tr>
    <td>API</td><td>[link](<%= apiurl %>Oskari.mapframework.bundle.featuredata2.FeatureDataBundleInstance.html)</td>
  </tr>
</table>

## Description

The Bundle provides a grid view for WFS object data.

## Screenshot

![screenshot](/images/bundles/featuredata.png)

## Bundle configuration

No configuration is required, but setting selectionTools to true will add a new button to toolbar that opens a selection tool dialog.

Using selection tools the user can get wfs grid data from smaller area than whole screen.

Selection tool can be simplified by setting singleSelection to true. This makes the dialog more simple and selection tool makes the filtering straight after defining the geometry. Only single geometry can be given per filter with this option.

```javascript
{
  "selectionTools" : true,
  "singleSelection" : true
}
```

## Bundle state

No statehandling has been implemented.

## Requests the bundle handles

This bundle doesn't handle any requests.

## Requests the bundle sends out

<table class="table">
  <tr>
    <th>Request</th><th> Where/why it's used</th>
  </tr>
  <tr>
    <td>userinterface.AddExtensionRequest</td><td> Register as part of the UI in start()-method</td>
  </tr>
  <tr>
    <td>Toolbar.AddToolButtonRequest</td><td> Requests selection toolbar button</td>
  </tr>
  <tr>
    <td> userinterface.RemoveExtensionRequest </td><td> Unregister from the UI in stop()-method</td>
  </tr>
  <tr>
    <td> HighlightMapLayerRequest </td><td> Requests that a layer is "highlighted" (old mechanic) so highlighting and feature selection will occur using this layer. Sent when a tab is selected (tab presents one layers data)</td>
  </tr>
  <tr>
    <td> DimMapLayerRequest </td><td> Requests that highlighting is removed from a layer (old mechanic) so highlighting and feature selection will be disabled on this layer. Sent when a tab is unselected/removed (tab presents one layers data).</td>
  </tr>
  <tr>
    <td> userguide.ShowUserGuideRequest </td><td> Used to show additional data that wouldn't fit the normal grid. A link is shown instead on grid and clicking the link will open the additional data on user guide "popup".</td>
  </tr>
</table>

## Events the bundle listens to

<table class="table">
<tr>
  <th> Event </th><th> How does the bundle react</th>
</tr>
<tr>
  <td> AfterMapLayerAddEvent </td><td> A tab panel is added to the flyout for the added layer.</td>
</tr>
<tr>
  <td> AfterMapLayerRemoveEvent </td><td> Tab panel presenting the layer is removed from the flyout.</td>
</tr>
<tr>
  <td> WFSPropertiesEvent </td><td> Grid data is updated if the flyout is open. Data is only updated for the layer whose tab is currently selected.</td>
</tr>
<tr>
  <td> WFSFeatureEvent </td><td> Grid data is updated if the flyout is open. Data is only updated for the layer whose tab is currently selected.</td>
</tr>
<tr>
  <td> WFSFeaturesSelectedEvent </td><td> Highlights the feature on the grid.</td>
</tr>
<tr>
  <td> userinterface.ExtensionUpdatedEvent </td><td> Determines if the layer was closed or opened and enables/disables data updates accordingly.</td>
</tr>
</table>

## Events the bundle sends out

<table class="table">
  <tr>
    <th> Event </th><th> When it is triggered/what it tells other components</th>
  </tr>
  <tr>
    <td> WFSFeaturesSelectedEvent </td><td> Sent when a selection is made on the grid to notify other components that a feature has been selected</td>
  </tr>
  <tr>
    <td> AddedFeatureEvent </td><td> Sent when a selection feature has been added</td>
  </tr>
  <tr>
    <td> FinishedDrawingEvent </td><td> Sent when a selection has been finished drawing</td>
  </tr>
</table>

## Dependencies

<table class="table">
  <tr>
    <th> Dependency </th><th> Linked from </th><th> Purpose</th>
  </tr>
  <tr>
    <td> [jQuery](http://api.jquery.com/) </td>
    <td> Linked in portal theme </td>
    <td> Used to create the component UI from begin to end</td>
  </tr>
  <tr>
    <td> [Oskari toolbar](<%= docsurl %>framework/toolbar.html) </td>
    <td> Expects to be present in application setup </td>
    <td> To register plugin to toolbar</td>
  </tr>
</table>
